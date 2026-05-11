<script>
	import mapboxgl from "mapbox-gl";
	import "mapbox-gl/dist/mapbox-gl.css";
	import { onMount, onDestroy } from "svelte";

	// ── Props ──────────────────────────────────────────────────────────────────
	export let styleUrl;         // Mapbox Studio style URL
	export let valueProperty;    // GeoJSON property key holding the numeric value
	export let townNameProperty; // GeoJSON property key holding the town name
	export let criteriaName;     // shown in the info panel header and legend title
	export let valueLabel;       // tooltip label, e.g. '% Multifamily'
	export let unit = "%";       // appended to the hover value
	export let legendBins = [];  // [{ label: "0 – 5%", color: "#f7fbff" }, ...]
	export let colorStops = [];  // [{ min: 0, color: "#f7fbff" }, ...] using raw data values
	export let noDataColor = "#d9d9d9";

	mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;
	const numberFormat = new Intl.NumberFormat("en-US", { maximumFractionDigits: 1 });

	// ── State ──────────────────────────────────────────────────────────────────
	let mapContainer;
	let map;
	let loading = true;
	let error   = null;
	let hoveredInfo = null;

	// ── Map initialization ────────────────────────────────────────────────────
	async function initMap() {
		if (!mapboxgl.accessToken) {
			error = "Mapbox token missing. Add VITE_MAPBOX_TOKEN to your .env file.";
			loading = false;
			return;
		}

		try {
			const preparedStyle = await getPreparedStyle();

			map = new mapboxgl.Map({
				container: mapContainer,
				style: preparedStyle,
				center: [-71.2, 42.3],
				zoom: 8,
				minZoom: 7,
				maxZoom: 11,
			});
		} catch (err) {
			error = err instanceof Error ? err.message : "Map style failed to load.";
			loading = false;
			return;
		}

		map.addControl(new mapboxgl.NavigationControl(), "top-right");

		map.on("error", (e) => {
			if (!error) {
				error = e.error?.message ?? "Map failed to load.";
				loading = false;
			}
		});

		await new Promise(resolve => map.on("load", resolve));

		map.getCanvas().style.cursor = "default";

		// Find the first fill layer in the published style for hover queries.
		const styleLayers = map.getStyle().layers ?? [];
		const fillLayer = getTargetFillLayers(styleLayers)[0];

		if (!fillLayer) {
			loading = false;
			return;
		}

		const fillColorExpression = getFillColorExpression();

		if (fillColorExpression) {
			map.setPaintProperty(fillLayer.id, "fill-color-transition", { duration: 0, delay: 0 });
			map.setPaintProperty(fillLayer.id, "fill-color", fillColorExpression);
		}

		loading = false;

		map.on("mousemove", event => {
			const features = map.queryRenderedFeatures(event.point, {
				layers: [fillLayer.id],
			});

			if (features.length > 0) {
				const props = features[0].properties;
				const townName = props[townNameProperty] ?? "Unknown";
				const rawValue = props[valueProperty];
				const displayValue = formatDisplayValue(rawValue);
				hoveredInfo = { townName, displayValue };
			} else {
				hoveredInfo = null;
			}
		});
	}

	async function getPreparedStyle() {
		if (!styleUrl.startsWith("mapbox://styles/")) {
			return styleUrl;
		}

		const stylePath = styleUrl.replace("mapbox://styles/", "");
		const [username, styleId] = stylePath.split("/");

		if (!username || !styleId) {
			return styleUrl;
		}

		const response = await fetch(
			`https://api.mapbox.com/styles/v1/${username}/${styleId}?access_token=${mapboxgl.accessToken}`
		);

		if (!response.ok) {
			throw new Error(`Mapbox style request failed (${response.status}).`);
		}

		const style = await response.json();

		return applyFillColorOverride(style);
	}

	function applyFillColorOverride(style) {
		const fillColorExpression = getFillColorExpression();

		if (!fillColorExpression) {
			return style;
		}

		const fillLayers = getTargetFillLayers(style.layers ?? []);

		for (const layer of fillLayers) {
			layer.paint = {
				...(layer.paint ?? {}),
				"fill-color": fillColorExpression,
				"fill-color-transition": { duration: 0, delay: 0 }
			};
		}

		return style;
	}

	function getTargetFillLayers(layers = []) {
		const matchingLayers = layers.filter(
			l => l.type === "fill" &&
			/dens|mix|multi|rent|disab|park|heat|choropleth/i.test(
				l.id + " " + (l["source-layer"] ?? "")
			)
		);

		return matchingLayers.length > 0
			? matchingLayers
			: layers.filter(l => l.type === "fill").slice(0, 1);
	}

	function formatDisplayValue(rawValue) {
		const numericValue = Number(rawValue);

		if (!Number.isFinite(numericValue)) {
			return null;
		}

		return unit === "%"
			? (numericValue * 100).toFixed(1)
			: numberFormat.format(numericValue);
	}

	function getFillColorExpression() {
		if (!valueProperty || colorStops.length === 0) {
			return null;
		}

		const sortedStops = colorStops
			.filter(({ min, color }) => Number.isFinite(min) && color)
			.sort((a, b) => a.min - b.min);

		if (sortedStops.length === 0) {
			return null;
		}

		const [firstStop, ...remainingStops] = sortedStops;
		const stepExpression = [
			"step",
			["to-number", ["get", valueProperty]],
			firstStop.color
		];

		for (const stop of remainingStops) {
			stepExpression.push(stop.min, stop.color);
		}

		return [
			"case",
			["has", valueProperty],
			stepExpression,
			noDataColor
		];
	}

	onMount(() => { initMap(); });
	onDestroy(() => { map?.remove(); });
</script>

<!-- ── Info panel ─────────────────────────────────────────────────────────── -->
<div class="info-panel">
	<!-- <h3>{criteriaName}</h3> -->
	{#if hoveredInfo}
		<p><strong>{hoveredInfo.townName}</strong></p>
		<p>
			{valueLabel}:
			{#if hoveredInfo.displayValue != null}
				<strong>{hoveredInfo.displayValue}{unit}</strong>
			{:else}
				<em>No data</em>
			{/if}
		</p>
	{:else}
		<p style="text-align:center;">Hover over a town!</p>
	{/if}
</div>

<!-- ── Map container ─────────────────────────────────────────────────────── -->
{#if loading && !error}
	<div class="status">Loading map…</div>
{:else if error}
	<div class="status error">Could not load map: {error}</div>
{/if}

<div bind:this={mapContainer} class="map-container" class:loading={loading && !error} class:hidden={!!error} />

<!-- ── Legend ────────────────────────────────────────────────────────────── -->
<div class="legend" class:hidden={loading || !!error}>
	<strong>{criteriaName}</strong>

	{#each legendBins as bin}
		<div class="legend-row">
			<span class="legend-key" style="background-color: {bin.color};"></span>
			<span>{bin.label}</span>
		</div>
	{/each}

	<div class="legend-row">
		<span class="legend-key" style="background-color: {noDataColor};"></span>
		<span>No data</span>
	</div>
</div>

<style>
	.map-container {
		min-height: 520px;
	}

	.map-container.loading {
		visibility: hidden;
	}

	.map-container.hidden { display: none; }

	.status {
		padding: 1rem;
		text-align: center;
		color: #555;
	}

	.status.error { color: #c0392b; }

	.info-panel {
		position: absolute;
		top: 12px;
		left: 12px;
		z-index: 1;
		background: rgba(255, 255, 255, 0.92);
		border-radius: 6px;
		padding: 10px 14px;
		font-size: 13px;
		/* box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1); */
		box-shadow: 0 0 10px #e6e600;
		min-width: 180px;
		pointer-events: none;
	}

	.info-panel h3 { margin: 0 0 6px 0; font-size: 14px; }
	.info-panel p  { margin: 2px 0; }

	.legend {
		position: absolute;
		bottom: 32px;
		right: 12px;
		z-index: 1;
		background: rgba(255, 255, 255, 0.92);
		border-radius: 6px;
		padding: 10px 14px;
		font-size: 12px;
		box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
		pointer-events: none;
		line-height: 18px;
	}

	.legend.hidden { display: none; }

	.legend strong {
		display: block;
		margin-bottom: 6px;
	}

	.legend-row {
		display: flex;
		align-items: center;
		gap: 6px;
		margin: 2px 0;
	}

	.legend-key {
		display: inline-block;
		width: 10px;
		height: 10px;
		border-radius: 20%;
		flex-shrink: 0;
	}
</style>
