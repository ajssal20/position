<script>
  import { onMount } from "svelte";
  import { writable, derived } from "svelte/store";
  import {
    isPointWithinRadius,
    getPreciseDistance,
    getGreatCircleBearing,
  } from "geolib";

  const ziel = { latitude: 42.05913363079434, longitude: 19.51725368912721 };

  const position = writable(null);
  const heading = writable(0);
  const beta = writable(0);
  const gamma = writable(0);

  const distance = derived(position, ($pos) =>
    $pos ? getPreciseDistance($pos, ziel) : null
  );
  const inRadius = derived(position, ($pos) =>
    $pos ? isPointWithinRadius($pos, ziel, 5) : false
  );
  const bearing = derived(position, ($pos) =>
    $pos ? getGreatCircleBearing($pos, ziel) : 0
  );
  const arrowRotation = derived([bearing, heading], ([$bearing, $heading]) => {
    let rot = $bearing - $heading;
    if (rot < 0) rot += 360;
    return rot;
  });
  const circleSize = derived(distance, ($d) => {
    if ($d === null) return 150;
    return Math.max(80, 300 - Math.min($d, 200));
  });

  let errorMsg = "";
  let permissionGranted = false;

  async function requestPermission() {
    if (
      typeof DeviceOrientationEvent !== "undefined" &&
      typeof DeviceOrientationEvent.requestPermission === "function"
    ) {
      try {
        const response = await DeviceOrientationEvent.requestPermission();
        if (response === "granted") {
          permissionGranted = true;
          setupOrientation();
        } else {
          errorMsg = "Keine Berechtigung für Kompassdaten.";
        }
      } catch (err) {
        console.error(err);
        errorMsg = "Kompass-Berechtigung fehlgeschlagen.";
      }
    } else {
      permissionGranted = true;
      setupOrientation();
    }
  }

  function setupOrientation() {
    function handleOrientation(event) {
      if (event.webkitCompassHeading !== undefined) {
        heading.set(event.webkitCompassHeading);
      } else if (event.absolute === true) {
        heading.set(event.alpha || 0);
      }
      beta.set(event.beta || 0);
      gamma.set(event.gamma || 0);
    }
    window.addEventListener("deviceorientationabsolute", handleOrientation, true);
    window.addEventListener("deviceorientation", handleOrientation, true);
  }

  onMount(() => {
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
          errorMsg = "Standort konnte nicht ermittelt werden.";
        },
        { enableHighAccuracy: true, maximumAge: 0, timeout: 10000 }
      );
      return () => navigator.geolocation.clearWatch(watchId);
    }
  });
</script>

<style>
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100vw;
    height: 100vh;
    padding: 1rem;
    box-sizing: border-box;
    text-align: center;
    justify-content: space-between;
    background: linear-gradient(135deg, #1e3c72, #2a5298);
    color: #fff;
  }

  .kreis {
    border-radius: 50%;
    box-shadow: 0 0 30px rgba(0,0,0,0.5);
    transition: all 0.4s ease;
    animation: pulse 2s infinite;
  }
  .gruen {
    background: radial-gradient(circle, #00ff88, #007f44);
    box-shadow: 0 0 25px rgba(0,255,136,0.8);
  }
  .rot {
    background: radial-gradient(circle, #ff4444, #990000);
    box-shadow: 0 0 25px rgba(255,68,68,0.8);
  }

  @keyframes pulse {
    0% { transform: scale(1); opacity: 0.9; }
    50% { transform: scale(1.1); opacity: 1; }
    100% { transform: scale(1); opacity: 0.9; }
  }

  .kompass-container {
    margin-bottom: 2rem;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .kompass {
    width: 150px;
    height: 150px;
    border-radius: 50%;
    border: 4px solid #fff;
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    background: radial-gradient(circle, #222, #444);
    box-shadow: 0 0 20px rgba(0,255,255,0.5);
  }

  /* Pfeil als richtiger Kompasspfeil */
  .pfeil {
    position: absolute;
    top: 15px;
    display: flex;
    flex-direction: column;
    align-items: center;
    transform-origin: 50% 85%;
    transition: transform 0.15s linear;
  }
  .pfeil-spitze {
    width: 0;
    height: 0;
    border-left: 18px solid transparent;
    border-right: 18px solid transparent;
    border-bottom: 30px solid #ffcc00;
  }
  .pfeil-schaft {
    width: 8px;
    height: 45px;
    background: linear-gradient(to bottom, #ff0000, #ffcc00);
    border-radius: 4px;
  }

  /* Nord-Markierung */
  .kompass::before {
    content: "N";
    position: absolute;
    top: 5px;
    font-size: 1.2rem;
    font-weight: bold;
    color: #fff;
  }

  p { font-size: 1rem; margin: 0.4rem 0; }
  .error { color: #ff6666; font-weight: bold; margin-top: 1rem; }
  button {
    padding: 0.6rem 1rem;
    border: none;
    border-radius: 8px;
    background: linear-gradient(45deg, #00c6ff, #0072ff);
    color: #fff;
    font-size: 1rem;
    cursor: pointer;
    margin-bottom: 1rem;
    box-shadow: 0 4px 10px rgba(0,0,0,0.3);
  }
</style>

<div class="container">
  <!-- Kreis für Zielradius -->
  <div
    class="kreis { $inRadius ? 'gruen' : 'rot'}"
    style="width: {$circleSize}px; height: {$circleSize}px;"
  ></div>

  <!-- Distanzanzeige -->
  {#if $distance !== null}
    <p>Abstand: {$distance.toFixed(1)} m</p>
  {:else}
    <p>Abstand: wird berechnet...</p>
  {/if}

  <!-- Kompass -->
  <div class="kompass-container">
    {#if !permissionGranted}
      <button on:click={requestPermission}>📍 Kompass aktivieren</button>
    {/if}

    <div class="kompass">
      <div class="pfeil" style="transform: rotate({$arrowRotation}deg);">
        <div class="pfeil-spitze"></div>
        <div class="pfeil-schaft"></div>
      </div>
    </div>

    <p>➡ Ziel: {$bearing.toFixed(0)}°</p>
    <p>🧭 Heading: {$heading.toFixed(0)}°</p>
    <p>β: {$beta.toFixed(0)}° | γ: {$gamma.toFixed(0)}°</p>
  </div>

  {#if errorMsg}
    <p class="error">{errorMsg}</p>
  {/if}
</div>
