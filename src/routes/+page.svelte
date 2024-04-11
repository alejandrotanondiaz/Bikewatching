<style>
    @import url("$lib/global.css");
</style>
<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";
    mapboxgl.accessToken = "pk.eyJ1IjoidGFub25hbGVqYW5kcm8iLCJhIjoiY2x1dWJkbHZyMDhmYzJpbXV4NGZmeDB4OCJ9.afyxkaTggCebErFKmYd2mA";

    
    function getCoords (station) {
            let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
            let {x, y} = map.project(point);
            return {cx: x, cy: y};
    }
    let stations = [];
    let trips = [];
    let map;
    let arrivals;
    let departures;
    onMount(async () => { 
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09415, 42.36027],
            style: "mapbox://styles/mapbox/light-v11",
            zoom: 12
        });
        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
        type: "geojson",
        data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        })
        map.addLayer({
        id: "boston lanes", // A name for our layer (up to you)
        type: "line", // one of the supported layer types, e.g. line, circle, etc.
        source: "boston_route", // The id we specified in `addSource()`
        paint: {
            "line-color": "#00ced1",
            "line-width": 2,
            "line-opacity": 0.4,
        },
        });
        map.addSource("cambridge_route", {
        type: "geojson",
        data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
        id: "cambridge lanes", // A name for our layer (up to you)
        type: "line", // one of the supported layer types, e.g. line, circle, etc.
        source: "cambridge_route", // The id we specified in `addSource()`
        paint: {
            "line-color": "#00ced1",
            "line-width": 2,
            "line-opacity": 0.4,
        },
        });
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");    
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
            }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    });
    let mapViewChanged = 0;
    $: map?.on("move", evt => mapViewChanged++);
    $: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic)])
	.range([0, 25]);
    let timeFilter = -1;
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});
                     
    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }
    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });

    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
    $: filteredStations = stations.map(station => {
        let id = station.Number;
        station = {...station};
        station.arrivals = filteredArrivals.get(id) ?? 0;
        station.departures = filteredDepartures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station;
    });

    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);


</script>


<title>
    Bikewatching
</title>
<header>
    <h1>🚴🏼‍♀️ Bikewatching</h1>
    <label>Filter by time:
        <input type=range min="-1" max="1440" bind:value={timeFilter}/>
        {#if timeFilter === -1}
            <em>(any time)</em>
        {:else}
            <time>{ timeFilterLabel }</time>
        {/if}
    </label>

        
</header>



<p>Visualization allowing us to see the state of BlueBike stations accross Boston</p>
<div id="map">
    <svg>
        {#key mapViewChanged}
            {#each filteredStations as station}
                <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
                <circle style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }" { ...getCoords(station) } r={ radiusScale(station.totalTraffic) } fill="steelblue" />
            {/each}
        {/key}
    </svg>
</div>
<div class="legend">
	<div style="--departure-ratio: 1">More departures</div>
	<div style="--departure-ratio: 0.5">Balanced</div>
	<div style="--departure-ratio: 0">More arrivals</div>
</div>
