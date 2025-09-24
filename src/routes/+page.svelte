<script>
  import { onMount } from "svelte";
  import { writable, derived } from "svelte/store";
  import { isPointWithinRadius, getPreciseDistance, getGreatCircleBearing } from "geolib";

  const ziel = { latitude: 42.05913363079434, longitude: 19.51725368912721 };

  // Standort und Sensoren
  const position = writable(null);
  const heading = writable(0);
  const beta = writable(0);
  const gamma = writable(0);

  const distance = derived(position, ($pos) =>
    $pos ? getPreciseDistance($pos, ziel) : null
  );

  // InRadius kontrollieren für Farbwechsel & Vibration
  const inRadius = writable(false);
  let lastInRadius = false;

  const bearing = derived(position, ($pos) =>
    $pos ? getGreatCircleBearing($pos, ziel) : 0
  );

  const arrowRotation = derived([bearing, heading], ([$bearing, $heading]) => {
    let rot = $bearing - $heading;
    if (rot < 0) rot += 360;
    return rot;
  });

  const circleSize = derived(distance, ($d) => {
    if ($d === null) return 150; // Default-Wert
    return Math.max(80, 300 - Math.min($d, 200));
  });

  // Battery
  const batteryLevel = writable(null);
  const isCharging = writable(false);

  let errorMsg = "";
  let permissionGranted = false;

  async function requestPermission() {
    if (typeof DeviceOrientationEvent !== "undefined" &&
        typeof DeviceOrientationEvent.requestPermission === "function") {
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
    // Standort
    if ("geolocation" in navigator) {
      const watchId = navigator.geolocation.watchPosition(
        (pos) => {
          if (pos.coords.accuracy <= 20) {
            const current = { latitude: pos.coords.latitude, longitude: pos.coords.longitude };
            position.set(current);

            // Prüfen Radius und Vibration auslösen
            const inside = isPointWithinRadius(current, ziel, 5);
            inRadius.set(inside);

            if (inside && !lastInRadius && navigator.vibrate) {
              navigator.vibrate(300); // 300ms Vibration
            }
            lastInRadius = inside;
          }
        },
        (err) => {
          console.error(err);
          errorMsg = "Standort konnte nicht ermittelt werden.";
        },
        { enableHighAccuracy: true, maximumAge: 0, timeout: 10000 }
      );
      return () => navigator.geolocation.clearWatch(watchId);
    }

    // Battery API
    if (navigator.getBattery) {
      navigator.getBattery().then((battery) => {
        batteryLevel.set(Math.floor(battery.level * 100));
        isCharging.set(battery.charging);

        battery.addEventListener("levelchange", () =>
          batteryLevel.set(Math.floor(battery.level * 100))
        );
        battery.addEventListener("chargingchange", () =>
          isCharging.set(battery.charging)
        );
      }).catch(() => batteryLevel.set(null));
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
    position: relative;
    font-family: "Helvetica Neue", sans-serif;
  }

  .kreis {
    border-radius: 50%;
    box-shadow: 0 0 20px rgba(0,0,0,0.4);
    transition: all 0.4s ease;
    animation: pulse 2s infinite;
  }
  .gruen {
    background: radial-gradient(circle, #00ff88, #007f44);
    box-shadow: 0 0 20px rgba(0,255,136,0.6);
  }
  .rot {
    background: radial-gradient(circle, #ff4444, #990000);
    box-shadow: 0 0 20px rgba(255,68,68,0.6);
  }

  @keyframes pulse {
    0% { transform: scale(1); opacity: 0.9; }
    50% { transform: scale(1.08); opacity: 1; }
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
    border: 3px solid #fff;
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    background: radial-gradient(circle, #222, #444);
    box-shadow: 0 0 15px rgba(0,255,255,0.4);
  }

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
    border-left: 16px solid transparent;
    border-right: 16px solid transparent;
    border-bottom: 28px solid #ffcc00;
  }
  .pfeil-schaft {
    width: 7px;
    height: 40px;
    background: linear-gradient(to bottom, #ff0000, #ffcc00);
    border-radius: 4px;
  }

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
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 6px;
    background: rgba(0,200,255,0.8);
    color: #fff;
    font-size: 0.95rem;
    cursor: pointer;
    margin-bottom: 1rem;
    box-shadow: 0 2px 6px rgba(0,0,0,0.3);
  }

  .battery-small {
    position: absolute;
    top: 10px;
    right: 10px;
    padding: 4px 7px;
    background: rgba(0,0,0,0.25);
    color: #fff;
    border-radius: 6px;
    font-size: 0.85rem;
    font-weight: 600;
    display: flex;
    align-items: center;
    box-shadow: 0 0 4px rgba(0,0,0,0.3);
    backdrop-filter: blur(3px);
  }
</style>

<div class="container">
  <!-- Kleine Akkuanzeige -->
  {#if $batteryLevel !== null}
    <div class="battery-small">
      🔋 {$batteryLevel.toFixed(0)}% {#if $isCharging}⚡{/if}
    </div>
  {/if}

  <!-- Kreis für Zielradius -->
  <div
    class="kreis { $inRadius ? 'gruen' : 'rot'}"
    style="width: {$circleSize || 150}px; height: {$circleSize || 150}px;"
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
