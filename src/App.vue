<template>
  <div class="map-app">
    <div class="header">
      <h2>🗺️ Ruta de Hoy: {{ totalRuta }} puntos</h2>
      <div class="controls">
        <input type="file" id="excelInput" accept=".xlsx, .xls, .csv" style="display: none;" @change="handleFileUpload">
        <label for="excelInput" class="btn-upload">📁 Nuevo Excel</label>
        <button class="btn-locate" @click="ubicarMiPosicion">📍 Mi Ubicación</button>
      </div>
      <div class="status" v-if="procesando">
        ⏳ Cargando: {{ procesadas }} de {{ totalRuta }}
      </div>
      <div class="leyenda" v-if="totalRuta > 0">
        <span class="dot red"></span> Entregas | <span class="dot purple"></span> Recojos | <span class="dot orange"></span> Intercambios
      </div>
    </div>
    <div id="map" class="map-container"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as XLSX from 'xlsx';

const API_KEY = "AIzaSyCwTiEgonLx7sg63XR5BFwTjIsHI7L4wzw"; // Tu clave ya activada
const ruta = ref([]);
const procesando = ref(false);
const procesadas = ref(0);
const totalRuta = ref(0);
let map, geocoder, markers = [], miUbicacionMarker = null;

onMounted(() => {
  const script = document.createElement("script");
  script.src = `https://maps.googleapis.com/maps/api/js?key=${API_KEY}&libraries=places`;
  script.async = true;
  script.onload = () => {
    initMap();
    // RECUPERAR DATOS GUARDADOS AL ABRIR
    const guardado = localStorage.getItem('ultima_ruta');
    if (guardado) {
      const datos = JSON.parse(guardado);
      ruta.value = datos;
      totalRuta.value = datos.length;
      setTimeout(() => dibujarPuntosEnMapa(datos), 1000);
    }
  };
  document.head.appendChild(script);
});

const initMap = () => {
  map = new window.google.maps.Map(document.getElementById("map"), {
    zoom: 12, center: { lat: -12.046, lng: -77.042 },
    mapTypeControl: false, streetViewControl: false,
  });
  geocoder = new window.google.maps.Geocoder();
};

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (e) => {
    const data = new Uint8Array(e.target.result);
    const workbook = XLSX.read(data, { type: 'array' });
    let extraidos = [];
    workbook.SheetNames.forEach(name => {
      const json = XLSX.utils.sheet_to_json(workbook.Sheets[name]);
      json.forEach(row => {
        if (row.NOMBRE && row.DIRECCIÓN) {
          let tipo = name.toUpperCase().includes("RECOJO") ? "RECOJO" : name.toUpperCase().includes("INTERCAMBIO") ? "INTERCAMBIO" : "ENTREGA";
          extraidos.push({ 
            nom: row.NOMBRE, 
            addr: `${row.DIRECCIÓN}, ${row.DISTRITO || ''}, Lima`,
            tipo, motivo: row.MOTIVO || '' 
          });
        }
      });
    });
    // GUARDAR EN MEMORIA DEL CELULAR
    localStorage.setItem('ultima_ruta', JSON.stringify(extraidos));
    ruta.value = extraidos;
    totalRuta.value = extraidos.length;
    dibujarPuntosEnMapa(extraidos);
  };
  reader.readAsArrayBuffer(file);
};

const dibujarPuntosEnMapa = (datos) => {
  markers.forEach(m => m.setMap(null));
  markers = [];
  procesando.value = true;
  procesadas.value = 0;

  datos.forEach((cliente, i) => {
    setTimeout(() => {
      geocoder.geocode({ address: cliente.addr }, (res, status) => {
        if (status === "OK") {
          const pos = res[0].geometry.location;
          const color = cliente.tipo === "RECOJO" ? "purple" : cliente.tipo === "INTERCAMBIO" ? "orange" : "red";
          const marker = new window.google.maps.Marker({
            map, position: pos, title: cliente.nom,
            icon: `http://maps.google.com/mapfiles/ms/icons/${color}-dot.png`
          });
          
          // BOTÓN DE NAVEGACIÓN GPS
          const googleMapsUrl = `https://www.google.com/maps/dir/?api=1&destination=${encodeURIComponent(cliente.addr)}`;
          const wazeUrl = `https://waze.com/ul?q=${encodeURIComponent(cliente.addr)}&navigate=yes`;

          const info = new window.google.maps.InfoWindow({
            content: `
              <div style="color:black; padding:5px;">
                <b style="font-size:14px;">${cliente.nom}</b><br>
                <small>${cliente.addr}</small><br><br>
                <div style="display:flex; gap:5px;">
                  <a href="${googleMapsUrl}" target="_blank" style="background:#4285F4; color:white; padding:8px; border-radius:5px; text-decoration:none; font-size:12px; font-weight:bold;">🚀 IR CON GOOGLE</a>
                  <a href="${wazeUrl}" target="_blank" style="background:#33CCFF; color:white; padding:8px; border-radius:5px; text-decoration:none; font-size:12px; font-weight:bold;">🚙 WAZE</a>
                </div>
              </div>`
          });
          marker.addListener("click", () => info.open(map, marker));
          markers.push(marker);
        }
        procesadas.value++;
        if (procesadas.value === datos.length) procesando.value = false;
      });
    }, i * 400);
  });
};

const ubicarMiPosicion = () => {
  navigator.geolocation.getCurrentPosition((p) => {
    const pos = { lat: p.coords.latitude, lng: p.coords.longitude };
    if (miUbicacionMarker) miUbicacionMarker.setMap(null);
    miUbicacionMarker = new window.google.maps.Marker({
      position: pos, map, icon: "http://maps.google.com/mapfiles/ms/icons/blue-dot.png"
    });
    map.setCenter(pos);
    map.setZoom(15);
  });
};
</script>

<style scoped>
.map-app { display: flex; flex-direction: column; height: 100vh; }
.header { background: #1a1a1a; color: white; padding: 10px; text-align: center; }
.controls { display: flex; gap: 8px; margin: 10px 0; }
.btn-upload, .btn-locate { background: #007bff; color: white; padding: 12px; border-radius: 8px; flex: 1; cursor: pointer; border:none; font-weight:bold; font-size: 0.9em; }
.btn-locate { background: #28a745; }
.status { color: #ffc107; font-size: 0.8em; margin-bottom: 5px; }
.leyenda { font-size: 0.75em; margin-bottom: 5px; }
.dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin: 0 3px 0 8px; }
.red { background: #ff0000; } .purple { background: #a020f0; } .orange { background: #ffa500; }
.map-container { flex-grow: 1; width: 100%; }
</style>