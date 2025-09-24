<script>
  import { onMount } from "svelte";
  import { writable, derived } from "svelte/store";
  import {
    isPointWithinRadius,
    getPreciseDistance,
    getGreatCircleBearing,
  } from "geolib";

  const ziel = {
    latitude: 42.05913363079434,
    longitude: 19.51725368912721,
  };

  const position = writable(null);
  const heading = writable(0); // jetzt aus absoluter Orientierung
  const beta = writable(0);
  const gamma = writable(0);

  const distance = derived(position, ($pos) =>
    $pos ? getPreciseDistance($pos, ziel) : null
  );

  const inRadius = derived(position, ($pos) =>
    $pos ? isPointWithinRadius($pos, ziel, 15) : false
  );

  const bearing = derived(position, ($pos) =>
    $pos ? getGreatCircleBearing($pos, ziel) : 0
  );

  const arrowRotation = derived(
    [bearing, heading],
    ([$bearing, $heading]) => {
      let rot = $bearing - $heading;
      if (rot < 0) rot += 360;
      return rot;
    }
  );

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
      // Android oder Browser mit direkter Unterstützung
      permissionGranted = true;
      setupOrientation();
    }
  }

  function setupOrientation() {
    function handleOrientation(event) {
      // Prüfen ob absolute Werte geliefert werden
      if (event.absolute === true || event.webkitCompassHeading !== undefined) {
        let h = 0;

        if (event.webkitCompassHeading !== undefined) {
          // iOS liefert direkten Kompasswinkel
          h = event.webkitCompassHeading; // in Grad
        } else {
          // Android liefert alpha (0 = Norden)
          h = event.alpha || 0;
        }

        heading.set(h);
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
</style>

<div>
  {#if !permissionGranted}
    <button on:click={requestPermission}>📍 Kompass aktivieren</button>
  {/if}

  {#if $distance !== null}
    <p>Abstand: {$distance.toFixed(1)} m</p>
  {:else}
    <p>Abstand: wird berechnet...</p>
  {/if}

  <div class="kompass">
    <div class="pfeil" style="transform: rotate({$arrowRotation}deg);"></div>
  </div>

  <p>➡ Ziel: {$bearing.toFixed(0)}°</p>
  <p>🧭 Heading: {$heading.toFixed(0)}°</p>
  <p>β: {$beta.toFixed(0)}° | γ: {$gamma.toFixed(0)}°</p>

  {#if errorMsg}
    <p style="color:red">⚠ {errorMsg}</p>
  {/if}
</div>
