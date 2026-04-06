<script>
  import { onDestroy, onMount } from 'svelte';
  import {
    ascending,
    csv,
    extent,
    hsl,
    max,
    rollups,
    scaleOrdinal,
    scaleSqrt,
    schemeTableau10
  } from 'd3';

  export let csvUrl = '/wordcloud-classified-data.csv';
  export let fontFamily = 'Inter, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif';

  let cloudFactory;
  let container;
  let resizeObserver;
  let currentLayout;

  let width = 960;
  let height = 620;

  let rows = [];
  let positionedWords = [];
  let selectedCategory = null;
  let loading = true;
  let error = '';
  let displayWords = [];

  

  const numberFormat = new Intl.NumberFormat('en-US');

  function clamp(value, min, max) {
    return Math.min(max, Math.max(min, value));
  }

  function parsePopulation(value) {
    const numeric = Number(String(value ?? '').replace(/,/g, '').trim());
    return Number.isFinite(numeric) ? numeric : 0;
  }

  function hashString(input = '') {
    let hash = 1779033703;

    for (let i = 0; i < input.length; i += 1) {
      hash = Math.imul(hash ^ input.charCodeAt(i), 3432918353);
      hash = (hash << 13) | (hash >>> 19);
    }

    return hash >>> 0;
  }

  function mulberry32(seed) {
    return function seededRandom() {
      let t = (seed += 0x6d2b79f5);
      t = Math.imul(t ^ (t >>> 15), t | 1);
      t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
      return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
    };
  }

  function buildPalette(count) {
    if (count <= schemeTableau10.length) {
      return schemeTableau10.slice(0, count);
    }

    return Array.from({ length: count }, (_, index) =>
      hsl((index / count) * 360, 0.62, 0.5).formatHex()
    );
  }

  $: categories = Array.from(new Set(rows.map((row) => row.category))).sort(ascending);
  $: categoryColor = scaleOrdinal().domain(categories).range(buildPalette(categories.length));

  $: selectedRows = selectedCategory
    ? rows.filter((row) => row.category === selectedCategory)
    : [];

  $: summaryText = selectedCategory
    ? `Showing ${new Set(selectedRows.map((row) => row.awardee)).size} awardees in “${selectedCategory}”. Font size reflects population; saturation reflects larger vs. smaller populations.`
    : 'Click a category to drill down to the awardees in that category. Font size reflects how many rows each category appears in.';

  function createFontScale(values, minFont, maxFont) {
    const [minValue, maxValue] = extent(values);

    if (!Number.isFinite(minValue) || !Number.isFinite(maxValue)) {
      return () => minFont;
    }

    if (minValue === maxValue) {
      return () => (minFont + maxFont) / 2;
    }

    return scaleSqrt().domain([minValue, maxValue]).range([minFont, maxFont]);
  }

  function saturateCategoryColor(category, population, [minPopulation, maxPopulation]) {
    const base = hsl(categoryColor(category) ?? '#64748b');

    if (!Number.isFinite(minPopulation) || !Number.isFinite(maxPopulation) || minPopulation === maxPopulation) {
      return hsl(base.h, 0.75, base.l).formatHex();
    }

    const t = (population - minPopulation) / (maxPopulation - minPopulation);
    const saturation = 0.2 + t * 0.8;

    return hsl(base.h, saturation, base.l).formatHex();
  }

  function getCategoryWords() {
    const counts = rollups(
      rows,
      (group) => group.length,
      (row) => row.category
    ).sort((a, b) => b[1] - a[1] || ascending(a[0], b[0]));

    const minFont = clamp(width * 0.03, 20, 30);
    const maxFont = clamp(width * 0.11, 48, 96);
    const fontScale = createFontScale(
      counts.map(([, count]) => count),
      minFont,
      maxFont
    );

    return counts.map(([category, count]) => ({
      kind: 'category',
      text: category,
      category,
      value: count,
      size: fontScale(count),
      fill: categoryColor(category),
      title: `${category}: ${count} row${count === 1 ? '' : 's'}`
    }));
  }

  function getAwardeeWords(category) {
    const filtered = rows.filter((row) => row.category === category);

    // We keep one population per awardee. If your source data ever stores partial
    // populations across multiple rows, switch max(...) to a sum(...).
    const awardees = rollups(
      filtered,
      (group) => max(group, (row) => row.population) ?? 0,
      (row) => row.awardee
    ).sort((a, b) => b[1] - a[1] || ascending(a[0], b[0]));

    const populations = awardees.map(([, population]) => population);
    const minFont = clamp(width * 0.02, 14, 22);
    const maxFont = clamp(width * 0.085, 38, 76);
    const fontScale = createFontScale(populations, minFont, maxFont);
    const populationExtent = extent(populations);

    return awardees.map(([awardee, population]) => ({
      kind: 'awardee',
      text: awardee,
      category,
      awardee,
      value: population,
      population,
      size: fontScale(population),
      fill: saturateCategoryColor(category, population, populationExtent),
      title: `${awardee}: population ${numberFormat.format(population)}`
    }));
  }
  // console.log("selected category is", selectedCategory);
  // $: displayWords = selectedCategory ? getAwardeeWords(selectedCategory) : getCategoryWords();

  // $: console.log('selected category is', selectedCategory);

$: {
  const _rows = rows;
  const _width = width;
  const _categoryColor = categoryColor;
  const _selectedCategory = selectedCategory;

  console.log(selectedCategory ? selectedCategory : "selected cat doesn't exist");

  displayWords = selectedCategory
    ? getAwardeeWords(selectedCategory)
    : getCategoryWords();
}


  async function loadRows() {
    loading = true;
    error = '';

    try {
      const parsed = await csv(csvUrl, (row) => {
        const awardee = (row.Awardee ?? row.awardee ?? '').trim();
        const category = (row.Category ?? row.category ?? '').trim();
        const population = parsePopulation(row.Population ?? row.population);

        if (!awardee || !category) {
          return null;
        }

        return {
          awardee,
          category,
          population
        };
        
      });

      rows = parsed.filter(Boolean);
      

    } catch (err) {
      error = err instanceof Error ? err.message : 'Unable to load CSV.';
      rows = [];
    } finally {
      loading = false;
    }
  }

  function stopLayout() {
    currentLayout?.stop?.();
  }

  function runLayout() {
    if (!cloudFactory || loading || error || displayWords.length === 0 || width <= 0 || height <= 0) {
      return;
    }

    stopLayout();
    positionedWords = [];

    const seedKey = `${selectedCategory ?? 'categories'}-${width}-${height}`;
    const seededRandom = mulberry32(hashString(seedKey));
    const wordsForLayout = displayWords.map((word) => ({ ...word }));

    currentLayout = cloudFactory()
      .size([width, height])
      .words(wordsForLayout)
      .padding(selectedCategory ? 3 : 8)
      .rotate(() => 0)
      .font(fontFamily)
      .fontSize((word) => word.size)
      .timeInterval(20)
      .random(seededRandom)
      .on('end', (words) => {
        positionedWords = words;
      });

    currentLayout.start();
  }

  function handleCategorySelect(word) {
    if (word.kind === 'category') {
      selectedCategory = word.category;
    }
  }

  function handleKeydown(event, word) {
    if (word.kind !== 'category') {
      return;
    }

    if (event.key === 'Enter' || event.key === ' ') {
      event.preventDefault();
      handleCategorySelect(word);
    }
  }

  function resetView() {
    selectedCategory = null;
  }

  onMount(async () => {
    const [{ default: importedCloud }] = await Promise.all([import('d3-cloud')]);

    cloudFactory = importedCloud;
    await loadRows();
    // console.log(rows);

    resizeObserver = new ResizeObserver((entries) => {
      const nextWidth = Math.floor(entries[0]?.contentRect?.width ?? 0);

      if (nextWidth > 0) {
        width = nextWidth;
      }
    });

    if (container) {
      resizeObserver.observe(container);
      width = Math.floor(container.getBoundingClientRect().width);
    }
  });

  onDestroy(() => {
    resizeObserver?.disconnect?.();
    stopLayout();
  });

  $: height = clamp(Math.round(width * 0.68), 420, 700);

  // $: if (cloudFactory && rows.length && width > 0 && height > 0 && !loading && !error) {
  //   runLayout();
  // }
  $: {
    // displayWords;
    // selectedCategory;
    const _displayWords = displayWords;
    const _selectedCategory = selectedCategory;
    const _cloudFactory = cloudFactory;
    const _loading = loading;
    const _error = error;
    const _width = width;
    const _height = height;

    // if (cloudFactory && width > 0 && height > 0 && !loading && !error && displayWords.length > 0) {
    //   runLayout();
    // } else if (!loading && !error) {
    //   stopLayout();
    //   positionedWords = [];
    // }
    if (_cloudFactory && !_loading && !_error && _width > 0 && _height > 0 && _displayWords.length > 0) {
      runLayout();
    } else if (!_loading && !_error) {
      stopLayout();
      positionedWords = [];
    }
}
</script>

<section class="panel">
  <div class="panel__header">
    <div>
      <h2>{selectedCategory ? selectedCategory : 'Categories'}</h2>
      <p>{summaryText}</p>
    </div>

    {#if selectedCategory}
      <button type="button" class="back-button" on:click={resetView}>
        ← Back to categories
      </button>
    {/if}
  </div>

  <div class="cloud-shell" bind:this={container}>
    {#if loading}
      <div class="state-message">Loading CSV…</div>
    {:else if error}
      <div class="state-message error">
        Could not load <code>{csvUrl}</code>: {error}
      </div>
    {:else if displayWords.length === 0}
      <div class="state-message">No rows matched the current view.</div>
    {:else}
      <svg
        width="100%"
        height={height}
        viewBox={`0 0 ${width} ${height}`}
        aria-label={selectedCategory ? `${selectedCategory} awardees word cloud` : 'Category word cloud'}
      >
        <rect width={width} height={height} rx="18" fill="#f8fafc" />
        <g>
          {#each positionedWords as word (word.text)}
            <g
              transform={`translate(${word.x + width / 2}, ${word.y + height / 2}) rotate(${word.rotate})`}
              tabindex={word.kind === 'category' ? 0 : undefined}
              role={word.kind === 'category' ? 'button' : undefined}
              aria-label={word.title}
              class:word-group={true}
              class:word-group--clickable={word.kind === 'category'}
              on:click={() => handleCategorySelect(word)}
              on:keydown={(event) => handleKeydown(event, word)}
            >
              <title>{word.title}</title>
              <text
                text-anchor="middle"
                dominant-baseline="central"
                font-size={word.size}
                font-family={fontFamily}
                font-weight={word.kind === 'category' ? 700 : 600}
                fill={word.fill}
              >
                {word.text}
              </text>
            </g>
          {/each}
        </g>
      </svg>
    {/if}
  </div>
</section>

<style>
  :global(body) {
    background: #f1f5f9;
  }

  .panel {
    display: grid;
    gap: 1rem;
  }

  .panel__header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
    flex-wrap: wrap;
  }

  h2 {
    margin: 0;
    font-size: clamp(1.4rem, 2vw, 2rem);
    line-height: 1.1;
    color: #0f172a;
  }

  p {
    margin: 0.45rem 0 0;
    max-width: 70ch;
    color: #475569;
    line-height: 1.5;
  }

  .cloud-shell {
    width: 100%;
    min-height: 420px;
  }

  svg {
    display: block;
    filter: drop-shadow(0 12px 30px rgba(15, 23, 42, 0.06));
  }

  .word-group text {
    user-select: none;
    transition: opacity 120ms ease, transform 120ms ease;
  }

  .word-group:hover text {
    opacity: 0.86;
  }

  .word-group--clickable {
    cursor: pointer;
  }

  .word-group--clickable:focus {
    outline: none;
  }

  .word-group--clickable:focus text {
    paint-order: stroke fill;
    stroke: rgba(15, 23, 42, 0.18);
    stroke-width: 8px;
    stroke-linejoin: round;
  }

  .back-button {
    border: 1px solid #cbd5e1;
    background: white;
    color: #0f172a;
    border-radius: 999px;
    padding: 0.7rem 1rem;
    font: inherit;
    cursor: pointer;
  }

  .back-button:hover {
    background: #f8fafc;
  }

  .state-message {
    display: grid;
    place-items: center;
    min-height: 420px;
    border-radius: 18px;
    background: #f8fafc;
    color: #334155;
    text-align: center;
    padding: 2rem;
  }

  .state-message.error {
    color: #991b1b;
    background: #fef2f2;
  }

  code {
    font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
    font-size: 0.92em;
  }
</style>
