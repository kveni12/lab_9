<script>
	import { PUBLIC_MAPBOX_ACCESS_TOKEN } from '$env/static/public';
	import { onMount } from 'svelte';
	import mapboxgl from 'mapbox-gl';
	import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';

	mapboxgl.accessToken = PUBLIC_MAPBOX_ACCESS_TOKEN;

	async function initializeMap() {
		const map = new mapboxgl.Map({
			container: 'map',
			style: 'mapbox://styles/mapbox/streets-v12',
			center: [-71.1097, 42.3736],
			zoom: 12,
			minZoom: 10,
			maxZoom: 18
		});

		await new Promise((resolve) => map.on('load', resolve));

		map.addSource('boston_route', {
			type: 'geojson',
			data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D'
		});

		map.addLayer({
			id: 'boston-bike-lanes',
			type: 'line',
			source: 'boston_route',
			paint: {
				'line-color': '#32a852',
				'line-width': 3,
				'line-opacity': 0.4
			}
		});

		map.addSource('cambridge_route', {
			type: 'geojson',
			data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson'
		});

		map.addLayer({
			id: 'cambridge-bike-lanes',
			type: 'line',
			source: 'cambridge_route',
			paint: {
				'line-color': '#1d4ed8',
				'line-width': 3,
				'line-opacity': 0.45
			}
		});

		return map;
	}

	onMount(() => {
		/** @type {import('mapbox-gl').Map | undefined} */
		let map;

		initializeMap().then((createdMap) => {
			map = createdMap;
		});

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

<div id="map"></div>

<style>
	@import url("$lib/global.css");

	#map {
		flex: 1;
		background: #d8f0ff;
		min-height: 24rem;
		border-radius: 1rem;
	}
</style>
