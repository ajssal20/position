<script>
  import { onMount } from "svelte";
  import { writable, derived } from "svelte/store";
  import {
    isPointWithinRadius,
    getPreciseDistance,
    getGreatCircleBearing,
  } from "geolib";

  // 📍 Zielkoordinaten
  const ziel = {
    latitude: 42.05913363079434,
    longitude: 19.51725368912721,
  };

  // 🔹 Stores
  const position = writable(null);
  const alpha = writable(0); // Kompassrichtung
  const beta = writable(0);  // Vorwärts/Zurück kippen
  const gamma = writable(0); // Links/Rechts kippen

  // 🔹 Derived Stores
  const distance = derived(position, ($pos) =>
    $pos ? getPreciseDistance($pos, ziel) : null
  );

  const inRadius = derived(position, ($pos) =>
    $pos ? isPointWithinRadius($pos, ziel, 15) : false
  );

  const bearing = derived(position, ($pos) =>
    $pos ? getGreatCircleBearing($pos, ziel) : 0
  );

  // Pfeilrichtung berechnen
  const arrowRotation = derived(
    [bearing, alpha],
    ([$bearing, $alpha]) => {
      let rot = $bearing - $alpha;
      if (rot < 0) rot += 360;
      return rot;
    }
  );

  let errorMsg = "";

  onMount(() => {
    // 🌍 Standort
    if ("geolocation" in navigator) {
      const watchId = navigator.geolocation.watchPosition(
        (pos) => {
          if (pos.coords.accuracy <= 20) {
            position.set({
              latitude: pos.coords.latitude,
              longitude: pos.coords.longitude,
            });
          }
        },
        (err) => {
          console.error("Fehler bei Standort:", err);
          errorMsg = "Standort konnte nicht ermittelt werden. Bitte GPS aktivieren.";
        },
        { enableHighAccuracy: true, maximumAge: 0, timeout: 10000 }
      );
      return () => navigator.geolocation.clearWatch(watchId);
    } else {
      errorMsg = "Geolocation wird nicht unterstützt.";
    }

    // 📱 Geräteausrichtung
    function handleOrientation(event) {
      alpha.set(event.alpha || 0);
      beta.set(event.beta || 0);
      gamma.set(event.gamma || 0);
    }
    window.addEventListener("deviceorientation", handleOrientation, true);
    return () => window.removeEventListener("deviceorientation", handleOrientation);
  });
</script>

<style>
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
  }

  .kreis {
    border-radius: 50%;
    margin-bottom: 1.5rem;
    width: 40vw;
    height: 40vw;
    max-width: 160px;
    max-height: 160px;
  }
  .gruen { background-color: green; }
  .rot { background-color: red; }

  .kompass-container {
    position: absolute;
    bottom: 2rem;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .kompass {
    width: 120px;
    height: 120px;
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
    border-left: 12px solid transparent;
    border-right: 12px solid transparent;
    border-bottom: 35px solid red;
    position: absolute;
    top: 20px;
    transform-origin: 50% 90%;
    transition: transform 0.15s linear;
  }

  p { font-size: 1rem; margin: 0.4rem 0; }
  .error { color: #c00; font-weight: bold; margin-top: 1rem; }
</style>

<div class="container">
  <div class="kreis { $inRadius ? 'gruen' : 'rot'}"></div>

  {#if $distance !== null}
    <p>Abstand zum Ziel:<br>{$distance.toFixed(1)} m</p>
  {:else}
    <p>Abstand zum Ziel: wird berechnet...</p>
  {/if}

  {#if errorMsg}
    <p class="error">{errorMsg}</p>
  {/if}

  <div class="kompass-container">
    <div class="kompass">
      <div class="pfeil" style="transform: rotate({$arrowRotation}deg);"></div>
    </div>
    <p>➡ Zielrichtung: {$bearing.toFixed(0)}°</p>
    <p>📍 Heading (α): {$alpha.toFixed(0)}°</p>
    <p>📐 Neigung β: {$beta.toFixed(0)}° | γ: {$gamma.toFixed(0)}°</p>
  </div>
</div>
