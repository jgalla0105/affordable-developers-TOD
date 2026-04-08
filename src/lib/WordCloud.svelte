<script>
  import { onDestroy, onMount, tick } from 'svelte';
  import {
    ascending,
    csv,
    extent,
    hsl,
    max,
    rollups,
    scaleOrdinal,
    scaleSqrt,
    sum
  } from 'd3';
  import { base } from '$app/paths';

  export let csvUrl = `wordcloud-classified-data.csv`;
  export let fontFamily = 'Inter, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif';

  let cloudFactory;
  let container;
  let panelRoot;
  let resizeObserver;
  let currentLayout;

  let width = 960;
  let height = 620;

  let rows = [];
  let positionedWords = [];
  let selectedCategory = null;
  let hoveredCategoryKey = null;
  let hoveredAwardee = null;
  let pinnedAwardee = null;
  let tooltip = null;
  let loading = true;
  let error = '';
  let displayWords = [];

  

  const numberFormat = new Intl.NumberFormat('en-US');
  const fundsFormat = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 0
  });
  const fundingPerCapitaFormat = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 2,
    maximumFractionDigits: 2
  });

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

  function parseYear(value) {
    const numeric = Number(String(value ?? '').trim());
    return Number.isFinite(numeric) ? numeric : null;
  }

  function normalizeCategory(value) {
    const category = String(value ?? '').trim().toLowerCase();
    return category === 'afforable housing' ? 'affordable housing' : category;
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
    ? `Viewing ${new Set(selectedRows.map((row) => row.awardee)).size} awardees in "${selectedCategory}". Larger, darker names indicate higher funding per capita.`
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

  function getFundingPerCapita(funds, population) {
    if (!Number.isFinite(funds) || !Number.isFinite(population) || population <= 0) {
      return 0;
    }

    return funds / population;
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

  function getAwardeeWords(category, awardees = getAwardeeMetrics(category)) {
    const fundingPerCapitaValues = awardees.map(({ fundingPerCapita }) => fundingPerCapita);
    const minFont = clamp(width * 0.02, 14, 22);
    const maxFont = clamp(width * 0.085, 38, 76);
    const fontScale = createFontScale(fundingPerCapitaValues, minFont, maxFont);
    const fundingPerCapitaExtent = extent(fundingPerCapitaValues);

    return awardees.map(({ awardee, population, funds, fundingPerCapita }) => ({
      kind: 'awardee',
      text: awardee,
      category,
      awardee,
      value: fundingPerCapita,
      population,
      funds,
      fundingPerCapita,
      size: fontScale(fundingPerCapita),
      fill: saturateCategoryColor(category, fundingPerCapita, fundingPerCapitaExtent),
      title: `${awardee}: Funding per capita ${fundingPerCapitaFormat.format(fundingPerCapita)}, Total funds ${fundsFormat.format(funds)}, Population ${numberFormat.format(population)}`
    }));
    
  }

  function getAwardeeMetrics(category) {
    return rollups(
      rows.filter((row) => row.category === category),
      (group) => {
        const population = max(group, (row) => row.population) ?? 0;
        const funds = sum(group, (row) => row.funds) ?? 0;

        return {
          population,
          funds,
          fundingPerCapita: getFundingPerCapita(funds, population)
        };
      },
      (row) => row.awardee
    )
      .map(([awardee, metrics]) => ({
        awardee,
        ...metrics
      }))
      .sort(
        (a, b) =>
          b.fundingPerCapita - a.fundingPerCapita ||
          b.funds - a.funds ||
          ascending(a.awardee, b.awardee)
      );
  }

  function getAwardeeProjectEntries(category, awardee) {
    return rows
      .filter((row) => row.category === category && row.awardee === awardee)
      .sort(
        (a, b) =>
          (b.fiscalYear ?? -Infinity) - (a.fiscalYear ?? -Infinity) ||
          ascending(a.project || '', b.project || '')
      )
      .map((row, index) => ({
        key: `${awardee}-${category}-${row.fiscalYear ?? 'unknown'}-${index}`,
        fiscalYear: row.fiscalYear,
        project: row.project || 'Project description unavailable.'
      }));
  }

  function createAwardeeSelection(word) {
    return {
      text: word.text,
      awardee: word.awardee,
      category: word.category
    };
  }

  function clearAwardeeSelection() {
    hoveredAwardee = null;
    pinnedAwardee = null;
  }

  function clearAwardeePreview() {
    hoveredAwardee = null;
  }

  function hasAwardeeProjectEntries(word) {
    return getAwardeeProjectEntries(word.category, word.awardee).length > 0;
  }

  function previewAwardee(word) {
    if (!hasAwardeeProjectEntries(word)) {
      clearAwardeePreview();
      return;
    }

    hoveredAwardee = createAwardeeSelection(word);
  }

  function pinAwardee(word) {
    if (!hasAwardeeProjectEntries(word)) {
      clearAwardeeSelection();
      return;
    }

    pinnedAwardee = createAwardeeSelection(word);
    hoveredAwardee = null;
  }

  $: activeAwardee = pinnedAwardee ?? hoveredAwardee;

  $: tooltip = activeAwardee
    ? (() => {
        const entries = getAwardeeProjectEntries(activeAwardee.category, activeAwardee.awardee);

        return entries.length > 0
          ? {
              awardee: activeAwardee.awardee,
              category: activeAwardee.category,
              entries
            }
          : null;
      })()
    : null;

  $: activeWordKey = selectedCategory ? activeAwardee?.text ?? null : hoveredCategoryKey;
  $: awardeeMetrics = selectedCategory ? getAwardeeMetrics(selectedCategory) : [];


$: {
  const _rows = rows;
  const _width = width;
  const _categoryColor = categoryColor;
  const _selectedCategory = selectedCategory;
  const _awardeeMetrics = awardeeMetrics;

  displayWords = selectedCategory
    ? getAwardeeWords(selectedCategory, awardeeMetrics)
    : getCategoryWords();
}


  async function loadRows() {
    loading = true;
    error = '';

    try {
      const parsed = await csv(resolvedCsvUrl, (row) => {
        const awardee = (row.Awardee ?? row.awardee ?? '').trim();
        const category = normalizeCategory(row.Category ?? row.category);
        const population = parsePopulation(row.Population ?? row.population);
        const funds = parseFunds(row.funds ?? row.Funds);
        const fiscalYear = parseYear(row['fiscal-year'] ?? row.fiscalYear ?? row['Fiscal Year']);
        const project = String(row.project ?? row.Project ?? '').trim();

        if (!awardee || !category) {
          return null;
        }

        return {
          awardee,
          category,
          population,
          funds,
          fiscalYear,
          project
        };
        
      });

      const dedupedRows = [];
      const seenRows = new Set();

      // Some rows are duplicated verbatim in the source CSV; collapse only exact matches.
      for (const row of parsed.filter(Boolean)) {
        const key = [
          row.awardee,
          row.category,
          row.population,
          row.funds,
          row.fiscalYear ?? '',
          row.project
        ].join('||');

        if (!seenRows.has(key)) {
          seenRows.add(key);
          dedupedRows.push(row);
        }
      }

      rows = dedupedRows;
      

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

  async function preservePanelViewport(updateView) {
    const initialTop = panelRoot?.getBoundingClientRect().top ?? null;

    updateView();
    await tick();

    if (initialTop === null || !panelRoot) {
      return;
    }

    const nextTop = panelRoot.getBoundingClientRect().top;
    const scrollDelta = nextTop - initialTop;

    if (Math.abs(scrollDelta) > 1) {
      window.scrollBy({ top: scrollDelta });
    }
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

  async function handleCategorySelect(word) {
    if (word.kind === 'category') {
      await preservePanelViewport(() => {
        clearAwardeeSelection();
        hoveredCategoryKey = null;
        selectedCategory = word.category;
      });
    }
  }

  function handleKeydown(event, word) {
    if (word.kind !== 'category' && word.kind !== 'awardee') {
      return;
    }

    if (event.key === 'Enter' || event.key === ' ') {
      event.preventDefault();

      if (word.kind === 'category') {
        handleCategorySelect(word);
        return;
      }

      if (selectedCategory && word.kind === 'awardee') {
        pinAwardee(word);
      }
    }
  }

  async function resetView() {
    await preservePanelViewport(() => {
      clearAwardeeSelection();
      hoveredCategoryKey = null;
      selectedCategory = null;
    });
  }

  function handleWordClick(word) {
    if (word.kind === 'category') {
      handleCategorySelect(word);
      return;
    }

    if (selectedCategory && word.kind === 'awardee') {
      pinAwardee(word);
    }
  }

  function handleWordEnter(word) {
    if (!selectedCategory && word.kind === 'category') {
      hoveredCategoryKey = word.text;
      return;
    }

    if (selectedCategory && word.kind === 'awardee' && !pinnedAwardee) {
      previewAwardee(word);
    }
  }

  function handleWordLeave(word) {
    if (selectedCategory && word.kind === 'awardee') {
      if (!pinnedAwardee) {
        clearAwardeePreview();
      }

      return;
    }

    hoveredCategoryKey = null;
  }

  function handleWindowClick(event) {
    if (!selectedCategory) {
      return;
    }

    const target = event.target;

    if (!(target instanceof Element)) {
      return;
    }

    if (target.closest('[data-word-kind="awardee"]')) {
      return;
    }

    clearAwardeeSelection();
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

// Legend for the detail cloud uses the same funding-per-capita metric as size and color.
$: awardeeFundingPerCapita = awardeeMetrics
  .map(({ fundingPerCapita }) => fundingPerCapita)
  .filter((value) => Number.isFinite(value));

$: awardeeFundingPerCapitaExtent = extent(awardeeFundingPerCapita);

$: legendMinFundingPerCapita = awardeeFundingPerCapitaExtent[0] ?? 0;
$: legendMaxFundingPerCapita = awardeeFundingPerCapitaExtent[1] ?? 0;

$: legendTopColor = selectedCategory
  ? saturateCategoryColor(selectedCategory, legendMaxFundingPerCapita, awardeeFundingPerCapitaExtent)
  : null;

$: legendBottomColor = selectedCategory
  ? saturateCategoryColor(selectedCategory, legendMinFundingPerCapita, awardeeFundingPerCapitaExtent)
  : null;

</script>

<svelte:window on:click={handleWindowClick} />

<section class="panel" bind:this={panelRoot}>
  <div class="panel__layout" class:panel__layout--detail={selectedCategory}>
    <div class="panel__meta" class:panel__meta--detail={selectedCategory}>
      {#if selectedCategory}
        <div class="panel__controls">
          <button type="button" class="back-button" on:click={resetView}>
            <span class="back-button__icon" aria-hidden="true">←</span>
            <span>Back to categories</span>
          </button>
        </div>
      {/if}

      <div class="panel__header">
        <h2 class="panel__title" class:panel__title--detail={selectedCategory}>
          {panelTitle}
        </h2>
        <p class="panel__summary">{summaryText}</p>
      </div>

      {#if selectedCategory}
        <section
          class="awardee-panel"
          style={`--tooltip-accent: ${tooltipAccentColor};`}
          aria-live="polite"
        >
          <p class="awardee-panel__eyebrow">
            {tooltip ? 'Project details' : 'Hover or click for details'}
          </p>

          {#if tooltip}
            <h3 class="awardee-panel__title">{tooltip.awardee}</h3>

            <div class="awardee-panel__content">
              {#each tooltip.entries as entry (entry.key)}
                <article class="awardee-panel__entry">
                  <p class="awardee-panel__year">{entry.fiscalYear ?? 'Year unavailable'}</p>
                  <p class="awardee-panel__description">{entry.project}</p>
                </article>
              {/each}
            </div>
          {:else}
            <p class="awardee-panel__placeholder">
              Hover over a community name in the word cloud to preview each project year and description here, or click one to keep it selected.
            </p>
          {/if}
        </section>
      {/if}
    </div>

    <div class="cloud-shell" class:cloud-shell--detail={selectedCategory}>
      <div class="cloud-frame" bind:this={container}>
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
                <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
                <g
                  transform={`translate(${word.x + width / 2}, ${word.y + height / 2}) rotate(${word.rotate})`}
                  data-word-kind={word.kind}
                  tabindex={word.kind === 'category' || word.kind === 'awardee' ? 0 : undefined}
                  role={word.kind === 'category' || word.kind === 'awardee' ? 'button' : undefined}
                  aria-pressed={word.kind === 'awardee' ? pinnedAwardee?.text === word.text : undefined}
                  aria-label={word.title}
                  class:word-group={true}
                  class:word-group--clickable={word.kind === 'category' || word.kind === 'awardee'}
                  class:word-group--active={activeWordKey === word.text}
                  class:word-group--dimmed={!selectedCategory && hoveredCategoryKey && hoveredCategoryKey !== word.text}
                  on:click={() => handleWordClick(word)}
                  on:keydown={(event) => handleKeydown(event, word)}
                  on:mouseenter={() => handleWordEnter(word)}
                  on:mouseleave={() => handleWordLeave(word)}
                  on:focus={() => handleWordEnter(word)}
                  on:blur={() => handleWordLeave(word)}
                >
                  {#if word.kind === 'category' || word.kind === 'awardee'}
                    <title>{word.title}</title>
                  {/if}
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

      {#if selectedCategory}
        <div class="legend legend--side">
          <p class="legend__title">Funding intensity</p>

          <div class="legend__side-label legend__side-label--top">
            <span>Highest per capita</span>
            <span>{fundingPerCapitaFormat.format(legendMaxFundingPerCapita)}</span>
          </div>

          <div
            class="legend__bar legend__bar--vertical"
            style={`background: linear-gradient(to bottom, ${legendTopColor}, ${legendBottomColor});`}
            aria-label={`Funding per capita color scale for ${selectedCategory}`}
          ></div>

          <div class="legend__side-label legend__side-label--bottom">
            <span>Lowest per capita</span>
            <span>{fundingPerCapitaFormat.format(legendMinFundingPerCapita)}</span>
          </div>
        </div>
      {/if}
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

  .cloud-shell--detail {
    grid-template-columns: minmax(0, 1fr) auto;
    gap: clamp(0.75rem, 2vw, 1.25rem);
    align-items: center;
  }

  .cloud-frame {
    min-width: 0;
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
    stroke: rgba(15, 23, 42, 0.16);
    stroke-width: 5px;
    stroke-linejoin: round;
    filter: drop-shadow(0 0 0.55rem rgba(255, 255, 255, 0.95));
  }

  .awardee-panel {
    width: min(100%, var(--content-width, 72ch));
    margin: 0 auto;
    border-radius: 18px;
    border: 1px solid rgba(15, 23, 42, 0.08);
    border-top: 4px solid var(--tooltip-accent);
    background: linear-gradient(180deg, rgba(255, 255, 255, 0.98) 0%, #f8fbff 100%);
    box-shadow: 0 18px 36px rgba(15, 23, 42, 0.1);
    padding: 1rem;
    height: clamp(14rem, 30vw, 18rem);
    display: grid;
    align-content: start;
    grid-template-rows: auto auto minmax(0, 1fr);
  }

  .awardee-panel__eyebrow {
    margin: 0 0 0.6rem;
    font-size: 0.82rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: #64748b;
  }

  .awardee-panel__title {
    margin: 0 0 0.8rem;
    font-size: clamp(1.2rem, 2vw, 1.55rem);
    line-height: 1.1;
    color: #0f172a;
  }

  .awardee-panel__content {
    display: grid;
    gap: 0.7rem;
    min-height: 0;
    overflow-y: auto;
    padding-right: 0.2rem;
  }

  .awardee-panel__entry {
    display: grid;
    gap: 0.28rem;
    padding-bottom: 0.7rem;
    border-bottom: 1px solid rgba(148, 163, 184, 0.26);
  }

  .awardee-panel__entry:last-child {
    padding-bottom: 0;
    border-bottom: 0;
  }

  .awardee-panel__year,
  .awardee-panel__description,
  .awardee-panel__placeholder {
    margin: 0;
  }

  .awardee-panel__year {
    font-size: 0.84rem;
    font-weight: 700;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: var(--tooltip-accent);
  }

  .awardee-panel__description,
  .awardee-panel__placeholder {
    font-size: 0.95rem;
    line-height: 1.45;
    color: #334155;
  }

  .awardee-panel__placeholder {
    max-width: 28ch;
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

  .legend--side {
    width: clamp(4.75rem, 10vw, 6rem);
    padding: 0;
    border: 0;
    background: transparent;
    box-shadow: none;
    justify-items: end;
    align-content: center;
    text-align: right;
  }

  .legend__title {
    margin: 0;
    font-size: 0.82rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: #475569;
  }

  .legend--side .legend__title {
    width: 100%;
  }

  .legend__bar {
    height: 16px;
    border-radius: 999px;
    border: 1px solid rgba(15, 23, 42, 0.12);
  }

  .legend__bar--vertical {
    width: 16px;
    height: clamp(12rem, 34vw, 18rem);
    justify-self: end;
  }

  .legend__side-label {
    display: grid;
    gap: 0.18rem;
    font-size: 0.9rem;
    color: #475569;
  }

  .legend__side-label--top {
    align-self: end;
  }

  .legend__side-label--bottom {
    align-self: start;
  }

  @media (min-width: 900px) {
    .panel__layout {
      grid-template-columns: minmax(19rem, 28rem) minmax(0, 1fr);
    }

    .panel__layout--detail {
      align-items: stretch;
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

    .panel__meta--detail {
      width: 100%;
      min-height: 100%;
      grid-template-rows: auto auto minmax(0, 1fr);
    }

    .awardee-panel {
      margin: auto 0 0;
    }

    .back-button {
      justify-self: start;
    }
  }
</style>
