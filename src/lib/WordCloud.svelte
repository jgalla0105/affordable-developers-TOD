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

  let container;
  let measureLayer;
  let panelRoot;
  let resizeObserver;
  let layoutRun = 0;

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

  

  const SVG_NAMESPACE = 'http://www.w3.org/2000/svg';
  const GOLDEN_ANGLE = Math.PI * (3 - Math.sqrt(5));

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

  function buildPalette(count) {
    const basePalette = ['#00843D', '#7C878E', '#ED8B00', '#FFC72C', '#DA291C', '#80276C', '#003DA5'];

    return Array.from({ length: count }, (_, index) => basePalette[index % basePalette.length]);
  }

  $: categories = Array.from(new Set(rows.map((row) => row.category))).sort(ascending);
  $: categoryColor = scaleOrdinal().domain(categories).range(buildPalette(categories.length));

  $: selectedRows = selectedCategory
    ? rows.filter((row) => row.category === selectedCategory)
    : [];

  $: panelTitle = selectedCategory ? selectedCategory : 'TOD CATEGORIES';
  $: tooltipAccentColor = selectedCategory ? categoryColor(selectedCategory) ?? '#0ea5c6' : '#0ea5c6';

  $: summaryText = selectedCategory
    ? `Viewing ${new Set(selectedRows.map((row) => row.awardee)).size} awardees in "${selectedCategory}". Longer, darker bars indicate higher funding per capita.`
    : 'Start with the overview to compare which TOD project categories appear most often across the dataset. Select a category to reveal the awardees spending on it.';

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

    const minFont = clamp(width * 0.026, 18, 28);
    const maxFont = clamp(width * 0.078, 44, 78);
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

  function getAwardeeBars(category, awardees = getAwardeeMetrics(category)) {
    const fundingPerCapitaValues = awardees.map(({ fundingPerCapita }) => fundingPerCapita);
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
  $: awardeeBars = selectedCategory ? getAwardeeBars(selectedCategory, awardeeMetrics) : [];
  $: visualItemCount = selectedCategory ? awardeeBars.length : displayWords.length;


$: {
  const _rows = rows;
  const _width = width;
  const _categoryColor = categoryColor;
  const _selectedCategory = selectedCategory;

  displayWords = selectedCategory ? [] : getCategoryWords();
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
    layoutRun += 1;
  }

  function getLayoutBounds() {
    const horizontalPadding = clamp(width * 0.052, selectedCategory ? 22 : 30, selectedCategory ? 46 : 58);
    const verticalPadding = clamp(height * 0.085, 34, 64);

    return {
      left: horizontalPadding,
      right: width - horizontalPadding,
      top: verticalPadding,
      bottom: height - verticalPadding,
      width: Math.max(0, width - horizontalPadding * 2),
      height: Math.max(0, height - verticalPadding * 2)
    };
  }

  function estimateTextBounds(word, fontSize) {
    const estimatedWidth = Array.from(word.text).reduce((total, character) => {
      if (character === ' ') {
        return total + fontSize * 0.33;
      }

      if (/[A-Z0-9]/.test(character)) {
        return total + fontSize * 0.66;
      }

      return total + fontSize * 0.54;
    }, 0);

    return {
      x: -estimatedWidth / 2,
      y: -fontSize * 0.55,
      width: estimatedWidth,
      height: fontSize * 1.1
    };
  }

  function measureRenderedText(word, fontSize) {
    if (!measureLayer || typeof document === 'undefined') {
      return estimateTextBounds(word, fontSize);
    }

    const textNode = document.createElementNS(SVG_NAMESPACE, 'text');
    textNode.textContent = word.text;
    textNode.setAttribute('x', '0');
    textNode.setAttribute('y', '0');
    textNode.setAttribute('text-anchor', 'middle');
    textNode.setAttribute('dominant-baseline', 'central');
    textNode.setAttribute('font-family', fontFamily);
    textNode.setAttribute('font-size', String(fontSize));
    textNode.setAttribute('font-weight', word.kind === 'category' ? '700' : '600');

    measureLayer.appendChild(textNode);

    try {
      const bbox = textNode.getBBox();

      if (Number.isFinite(bbox.width) && Number.isFinite(bbox.height) && bbox.width > 0 && bbox.height > 0) {
        return {
          x: bbox.x,
          y: bbox.y,
          width: bbox.width,
          height: bbox.height
        };
      }
    } catch (err) {
      // Fall through to a conservative estimate if the browser cannot measure yet.
    } finally {
      textNode.remove();
    }

    return estimateTextBounds(word, fontSize);
  }

  function getWordBuffer(fontSize, kind) {
    return clamp(fontSize * (kind === 'category' ? 0.19 : 0.16), 6, kind === 'category' ? 18 : 14);
  }

  function getMinimumFontSize(word) {
    const sensibleMinimum = word.kind === 'category'
      ? clamp(width * 0.018, 15, 21)
      : clamp(width * 0.014, 11, 15);

    return Math.min(word.size, sensibleMinimum);
  }

  function createCollisionBox(x, y, textBounds, buffer) {
    return {
      left: x + textBounds.x - buffer,
      right: x + textBounds.x + textBounds.width + buffer,
      top: y + textBounds.y - buffer,
      bottom: y + textBounds.y + textBounds.height + buffer
    };
  }

  function boxFitsLayout(box, bounds) {
    return (
      box.left >= bounds.left &&
      box.right <= bounds.right &&
      box.top >= bounds.top &&
      box.bottom <= bounds.bottom
    );
  }

  function boxesOverlap(a, b) {
    return a.left < b.right && a.right > b.left && a.top < b.bottom && a.bottom > b.top;
  }

  function anchorForWord(index, total, bounds, seedOffset) {
    const centerX = bounds.left + bounds.width / 2;
    const centerY = bounds.top + bounds.height / 2;

    if (index === 0 || total <= 1) {
      return { x: centerX, y: centerY, angle: seedOffset };
    }

    const progress = index / Math.max(1, total - 1);
    const spread = 0.2 + Math.sqrt(progress) * 0.8;
    const angle = seedOffset + index * GOLDEN_ANGLE;

    return {
      x: centerX + Math.cos(angle) * bounds.width * 0.34 * spread,
      y: centerY + Math.sin(angle) * bounds.height * 0.32 * spread,
      angle
    };
  }

  function findPlacement(textBounds, buffer, index, total, placedBoxes, bounds, seedOffset) {
    if (textBounds.width + buffer * 2 > bounds.width || textBounds.height + buffer * 2 > bounds.height) {
      return null;
    }

    const anchor = anchorForWord(index, total, bounds, seedOffset);
    const maxRadius = Math.hypot(bounds.width, bounds.height);
    const radialStep = Math.max(5, Math.min(textBounds.width, textBounds.height) * 0.2);

    for (let radius = 0; radius <= maxRadius; radius += radialStep) {
      const attempts = radius === 0 ? 1 : Math.ceil(14 + radius / radialStep);

      for (let attempt = 0; attempt < attempts; attempt += 1) {
        const angle = anchor.angle + attempt * GOLDEN_ANGLE + radius * 0.012;
        const x = anchor.x + Math.cos(angle) * radius;
        const y = anchor.y + Math.sin(angle) * radius * 0.76;
        const box = createCollisionBox(x, y, textBounds, buffer);

        if (!boxFitsLayout(box, bounds)) {
          continue;
        }

        if (!placedBoxes.some((placedBox) => boxesOverlap(box, placedBox))) {
          return { x, y, box };
        }
      }
    }

    return null;
  }

  function buildMeasuredLayout(words) {
    const bounds = getLayoutBounds();
    const seedKey = `${selectedCategory ?? 'categories'}-${width}-${height}`;
    const seedOffset = (hashString(seedKey) / 4294967296) * Math.PI * 2;
    const placedWords = [];
    const placedBoxes = [];
    const sortedWords = [...words].sort(
      (a, b) => b.size - a.size || b.value - a.value || ascending(a.text, b.text)
    );

    for (let index = 0; index < sortedWords.length; index += 1) {
      const word = sortedWords[index];
      const minimumFontSize = getMinimumFontSize(word);
      const fontSizes = [];

      for (let size = word.size; size > minimumFontSize; size *= 0.93) {
        fontSizes.push(size);
      }

      fontSizes.push(minimumFontSize);

      for (const fontSize of fontSizes) {
        const textBounds = measureRenderedText(word, fontSize);
        const buffer = getWordBuffer(fontSize, word.kind);
        const placement = findPlacement(
          textBounds,
          buffer,
          index,
          sortedWords.length,
          placedBoxes,
          bounds,
          seedOffset
        );

        if (placement) {
          placedBoxes.push(placement.box);
          placedWords.push({
            ...word,
            size: fontSize,
            x: placement.x,
            y: placement.y,
            rotate: 0
          });
          break;
        }
      }
    }

    return placedWords;
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

  async function runLayout() {
    if (!measureLayer || loading || error || displayWords.length === 0 || width <= 0 || height <= 0) {
      return;
    }

    const runId = layoutRun + 1;
    layoutRun = runId;
    positionedWords = [];
    await tick();

    if (typeof document !== 'undefined' && document.fonts?.ready) {
      await document.fonts.ready;
    }

    if (runId !== layoutRun) {
      return;
    }

    const nextWords = buildMeasuredLayout(displayWords);

    if (runId === layoutRun) {
      positionedWords = nextWords;
    }
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

  $: height = clamp(Math.round(width * 0.72), 460, 720);

  $: {
    
    const _displayWords = displayWords;
    const _selectedCategory = selectedCategory;
    const _measureLayer = measureLayer;
    const _loading = loading;
    const _error = error;
    const _width = width;
    const _height = height;

    
    if (!_selectedCategory && _measureLayer && !_loading && !_error && _width > 0 && _height > 0 && _displayWords.length > 0) {
      void runLayout();
    } else if (!_loading && !_error) {
      stopLayout();
      positionedWords = [];
    }
}

// Legend and bars use the same funding-per-capita metric as length and color.
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

$: chartTop = selectedCategory ? clamp(height * 0.18, 96, 126) : 0;
$: chartInsetX = selectedCategory ? clamp(width * 0.035, 24, 50) : 0;
$: chartLabelWidth = selectedCategory ? clamp(width * 0.19, 118, 220) : 0;
$: chartValueWidth = selectedCategory ? clamp(width * 0.11, 86, 112) : 0;
$: chartValueGap = selectedCategory ? clamp(width * 0.024, 20, 34) : 0;
$: chartLeft = chartInsetX + chartLabelWidth;
$: chartValueX = width - chartInsetX - chartValueWidth;
$: chartValueEndX = width - chartInsetX;
$: chartAxisLabelY = selectedCategory ? chartTop - clamp(height * 0.08, 44, 58) : 0;
$: chartAxisY = selectedCategory ? chartTop - clamp(height * 0.04, 22, 30) : 0;
$: chartGridTop = selectedCategory ? chartTop - clamp(height * 0.018, 9, 13) : 0;
$: chartGridBottom = selectedCategory ? height - clamp(height * 0.16, 88, 118) : 0;
$: chartPlotWidth = Math.max(80, chartValueX - chartValueGap - chartLeft);
$: barRowHeight = awardeeBars.length > 0
  ? Math.max(0, (chartGridBottom - chartTop) / awardeeBars.length)
  : 0;
$: barThickness = clamp(barRowHeight * 0.38, 14, 26);
$: maxBarValue = max(awardeeBars, (bar) => bar.fundingPerCapita) ?? 0;
$: barTickValues = maxBarValue > 0 ? [0, maxBarValue / 2, maxBarValue] : [0];

  function getBarWidth(value) {
    if (!Number.isFinite(value) || maxBarValue <= 0) {
      return 0;
    }

    return (value / maxBarValue) * chartPlotWidth;
  }

  function getBarCenterY(index) {
    return chartTop + index * barRowHeight + barRowHeight / 2;
  }

  function getBarY(index) {
    return getBarCenterY(index) - barThickness / 2;
  }

  function getBarHitY(index) {
    return chartTop + index * barRowHeight;
  }

  function getBarTickX(value) {
    return chartLeft + getBarWidth(value);
  }

  function getBarTickAnchor(index) {
    if (barTickValues.length === 1) {
      return 'middle';
    }

    if (index === 0) {
      return 'start';
    }

    if (index === barTickValues.length - 1) {
      return 'end';
    }

    return 'middle';
  }

</script>

<svelte:window on:click={handleWindowClick} />

<svg class="measurement-svg" aria-hidden="true" focusable="false">
  <g bind:this={measureLayer}></g>
</svg>

<section class="panel" bind:this={panelRoot}>
  <div class="panel_layout" class:panel_layout--detail={selectedCategory}>
    <div class="panel_meta" class:panel_meta--detail={selectedCategory}>
      {#if selectedCategory}
        <div class="panel_controls">
          <button type="button" class="back-button" on:click={resetView}>
            <span class="back-button_icon" aria-hidden="true">←</span>
            <span>Back to categories</span>
          </button>
        </div>
      {/if}

      <div class="panel_header">
        <h2 class="panel_title" class:panel_title--detail={selectedCategory}>
          {panelTitle}
        </h2>
        <p class="panel_summary">{summaryText}</p>
      </div>

      {#if selectedCategory}
        <section
          class="awardee-panel"
          style={`--tooltip-accent: ${tooltipAccentColor};`}
          aria-live="polite"
        >
          <p class="awardee-panel_eyebrow">
            {tooltip ? 'Project details' : 'Hover or click for details'}
          </p>

          {#if tooltip}
            <h3 class="awardee-panel_title">{tooltip.awardee}</h3>

            <div class="awardee-panel_content">
              {#each tooltip.entries as entry (entry.key)}
                <article class="awardee-panel_entry">
                  <p class="awardee-panel_year">{entry.fiscalYear ?? 'Year unavailable'}</p>
                  <p class="awardee-panel_description">{entry.project}</p>
                </article>
              {/each}
            </div>
          {:else}
            <p class="awardee-panel_placeholder">
              Hover over a community bar to preview each project year and description here, or click one to keep it selected.
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
        {:else if visualItemCount === 0}
          <div class="state-message">No rows matched the current view.</div>
        {:else}
          <svg
            width="100%"
            height={height}
            viewBox={`0 0 ${width} ${height}`}
            aria-label={selectedCategory ? `${selectedCategory} awardees funding per capita bar chart` : 'Category word cloud'}
          >
            <rect class="cloud-canvas" width={width} height={height} rx="8" />
            {#if selectedCategory}
	              <g class="bar-chart" aria-hidden="true">
	                {#each barTickValues as tick, index}
	                  <g class="bar-chart_tick" transform={`translate(${getBarTickX(tick)}, 0)`}>
	                    <line y1={chartGridTop} y2={chartGridBottom} />
	                    <text y={chartAxisY} text-anchor={getBarTickAnchor(index)} dominant-baseline="middle">
	                      {fundingPerCapitaFormat.format(tick)}
	                    </text>
	                  </g>
	                {/each}
	                <text
	                  class="bar-chart_axis-label"
	                  x={chartLeft + chartPlotWidth / 2}
	                  y={chartAxisLabelY}
	                  text-anchor="middle"
	                  dominant-baseline="middle"
	                >
	                  Funding per capita
	                </text>
	                <text
	                  class="bar-chart_value-label"
	                  x={chartValueEndX}
	                  y={chartAxisLabelY}
	                  text-anchor="end"
	                  dominant-baseline="middle"
	                >
	                  Value
	                </text>
              </g>

              <g>
                {#each awardeeBars as bar, index (bar.text)}
                  <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
                  <g
                    data-word-kind={bar.kind}
                    tabindex="0"
                    role="button"
                    aria-pressed={pinnedAwardee?.text === bar.text}
                    aria-label={bar.title}
                    class:bar-row={true}
                    class:bar-row--active={activeWordKey === bar.text}
                    on:click={() => handleWordClick(bar)}
                    on:keydown={(event) => handleKeydown(event, bar)}
                    on:mouseenter={() => handleWordEnter(bar)}
                    on:mouseleave={() => handleWordLeave(bar)}
                    on:focus={() => handleWordEnter(bar)}
                    on:blur={() => handleWordLeave(bar)}
                  >
                    <title>{bar.title}</title>
                    <rect
                      class="bar-row_hit-area"
                      x="0"
                      y={getBarHitY(index)}
                      width={width}
                      height={barRowHeight}
                    />
                    <text
                      class="bar-row_label"
                      x={chartLeft - 14}
                      y={getBarCenterY(index)}
                      text-anchor="end"
                      dominant-baseline="central"
                    >
                      {bar.awardee}
                    </text>
                    <rect
                      class="bar-row_track"
                      x={chartLeft}
                      y={getBarY(index)}
                      width={chartPlotWidth}
                      height={barThickness}
                      rx="5"
                    />
                    <rect
                      class="bar-row_bar"
                      x={chartLeft}
                      y={getBarY(index)}
                      width={getBarWidth(bar.fundingPerCapita)}
                      height={barThickness}
                      rx="5"
                      fill={bar.fill}
                    />
                    <text
                      class="bar-row_value"
                      x={chartValueEndX}
                      y={getBarCenterY(index)}
                      text-anchor="end"
                      dominant-baseline="central"
                    >
                      {fundingPerCapitaFormat.format(bar.fundingPerCapita)}
                    </text>
                  </g>
                {/each}
              </g>
            {:else}
              <g>
                {#each positionedWords as word (word.text)}
                  <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
                  <g
                    transform={`translate(${word.x}, ${word.y}) rotate(${word.rotate})`}
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
            {/if}
          </svg>

        {/if}
      </div>

      {#if selectedCategory}
        <div class="legend legend--side">
          <p class="legend_title">Per capita funds</p>

          <div class="legend_side-label legend_side-label--top">
            <span>Highest per capita</span>
            <span>{fundingPerCapitaFormat.format(legendMaxFundingPerCapita)}</span>
          </div>

          <div
            class="legend_bar legend_bar--vertical"
            style={`background: linear-gradient(to bottom, ${legendTopColor}, ${legendBottomColor});`}
            aria-label={`Funding per capita color scale for ${selectedCategory}`}
          ></div>

          <div class="legend_side-label legend_side-label--bottom">
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

  .measurement-svg {
    position: fixed;
    top: -10000px;
    left: -10000px;
    width: 1px;
    height: 1px;
    overflow: visible;
    opacity: 0;
    pointer-events: none;
  }

  .panel {
    display: grid;
    gap: clamp(1rem, 2vw, 1.5rem);
    width: 100%;
  }

  .panel_layout {
    display: grid;
    gap: clamp(1rem, 3vw, 2.25rem);
    align-items: start;
  }

  .panel_meta {
    display: grid;
    gap: 1rem;
    align-content: start;
    align-self: start;
    min-width: 0;
    border: 1px solid rgba(148, 163, 184, 0.28);
    border-radius: 8px;
    background: linear-gradient(180deg, rgba(255, 255, 255, 0.98) 0%, rgba(248, 251, 255, 0.82) 100%);
    padding: 1.1rem;
  }

  .panel_header {
    display: grid;
    grid-template-columns: minmax(0, 1fr);
    justify-items: start;
    gap: 0.7rem;
    width: 100%;
    margin: 0;
    text-align: left;
  }

  .panel_title {
    margin: 0;
    font-size: 1rem;
    line-height: 1.1;
    letter-spacing: 0;
    text-align: left;
    text-transform: uppercase;
    color: rgb(59, 59, 59);
    text-wrap: balance;
  }

  .panel_title--detail {
    font-size: 1.35rem;
    line-height: 1.14;
    letter-spacing: 0;
    text-transform: uppercase;
  }

  .panel_summary {
    margin: 0;
    width: 100%;
    font-size: 0.98rem;
    line-height: 1.55;
    text-align: left;
    color: #475569;
  }

  .panel_controls {
    display: grid;
    gap: 1rem;
    width: 100%;
    margin: 0;
  }

  .cloud-shell {
    width: 100%;
    display: grid;
    align-items: start;
  }

  .cloud-shell--detail {
    grid-template-columns: minmax(0, 1fr) auto;
    gap: clamp(0.75rem, 2vw, 1.25rem);
    align-items: start;
  }

  .cloud-frame {
    min-width: 0;
    border-radius: 8px;
    border: 1px solid rgba(148, 163, 184, 0.34);
    background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
    box-shadow: 0 14px 28px rgba(15, 23, 42, 0.07);
    overflow: hidden;
    padding: clamp(1rem, 2vw, 1.45rem);
  }

  svg {
    display: block;
  }

  .cloud-frame > svg {
    width: 100%;
    height: auto;
  }

  .cloud-canvas {
    fill: rgba(255, 255, 255, 0.82);
    stroke: rgba(148, 163, 184, 0.2);
    stroke-width: 1;
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
  .word-group--clickable:focus-visible text {
    paint-order: stroke fill;
    stroke: rgba(255, 255, 255, 0.92);
    stroke-width: 4px;
    stroke-linejoin: round;
    filter: drop-shadow(0 0.35rem 0.55rem rgba(15, 23, 42, 0.16));
  }

  .bar-chart_tick line {
    stroke: rgba(148, 163, 184, 0.32);
    stroke-width: 1;
  }

  .bar-chart_tick text,
  .bar-chart_axis-label,
  .bar-chart_value-label,
  .bar-row_label,
  .bar-row_value {
    font-family: inherit;
    letter-spacing: 0;
    user-select: none;
  }

  .bar-chart_tick text {
    fill: #64748b;
    font-size: 0.76rem;
  }

  .bar-chart_axis-label {
    fill: #475569;
    font-size: 0.8rem;
    font-weight: 700;
    text-transform: uppercase;
  }

  .bar-chart_value-label {
    fill: #64748b;
    font-size: 0.76rem;
    font-weight: 800;
    text-transform: uppercase;
  }

  .bar-row {
    cursor: pointer;
  }

  .bar-row:focus {
    outline: none;
  }

  .bar-row_hit-area {
    fill: transparent;
  }

  .bar-row_track {
    fill: rgba(226, 232, 240, 0.7);
    stroke: rgba(148, 163, 184, 0.22);
    stroke-width: 1;
  }

  .bar-row_bar {
    transition: filter 160ms ease, stroke 160ms ease, stroke-width 160ms ease;
  }

  .bar-row_label {
    fill: #334155;
    font-size: 0.94rem;
    font-weight: 700;
  }

  .bar-row_value {
    fill: #273449;
    font-size: 0.9rem;
    font-weight: 800;
    paint-order: stroke fill;
    stroke: rgba(255, 255, 255, 0.94);
    stroke-linejoin: round;
    stroke-width: 4px;
  }

  .bar-row--active .bar-row_label,
  .bar-row:hover .bar-row_label,
  .bar-row:focus-visible .bar-row_label {
    fill: #0f172a;
  }

  .bar-row--active .bar-row_bar,
  .bar-row:hover .bar-row_bar,
  .bar-row:focus-visible .bar-row_bar {
    paint-order: stroke fill;
    stroke: rgba(255, 255, 255, 0.92);
    stroke-width: 2px;
    filter: drop-shadow(0 0.35rem 0.55rem rgba(15, 23, 42, 0.16));
  }

  .bar-row:focus-visible .bar-row_track {
    stroke: rgba(15, 23, 42, 0.38);
    stroke-width: 2;
  }

  .awardee-panel {
    width: 100%;
    margin: 0;
    border-top: 3px solid var(--tooltip-accent);
    padding: 1rem 0 0;
    height: 16rem;
    display: grid;
    align-content: start;
    grid-template-rows: auto auto minmax(0, 1fr);
  }

  .awardee-panel_eyebrow {
    margin: 0 0 0.6rem;
    font-size: 0.82rem;
    font-weight: 700;
    letter-spacing: 0;
    text-transform: uppercase;
    color: #64748b;
  }

  .awardee-panel_title {
    margin: 0 0 0.8rem;
    font-size: 1.15rem;
    line-height: 1.1;
    color: #0f172a;
  }

  .awardee-panel_content {
    display: grid;
    gap: 0.7rem;
    min-height: 0;
    overflow-y: auto;
    padding-right: 0.2rem;
  }

  .awardee-panel_entry {
    display: grid;
    gap: 0.28rem;
    padding-bottom: 0.7rem;
    border-bottom: 1px solid rgba(148, 163, 184, 0.26);
  }

  .awardee-panel_entry:last-child {
    padding-bottom: 0;
    border-bottom: 0;
  }

  .awardee-panel_year,
  .awardee-panel_description,
  .awardee-panel_placeholder {
    margin: 0;
  }

  .awardee-panel_year {
    font-size: 0.84rem;
    font-weight: 700;
    letter-spacing: 0;
    text-transform: uppercase;
    color: var(--tooltip-accent);
  }

  .awardee-panel_description,
  .awardee-panel_placeholder {
    font-size: 0.95rem;
    line-height: 1.45;
    color: #334155;
  }

  .awardee-panel_placeholder {
    max-width: 28ch;
  }

  .back-button {
    justify-self: start;
    display: inline-flex;
    align-items: center;
    gap: 0.65rem;
    border: 1px solid rgba(148, 163, 184, 0.55);
    background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
    color: rgb(59, 59, 59);
    border-radius: 999px;
    padding: 0.68rem 0.95rem;
    font: inherit;
    font-size: 0.92rem;
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

  .back-button_icon {
    font-size: 1.05em;
    line-height: 1;
  }

  .state-message {
    display: grid;
    place-items: center;
    min-height: 28rem;
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
    border-radius: 8px;
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

  .legend_title {
    margin: 0;
    font-size: 0.82rem;
    font-weight: 700;
    letter-spacing: 0;
    text-transform: uppercase;
    color: #475569;
  }

  .legend--side .legend_title {
    width: 100%;
  }

  .legend_bar {
    height: 16px;
    border-radius: 999px;
    border: 1px solid rgba(15, 23, 42, 0.12);
  }

  .legend_bar--vertical {
    width: 16px;
    height: clamp(12rem, 34vw, 18rem);
    justify-self: end;
  }

  .legend_side-label {
    display: grid;
    gap: 0.18rem;
    font-size: 0.9rem;
    color: #475569;
  }

  .legend_side-label--top {
    align-self: end;
  }

  .legend_side-label--bottom {
    align-self: start;
  }

  @media (min-width: 900px) {
    .panel_layout {
      grid-template-columns: minmax(13.5rem, 17.5rem) minmax(0, 1fr);
      gap: clamp(1.4rem, 3vw, 2.4rem);
    }

    .panel_layout--detail {
      align-items: start;
    }

    .panel_meta--detail {
      width: 100%;
      grid-template-rows: auto auto minmax(0, 1fr);
    }

    .awardee-panel {
      margin: 0;
    }

    .back-button {
      justify-self: start;
    }
  }

  @media (max-width: 899px) {
    .cloud-shell--detail {
      grid-template-columns: minmax(0, 1fr);
    }

    .legend--side {
      width: 100%;
      justify-items: start;
      text-align: left;
    }
  }
</style>
