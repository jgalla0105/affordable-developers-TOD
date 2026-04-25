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
	export let noDataColor = "#d9d9d9";

	mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;

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

		map = new mapboxgl.Map({
			container: mapContainer,
			style: styleUrl,
			center: [-71.2, 42.3],
			zoom: 8,
		});

		map.addControl(new mapboxgl.NavigationControl(), "top-right");

		map.on("error", (e) => {
			if (!error) {
				error = e.error?.message ?? "Map failed to load.";
				loading = false;
			}
		});

		await new Promise(resolve => map.on("load", resolve));

		loading = false;
		map.getCanvas().style.cursor = "default";

		// Find the first fill layer in the published style for hover queries.
		const styleLayers = map.getStyle().layers ?? [];
		const fillLayer =
			styleLayers.find(
				l => l.type === "fill" &&
				/dens|mix|multi|rent|disab|park|heat|choropleth/i.test(
					l.id + " " + (l["source-layer"] ?? "")
				)
			) ??
			styleLayers.find(l => l.type === "fill");

		if (!fillLayer) return;

		map.on("mousemove", event => {
			const features = map.queryRenderedFeatures(event.point, {
				layers: [fillLayer.id],
			});

			if (features.length > 0) {
				const props = features[0].properties;
				const townName = props[townNameProperty] ?? "Unknown";
				const rawValue = props[valueProperty];
				const displayValue = rawValue != null ? (rawValue * 100).toFixed(1) : null;
				hoveredInfo = { townName, displayValue };
			} else {
				hoveredInfo = null;
			}
		});
	}

	onMount(() => { initMap(); });
	onDestroy(() => { map?.remove(); });
</script>

<!-- ── Info panel ─────────────────────────────────────────────────────────── -->
<div class="info-panel">
	<h3>{criteriaName}</h3>
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
		<p>Hover over a town!</p>
	{/if}
</div>

<!-- ── Map container ─────────────────────────────────────────────────────── -->
{#if loading && !error}
	<div class="status">Loading map…</div>
{:else if error}
	<div class="status error">Could not load map: {error}</div>
{/if}

<div bind:this={mapContainer} class="map-container" class:hidden={!!error} />

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
		width: 100%;
		min-height: 520px;
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
		box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
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
