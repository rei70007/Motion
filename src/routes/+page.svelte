<script>
  // ---------- Einstellungen ----------
  const TOLERANCE_DEG = 35;     // so weit darf das Handy schief sein und es zählt trotzdem
  const SHAKE_THRESHOLD = 10;   // m/s² – wie stark geschüttelt werden muss
  const SETTLE_MS = 700;        // so lange nach dem letzten Schütteln warten
  const STABLE_MS = 400;        // so lange muss das Handy ruhig in einer Lage liegen

  const MIN_DOT = Math.cos((TOLERANCE_DEG * Math.PI) / 180);

  // Augen-Positionen im 3x3-Raster (1 = oben links … 9 = unten rechts)
  const PIPS = {
    1: [5],
    2: [1, 9],
    3: [1, 5, 9],
    4: [1, 3, 7, 9],
    5: [1, 3, 5, 7, 9],
    6: [1, 3, 4, 6, 7, 9]
  };

  // ---------- Zustand ----------
  let started = $state(false);
  let error = $state('');
  let rolling = $state(false);
  let result = $state(null);       // fester Würfelwert
  let display = $state(null);      // was gerade angezeigt wird
  let face = $state(null);         // aktuelle Lage des Handys (1–6 oder null)

  let alpha = $state(0);
  let beta = $state(0);
  let gamma = $state(0);

  let lastShake = 0;
  let lastFace = null;
  let faceSince = 0;
  let rollTimer = null;

  // ---------- Lage -> Zahl ----------
  // Aus beta/gamma berechnen, wohin "oben" (Schwerkraft) im Handy zeigt.
  function faceFromOrientation(b, g) {
    const rb = (b * Math.PI) / 180;
    const rg = (g * Math.PI) / 180;
    const ux = -Math.sin(rg) * Math.cos(rb); // rechte Seite zeigt nach oben
    const uy = Math.sin(rb);                 // Oberkante zeigt nach oben
    const uz = Math.cos(rg) * Math.cos(rb);  // Display zeigt nach oben

    const ax = Math.abs(ux), ay = Math.abs(uy), az = Math.abs(uz);

    if (az >= ax && az >= ay) return az >= MIN_DOT ? (uz > 0 ? 6 : 1) : null; // flach: oben 6, unten 1
    if (ay >= ax)             return ay >= MIN_DOT ? (uy > 0 ? 2 : 5) : null; // hochkant: 2 / 5
    return                           ax >= MIN_DOT ? (ux > 0 ? 3 : 4) : null; // quer: 3 / 4
  }

  // ---------- Sensoren ----------
  function handleOrientation(e) {
    if (e.beta == null || e.gamma == null) return;
    alpha = e.alpha ?? 0;
    beta = e.beta;
    gamma = e.gamma;

    const f = faceFromOrientation(beta, gamma);
    if (f !== lastFace) {
      lastFace = f;
      faceSince = performance.now();
    }
    face = f;
  }

  function handleMotion(e) {
    const a = e.accelerationIncludingGravity;
    if (!a || a.x == null) return;
    const magnitude = Math.hypot(a.x, a.y, a.z);

    if (Math.abs(magnitude - 9.81) > SHAKE_THRESHOLD) {
      lastShake = performance.now();
      if (!rolling) startRolling();
    }
  }

  // ---------- Würfeln ----------
  function startRolling() {
    rolling = true;
    result = null;
    clearInterval(rollTimer);

    rollTimer = setInterval(() => {
      // zufällige Zahlen zeigen, solange gewürfelt wird
      display = Math.floor(Math.random() * 6) + 1;

      const now = performance.now();
      const calm = now - lastShake > SETTLE_MS;
      const stable = face !== null && now - faceSince > STABLE_MS;

      if (calm && stable) {
        clearInterval(rollTimer);
        result = face;
        display = face;
        rolling = false;
      }
    }, 80);
  }

  // ---------- Start (Berechtigung für iOS) ----------
  async function start() {
    error = '';
    try {
      // Beide Anfragen direkt im Klick starten (iOS verlangt eine Nutzer-Aktion)
      const requests = [];
      if (typeof DeviceMotionEvent?.requestPermission === 'function') {
        requests.push(DeviceMotionEvent.requestPermission());
      }
      if (typeof DeviceOrientationEvent?.requestPermission === 'function') {
        requests.push(DeviceOrientationEvent.requestPermission());
      }
      const answers = await Promise.all(requests);
      if (answers.some((r) => r !== 'granted')) {
        error = 'Zugriff auf Bewegungssensoren verweigert.';
        return;
      }
    } catch (e) {
      error = 'Fehler: ' + e;
      return;
    }
    started = true;
  }

  $effect(() => {
    if (!started) return;
    window.addEventListener('deviceorientation', handleOrientation);
    window.addEventListener('devicemotion', handleMotion);
    return () => {
      window.removeEventListener('deviceorientation', handleOrientation);
      window.removeEventListener('devicemotion', handleMotion);
      clearInterval(rollTimer);
    };
  });
</script>

<main>
  <h1>🎲 Handy-Würfel</h1>

  {#if !started}
    <p>Tippe auf Start und erlaube den Zugriff auf Bewegung &amp; Ausrichtung.</p>
    <button onclick={start}>Start</button>
  {:else}
    <div class="die" class:rolling>
      {#if display}
        {#each Array(9) as _, i}
          <span class="cell">
            {#if PIPS[display].includes(i + 1)}<span class="pip"></span>{/if}
          </span>
        {/each}
      {:else}
        <span class="question">?</span>
      {/if}
    </div>

    <p class="status">
      {#if rolling}
        {#if face === null}
          Halte das Handy ruhig in einer geraden Lage …
        {:else}
          Würfelt …
        {/if}
      {:else if result}
        Ergebnis: <strong>{result}</strong> – schütteln für neue Zahl
      {:else}
        Schüttle das Handy zum Würfeln!
      {/if}
    </p>

    <details>
      <summary>Sensorwerte</summary>
      <p>Alpha: {alpha.toFixed(1)} · Beta: {beta.toFixed(1)} · Gamma: {gamma.toFixed(1)}</p>
      <p>Aktuelle Lage: {face ?? 'keine (zu schief)'}</p>
      <p>
        Display oben = 6 · Display unten = 1 · Hochkant = 2 · Kopfüber = 5 ·
        Rechte Seite oben = 3 · Linke Seite oben = 4
      </p>
    </details>
  {/if}

  {#if error}<p class="error">{error}</p>{/if}
</main>

<style>
  main {
    font-family: system-ui, sans-serif;
    text-align: center;
    padding: 1.5rem;
  }
  button {
    font-size: 1.3rem;
    padding: 0.8rem 2.2rem;
    border: none;
    border-radius: 12px;
    background: #222;
    color: white;
  }
  .die {
    width: 200px;
    height: 200px;
    margin: 2rem auto;
    padding: 20px;
    box-sizing: border-box;
    background: white;
    border: 3px solid #222;
    border-radius: 28px;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.25);
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(3, 1fr);
  }
  .die.rolling {
    animation: shake 0.25s infinite;
  }
  .cell {
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .pip {
    width: 34px;
    height: 34px;
    border-radius: 50%;
    background: #222;
  }
  .question {
    grid-column: 1 / 4;
    grid-row: 1 / 4;
    align-self: center;
    font-size: 6rem;
    font-weight: bold;
  }
  .status {
    font-size: 1.2rem;
    min-height: 1.5em;
  }
  .error {
    color: red;
  }
  details {
    margin-top: 2rem;
    color: #555;
    font-size: 0.9rem;
  }
  @keyframes shake {
    0%   { transform: rotate(0deg); }
    25%  { transform: rotate(8deg); }
    75%  { transform: rotate(-8deg); }
    100% { transform: rotate(0deg); }
  }
</style>