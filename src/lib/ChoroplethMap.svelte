<script>
	import mapboxgl from "mapbox-gl";
	import "mapbox-gl/dist/mapbox-gl.css";
	import { onMount, onDestroy } from "svelte";
	import { base } from "$app/paths";

	// ── Props ──────────────────────────────────────────────────────────────────
	export let styleUrl;         // Mapbox Studio style URL
	export let valueProperty;    // GeoJSON property key holding the numeric value
	export let townNameProperty; // GeoJSON property key holding the town name
	export let criteriaName;     // shown in the info panel header and legend title
	export let valueLabel;       // tooltip label, e.g. '% Multifamily'
	export let unit = "%";       // appended to the hover value
	export let displayMultiplier = null;
	export let legendBins = [];  // [{ label: "0 – 5%", color: "#f7fbff" }, ...]
	export let colorStops = [];  // [{ min: 0, color: "#f7fbff" }, ...] using raw data values
	export let noDataColor = "#d9d9d9";
	export let visualEncoding = "color";
	export let dataUrl = "";
	export let circleColor = "#80276C";
	export let radiusStops = [];

	mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;
	const numberFormat = new Intl.NumberFormat("en-US", { maximumFractionDigits: 1 });

	// ── State ──────────────────────────────────────────────────────────────────
	let mapContainer;
	let map;
	let loading = true;
	let error   = null;
	let hoveredInfo = null;
	let hoverLayerIds = [];

	$: usesRadiusEncoding = visualEncoding === "radius";
	$: hoverPrompt = usesRadiusEncoding ? "Hover over a station!" : "Hover over a town!";

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

		try {
			if (usesRadiusEncoding) {
				await addRadiusLayer();
			} else {
				addColorLayerOverrides();
			}
		} catch (err) {
			error = err instanceof Error ? err.message : "Map layer failed to load.";
			loading = false;
			return;
		}

		if (hoverLayerIds.length === 0) {
			loading = false;
			return;
		}

		loading = false;

		map.on("mousemove", event => {
			const features = map.queryRenderedFeatures(event.point, {
				layers: hoverLayerIds,
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

		map.on("mouseleave", () => {
			hoveredInfo = null;
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

		return applyStyleOverrides(style);
	}

	function applyStyleOverrides(style) {
		if (usesRadiusEncoding) {
			return style;
		}

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

	function addColorLayerOverrides() {
		const styleLayers = map.getStyle().layers ?? [];
		const fillLayers = getTargetFillLayers(styleLayers);
		const fillColorExpression = getFillColorExpression();

		if (fillLayers.length === 0) {
			return;
		}

		if (fillColorExpression) {
			for (const layer of fillLayers) {
				map.setPaintProperty(layer.id, "fill-color-transition", { duration: 0, delay: 0 });
				map.setPaintProperty(layer.id, "fill-color", fillColorExpression);
			}
		}

		hoverLayerIds = fillLayers.map((layer) => layer.id);
	}

	async function addRadiusLayer() {
		if (!dataUrl) {
			return;
		}

		const response = await fetch(resolveAssetUrl(dataUrl));

		if (!response.ok) {
			throw new Error(`Map data request failed (${response.status}).`);
		}

		hidePublishedThematicLayers();

		const sourceId = "tod-radius-source";
		const layerId = "tod-radius-layer";
		const outlineLayerId = "tod-radius-outline-layer";
		const sourceData = polygonFeaturesToCentroidPoints(await response.json());

		map.addSource(sourceId, {
			type: "geojson",
			data: sourceData
		});

		map.addLayer({
			id: layerId,
			type: "circle",
			source: sourceId,
			paint: {
				"circle-radius": getCircleRadiusExpression(),
				"circle-color": getCircleColorExpression(),
				"circle-opacity": 0.64,
				"circle-stroke-color": getCircleColorExpression(),
				"circle-stroke-width": 1.5,
				"circle-stroke-opacity": 0.85
			}
		});

		map.addLayer({
			id: outlineLayerId,
			type: "circle",
			source: sourceId,
			paint: {
				"circle-radius": getCircleRadiusExpression(1.12),
				"circle-color": "rgba(0, 0, 0, 0)",
				"circle-stroke-color": "#ffffff",
				"circle-stroke-width": 1,
				"circle-stroke-opacity": 0.55
			}
		});

		hoverLayerIds = [outlineLayerId, layerId];
	}

	function hidePublishedThematicLayers() {
		for (const layer of getTargetThematicLayers(map.getStyle().layers ?? [])) {
			if (map.getLayer(layer.id)) {
				map.setLayoutProperty(layer.id, "visibility", "none");
			}
		}
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

	function getTargetThematicLayers(layers = []) {
		return layers.filter(
			l => ["fill", "circle"].includes(l.type) &&
			/dens|mix|multi|rent|disab|park|parking|heat|choropleth|station|rail/i.test(
				l.id + " " + (l["source-layer"] ?? "")
			)
		);
	}

	function formatDisplayValue(rawValue) {
		const numericValue = Number(rawValue);

		if (!Number.isFinite(numericValue)) {
			return null;
		}

		const value = displayMultiplier == null
			? unit === "%" ? numericValue * 100 : numericValue
			: numericValue * displayMultiplier;

		return unit === "%"
			? value.toFixed(1)
			: numberFormat.format(value);
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

	function getCircleRadiusExpression(multiplier = 1) {
		const sortedStops = radiusStops
			.filter(({ value, radius }) => Number.isFinite(value) && Number.isFinite(radius))
			.sort((a, b) => a.value - b.value);

		if (sortedStops.length === 0) {
			return 6 * multiplier;
		}

		return [
			"interpolate",
			["linear"],
			["to-number", ["get", valueProperty], 0],
			...sortedStops.flatMap(({ value, radius }) => [value, radius * multiplier])
		];
	}

	function getCircleColorExpression() {
		return [
			"case",
			["==", ["to-number", ["get", valueProperty], 0], 0],
			"#000000",
			circleColor
		];
	}

	function polygonFeaturesToCentroidPoints(geojson) {
		return {
			type: "FeatureCollection",
			features: (geojson.features ?? []).map((feature) => ({
				type: "Feature",
				properties: feature.properties ?? {},
				geometry: {
					type: "Point",
					coordinates: getFeatureCentroid(feature)
				}
			}))
		};
	}

	function getFeatureCentroid(feature) {
		const geometry = feature.geometry ?? {};
		const ring = geometry.type === "Polygon"
			? geometry.coordinates?.[0]
			: geometry.type === "MultiPolygon"
				? geometry.coordinates?.[0]?.[0]
				: null;

		if (!Array.isArray(ring) || ring.length === 0) {
			return [0, 0];
		}

		const usableCoordinates = ring.filter(
			(coord) => Number.isFinite(coord?.[0]) && Number.isFinite(coord?.[1])
		);

		if (usableCoordinates.length === 0) {
			return [0, 0];
		}

		const [longitudeSum, latitudeSum] = usableCoordinates.reduce(
			([longitudeTotal, latitudeTotal], [longitude, latitude]) => [
				longitudeTotal + longitude,
				latitudeTotal + latitude
			],
			[0, 0]
		);

		return [
			longitudeSum / usableCoordinates.length,
			latitudeSum / usableCoordinates.length
		];
	}

	function resolveAssetUrl(path = "") {
		if (!path || /^(?:[a-z]+:)?\/\//i.test(path)) {
			return path;
		}

		if (path.startsWith(base)) {
			return path;
		}

		return path.startsWith("/") ? `${base}${path}` : `${base}/${path}`;
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
		<p style="text-align:center;">{hoverPrompt}</p>
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
			<span
				class="legend-key"
				class:legend-key--radius={usesRadiusEncoding}
				style:background-color={bin.color ?? circleColor}
				style:width={usesRadiusEncoding ? `${bin.radius ?? 10}px` : null}
				style:height={usesRadiusEncoding ? `${bin.radius ?? 10}px` : null}
			></span>
			<span>{bin.label}</span>
		</div>
	{/each}

	{#if !usesRadiusEncoding}
		<div class="legend-row">
			<span class="legend-key" style="background-color: {noDataColor};"></span>
			<span>No data</span>
		</div>
	{/if}
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

	.legend-key--radius {
		border-radius: 50%;
		opacity: 0.64;
		border: 1px solid rgba(128, 39, 108, 0.8);
	}
</style>
