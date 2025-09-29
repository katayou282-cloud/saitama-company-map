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
    #controls { position: absolute; top: 10px; left: 10px; z-index: 1000; background: white; padding: 8px; border-radius: 6px; }
  </style>
</head>
<body>
  <div id="controls">
    <div>
      <label>CSVを読み込む: <input type="file" id="file" accept=".csv,text/csv" /></label>
      <button id="btnDemo">サンプル読込</button>
    </div>
    <div>
      <label>市町村で絞り込み: <select id="cityFilter"><option value="">全て</option></select></label>
    </div>
    <div>
      <label>法人格で絞り込み:
        <select id="typeFilter">
          <option value="">全て</option>
          <option value="株式会社">株式会社</option>
          <option value="有限会社">有限会社</option>
          <option value="NPO法人">NPO法人</option>
          <option value="その他">その他</option>
        </select>
      </label>
    </div>
  </div>

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

    let rows = [];

    function plotRows(){
      markers.clearLayers();
      const city = document.getElementById('cityFilter').value;
      const type = document.getElementById('typeFilter').value;
      const filtered = rows.filter(r => {
        if(city && r.city !== city) return false;
        if(type && r.type !== type) return false;
        return true;
      });
      filtered.forEach(r => {
        if(r._geo){
          const marker = L.marker([r._geo.lat, r._geo.lon])
            .bindPopup(`<b>${r.name}</b><br>${r.address}<br>No.${r.company_number}<br>${r.type}`);
          markers.addLayer(marker);
        }
      });
      if(filtered.length){
        const pts = filtered.filter(r=>r._geo).map(r=>[r._geo.lat, r._geo.lon]);
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

    async function processRows(items){
      for(const r of items){
        r._geo = await geocodeOne(r.address);
        await new Promise(res=>setTimeout(res,800)); // API負荷軽減
      }
      renderCityOptions();
      plotRows();
    }

    function renderCityOptions(){
      const cityFilter = document.getElementById('cityFilter');
      const cities = Array.from(new Set(rows.map(r=>r.city).filter(Boolean))).sort();
      cityFilter.innerHTML = '<option value=\"\">全て</option>' + cities.map(c=>`<option>${c}</option>`).join('');
    }

    function loadCSVText(text){
      Papa.parse(text, { header: true, skipEmptyLines: true, complete: async (res) => {
        rows = res.data.map((r, i) => {
          const name = (r.name || r['会社名'] || '').trim();
          let type = 'その他';
          if(name.includes('株式会社')) type = '株式会社';
          else if(name.includes('有限会社')) type = '有限会社';
          else if(name.includes('NPO法人')) type = 'NPO法人';
          return {
            id: i+1,
            name,
            address: (
              r.address ||
              ((r['県名']||'') + (r['市町村']||'') + (r['番地']||''))
            ).trim(),
            city: (r['市町村']||'').trim(),
            company_number: (r.company_number || r['会社番号'] || '').trim(),
            type
          };
        });
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
000000000001,株式会社サンプル製作所,埼玉県,熊谷市,本石2-135
000000000002,NPO法人テスト商事,埼玉県,鴻巣市,本町1-2-3
000000000003,有限会社北埼玉工業,埼玉県,行田市,栄町2-2`;
      loadCSVText(demo);
    });

    document.getElementById('cityFilter').addEventListener('change', plotRows);
    document.getElementById('typeFilter').addEventListener('change', plotRows);
  </script>
</body>
</html>
