# rutas-bibliotecas
Rutas de visitas técnicas nivel 1
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Rutas Diarias del Técnico</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css"/>
  <style>
    body { font-family: Arial, sans-serif; margin: 0; }
    #map { height: 60vh; }
    .panel { padding: 10px; background: #f9f9f9; }
    select, button { margin: 5px; }
    table { width: 100%; border-collapse: collapse; margin-top: 10px; }
    th, td { border: 1px solid #ccc; padding: 6px; text-align: left; }
  </style>
</head>
<body>

<div class="panel">
  <h2>📍 Selección de rutas UI</h2>
  <select id="bibliotecaSelect">
    <option value="">-- Selecciona una biblioteca --</option>
  </select>
  <button onclick="agregarParada()">Agregar Parada</button>
  <button onclick="trazarRuta()">Trazar Ruta</button>
  <button onclick="guardarRuta()">Guardar Ruta</button>
  <button onclick="resetearRecorridos()">🧹 Resetear Recorridos</button>
  <div><b>Paradas:</b> <span id="listaParadas">–</span></div>
  <div><b>Distancia:</b> <span id="dist">–</span> km · <b>Duración:</b> <span id="dur">–</span> min</div>
</div>

<div id="map"></div>

<div class="panel">
  <h2>📄 Informe Diario</h2>
  <table id="tablaRutas">
    <thead>
      <tr><th>Fecha</th><th>Ruta</th><th>Distancia (km)</th><th>Duración (min)</th></tr>
    </thead>
    <tbody></tbody>
  </table>
</div>

<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<script>
const bibliotecas = [
  { nombre: "Parque Biblioteca Nuevo Occidente, Lusitania", coord: [6.27194, -75.64278] },
  { nombre: "Parque Biblioteca Belén", coord: [6.2236950, -75.5983286] },
  { nombre: "Parque Biblioteca Manuel Mejía Vallejo, Guayabal", coord: [6.2206491, -75.5858041] },
  { nombre: "Biblioteca Pública La Floresta", coord: [6.2580560, -75.5986110] },
  { nombre: "Biblioteca Pública El Poblado", coord: [6.1966100, -75.5617464] },
  { nombre: "Biblioteca Pública El Limonar", coord: [6.18417, -75.57972] },
  { nombre: "Biblioteca Público Corregimental Santa Elena", coord: [6.2475, -75.5175] },
  { nombre: "Biblioteca Público Corregimental San Sebastián de Palmitas", coord: [6.3222220, -75.7166670] },
  { nombre: "Casa de la Lectura Infantil", coord: [6.25111, -75.56861] },
  { nombre: "Biblioteca Pública Santa Cruz", coord: [6.29194, -75.54778] },
  { nombre: "Biblioteca Pública Ávila, María Agudelo Mejía", coord: [6.30028, -75.55333] },
  { nombre: "Parque Biblioteca Presbítero José Luis Arroyave, San Javier", coord: [6.2544886, -75.6133997] },
  { nombre: "Centro de Documentación Musical El Jordán", coord: [6.27222, -75.59056] },
  { nombre: "Biblioteca Pública Centro Occidental", coord: [6.25778, -75.62278] },
  { nombre: "Centro de Documentación Buen Comienzo", coord: [6.2236950, -75.5983286] },
  { nombre: "Parque Biblioteca Fernando Botero, San Cristóbal", coord: [6.3002780, -75.6458330] },
  { nombre: "Biblioteca Público Escolar Granizal", coord: [6.30028, -75.55833] },
  { nombre: "Casa de la Literatura, San Germán", coord: [6.26997, -75.58905] },
  { nombre: "Parque Biblioteca León de Greiff, La Ladera", coord: [6.25148, -75.55395] },
  { nombre: "Biblioteca Parque al Barrio", coord: [6.3008330, -75.5586110] },
  { nombre: "Parque Biblioteca José Horacio Betancur, San Antonio de Prado", coord: [6.18750, -75.67500] },
  { nombre: "Biblioteca Fernando Gómez Martínez, Robledo", coord: [6.28295, -75.59256] },
  { nombre: "Biblioteca Pública Altavista", coord: [6.22199, -75.62899] },
  { nombre: "Biblioteca Público Escolar Popular No.2", coord: [6.30237, -75.54706] },
  { nombre: "Parque Biblioteca Tomás Carrasquilla, La Quintana", coord: [6.28536, -75.58347] },
  { nombre: "Parque Biblioteca Gabriel García Márquez, Doce de Octubre", coord: [6.30432, -75.57775] }
];
  // Puedes agregar más aquí

const map = L.map('map').setView([6.25, -75.58], 12);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '&copy; OpenStreetMap contributors' }).addTo(map);

// Marcadores
bibliotecas.forEach(b => {
  L.marker(b.coord).addTo(map).bindPopup(b.nombre);
});

// Select
const select = document.getElementById('bibliotecaSelect');
bibliotecas.forEach((b, i) => {
  const opt = new Option(b.nombre, i);
  select.add(opt);
});

let paradas = [];
let rutaLayer;

function agregarParada() {
  const idx = select.value;
  if (idx === "") return;
  paradas.push(bibliotecas[idx]);
  document.getElementById('listaParadas').textContent = paradas.map(p => p.nombre).join(" → ");
}

function resetearRecorridos() {
  if (confirm("¿Estás seguro de que quieres borrar todos los recorridos y limpiar el mapa?")) {
    // Eliminar registros guardados
    localStorage.removeItem("rutasDiarias");

    // Limpiar tabla
    mostrarInforme();

    // Limpiar paradas
    paradas = [];
    document.getElementById('listaParadas').textContent = "–";

    // Limpiar ruta del mapa
    if (rutaLayer) {
      map.removeLayer(rutaLayer);
      rutaLayer = null;
    }

    // Resetear distancia y duración
    document.getElementById('dist').textContent = "–";
    document.getElementById('dur').textContent = "–";
  }
}
async function trazarRuta() {
  if (rutaLayer) map.removeLayer(rutaLayer);
  if (paradas.length < 2) return alert("Agrega al menos dos paradas");

  const coords = paradas.map(p => `${p.coord[1]},${p.coord[0]}`).join(";");
  const url = `https://router.project-osrm.org/route/v1/driving/${coords}?geometries=geojson&overview=full`;
  const r = await fetch(url);
  const data = await r.json();
  const route = data.routes[0];
  const km = route.distance / 1000;
  const min = route.duration / 60;
  const minutosExtra = paradas.length * 15;
  const minMoto = min * 0.85 + minutosExtra;


  rutaLayer = L.geoJSON(route.geometry, { style: { color: '#ff6600', weight: 5 } }).addTo(map);
  map.fitBounds(rutaLayer.getBounds(), { padding: [30,30] });

  document.getElementById('dist').textContent = km.toFixed(1);
  document.getElementById('dur').textContent = Math.round(minMoto);
}

function guardarRuta() {
  if (paradas.length < 2) return alert("Primero traza una ruta");
  const fecha = new Date().toLocaleDateString('es-CO', { timeZone: 'America/Bogota' });
  const ruta = paradas.map(p => p.nombre).join(" → ");
  const dist = document.getElementById('dist').textContent;
  const dur = document.getElementById('dur').textContent;
  const registro = { fecha, ruta, dist, dur };

  const registros = JSON.parse(localStorage.getItem("rutasDiarias") || "[]");
  registros.push(registro);
  localStorage.setItem("rutasDiarias", JSON.stringify(registros));
  mostrarInforme();
  paradas = [];
  document.getElementById('listaParadas').textContent = "–";
  if (rutaLayer) map.removeLayer(rutaLayer);
}

function mostrarInforme() {
  const registros = JSON.parse(localStorage.getItem("rutasDiarias") || "[]");
  const tbody = document.querySelector("#tablaRutas tbody");
  tbody.innerHTML = "";
  registros.forEach(r => {
    const fila = `<tr><td>${r.fecha}</td><td>${r.ruta}</td><td>${r.dist}</td><td>${r.dur}</td></tr>`;
    tbody.innerHTML += fila;
  });
}

mostrarInforme();
</script>
</body>
</html>
