<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Taiwan OSINT Monitor</title>
  
  <!-- Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>
  
  <!-- MapLibre GL JS CSS & JS -->
  <link href="https://unpkg.com/maplibre-gl@3.6.2/dist/maplibre-gl.css" rel="stylesheet" />
  <script src="https://unpkg.com/maplibre-gl@3.6.2/dist/maplibre-gl.js"></script>

  <style>
    /* Dark tactical scrollbar */
    ::-webkit-scrollbar { width: 5px; }
    ::-webkit-scrollbar-track { background: #0f172a; }
    ::-webkit-scrollbar-thumb { background: #334155; border-radius: 3px; }
  </style>
</head>
<body class="bg-slate-950 text-slate-100 font-mono h-screen flex flex-col overflow-hidden">

  <!-- Header -->
  <header class="h-14 border-b border-slate-800 bg-slate-900/80 px-4 flex items-center justify-between z-10">
    <div class="flex items-center gap-3">
      <span class="relative flex h-3 w-3">
        <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-red-400 opacity-75"></span>
        <span class="relative inline-flex rounded-full h-3 w-3 bg-red-500"></span>
      </span>
      <h1 class="text-lg font-bold tracking-wider text-slate-100">TAIWAN OSINT // MONITOR</h1>
    </div>
    <div class="text-xs text-slate-400" id="clock">UTC 00:00:00</div>
  </header>

  <!-- Main Grid -->
  <main class="flex-1 grid grid-cols-1 md:grid-cols-4 gap-2 p-2 overflow-hidden">
    
    <!-- Left Sidebar: Intelligence Feeds -->
    <section class="md:col-span-1 bg-slate-900/60 border border-slate-800 rounded-lg p-3 flex flex-col h-full overflow-hidden">
      <h2 class="text-xs font-semibold text-slate-400 uppercase tracking-widest mb-3 border-b border-slate-800 pb-1">
        LIVE NEWS FEEDS
      </h2>
      <div id="news-feed" class="flex-1 overflow-y-auto space-y-3 text-sm pr-1">
        <p class="text-slate-500 text-xs italic">Loading intel streams...</p>
      </div>
    </section>

    <!-- Center: Tactical Map (Taiwan Focus) -->
    <section class="md:col-span-2 border border-slate-800 rounded-lg relative overflow-hidden bg-slate-900">
      <div id="map" class="w-full h-full"></div>
      <div class="absolute top-3 left-3 bg-slate-950/80 border border-slate-800 px-3 py-1.5 rounded text-xs">
        <span class="text-emerald-400">STATUS:</span> ACTIVE MONITORING
      </div>
    </section>

    <!-- Right Sidebar: Seismic & Status Widgets -->
    <section class="md:col-span-1 bg-slate-900/60 border border-slate-800 rounded-lg p-3 flex flex-col h-full overflow-hidden gap-3">
      <div class="flex-1 flex flex-col overflow-hidden">
        <h2 class="text-xs font-semibold text-slate-400 uppercase tracking-widest mb-3 border-b border-slate-800 pb-1">
          REGIONAL SEISMIC ACTIVITY
        </h2>
        <div id="quake-feed" class="flex-1 overflow-y-auto space-y-2 text-xs">
          <p class="text-slate-500 italic">Fetching seismic data...</p>
        </div>
      </div>
      
      <div class="h-32 border-t border-slate-800 pt-2">
        <h2 class="text-xs font-semibold text-slate-400 uppercase tracking-widest mb-2">QUICK CONTROLS</h2>
        <button onclick="recenterMap()" class="w-full py-1.5 bg-slate-800 hover:bg-slate-700 text-xs text-slate-200 rounded transition border border-slate-700">
          Reset Map View (TW)
        </button>
      </div>
    </section>

  </main>

  <script>
    // 1. Clock Updates
    function updateClock() {
      const now = new Date();
      document.getElementById('clock').innerText = 'UTC ' + now.toISOString().substring(11, 19);
    }
    setInterval(updateClock, 1000);
    updateClock();

    // 2. Map Initialization (MapLibre GL)
    const map = new maplibregl.Map({
      container: 'map',
      style: {
        'version': 8,
        'sources': {
          'carto-dark': {
            'type': 'raster',
            'tiles': ['https://a.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png'],
            'tileSize': 256,
            'attribution': '&copy; CARTO & OpenStreetMap'
          }
        },
        'layers': [{
          'id': 'carto-dark-layer',
          'type': 'raster',
          'source': 'carto-dark',
          'minzoom': 0,
          'maxzoom': 18
        }]
      },
      center: [120.9605, 23.6978], // Centered over Taiwan
      zoom: 6.8
    });

    function recenterMap() {
      map.flyTo({ center: [120.9605, 23.6978], zoom: 6.8 });
    }

    // Add Taiwan Strategic Markers (Example Data)
    const strategicPoints = [
      { name: "Taipei (Capital)", coords: [121.5654, 25.0330] },
      { name: "Kaohsiung Port", coords: [120.2821, 22.6047] },
      { name: "Taichung Port", coords: [120.5186, 24.2562] }
    ];

    map.on('load', () => {
      strategicPoints.forEach(pt => {
        new maplibregl.Marker({ color: '#ef4444' })
          .setLngLat(pt.coords)
          .setPopup(new maplibregl.Popup().setHTML(`<div class="text-slate-900 font-bold">${pt.name}</div>`))
          .addTo(map);
      });
    });

    // 3. Fetch RSS Intel Feed (via AllOrigins CORS Proxy)
    async function fetchIntelFeeds() {
      const feedContainer = document.getElementById('news-feed');
      // CNA News RSS as an example
      const rssUrl = encodeURIComponent('https://feeds.feedburner.com/rsscna/realtime'); 
      const proxyUrl = `https://api.allorigins.win/get?url=${rssUrl}`;

      try {
        const response = await fetch(proxyUrl);
        const data = await response.json();
        const parser = new DOMParser();
        const xml = parser.parseFromString(data.contents, "text/xml");
        const items = xml.querySelectorAll("item");

        feedContainer.innerHTML = '';
        items.forEach((item, index) => {
          if (index > 8) return;
          const title = item.querySelector("title")?.textContent || "Untitled";
          const link = item.querySelector("link")?.textContent || "#";

          const card = document.createElement('div');
          card.className = 'p-2 bg-slate-950/50 border border-slate-800 rounded hover:border-slate-600 transition';
          card.innerHTML = `<a href="${link}" target="_blank" class="text-slate-300 hover:text-emerald-400 text-xs line-clamp-2">${title}</a>`;
          feedContainer.appendChild(card);
        });
      } catch (err) {
        feedContainer.innerHTML = `<p class="text-red-400 text-xs">Failed to load RSS feeds.</p>`;
      }
    }

    // 4. Fetch USGS Earthquakes (M2.5+ worldwide, filter near Taiwan)
    async function fetchEarthquakes() {
      const quakeContainer = document.getElementById('quake-feed');
      const usgsUrl = 'https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/2.5_day.geojson';

      try {
        const res = await fetch(usgsUrl);
        const geojson = await res.json();
        
        // Filter events around Taiwan region roughly (118 to 124 E, 20 to 26 N)
        const twQuakes = geojson.features.filter(f => {
          const [lon, lat] = f.geometry.coordinates;
          return lon >= 118 && lon <= 124 && lat >= 20 && lat <= 26;
        });

        quakeContainer.innerHTML = '';
        if (twQuakes.length === 0) {
          quakeContainer.innerHTML = '<p class="text-slate-500">No recent seismic events recorded in Taiwan sector.</p>';
          return;
        }

        twQuakes.forEach(q => {
          const mag = q.properties.mag;
          const place = q.properties.place;
          const card = document.createElement('div');
          card.className = 'p-2 bg-slate-950/50 border border-slate-800 rounded flex items-center justify-between';
          card.innerHTML = `
            <span class="text-slate-300">${place}</span>
            <span class="px-1.5 py-0.5 rounded text-[10px] font-bold ${mag >= 4.5 ? 'bg-red-900 text-red-200' : 'bg-amber-900 text-amber-200'}">M${mag}</span>
          `;
          quakeContainer.appendChild(card);
        });
      } catch (e) {
        quakeContainer.innerHTML = '<p class="text-red-400">Failed to fetch earthquake data.</p>';
      }
    }

    fetchIntelFeeds();
    fetchEarthquakes();
  </script>
</body>
</html># OSINI
