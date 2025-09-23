<script>
  import { onMount } from "svelte";
  import { isPointWithinRadius, getDistance } from "geolib";

  let distance = null;
  let inRadius = false;
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
  .container {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    width: 100vw;
    height: 100vh;
    text-align: center;
    padding: 1rem;
    box-sizing: border-box;
  }

  .kreis {
    border-radius: 50%;
    box-shadow: 0 0 15px rgba(0, 0, 0, 0.3);
    margin-bottom: 1.5rem;
    background-color: red; /* default */
  }

  .gruen {
    background-color: green;
  }
  .rot {
    background-color: red;
  }

  p {
    font-size: 1.2rem;
    line-height: 1.4;
  }

  .error {
    color: #c00;
    font-weight: bold;
    margin-top: 1rem;
  }

  /* 📱 Standard: Handy */
  .kreis {
    width: 40vw;
    height: 40vw;
    max-width: 160px;
    max-height: 160px;
  }

  /* 📲 Tablets (ab 600px) */
  @media (min-width: 600px) {
    .kreis {
      width: 25vw;
      height: 25vw;
      max-width: 200px;
      max-height: 200px;
    }

    p {
      font-size: 1.4rem;
    }
  }

  /* 💻 Desktop (ab 1024px) */
  @media (min-width: 1024px) {
    .kreis {
      width: 15vw;
      height: 15vw;
      max-width: 250px;
      max-height: 250px;
    }

    p {
      font-size: 1.6rem;
    }
  }
</style>

<div class="container">
  <div class="kreis {inRadius ? 'gruen' : 'rot'}"></div>

  {#if distance !== null}
    <p>Abstand zum Ziel:<br>{distance.toFixed(1)} m</p>
  {:else}
    <p>Abstand zum Ziel: wird berechnet...</p>
  {/if}

  {#if errorMsg}
    <p class="error">{errorMsg}</p>
  {/if}
</div>
