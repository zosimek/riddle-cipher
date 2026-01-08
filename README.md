<!doctype html>
<html lang="pl">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Szyfr – LEGO BONSAI</title>
  <style>
    :root { --letter-gap: 0.55em; }

    body{
      margin:0;
      min-height:100vh;
      display:grid;
      place-items:center;
      font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif;
      background:#0b0c10;
      color:#eaf2ff;
    }

    .wrap{
      width:min(980px,92vw);
      text-align:center;
      padding:24px 16px;
    }

    .line1{
      font-size:clamp(26px,4vw,44px);
      font-weight:800;
      letter-spacing:0.10em;
      margin-bottom:10px;
      text-transform:uppercase;
      transition:text-shadow 0.3s ease, color 0.3s ease;
    }

    .success{
      color:#9dffb0;
      text-shadow: 0 0 8px rgba(157,255,176,0.8), 0 0 18px rgba(157,255,176,0.6);
    }

    .status{
      height:1.2em;
      font-size:14px;
      letter-spacing:0.2em;
      opacity:0;
      transition:opacity 0.3s ease;
    }
    .status.show{ opacity:0.9; }

    .alpha{
      font-family:ui-monospace,SFMono-Regular,Menlo,Monaco,Consolas,"Liberation Mono",monospace;
      font-size:clamp(16px,2.2vw,26px);
      letter-spacing:var(--letter-gap);
      white-space:nowrap;
      margin:12px 0;
      opacity:0.95;
    }

    .controls{
      display:inline-flex;
      gap:10px;
      align-items:center;
      justify-content:center;
      margin-top:8px;
    }

    button{
      appearance:none;
      border:1px solid rgba(255,255,255,0.18);
      background:rgba(255,255,255,0.06);
      color:inherit;
      padding:10px 12px;
      border-radius:12px;
      font-size:16px;
      cursor:pointer;
      transition:transform 0.05s ease, background 0.15s ease;
      user-select:none;
    }
    button:hover{ background:rgba(255,255,255,0.10); }
    button:active{ transform:translateY(1px); }

    .hint{
      margin-top:10px;
      font-size:13px;
      opacity:0.75;
    }

    @media (max-width:420px){
      :root{ --letter-gap:0.38em; }
    }
  </style>
</head>

<body>
  <div class="wrap" role="application">
    <div class="line1" id="decoded">_____</div>
    <div class="status" id="status">✓ OK</div>

    <div class="alpha" id="alphaStatic"></div>
    <div class="alpha" id="alphaShifted"></div>

    <div class="controls">
      <button id="left">←</button>
      <button id="right">→</button>
    </div>

    <div class="hint">Przesuwaj dolny alfabet (← / →).</div>
  </div>

  <script>
    const alphabet = "abcdefghijklmnopqrstuvwxyz".split("");

    const decodedEl = document.getElementById("decoded");
    const statusEl  = document.getElementById("status");
    const staticEl  = document.getElementById("alphaStatic");
    const shiftedEl = document.getElementById("alphaShifted");
    const leftBtn   = document.getElementById("left");
    const rightBtn  = document.getElementById("right");

    // "lego bonsai" zaszyfrowane przesunięciem +11
    const cipherText = "wprz mzydlt";

    let current = [...alphabet];
    let shiftCount = 0;          // 0–25
    let wasSuccess = false;      // żeby klik zagrał tylko przy "wejściu" w +11

    // --- Subtelny klik bez plików audio (Web Audio API) ---
    let audioCtx = null;

    function ensureAudio() {
      if (!audioCtx) {
        audioCtx = new (window.AudioContext || window.webkitAudioContext)();
      }
      // niektóre przeglądarki startują w "suspended" dopóki nie ma interakcji
      if (audioCtx.state === "suspended") audioCtx.resume();
    }

    function playClick() {
      // Krótki "klik": bardzo krótki impuls + szybkie wygaszenie
      ensureAudio();

      const t = audioCtx.currentTime;

      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      const filter = audioCtx.createBiquadFilter();

      // Ustawienia kliknięcia (subtelne)
      osc.type = "square";
      osc.frequency.setValueAtTime(900, t);

      filter.type = "highpass";
      filter.frequency.setValueAtTime(600, t);

      gain.gain.setValueAtTime(0.0001, t);
      gain.gain.exponentialRampToValueAtTime(0.04, t + 0.005); // szybki atak
      gain.gain.exponentialRampToValueAtTime(0.0001, t + 0.06); // szybkie wygaszenie

      osc.connect(filter);
      filter.connect(gain);
      gain.connect(audioCtx.destination);

      osc.start(t);
      osc.stop(t + 0.07);
    }
    // -------------------------------------------------------

    function renderAlpha(el, arr){
      el.textContent = arr.join("");
    }

    function decode(){
      let out = "";
      for(const ch of cipherText){
        if(ch === " "){ out += " "; continue; }
        const idx = alphabet.indexOf(ch);
        out += idx === -1 ? ch : current[idx];
      }
      decodedEl.textContent = out.toUpperCase();

      const isSuccess = shiftCount === 11;

      // trigger wizualny
      decodedEl.classList.toggle("success", isSuccess);
      statusEl.classList.toggle("show", isSuccess);

      // trigger dźwiękowy tylko przy wejściu w sukces
      if (isSuccess && !wasSuccess) {
        playClick();
      }
      wasSuccess = isSuccess;
    }

    function shiftAlphabet(dir){
      if(dir === 1){
        current.unshift(current.pop());
        shiftCount = (shiftCount + 1) % 26;
      } else {
        current.push(current.shift());
        shiftCount = (shiftCount + 25) % 26;
      }
      renderAlpha(shiftedEl, current);
      decode();
    }

    // init
    renderAlpha(staticEl, alphabet);
    renderAlpha(shiftedEl, current);
    decode();

    // Interakcje (zapewniają "user gesture" dla audio)
    leftBtn.addEventListener("click", () => shiftAlphabet(-1));
    rightBtn.addEventListener("click", () => shiftAlphabet(1));

    window.addEventListener("keydown", e => {
      if(e.key === "ArrowLeft")  shiftAlphabet(-1);
      if(e.key === "ArrowRight") shiftAlphabet(1);
    });
  </script>
</body>
</html>
