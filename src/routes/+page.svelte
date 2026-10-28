<script>
  let alpha = $state(0);
  let beta = $state(0);
  let gamma = $state(0);
  let started = $state(false);
  let error = $state('');

  function handleOrientation(event) {
    alpha = event.alpha ?? 0;
    beta = event.beta ?? 0;
    gamma = event.gamma ?? 0;
  }

  async function start() {
    if (!window.DeviceOrientationEvent) {
      error = 'Device orientation not supported';
      return;
    }

    // iOS 13+ (Safari und Chrome auf iPhone)
    if (typeof DeviceOrientationEvent.requestPermission === 'function') {
      try {
        const state = await DeviceOrientationEvent.requestPermission();
        if (state !== 'granted') {
          error = 'Zugriff verweigert';
          return;
        }
      } catch (e) {
        error = 'Fehler: ' + e;
        return;
      }
    }

    window.addEventListener('deviceorientation', handleOrientation);
    started = true;
  }
</script>

<h1>Device Orientation</h1>

{#if !started}
  <button onclick={start}>Sensor starten</button>
{/if}

{#if error}<p style="color:red">{error}</p>{/if}

<p>Alpha: {alpha.toFixed(1)}</p>
<p>Beta: {beta.toFixed(1)}</p>
<p>Gamma: {gamma.toFixed(1)}</p>