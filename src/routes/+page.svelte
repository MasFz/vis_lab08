<script>
    import mapboxgl from "mapbox-gl";
    import "mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";
  
    // Mapbox access token
    mapboxgl.accessToken = "pk.eyJ1IjoibWFzZnoiLCJhIjoiY21hcGhqa3d2MGd3bzJtbzhldWwzajM1ZiJ9.VdYcnzrcjeoygwFx55GpXA";
  
    // — Variáveis reativas —
    let map;
    let stations = [];
    let trips = [];
    let arrivals, departures;
    let mapViewChanged = 0;
    let timeFilter = -1;
    let timeFilterLabel = "(any time)";
    let radiusScale;
  
    // — Função de inicialização do mapa e camadas de bike lanes —
    async function initMap() {
      map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/streets-v12',
        center: [-71.09415, 42.36027],
        zoom: 12
      });
      await new Promise(resolve => map.on("load", resolve));
  
      // bike lanes Boston
      map.addSource("boston_route", {
        type: "geojson",
        data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
      });
      map.addLayer({
        id: "boston_route_layer",
        type: "line",
        source: "boston_route",
        paint: {
          'line-color': 'green',
          'line-width': 3,
          'line-opacity': 0.4
        }
      });
  
      // bike lanes Cambridge (exemplo de fonte extra)
      map.addSource("cambridge_route", {
        type: "geojson",
        data: "https://raw.githubusercontent.com/CityOfCambridgeData/geo/master/bike_network.geojson"
      });
      map.addLayer({
        id: "cambridge_route_layer",
        type: "line",
        source: "cambridge_route",
        paint: {
          'line-color': 'purple',
          'line-width': 2,
          'line-opacity': 0.6
        }
      });
  
      // atualiza posição dos marcadores a cada movimento
      map.on("move", () => mapViewChanged++);
    }
  
    // — Carrega dados das estações Bluebikes —
    async function loadStationData() {
      try {
        const data = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-stations.csv');
        stations = data.map(d => ({
          id: d.Number,
          name: d.NAME,
          Lat: +d.Lat,
          Long: +d.Long
        }));
      } catch (err) {
        console.error('Error loading stations:', err);
      }
    }
  
    // — Carrega dados de tráfego (trips) e calcula chegadas/saídas —
    async function loadStationDemand() {
      try {
        const data = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv');
        trips = data.map(trip => ({
          ride_id: trip.ride_id,
          started_at: new Date(trip.started_at),
          ended_at: new Date(trip.ended_at),
          start_station_id: trip.start_station_id,
          end_station_id: trip.end_station_id
        }));
  
        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals   = d3.rollup(trips, v => v.length, d => d.end_station_id);
  
        stations = stations.map(s => {
          const arr = arrivals.get(s.id) ?? 0;
          const dep = departures.get(s.id) ?? 0;
          return { ...s, arrivals: arr, departures: dep, totalTraffic: arr + dep };
        });
  
      } catch (err) {
        console.error('Error loading demand:', err);
      }
    }
  
    // — Função de projeção de coordenadas para pixels no mapa —
    function getCoords(station) {
      const point = new mapboxgl.LngLat(station.Long, station.Lat);
      const { x, y } = map.project(point);
      return { cx: x, cy: y };
    }
  
    // — Escala reativa para raio de círculo —
    $: radiusScale = d3.scaleSqrt()
      .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
      .range([0, 25]);
  
    // — Reatividade para filtro de horário —
    $: timeFilterLabel = timeFilter === -1
      ? "(any time)"
      : new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", { timeStyle: "short" });
  
    // — Função para minutos desde meia-noite —
    function minutesSinceMidnight(date) {
      return date.getHours() * 60 + date.getMinutes();
    }
  
    // — Dados filtrados pelas últimas +/-60 minutos —
    $: filteredTrips = timeFilter === -1
      ? trips
      : trips.filter(trip => {
        const sm = minutesSinceMidnight(trip.started_at);
        const em = minutesSinceMidnight(trip.ended_at);
        return Math.abs(sm - timeFilter) <= 60 || Math.abs(em - timeFilter) <= 60;
      });
  
    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    $: filteredArrivals   = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
  
    $: filteredStations = stations.map(s => {
      const arr = filteredArrivals.get(s.id) ?? 0;
      const dep = filteredDepartures.get(s.id) ?? 0;
      return { ...s, arrivals: arr, departures: dep, totalTraffic: arr + dep };
    });
  
    onMount(async () => {
      await initMap();
      await loadStationData();
      await loadStationDemand();
    });
  </script>
  
  <header>
    <h1>BikeWatch</h1>
    <label>
      Filter by time:
      <input type="range" min="-1" max="1440" bind:value={timeFilter} />
      {#if timeFilter !== -1}
        <time style="display: block">{timeFilterLabel}</time>
      {:else}
        <em style="display: block">(any time)</em>
      {/if}
    </label>
  </header>
  
  <div id="map">
    <svg>
      {#key mapViewChanged}
        {#each (timeFilter === -1 ? stations : filteredStations) as station (station.id)}
          <circle
            {...getCoords(station)}
            r={radiusScale(station.totalTraffic)}
            fill="steelblue"
            fill-opacity="0.6"
            stroke="white"
          />
        {/each}
      {/key}
    </svg>
  </div>
  
  <style>
    @import url("$lib/global.css");
  
    #map {
      position: relative;
      width: 100%;
      height: 600px;
    }
    #map svg {
      position: absolute;
      top: 0; left: 0;
      width: 100%; height: 100%;
      pointer-events: none;
    }
    svg circle {
      transition: r 0.2s;
    }
  </style>
  