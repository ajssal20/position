<script>
  import { onMount } from "svelte";
  import { isPointWithinRadius, getDistance, getRhumbLineBearing } from "geolib";

  let distance = null;
  let inRadius = false;
  let bearing = 0; // Richtung in Grad
  let errorMsg = "";

  const ziel = {
    latitude: 42.05913363079434,
    longitude: 19.51725368912721,
  };

  onMount(() => {
    if ("geolocation" in navigator) {
      const watchId = navigator.geolocation.watchPosition(
        (pos) => {
          const current = {
            latitude: pos.coords.latitude,
            longitude: pos.coords.longitude,
          };

          distance = getDistance(current, ziel);
          inRadius = isPointWithinRadius(current, ziel, 15);

          // Richtung berechnen (0° = Norden)
          bearing = getRhumbLineBearing(current, ziel);
          errorMsg = "";
        },
        (err) => {
          console.error("Fehler bei Standort:", err);
          errorMsg = "Standort konnte nicht ermittelt werden. Bitte GPS aktivieren.";
        },
        {
          enableHighAccuracy: true,
          maximumAge: 1000,
          timeout: 5000,
        }
      );

      return () => navigator.geolocation.clearWatch(watchId);
    } else {
      errorMsg = "Geolocation wird von diesem Gerät nicht unterstützt.";
    }
  });
</script>

<style>
  body, html {
    margin: 0;
    padding: 0;
    height: 100%;
  }

  .container {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: center;
    width: 100vw;
    height: 100vh;
    padding: 1rem;
    box-sizing: border-box;
    text-align: center;
    position: relative;
  }

  /* Abstand-Kreis */
  .kreis {
    border-radius: 50%;
    box-shadow: 0 0 15px rgba(0, 0, 0, 0.3);
    margin-bottom: 1.5rem;
    width: 40vw;
    height: 40vw;
    max-width: 160px;
    max-height: 160px;
    background-color: red;
  }

  .gruen { background-color: green; }
  .rot { background-color: red; }

  p {
    font-size: 1.2rem;
    line-height: 1.4;
  }

  .error {
    color: #c00;
    font-weight: bold;
    margin-top: 1rem;
  }

  /* Kompass unten */
  .kompass-container {
    position: absolute;
    bottom: 2rem;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .kompass {
    width: 80px;
    height: 80px;
    border-radius: 50%;
    border: 3px solid #333;
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    background-color: #f0f0f0;
  }

  .pfeil {
    width: 0;
    height: 0;
    border-left: 10px solid transparent;
    border-right: 10px solid transparent;
    border-bottom: 25px solid #ff0000;
    position: absolute;
    top: 10px;
    transform: rotate(0deg);
    transform-origin: 50% 90%;
    transition: transform 0.3s ease;
  }

  /* Media Queries */
  @media (min-width: 600px) {
    .kreis { width: 25vw; height: 25vw; max-width: 200px; max-height: 200px; }
    p { font-size: 1.4rem; }
    .kompass { width: 100px; height: 100px; }
  }

  @media (min-width: 1024px) {
    .kreis { width: 15vw; height: 15vw; max-width: 250px; max-height: 250px; }
    p { font-size: 1.6rem; }
    .kompass { width: 120px; height: 120px; }
  }
</style>

<div class="container">
  <!-- Abstand-Kreis -->
  <div class="kreis {inRadius ? 'gruen' : 'rot'}"></div>

  {#if distance !== null}
    <p>Abstand zum Ziel:<br>{distance.toFixed(1)} m</p>
  {:else}
    <p>Abstand zum Ziel: wird berechnet...</p>
  {/if}

  {#if errorMsg}
    <p class="error">{errorMsg}</p>
  {/if}

  <!-- Kompass unten -->
  <div class="kompass-container">
    <div class="kompass">
      <div class="pfeil" style="transform: rotate({bearing}deg);"></div>
    </div>
    <p>Richtung: {bearing.toFixed(0)}°</p>
  </div>
</div>
