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
    scaleSqrt
  } from 'd3';
  import { base } from '$app/paths';

  export let csvUrl = `wordcloud-classified-data.csv`;
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
  let hoveredWordKey = null;
  let loading = true;
  let error = '';
  let displayWords = [];

  

  const numberFormat = new Intl.NumberFormat('en-US');

  $: resolvedCsvUrl = resolveAssetUrl(csvUrl);

  function clamp(value, min, max) {
    return Math.min(max, Math.max(min, value));
  }

  function resolveAssetUrl(path = '') {
    if (!path) {
      return path;
    }

    if (/^(?:[a-z]+:)?\/\//i.test(path)) {
      return path;
    }

    if (path.startsWith(base)) {
      return path;
    }

    if (path.startsWith('/')) {
      return `${base}${path}`;
    }

    return `${base}/${path}`;
  }

  function parsePopulation(value) {
    const numeric = Number(String(value ?? '').replace(/,/g, '').trim());
    return Number.isFinite(numeric) ? numeric : 0;
  }

  function parseFunds(value) {
    const numeric = Number(
        String(value ?? '')
          .replace(/\$/g, '')
          .replace(/\.00$/, '')
          .replace(/,/g, '')
          .trim()
      );

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
    const basePalette = ['#c69000', '#7a2f96', '#00778b'];

    return Array.from({ length: count }, (_, index) => basePalette[index % basePalette.length]);
  }

  $: categories = Array.from(new Set(rows.map((row) => row.category))).sort(ascending);
  $: categoryColor = scaleOrdinal().domain(categories).range(buildPalette(categories.length));

  $: selectedRows = selectedCategory
    ? rows.filter((row) => row.category === selectedCategory)
    : [];

  $: panelTitle = selectedCategory ? selectedCategory : 'TOD CATEGORIES';

  $: summaryText = selectedCategory
    ? `Viewing ${new Set(selectedRows.map((row) => row.awardee)).size} awardees in "${selectedCategory}". Larger names correspond to awardees in larger-population communities, while deeper color indicates higher funding levels.`
    : 'Start with the overview to compare which TOD project categories appear most often across the dataset. Select a category to reveal the awardees and communities behind it.';

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

  function saturateCategoryColor(category, fund, [minFunds, maxFunds]) {
    const base = hsl(categoryColor(category) ?? '#64748b');

    if (!Number.isFinite(minFunds) || !Number.isFinite(maxFunds) || minFunds === maxFunds) {
      return hsl(base.h, clamp(base.s + 0.18, 0.42, 0.88), clamp(base.l - 0.12, 0.36, 0.56)).formatHex();
    }

    const t = clamp((fund - minFunds) / (maxFunds - minFunds), 0, 1);
    const saturation = 0.32 + t * 0.58;
    const lightness = 0.72 - t * 0.30;

    return hsl(base.h, saturation, lightness).formatHex();
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

    // We keep one population and amount of funds per awardee
    const awardees = rollups(
      filtered,
      (group) => [
        max(group, (row) => row.population) ?? 0,
        max(group, (row) => row.funds) ?? 0
      ],
      (row) => row.awardee
    )
      .map(([awardee, [population, funds]]) => [awardee, population, funds])
      .sort((a, b) => b[1] - a[1] || ascending(a[0], b[0]));

    const populations = awardees.map(([, population, _funds]) => population);
    const minFont = clamp(width * 0.02, 14, 22);
    const maxFont = clamp(width * 0.085, 38, 76);
    const fontScale = createFontScale(populations, minFont, maxFont);
    const populationExtent = extent(populations);
    
    const funds = awardees.map(([, , fund]) => fund);
    const fundsExtent = extent(funds); 

    //saturation is based on how much money (funds) the awardee got
    return awardees.map(([awardee, population, fund]) => ({
      kind: 'awardee',
      text: awardee,
      category,
      awardee,
      value: population,
      population,
      size: fontScale(population),
      fill: saturateCategoryColor(category, fund, fundsExtent),
      // fill: saturateCategoryColor(category, population, populationExtent),
      title: `${awardee}: Population ${numberFormat.format(population)}, Funds ${numberFormat.format(fund)}`
    }));
    
  }
  

$: {
  const _rows = rows;
  const _width = width;
  const _categoryColor = categoryColor;
  const _selectedCategory = selectedCategory;

  displayWords = selectedCategory
    ? getAwardeeWords(selectedCategory)
    : getCategoryWords();
}


  async function loadRows() {
    loading = true;
    error = '';

    try {
      const parsed = await csv(resolvedCsvUrl, (row) => {
        const awardee = (row.Awardee ?? row.awardee ?? '').trim();
        const category = (row.Category ?? row.category ?? '').trim();
        const population = parsePopulation(row.Population ?? row.population);
        const funds = parseFunds(row.funds ?? row.funds);

        if (!awardee || !category) {
          return null;
        }

        return {
          awardee,
          category,
          population,
          funds
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

    // rotate each word by same amount every render (making sure "mixed use development" isn't off-screen)
    function getWordRotation(text) {
        if (text.length == 21) {
          return 0;
        }
      return hashString(text) % 2 === 0 ? 0 : -90;
    
    }

    currentLayout = cloudFactory()
      .size([width, height])
      .words(wordsForLayout)
      .padding(selectedCategory ? 3 : 8)
      .rotate((word) => getWordRotation(word.text))
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
      hoveredWordKey = null;
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
    hoveredWordKey = null;
    selectedCategory = null;
  }

  function handleWordEnter(word) {
    if (!selectedCategory && word.kind === 'category') {
      hoveredWordKey = word.text;
    }
  }

  function clearWordHover() {
    hoveredWordKey = null;
  }

  onMount(async () => {
    const [{ default: importedCloud }] = await Promise.all([import('d3-cloud')]);

    cloudFactory = importedCloud;
    await loadRows();

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

  $: {
    
    const _displayWords = displayWords;
    const _selectedCategory = selectedCategory;
    const _cloudFactory = cloudFactory;
    const _loading = loading;
    const _error = error;
    const _width = width;
    const _height = height;

    
    if (_cloudFactory && !_loading && !_error && _width > 0 && _height > 0 && _displayWords.length > 0) {
      runLayout();
    } else if (!_loading && !_error) {
      stopLayout();
      positionedWords = [];
    }
}

//making legend for awardee wordcloud
$: awardeeFunds = selectedRows.map((row) => row.funds).filter((v) => Number.isFinite(v));

$: awardeeFundsExtent = extent(awardeeFunds);

$: legendMinFunds = awardeeFundsExtent[0] ?? 0;
$: legendMaxFunds = awardeeFundsExtent[1] ?? 0;

$: legendStartColor = selectedCategory
  ? saturateCategoryColor(selectedCategory, legendMinFunds, awardeeFundsExtent)
  : null;

$: legendEndColor = selectedCategory
  ? saturateCategoryColor(selectedCategory, legendMaxFunds, awardeeFundsExtent)
  : null;
  
</script>

<section class="panel">
  <div class="panel__layout" class:panel__layout--detail={selectedCategory}>
    <div class="panel__meta">
      <div class="panel__header">
        <h2 class="panel__title" class:panel__title--detail={selectedCategory}>
          {panelTitle}
        </h2>
        <p class="panel__summary">{summaryText}</p>
      </div>

      {#if selectedCategory}
        <div class="panel__controls">
          <button type="button" class="back-button" on:click={resetView}>
            <span class="back-button__icon" aria-hidden="true">←</span>
            <span>Back to categories</span>
          </button>

          <div class="legend">
            <p class="legend__title">Funding intensity</p>

            <div class="legend__label-row">
              <span>Lowest funding</span>
              <span>Highest funding</span>
            </div>

            <div
              class="legend__bar"
              style={`background: linear-gradient(to right, ${legendStartColor}, ${legendEndColor});`}
              aria-label={`Funding color scale for ${selectedCategory}`}
            ></div>

            <div class="legend__value-row">
              <span>${numberFormat.format(legendMinFunds)}</span>
              <span>${numberFormat.format(legendMaxFunds)}</span>
            </div>
          </div>
        </div>
      {/if}
    </div>

    <div class="cloud-shell" bind:this={container}>
      <div class="cloud-frame">
        {#if loading}
          <div class="state-message">Loading CSV…</div>
        {:else if error}
          <div class="state-message error">
            Could not load <code>{resolvedCsvUrl}</code>: {error}
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
            <rect class="cloud-canvas" width={width} height={height} rx="24" />
            <g>
              {#each positionedWords as word (word.text)}
                <g
                  transform={`translate(${word.x + width / 2}, ${word.y + height / 2}) rotate(${word.rotate})`}
                  tabindex={word.kind === 'category' ? 0 : undefined}
                  role={word.kind === 'category' ? 'button' : undefined}
                  aria-label={word.title}
                  class:word-group={true}
                  class:word-group--clickable={word.kind === 'category'}
                  class:word-group--active={hoveredWordKey === word.text}
                  class:word-group--dimmed={!selectedCategory && hoveredWordKey && hoveredWordKey !== word.text}
                  on:click={() => handleCategorySelect(word)}
                  on:keydown={(event) => handleKeydown(event, word)}
                  on:mouseenter={() => handleWordEnter(word)}
                  on:mouseleave={clearWordHover}
                  on:focus={() => handleWordEnter(word)}
                  on:blur={clearWordHover}
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
    </div>
  </div>
</section>

<style>
  :global(body) {
    /* background: #f1f5f9; */
    font-family:'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }

  .panel {
    display: grid;
    gap: clamp(1rem, 2vw, 1.5rem);
  }

  .panel__layout {
    display: grid;
    gap: clamp(1.5rem, 3vw, 2.75rem);
    align-items: start;
  }

  .panel__meta {
    display: grid;
    gap: 1.1rem;
    align-content: start;
  }

  .panel__header {
    display: grid;
    grid-template-columns: minmax(0, 1fr);
    justify-items: center;
    gap: 0.85rem;
    width: min(100%, var(--content-width, 72ch));
    margin: 0 auto;
    text-align: center;
  }

  .panel__title {
    margin: 0;
    font-family: 'DotFont', sans-serif;
    font-size: clamp(2.5rem, 7vw, 3.75rem);
    line-height: 0.92;
    letter-spacing: 0.06em;
    text-align: center;
    color: rgb(59, 59, 59);
    text-wrap: balance;
  }

  .panel__title--detail {
    font-size: clamp(2.1rem, 5vw, 3.3rem);
    line-height: 0.98;
    letter-spacing: 0.05em;
    text-transform: uppercase;
  }

  .panel__summary {
    margin: 0;
    width: 100%;
    font-size: clamp(1.05rem, 2vw, 1.35rem);
    line-height: 1.6;
    text-align: left;
    color: rgb(59, 59, 59);
  }

  .panel__controls {
    display: grid;
    gap: 1rem;
    width: min(100%, var(--content-width, 72ch));
    margin: 0 auto;
  }

  .cloud-shell {
    width: 100%;
    display: grid;
    align-items: start;
  }

  .cloud-frame {
    min-height: 420px;
    border-radius: 24px;
    border: 2px solid rgba(148, 163, 184, 0.32);
    background: linear-gradient(180deg, #ffffff 0%, #fbfdff 100%);
    box-shadow: 0 18px 36px rgba(15, 23, 42, 0.08);
    overflow: hidden;
  }

  svg {
    display: block;
  }

  .cloud-canvas {
    fill: rgba(255, 255, 255, 0.96);
    stroke: rgba(148, 163, 184, 0.18);
    stroke-width: 2;
  }

  .word-group {
    opacity: 1;
    transition: opacity 160ms ease;
  }

  .word-group text {
    user-select: none;
    vector-effect: non-scaling-stroke;
    transition: opacity 160ms ease, filter 160ms ease, stroke 160ms ease, stroke-width 160ms ease;
  }

  .word-group--dimmed {
    opacity: 0.24;
  }

  .word-group--clickable {
    cursor: pointer;
  }

  .word-group--clickable:focus {
    outline: none;
  }

  .word-group--active text,
  .word-group--clickable:hover text,
  .word-group--clickable:focus text {
    paint-order: stroke fill;
    stroke: rgba(15, 23, 42, 0.24);
    stroke-width: 8px;
    stroke-linejoin: round;
    filter:
      drop-shadow(0 0 0.45rem rgba(255, 255, 255, 0.95))
      drop-shadow(0 0.7rem 1rem rgba(15, 23, 42, 0.16));
  }

  .back-button {
    justify-self: center;
    display: inline-flex;
    align-items: center;
    gap: 0.65rem;
    border: 1px solid rgba(148, 163, 184, 0.55);
    background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
    color: rgb(59, 59, 59);
    border-radius: 999px;
    padding: 0.8rem 1.15rem;
    font: inherit;
    font-weight: 700;
    cursor: pointer;
    box-shadow: 0 10px 20px rgba(15, 23, 42, 0.08);
    transition: transform 160ms ease, box-shadow 160ms ease, border-color 160ms ease, background 160ms ease;
  }

  .back-button:hover {
    transform: translateY(-1px);
    border-color: rgba(71, 85, 105, 0.35);
    background: #ffffff;
    box-shadow: 0 14px 24px rgba(15, 23, 42, 0.12);
  }

  .back-button__icon {
    font-size: 1.05em;
    line-height: 1;
  }

  .state-message {
    display: grid;
    place-items: center;
    min-height: inherit;
    background: transparent;
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

  .legend {
    display: grid;
    gap: 0.55rem;
    width: 100%;
    padding: 1rem 1rem 0.95rem;
    border-radius: 20px;
    border: 1px solid rgba(148, 163, 184, 0.26);
    background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
    box-shadow: 0 12px 24px rgba(15, 23, 42, 0.06);
  }

  .legend__title {
    margin: 0;
    font-size: 0.82rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: #475569;
  }

  .legend__label-row,
  .legend__value-row {
    display: flex;
    justify-content: space-between;
    gap: 1rem;
    font-size: 0.9rem;
    color: #475569;
  }

  .legend__bar {
    height: 16px;
    border-radius: 999px;
    border: 1px solid rgba(15, 23, 42, 0.12);
  }

  @media (min-width: 900px) {
    .panel__layout {
      grid-template-columns: minmax(19rem, 28rem) minmax(0, 1fr);
    }

    .panel__header,
    .panel__controls {
      width: 100%;
      margin: 0;
    }

    .panel__header {
      justify-items: start;
      text-align: left;
    }

    .panel__title,
    .panel__summary {
      text-align: left;
    }

    .panel__controls {
      justify-items: start;
    }

    .back-button {
      justify-self: start;
    }
  }
</style>
