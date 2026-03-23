<template>
  <div class="map-app">
    <div class="header">
      <h2>🗺️ Mapa de Rutas TP</h2>
      <div class="controls">
        <input type="file" id="excelInput" accept=".xlsx, .xls, .csv" style="display: none;" @change="handleFileUpload">
        <label for="excelInput" class="btn-upload">1️⃣ Cargar Excel de Ruta</label>
        <button class="btn-locate" @click="ubicarMiPosicion">📍 Mi Ubicación</button>
      </div>
      <div class="status" v-if="procesando">
        ⏳ Ubicando en el mapa: {{ procesadas }} de {{ totalRuta }}
      </div>
      <div class="leyenda" v-if="totalRuta > 0">
        <span class="dot red"></span> Entregas | 
        <span class="dot purple"></span> Recojos | 
        <span class="dot orange"></span> Intercambios
      </div>
    </div>

    <div id="map" class="map-container"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as XLSX from 'xlsx';

const ruta = ref([]);
const procesando = ref(false);
const procesadas = ref(0);
const totalRuta = ref(0);

let map = null;
let geocoder = null;
let markers = [];
let miUbicacionMarker = null;

onMounted(() => {
  cargarGoogleMapsAPI();
});

const cargarGoogleMapsAPI = () => {
  // REEMPLAZA ESTA LÍNEA CON LA CLAVE QUE COPIASTE
  const API_KEY = "AIzaSyCwTiEgonLx7sg63XR5BFwTjIsHI7L4wzw"; 
  
  if (window.google && window.google.maps) {
    initMap();
    return;
  }
  // ... resto del código
  const script = document.createElement("script");
  script.src = `https://maps.googleapis.com/maps/api/js?key=${API_KEY}&libraries=places`;
  script.async = true;
  script.defer = true;
  script.onload = initMap;
  document.head.appendChild(script);
};

const initMap = () => {
  const lima = { lat: -12.046374, lng: -77.042793 };
  map = new window.google.maps.Map(document.getElementById("map"), {
    zoom: 11,
    center: lima,
    mapTypeControl: false,
    streetViewControl: false,
  });
  geocoder = new window.google.maps.Geocoder();
};

const ubicarMiPosicion = () => {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      (position) => {
        const pos = {
          lat: position.coords.latitude,
          lng: position.coords.longitude,
        };
        if (miUbicacionMarker) miUbicacionMarker.setMap(null);
        miUbicacionMarker = new window.google.maps.Marker({
          position: pos,
          map: map,
          title: "¡Estás aquí!",
          icon: "http://maps.google.com/mapfiles/ms/icons/blue-dot.png"
        });
        map.setCenter(pos);
        map.setZoom(14);
      },
      () => alert("❌ No se pudo obtener tu ubicación. Activa el GPS.")
    );
  }
};

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;
  
  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const data = new Uint8Array(e.target.result);
      const workbook = XLSX.read(data, { type: 'array' });
      let clientesExtraidos = [];

      workbook.SheetNames.forEach(sheetName => {
        const sheet = workbook.Sheets[sheetName];
        const json = XLSX.utils.sheet_to_json(sheet);
        
        json.forEach(row => {
          if (!row.NOMBRE) return;
          
          const direccionCompleta = `${row.DIRECCIÓN || ''}, ${row.DISTRITO || ''}, Lima, Perú`.replace(/^, |, $/g, '');
          
          let tipo = "ENTREGA";
          const nombreHoja = sheetName.toUpperCase();
          if (nombreHoja.includes("RECOJO")) tipo = "RECOJO";
          if (nombreHoja.includes("INTERCAMBIO")) tipo = "INTERCAMBIO";

          clientesExtraidos.push({
            nom: row.NOMBRE,
            addr: direccionCompleta,
            motivo: row.MOTIVO || "Sin detalle",
            tipo: tipo
          });
        });
      });

      ruta.value = clientesExtraidos;
      totalRuta.value = clientesExtraidos.length;
      procesadas.value = 0;
      limpiarMapa();
      
      if (clientesExtraidos.length > 0) {
        procesarDireccionesEnLote();
      } else {
        alert("El Excel está vacío o no tiene la columna 'NOMBRE'.");
      }
    } catch(err) {
      alert("❌ Error leyendo el Excel.");
    }
  };
  reader.readAsArrayBuffer(file);
  event.target.value = null;
};

const limpiarMapa = () => {
  markers.forEach(marker => marker.setMap(null));
  markers = [];
};

const getIconUrl = (tipo) => {
  if (tipo === "RECOJO") return "http://maps.google.com/mapfiles/ms/icons/purple-dot.png";
  if (tipo === "INTERCAMBIO") return "http://maps.google.com/mapfiles/ms/icons/orange-dot.png";
  return "http://maps.google.com/mapfiles/ms/icons/red-dot.png"; // Entregas
};

const procesarDireccionesEnLote = () => {
  procesando.value = true;
  let i = 0;
  
  const intervalo = setInterval(() => {
    if (i >= ruta.value.length) {
      clearInterval(intervalo);
      procesando.value = false;
      return;
    }

    const cliente = ruta.value[i];
    geocoder.geocode({ address: cliente.addr }, (results, status) => {
      if (status === "OK") {
        const marker = new window.google.maps.Marker({
          map: map,
          position: results[0].geometry.location,
          title: cliente.nom,
          icon: getIconUrl(cliente.tipo)
        });

        const infoWindow = new window.google.maps.InfoWindow({
          content: `<div style="color:black;"><strong>${cliente.tipo}: ${cliente.nom}</strong><br>📍 ${cliente.addr}<br>📦 ${cliente.motivo}</div>`
        });

        marker.addListener("click", () => infoWindow.open(map, marker));
        markers.push(marker);
      } else if (status === "REQUEST_DENIED") {
        clearInterval(intervalo);
        procesando.value = false;
        alert("⛔ GOOGLE BLOQUEÓ LA BÚSQUEDA. \n\nPara que Google convierta texto a mapa, debes entrar a Google Cloud > Facturación, y agregar una tarjeta. (Te regalan $200 al mes, no te cobrarán nada por este uso).");
      } else if (status === "OVER_QUERY_LIMIT") {
        console.warn("Yendo muy rápido para Google...");
      }
      procesadas.value++;
    });

    i++;
  }, 500); // Medio segundo por dirección para no saturar
};
</script>

<style scoped>
.map-app { display: flex; flex-direction: column; height: 100vh; font-family: -apple-system, sans-serif; }
.header { background: #212529; color: white; padding: 15px; text-align: center; z-index: 10; box-shadow: 0 4px 6px rgba(0,0,0,0.2); }
.header h2 { margin: 0 0 15px 0; font-size: 1.3em;}
.controls { display: flex; gap: 10px; justify-content: center; margin-bottom: 10px; }
.btn-upload, .btn-locate { background: #0d6efd; color: white; border: none; padding: 10px 15px; border-radius: 8px; font-weight: bold; cursor: pointer; flex: 1; text-align: center; }
.btn-locate { background: #198754; }
.status { font-size: 0.9em; color: #ffc107; font-weight: bold; margin-bottom: 5px; }
.leyenda { font-size: 0.85em; color: #ced4da; }
.dot { display: inline-block; width: 10px; height: 10px; border-radius: 50%; margin-right: 3px; }
.dot.red { background: #ea4335; }
.dot.purple { background: #8e24aa; }
.dot.orange { background: #f29900; }
.map-container { flex-grow: 1; width: 100%; }
</style>