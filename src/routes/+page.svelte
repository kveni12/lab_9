<script>
	import { env } from '$env/dynamic/public';
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import mapboxgl from 'mapbox-gl';
	import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';

	const STATION_DATA_URL = 'https://vis-society.github.io/labs/9/data/bluebikes-stations.csv';
	const TRIP_DATA_URL = 'https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv';
	const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
	const profile = 'cycling';
	const minutes = [5, 10, 15, 20];
	const contourColors = ['03045e', '0077b6', '00b4d8', '90e0ef'];
	const stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

	const mapboxToken = env.PUBLIC_MAPBOX_ACCESS_TOKEN ?? '';
	mapboxgl.accessToken = mapboxToken;

	/** @typedef {{ Number: string, NAME: string, Lat: string | number, Long: string | number, 'Seasonal Status': string, Municipality: string, 'Total Docks': string | number }} Station */
	/** @typedef {{ ride_id: string, bike_type: string, started_at: Date, ended_at: Date, start_station_id: string, end_station_id: string, is_member: string }} Trip */
	/** @typedef {Station & { arrivals: number, departures: number, totalTraffic: number }} StationWithTraffic */
	/** @typedef {{ features: Array<{ geometry: { coordinates: number[][][] }, properties: { contour: number, fillColor: string } }> }} Isochrone */
	/** @type {Trip[][]} */
	const departuresByMinute = Array.from({ length: 1440 }, () => []);
	/** @type {Trip[][]} */
	const arrivalsByMinute = Array.from({ length: 1440 }, () => []);

	/** @type {Station[]} */
	let stations = $state([]);
	/** @type {Trip[]} */
	let trips = $state([]);
	/** @type {Map<string, number>} */
	let arrivals = $state(new Map());
	/** @type {Map<string, number>} */
	let departures = $state(new Map());
	/** @type {import('mapbox-gl').Map | undefined} */
	let map = $state();
	let mapViewChanged = $state(0);
	let timeFilter = $state(-1);
	/** @type {StationWithTraffic | null} */
	let selectedStation = $state(null);
	/** @type {Isochrone | null} */
	let isochrone = $state(null);
	let isoRequestId = 0;

	let timeFilterLabel = $derived(
		new Date(0, 0, 0, 0, Math.max(timeFilter, 0)).toLocaleString('en', { timeStyle: 'short' })
	);

	let filteredDepartures = $derived.by(() => {
		if (timeFilter === -1) return departures;
		return d3.rollup(
			filterByMinute(departuresByMinute, timeFilter),
			/** @param {Trip[]} v */ (v) => v.length,
			/** @param {Trip} d */ (d) => d.start_station_id
		);
	});

	let filteredArrivals = $derived.by(() => {
		if (timeFilter === -1) return arrivals;
		return d3.rollup(
			filterByMinute(arrivalsByMinute, timeFilter),
			/** @param {Trip[]} v */ (v) => v.length,
			/** @param {Trip} d */ (d) => d.end_station_id
		);
	});

	/** @type {StationWithTraffic[]} */
	let filteredStations = $derived.by(() =>
		stations.map((station) => {
			const id = station.Number;
			const arr = filteredArrivals.get(id) ?? 0;
			const dep = filteredDepartures.get(id) ?? 0;

			return {
				...station,
				arrivals: arr,
				departures: dep,
				totalTraffic: arr + dep
			};
		})
	);

	let radiusScale = $derived.by(() =>
		d3
			.scaleSqrt()
			.domain([0, d3.max(filteredStations, /** @param {StationWithTraffic} d */ (d) => d.totalTraffic) || 0])
			.range(timeFilter === -1 ? [0, 25] : [3, 30])
	);

	async function loadStations() {
		stations = await d3.csv(STATION_DATA_URL);
		console.log('Bluebikes stations:', stations);
	}

	async function loadTrips() {
		for (const bucket of departuresByMinute) bucket.length = 0;
		for (const bucket of arrivalsByMinute) bucket.length = 0;

		trips = await d3.csv(TRIP_DATA_URL).then(
			/** @param {any[]} rows */ (rows) => {
			for (const trip of rows) {
				trip.started_at = new Date(trip.started_at);
				trip.ended_at = new Date(trip.ended_at);

				const startedMinutes = minutesSinceMidnight(trip.started_at);
				const endedMinutes = minutesSinceMidnight(trip.ended_at);

				departuresByMinute[startedMinutes].push(trip);
				arrivalsByMinute[endedMinutes].push(trip);
			}

			return /** @type {Trip[]} */ (rows);
			}
		);

		departures = d3.rollup(
			trips,
			/** @param {Trip[]} v */ (v) => v.length,
			/** @param {Trip} d */ (d) => d.start_station_id
		);
		arrivals = d3.rollup(
			trips,
			/** @param {Trip[]} v */ (v) => v.length,
			/** @param {Trip} d */ (d) => d.end_station_id
		);
	}

	async function initializeMap() {
		if (!mapboxToken) return;

		const mapInstance = new mapboxgl.Map({
			container: 'map',
			style: 'mapbox://styles/mapbox/streets-v12',
			center: [-71.09415, 42.36027],
			zoom: 12,
			minZoom: 10,
			maxZoom: 18
		});
		map = mapInstance;

		await new Promise((resolve) => mapInstance.on('load', resolve));

		mapInstance.addSource('boston_route', {
			type: 'geojson',
			data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D'
		});

		mapInstance.addLayer({
			id: 'boston-bike-lanes',
			type: 'line',
			source: 'boston_route',
			paint: {
				'line-color': '#32a852',
				'line-width': 3,
				'line-opacity': 0.4
			}
		});

		mapInstance.addSource('cambridge_route', {
			type: 'geojson',
			data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson'
		});

		mapInstance.addLayer({
			id: 'cambridge-bike-lanes',
			type: 'line',
			source: 'cambridge_route',
			paint: {
				'line-color': '#1d4ed8',
				'line-width': 3,
				'line-opacity': 0.45
			}
		});

		mapInstance.on('move', () => {
			mapViewChanged += 1;
		});
	}

	/** @param {Date} date */
	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	/** @param {Trip[][]} tripsByMinute @param {number} minute */
	function filterByMinute(tripsByMinute, minute) {
		const minMinute = (minute - 60 + 1440) % 1440;
		const maxMinute = (minute + 60) % 1440;

		if (minMinute > maxMinute) {
			const beforeMidnight = tripsByMinute.slice(minMinute);
			const afterMidnight = tripsByMinute.slice(0, maxMinute);
			return beforeMidnight.concat(afterMidnight).flat();
		}

		return tripsByMinute.slice(minMinute, maxMinute).flat();
	}

	/** @param {StationWithTraffic} station */
	function getCoords(station) {
		if (!map) return { cx: 0, cy: 0 };

		const point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		const { x, y } = map.project(point);
		return { cx: x, cy: y };
	}

	/** @param {StationWithTraffic} station */
	function getDepartureRatio(station) {
		if (!station.totalTraffic) return stationFlow(0.5);
		return stationFlow(station.departures / station.totalTraffic);
	}

	/** @param {Isochrone['features'][number]} feature */
	function geoJSONPolygonToPath(feature) {
		if (!map) return '';

		const path = d3.path();
		const rings = feature.geometry.coordinates;

		for (const ring of rings) {
			for (let i = 0; i < ring.length; i += 1) {
				const [lng, lat] = ring[i];
				const { x, y } = map.project([lng, lat]);

				if (i === 0) path.moveTo(x, y);
				else path.lineTo(x, y);
			}
			path.closePath();
		}

		return path.toString();
	}

	/** @param {number} lon @param {number} lat */
	async function getIso(lon, lat) {
		const requestId = ++isoRequestId;
		const base = `${urlBase}${profile}/${lon},${lat}`;
		const params = new URLSearchParams({
			contours_minutes: minutes.join(','),
			contours_colors: contourColors.join(','),
			polygons: 'true',
			access_token: mapboxToken
		});
		const url = `${base}?${params.toString()}`;

		const query = await fetch(url, { method: 'GET' });
		const data = await query.json();

		if (requestId === isoRequestId) {
			isochrone = data;
			mapViewChanged += 1;
		}
	}

	$effect(() => {
		if (selectedStation) {
			getIso(+selectedStation.Long, +selectedStation.Lat);
		} else {
			isoRequestId += 1;
			isochrone = null;
		}
	});

	onMount(() => {
		loadStations();
		loadTrips();
		initializeMap();

		return () => {
			isoRequestId += 1;
			map?.remove();
		};
	});
</script>

<svelte:head>
	<title>Cycle-ytics: Watching the Wheels Turn</title>
	<meta
		name="description"
		content="Cycle-ytics explores Bluebikes demand, bike network coverage, and cycling access across Boston and Cambridge."
	/>
</svelte:head>

<header>
	<h1>Cycle-ytics: Watching the Wheels Turn 🚴🏼‍♀️</h1>

	<label>
		Filter by time:
		<input type="range" min="-1" max="1440" bind:value={timeFilter} />
		{#if timeFilter !== -1}
			<time>{timeFilterLabel}</time>
		{:else}
			<em>(any time)</em>
		{/if}
	</label>
</header>

<p>
	Cycle-ytics is a Svelte project for exploring bike traffic patterns, Bluebikes demand, and
	biking access across Boston and Cambridge.
</p>

<div id="map">
	<svg>
		{#if map}
			{#key mapViewChanged}
				{#if isochrone}
					{#each isochrone.features as feature}
						<path
							d={geoJSONPolygonToPath(feature)}
							fill={feature.properties.fillColor}
							fill-opacity="0.2"
							stroke="#0f172a"
							stroke-opacity="0.45"
							stroke-width="1"
						>
							<title>{feature.properties.contour} minutes of biking</title>
						</path>
					{/each}
				{/if}

				{#each filteredStations as station}
						<circle
							{...getCoords(station)}
							r={radiusScale(station.totalTraffic)}
							class={station.Number === selectedStation?.Number ? 'selected' : ''}
							style={`--departure-ratio: ${getDepartureRatio(station)}`}
							aria-label={`${station.NAME}: ${station.totalTraffic} trips (${station.departures} departures, ${station.arrivals} arrivals)`}
							onmousedown={() =>
								(selectedStation =
									selectedStation?.Number !== station.Number ? station : null)}
						>
							<title>{`${station.NAME}: ${station.totalTraffic} trips (${station.departures} departures, ${station.arrivals} arrivals)`}</title>
						</circle>
				{/each}
			{/key}
		{/if}
	</svg>
</div>

<div class="legend">
	<span>Flow:</span>
	<div style="--departure-ratio: 1">More departures</div>
	<div style="--departure-ratio: 0.5">Balanced</div>
	<div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
	@import url("$lib/global.css");

	header {
		display: flex;
		gap: 1em;
		align-items: baseline;
		flex-wrap: wrap;
	}

	label {
		margin-left: auto;
		font-size: 0.95rem;
	}

	input {
		width: min(24rem, 100%);
		display: block;
		margin-top: 0.3rem;
	}

	time,
	em {
		display: block;
		margin-top: 0.25rem;
	}

	em {
		color: #64748b;
	}

	#map {
		flex: 1;
		background: #d8f0ff;
		min-height: 32rem;
		border-radius: 1rem;
		position: relative;
		overflow: hidden;
		border: 1px solid #cbd5e1;
	}

	#map svg {
		position: absolute;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	#map circle,
	.legend > div {
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);
	}

	#map circle {
		fill: var(--color);
		fill-opacity: 0.6;
		stroke: white;
		stroke-width: 1.2;
		pointer-events: auto;
		cursor: pointer;
		transition: opacity 0.2s ease, stroke-width 0.2s ease;
	}

	#map circle.selected {
		stroke: #0f172a;
		stroke-width: 2;
	}

	#map svg:has(circle.selected) circle:not(.selected) {
		opacity: 0.3;
	}

	#map path {
		pointer-events: auto;
	}

	.legend {
		display: flex;
		align-items: center;
		gap: 0.35rem;
		margin-block: 0.9rem 0;
		font-size: 0.9rem;
	}

	.legend > span {
		font-weight: 600;
		margin-right: 0.25rem;
	}

	.legend > div {
		flex: 1;
		background: var(--color);
		color: white;
		padding: 0.55rem 0.9rem;
	}

	.legend > div:nth-child(2) {
		text-align: left;
	}

	.legend > div:nth-child(3) {
		text-align: center;
	}

	.legend > div:nth-child(4) {
		text-align: right;
	}
</style>
