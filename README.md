# rs3
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>Pelacak Rumah Sakit Terdekat</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css"/>
  <link rel="stylesheet" href="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.css"/>
  <style>
    html, body, #map {
      height: 100%;
      margin: 0;
    }
    #tombol {
      position: absolute;
      top: 10px;
      left: 10px;
      z-index: 999;
      background: white;
      padding: 10px;
      border-radius: 8px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.3);
    }
  </style>
</head>
<body>
  <div id="tombol">
    <button onclick="lacakLokasi()">Lacak Lokasi Saya</button>
    <p id="info">Klik tombol untuk mulai</p>
  </div>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script src="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.js"></script>

  <script>
    const map = L.map('map').setView([-2.5, 118], 5); // posisi default Indonesia

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap'
    }).addTo(map);

    let markerPengguna = null;
    let routing = null;

    function lacakLokasi() {
      if (!navigator.geolocation) {
        alert("Browser tidak mendukung geolokasi.");
        return;
      }

      navigator.geolocation.getCurrentPosition(pos => {
        const lat = pos.coords.latitude;
        const lon = pos.coords.longitude;

        if (markerPengguna) map.removeLayer(markerPengguna);

        markerPengguna = L.marker([lat, lon]).addTo(map)
          .bindPopup("Lokasi Anda").openPopup();

        map.setView([lat, lon], 14);
        document.getElementById('info').textContent = "Mencari rumah sakit terdekat...";

        cariRumahSakit(lat, lon);
      }, () => {
        alert("Gagal mendapatkan lokasi.");
      });
    }

    function cariRumahSakit(lat, lon) {
      const radius = 5000; // dalam meter
      const query = `
        [out:json];
        (
          node["amenity"="hospital"](around:${radius},${lat},${lon});
          way["amenity"="hospital"](around:${radius},${lat},${lon});
          relation["amenity"="hospital"](around:${radius},${lat},${lon});
        );
        out center;
      `;

      fetch("https://overpass-api.de/api/interpreter", {
        method: "POST",
        body: query
      })
      .then(res => res.json())
      .then(data => {
        if (!data.elements.length) {
          document.getElementById('info').textContent = "Tidak ada rumah sakit dalam radius 5 km.";
          return;
        }

        let rumahSakitTerdekat = null;
        let jarakTerdekat = Infinity;

        data.elements.forEach(el => {
          const latRS = el.lat || (el.center && el.center.lat);
          const lonRS = el.lon || (el.center && el.center.lon);
          const nama = el.tags && el.tags.name ? el.tags.name : "Rumah Sakit";

          const jarak = hitungJarak(lat, lon, latRS, lonRS);

          if (jarak < jarakTerdekat) {
            jarakTerdekat = jarak;
            rumahSakitTerdekat = { lat: latRS, lon: lonRS, nama };
          }

          const marker = L.marker([latRS, lonRS]).addTo(map);
          marker.bindPopup(`${nama}<br>Jarak: ${jarak.toFixed(2)} km<br><button onclick="tampilkanRute(${lat}, ${lon}, ${latRS}, ${lonRS})">Lihat Rute</button>`);
        });

        document.getElementById('info').textContent = `Terdekat: ${rumahSakitTerdekat.nama} (${jarakTerdekat.toFixed(2)} km)`;
        tampilkanRute(lat, lon, rumahSakitTerdekat.lat, rumahSakitTerdekat.lon);
      })
      .catch(err => {
        console.error(err);
        document.getElementById('info').textContent = "Terjadi kesalahan saat memuat rumah sakit.";
      });
    }

    function tampilkanRute(lat1, lon1, lat2, lon2) {
      if (routing) map.removeControl(routing);

      routing = L.Routing.control({
        waypoints: [
          L.latLng(lat1, lon1),
          L.latLng(lat2, lon2)
        ],
        routeWhileDragging: false,
        draggableWaypoints: false,
        createMarker: () => null
      }).addTo(map);
    }

    function hitungJarak(lat1, lon1, lat2, lon2) {
      const R = 6371;
      const dLat = (lat2 - lat1) * Math.PI / 180;
      const dLon = (lon2 - lon1) * Math.PI / 180;
      const a =
        Math.sin(dLat/2) * Math.sin(dLat/2) +
        Math.cos(lat1 * Math.PI/180) * Math.cos(lat2 * Math.PI/180) *
        Math.sin(dLon/2) * Math.sin(dLon/2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
      return R * c;
    }
  </script>
</body>
</html>
