<script>
	import { PUBLIC_MAPBOX_ACCESS_TOKEN } from '$env/static/public';
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import mapboxgl from 'mapbox-gl';
	import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';

	mapboxgl.accessToken = PUBLIC_MAPBOX_ACCESS_TOKEN;

	/** @typedef {{ Number: string, NAME: string, Lat: string | number, Long: string | number, 'Seasonal Status': string, Municipality: string, 'Total Docks': string | number }} Station */

	/** @type {Station[]} */
	let stations = $state([]);
	/** @type {import('mapbox-gl').Map | undefined} */
	let map = $state();
	let mapViewChanged = $state(0);

	async function loadStations() {
		stations = await d3.csv('https://vis-society.github.io/labs/9/data/bluebikes-stations.csv');
		console.log('Bluebikes stations:', stations);
	}

	async function initializeMap() {
		const mapInstance = new mapboxgl.Map({
			container: 'map',
			style: 'mapbox://styles/mapbox/streets-v12',
			center: [-71.1097, 42.3736],
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

	/** @param {Station} station */
	function getCoords(station) {
		if (!map) return { cx: 0, cy: 0 };

		const point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		const { x, y } = map.project(point);
		return { cx: x, cy: y };
	}

	onMount(() => {
		loadStations();
		initializeMap();

		return () => map?.remove();
	});
</script>

<svelte:head>
	<title>Cycle-ytics: Watching the Wheels Turn</title>
	<meta
		name="description"
		content="Bikewatching explores bike traffic patterns and street-level movement in a simple Svelte app."
	/>
</svelte:head>

<h1>Cycle-ytics: Watching the Wheels Turn 🚴🏼‍♀️</h1>
<p>
	Cycle-ytics is a Svelte project for exploring bike traffic patterns, mapping rides, and telling
	stories about how people move through the city.
</p>

<div id="map">
	<svg>
		{#if map}
			{#key mapViewChanged}
				{#each stations as station}
					<circle {...getCoords(station)} r="5" fill="steelblue" stroke="white" stroke-width="1" />
				{/each}
			{/key}
		{/if}
	</svg>
</div>

<style>
	@import url("$lib/global.css");

	#map {
		flex: 1;
		background: #d8f0ff;
		min-height: 24rem;
		border-radius: 1rem;
		position: relative;
		overflow: hidden;
	}

	#map svg {
		position: absolute;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}
</style>
