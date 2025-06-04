<script lang="ts">
  import mapboxgl from 'mapbox-gl';
  import 'mapbox-gl/dist/mapbox-gl.css';
  import { onMount } from 'svelte';

  // Replace with your actual Mapbox access token
  const mapboxToken = 'pk.eyJ1IjoibWFraGFuc2Fja28iLCJhIjoiY21iZmczOXFvMDNrYjJrcXZydnJ3dm1pdCJ9.Z6kRLFVdpkZ7Q-CUQUJljA';

  // Fonction pour générer des données aléatoires pour un client
  function generateCustomerData() {
    const status = ['À jour', 'En retard', 'En attente'][Math.floor(Math.random() * 3)];
    const months = Math.floor(Math.random() * 24) + 1; // 1 à 24 mois
    const lastPayment = new Date();
    lastPayment.setDate(lastPayment.getDate() - Math.floor(Math.random() * 30)); // Dernier paiement dans les 30 derniers jours
    
    // Taille de la poubelle et montant correspondant
    const binSizes = [
      { size: 'Petite (120L)', amount: 10000 },
      { size: 'Moyenne (240L)', amount: 15000 },
      { size: 'Grande (360L)', amount: 20000 },
      { size: 'Très grande (660L)', amount: 30000 }
    ];
    const binInfo = binSizes[Math.floor(Math.random() * binSizes.length)];
    
    return {
      status,
      months,
      lastPayment: lastPayment.toLocaleDateString('fr-FR'),
      binSize: binInfo.size,
      amount: binInfo.amount
    };
  }

  let totalClients = 0;
  let commercialClients = 0;
  let residentialClients = 0;

  async function loadCustomerData() {
    try {
      const response = await fetch('/macrowaste_customers.csv');
      const csvText = await response.text();
      const lines = csvText.split('\n');
      const headers = lines[0].split(',');
      
      const points = lines.slice(1).map((line, index) => {
        const [longitude, latitude, type] = line.split(',');
        const customerData = generateCustomerData();
        return {
          type: 'Feature',
          geometry: {
            type: 'Point',
            coordinates: [parseFloat(longitude), parseFloat(latitude)]
          },
          properties: {
            id: index,
            type: type.trim() === 'Autorisé' ? 'Commercial' : 'Résidence',
            ...customerData
          }
        };
      });

      // Calculer les statistiques
      totalClients = points.length;
      commercialClients = points.filter(p => p.properties.type === 'Commercial').length;
      residentialClients = points.filter(p => p.properties.type === 'Résidence').length;

      // Créer des zones de service
      const serviceAreas = points.map(point => ({
        type: 'Feature',
        geometry: {
          type: 'Point',
          coordinates: point.geometry.coordinates
        },
        properties: {
          ...point.properties,
          radius: 500
        }
      }));

      return { points, serviceAreas };
    } catch (error) {
      console.error('Error loading customer data:', error);
      return { points: [], serviceAreas: [] };
    }
  }

  onMount(async () => {
    try {
      mapboxgl.accessToken = mapboxToken;
      const map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/navigation-day-v1',
        center: [-8.0029, 12.6392],
        zoom: 12,
        pitch: 20,
        bearing: 0
      });

      map.on('load', async () => {
        console.log('Map loaded successfully');
        
        const { points, serviceAreas } = await loadCustomerData();

        // Ajouter les zones de service
        map.addSource('service-areas', {
          type: 'geojson',
          data: {
            type: 'FeatureCollection',
            features: serviceAreas
          }
        });

        // Ajouter une couche pour les zones de service
        map.addLayer({
          id: 'service-areas-circles',
          type: 'circle',
          source: 'service-areas',
          paint: {
            'circle-radius': [
              'interpolate',
              ['linear'],
              ['zoom'],
              12, 5,  // à zoom 12, rayon de 5 pixels
              20, 20  // à zoom 20, rayon de 20 pixels
            ],
            'circle-color': [
              'match',
              ['get', 'type'],
              'Commercial', 'rgba(255, 140, 0, 0.2)',
              'Résidence', 'rgba(0, 100, 0, 0.2)',
              'rgba(0, 0, 0, 0.2)'
            ],
            'circle-stroke-width': 1,
            'circle-stroke-color': [
              'match',
              ['get', 'type'],
              'Commercial', '#FF8C00',
              'Résidence', '#006400',
              '#000000'
            ]
          }
        });

        // Ajouter la source des points clients
        map.addSource('clients', {
          type: 'geojson',
          data: {
            type: 'FeatureCollection',
            features: points
          }
        });

        // Ajouter la couche des points clients au-dessus des zones
        map.addLayer({
          id: 'clients-points',
          type: 'circle',
          source: 'clients',
          paint: {
            'circle-radius': 6,
            'circle-color': [
              'match',
              ['get', 'type'],
              'Commercial', '#FF8C00',
              'Résidence', '#006400',
              '#000000'
            ],
            'circle-opacity': 0.8,
            'circle-stroke-width': 2,
            'circle-stroke-color': '#ffffff'
          }
        });

        // Ajouter un popup au clic sur un point
        map.on('click', 'clients-points', (e) => {
          if (e.features && e.features.length > 0) {
            const feature = e.features[0];
            const coordinates = feature.geometry.coordinates.slice();
            const properties = feature.properties;

            const popupContent = `
              <div style="padding: 10px;">
                <h3 style="margin: 0 0 10px 0;">Client #${properties.id + 1}</h3>
                <p><strong>Type:</strong> ${properties.type}</p>
                <p><strong>Statut:</strong> ${properties.status}</p>
                <p><strong>Mois d'abonnement:</strong> ${properties.months}</p>
                <p><strong>Dernier paiement:</strong> ${properties.lastPayment}</p>
                <p><strong>Taille de poubelle:</strong> ${properties.binSize}</p>
                <p><strong>Montant mensuel:</strong> ${properties.amount.toLocaleString()} FCFA</p>
              </div>
            `;

            new mapboxgl.Popup()
              .setLngLat(coordinates)
              .setHTML(popupContent)
              .addTo(map);
          }
        });

        // Changer le curseur au survol des points
        map.on('mousemove', 'clients-points', (e) => {
          if (e.features && e.features.length > 0) {
            map.getCanvas().style.cursor = 'pointer';
          }
        });

        map.on('mouseleave', 'clients-points', () => {
          map.getCanvas().style.cursor = '';
        });
      });

      map.on('error', (e) => {
        console.error('Map error:', e);
      });
    } catch (error) {
      console.error('Error initializing map:', error);
    }
  });
</script>

<div class="map-container">
  <h2 class="title">Clients Macrowaste</h2>
  <div class="stats">
    <div class="stat-item">
      <span class="stat-value">{totalClients}</span>
      <span class="stat-label">Total Clients</span>
    </div>
    <div class="stat-item">
      <span class="stat-value">{commercialClients}</span>
      <span class="stat-label">Clients Commerciaux</span>
    </div>
    <div class="stat-item">
      <span class="stat-value">{residentialClients}</span>
      <span class="stat-label">Clients Résidentiels</span>
    </div>
  </div>
  <div id="map"></div>
  <div class="legend">
    <div class="legend-item">
      <span class="legend-color" style="background-color: #FF8C00;"></span>
      <span>Entreprises</span>
    </div>
    <div class="legend-item">
      <span class="legend-color" style="background-color: #006400;"></span>
      <span>Ménages</span>
    </div>
  </div>
  <div class="logo">
    <img src="/Logo_Macrowaste.png" alt="Macrowaste Logo" />
  </div>
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
  }

  .map-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
  }

  .title {
    position: absolute;
    top: 20px;
    left: 20px;
    z-index: 1;
    color: white;
    background-color: rgba(0,0,0,0.5);
    padding: 15px;
    border-radius: 4px;
    margin: 0;
  }

  .logo {
    position: absolute;
    top: 20px;
    right: 20px;
    z-index: 1;
    padding: 10px;
  }

  .logo img {
    height: 80px;
    width: auto;
  }

  #map {
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
  }

  .legend {
    position: absolute;
    bottom: 20px;
    right: 20px;
    background-color: rgba(255, 255, 255, 0.9);
    padding: 10px;
    border-radius: 4px;
    z-index: 1;
  }

  .legend-item {
    display: flex;
    align-items: center;
    margin: 5px 0;
  }

  .legend-color {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    margin-right: 10px;
  }

  :global(.mapboxgl-popup-content) {
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  }

  .stats {
    position: absolute;
    top: 100px;
    left: 20px;
    z-index: 1;
    display: flex;
    gap: 20px;
    background-color: rgba(0,0,0,0.5);
    padding: 15px;
    border-radius: 4px;
  }

  .stat-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    color: white;
  }

  .stat-value {
    font-size: 24px;
    font-weight: bold;
  }

  .stat-label {
    font-size: 14px;
    opacity: 0.8;
  }
  @media (max-width: 768px) {
  .title {
    font-size: 16px;
    padding: 10px;
    top: 10px;
    left: 10px;
    max-width: 80%;
  }

  stats {
  top: 60px;
  left: 10px;
  right: auto;
  flex-direction: column;
  gap: 10px;
  padding: 10px;
  max-width: 250px;
  width: 90%;
  background-color: rgba(0, 0, 0, 0.6);
  border-radius: 4px;
}

  .stat-item {
    align-items: flex-start;
  }

  .stat-value {
    font-size: 20px;
  }

  .stat-label {
    font-size: 12px;
  }

  .legend {
    bottom: 10px;
    right: 10px;
    font-size: 12px;
    padding: 8px;
    max-width: 80%;
  }

  .legend-item {
    margin: 4px 0;
  }

  .legend-color {
    width: 16px;
    height: 16px;
    margin-right: 6px;
  }

  .logo {
    top: 10px;
    right: 10px;
    padding: 5px;
  }

  .logo img {
    height: 50px;
    width: auto;
  }

  :global(.mapboxgl-popup-content) {
    font-size: 13px;
    max-width: 250px;
    padding: 8px;
  }
}
</style>
