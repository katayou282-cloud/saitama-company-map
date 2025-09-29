<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>埼玉 会社マップ | CSV→地図表示・最短ルート・メモ共有</title>
  <link rel="preconnect" href="https://unpkg.com">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet.markercluster@1.5.3/dist/MarkerCluster.css" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet.markercluster@1.5.3/dist/MarkerCluster.Default.css" />
  <style>
    body { margin:0; font-family: system-ui, sans-serif; }
    #map { height: 100vh; }
  </style>
</head>
<body>
  <input type="file" id="file" accept=".csv,text/csv" />
  <button id="btnDemo">サンプル読込</button>
  <div id="map"></div>

  <script src="https://unpkg.com/papaparse@5.4.1/papaparse.min.js"></script>
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="https://unpkg.com/leaflet.markercluster@1.5.3/dist/leaflet.markercluster.js"></script>
  <script>
    const map = L.map('map').setView([36.1473, 139.3889], 10);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 20,
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    const markers = L.markerClusterGroup();
    map.addLayer(markers);

    function plotRows(rows){
      markers.clearLayers();
      rows.forEach(r => {
        if(r._geo){
          const marker = L.marker([r._geo.lat, r._geo.lon])
            .bindPopup(`<b>${r.name}</b><br>${r.address}<br>No.${r.company_number}`);
          markers.addLayer(marker);
        }
      });
      if(rows.length){
        const pts = rows.filter(r=>r._geo).map(r=>[r._geo.lat, r._geo.lon]);
        if(pts.length) map.fitBounds(pts);
      }
    }

    async function geocodeOne(address){
      if(!address) return null;
      const url = `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(address)}&format=jsonv2&countrycodes=jp`;
      const res = await fetch(url, { headers: { 'Accept-Language':'ja' }});
      const js = await res.json();
      const best = js[0];
      if(best){
        return { lat:+best.lat, lon:+best.lon };
      }
      return null;
    }

    async function processRows(rows){
      for(const r of rows){
        r._geo = await geocodeOne(r.address);
        await new Promise(res=>setTimeout(res,800)); // API優しめに
      }
      plotRows(rows);
    }

    function loadCSVText(text){
      Papa.parse(text, { header: true, skipEmptyLines: true, complete: async (res) => {
        const rows = res.data.map((r, i) => ({
          id: i+1,
          name: (r.name || r['会社名'] || '').trim(),
          address: (
            r.address ||
            ((r['県名']||'') + (r['市町村']||'') + (r['番地']||''))
          ).trim(),
          company_number: (r.company_number || r['会社番号'] || '').trim()
        }));
        await processRows(rows);
      }});
    }

    document.getElementById('file').addEventListener('change', async (e) => {
      const file = e.target.files[0];
      if(!file) return;
      const text = await file.text();
      loadCSVText(text);
    });

    document.getElementById('btnDemo').addEventListener('click', () => {
      const demo = `company_number,name,県名,市町村,番地
000000000001,サンプル製作所,埼玉県,熊谷市,本石2-135
000000000002,テスト商事,埼玉県,鴻巣市,本町1-2-3`;
      loadCSVText(demo);
    });
  </script>
</body>
</html>
