# flappy-js <!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Flappy JS</title>
  <style>
    :root {
      --bg: #0e1726;
      --panel: #151e2f;
      --text: #e6edf3;
      --accent: #3b82f6;
      --muted: #94a3b8;
      --good: #10b981;
      --bad: #ef4444;
      --pipe: #22c55e;
      --pipeDark: #16a34a;
    }
    html, body { height: 100%; }
    body {
      margin: 0;
      background: radial-gradient(1200px 600px at 70% 0%, #132038, var(--bg));
      color: var(--text);
      font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Inter, "Helvetica Neue", Arial;
      display: grid;
      place-items: center;
    }
    .wrap {
      width: min(92vw, 900px);
      aspect-ratio: 16/9;
      display: grid;
      grid-template-rows: 1fr auto;
      gap: .75rem;
    }
    #game {
      width: 100%;
      height: 100%;
      background: linear-gradient(#60a5fa, #38bdf8 60%, #86efac 60%, #86efac);
      border-radius: 16px;
      box-shadow: 0 20px 60px rgba(0,0,0,.4), inset 0 0 0 1px rgba(255,255,255,.1);
      display: block;
    }
    .hud {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: .75rem;
      background: color-mix(in srgb, var(--panel) 88%, transparent);
      border: 1px solid rgba(255,255,255,.08);
      padding: .6rem .8rem;
      border-radius: 12px;
      font-size: 14px;
    }
    .hud .left, .hud .right {
      display: flex;
      align-items: center;
      gap: .75rem;
      flex-wrap: wrap;
    }
    .badge {
      padding: .25rem .5rem;
      background: rgba(255,255,255,.06);
      border: 1px solid rgba(255,255,255,.09);
      border-radius: 999px;
    }
    .btn {
      cursor: pointer;
      padding: .5rem .75rem;
      background: var(--accent);
      color: white;
      border: 0;
      border-radius: 10px;
      box-shadow: 0 6px 16px rgba(59,130,246,.35);
      transition: transform .08s ease;
      font-weight: 600;
    }
    .btn:active { transform: translateY(1px); }
    .ghost { background: rgba(255,255,255,.08); box-shadow: none; color: var(--text); }
    .kbd { font-family: ui-monospace, Menlo, Consolas, monospace; font-size: .9em; padding: .15rem .4rem; border-radius: 6px; background: rgba(255,255,255,.08); border: 1px solid rgba(255,255,255,.12); }
    .overlay {
      position: absolute;
      inset: 0;
      display: grid;
      place-items: center;
      pointer-events: none;
    }
    .card {
      pointer-events: auto;
      background: color-mix(in srgb, var(--panel) 86%, transparent);
      border: 1px solid rgba(255,255,255,.08);
      padding: 18px 20px;
      border-radius: 16px;
      box-shadow: 0 10px 30px rgba(0,0,0,.35);
      text-align: center;
      max-width: min(90%, 520px);
    }
    .card h1 { margin: 6px 0 8px; font-size: clamp(22px, 3vw, 32px); }
    .card p { margin: 0 0 10px; color: var(--muted); }
    .row { display: flex; gap: .5rem; justify-content: center; flex-wrap: wrap; }
    .scoreBig { font-size: clamp(32px, 6vw, 60px); font-weight: 800; letter-spacing: .02em; }
    .sr-only { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0,0,0,0); white-space: nowrap; border: 0; }
  </style>
</head>
<body>
  <div class="wrap">
    <canvas id="game" width="800" height="450" aria-label="Flappy JS Canvas"></canvas>
    <div class="hud" role="status" aria-live="polite">
      <div class="left">
        <span class="badge">Счёт: <strong id="score">0</strong></span>
        <span class="badge">Рекорд: <strong id="best">0</strong></span>
        <span class="badge">Скорость: <strong id="speed">1.0x</strong></span>
      </div>
      <div class="right">
        <button class="btn ghost" id="btn-pause" title="P">Пауза</button>
        <button class="btn ghost" id="btn-mute" title="M">Звук: вкл</button>
        <button class="btn" id="btn-restart" title="R">Заново</button>
      </div>
    </div>
  </div>

  <div class="overlay" id="overlay">
    <div class="card" id="startCard">
      <div class="badge">Flappy JS</div>
      <h1>Нажми <span class="kbd">Пробел</span> / тапни — чтобы взлететь</h1>
      <p>Управление: <span class="kbd">Пробел</span> / <span class="kbd">W</span> / клики / тапы
         · Пауза: <span class="kbd">P</span> · Перезапуск: <span class="kbd">R</span> · Звук: <span class="kbd">M</span></p>
      <div class="row">
        <button class="btn" id="btn-start">Играть</button>
      </div>
    </div>
    <div class="card" id="gameOverCard" style="display:none">
      <div class="badge">Игра окончена</div>
      <div class="scoreBig" id="finalScore">0</div>
      <p id="bestNote" style="display:none;color:var(--good);font-weight:700;">Новый рекорд!</p>
      <div class="row">
        <button class="btn" id="btn-replay">Сыграть ещё</button>
        <button class="btn ghost" id="btn-share">Поделиться</button>
      </div>
    </div>
  </div>

  <script>
  (() => {
    // --- Canvas setup with HiDPI scaling ---
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    function fitHiDPI() {
      const ratio = window.devicePixelRatio || 1;
      const cssW = canvas.clientWidth;
      const cssH = canvas.clientHeight;
      canvas.width = Math.round(cssW * ratio);
      canvas.height = Math.round(cssH * ratio);
      ctx.setTransform(ratio, 0, 0, ratio, 0, 0); // draw in CSS pixels
    }
    new ResizeObserver(fitHiDPI).observe(canvas);
    fitHiDPI();

    // --- DOM/HUD ---
    const $ = (id) => document.getElementById(id);
    const scoreEl = $('score');
    const bestEl = $('best');
    const speedEl = $('speed');
    const overlay = $('overlay');
    const startCard = $('startCard');
    const gameOverCard = $('gameOverCard');
    const finalScoreEl = $('finalScore');
    const bestNote = $('bestNote');

    const btnStart = $('btn-start');
    const btnReplay = $('btn-replay');
    const btnShare = $('btn-share');
    const btnRestart = $('btn-restart');
    const btnPause = $('btn-pause');
    const btnMute = $('btn-mute');

    // --- Settings ---
    const G = 0.38;              // gravity (px/ms^2)
    const FLAP = -6.0;           // vertical impulse
    const PIPE_GAP_MIN = 120;    // min hole size
    const PIPE_GAP_MAX = 180;    // max hole size
    const PIPE_W = 68;           // pipe width
    const PIPE_SPACING = 220;    // horizontal spacing between pipes
    const GROUND_H = 64;         // ground thickness
    const SKY_H = 9999;          // sky cap (unused visually)

    const world = {
      state: 'idle', // 'idle' | 'play' | 'pause' | 'over'
      t: 0,
      speed: 2.1, // px per frame baseline
      score: 0,
      best: Number(localStorage.getItem('flappy-best') || 0),
      muted: false,
    };

    bestEl.textContent = world.best;

    // --- Simple bird ---
    const bird = {
      x: 140,
      y: 200,
      vy: 0,
      r: 16,
      tilt: 0,
      flapCooldown: 0,
    };

    // --- Pipes ---
    let pipes = [];
    function resetPipes() {
      pipes = [];
      // pre-fill a few pipes
      let x = canvas.clientWidth + 100;
      for (let i = 0; i < 5; i++) {
        pipes.push(makePipe(x));
        x += PIPE_SPACING;
      }
    }

    function rand(min, max) { return Math.random() * (max - min) + min; }

    function makePipe(x) {
      const h = canvas.clientHeight;
      const gap = rand(PIPE_GAP_MIN, PIPE_GAP_MAX);
      const margin = 40;
      const topHeight = rand(margin, h - GROUND_H - margin - gap);
      return { x, gap, top: topHeight, scored: false };
    }

    // --- Input ---
    const keys = new Set();
    function flap() {
      if (world.state === 'idle') startGame();
      if (world.state !== 'play') return;
      bird.vy = FLAP;
      bird.flapCooldown = 90; // ms for wing animation
      sfx.flap();
    }

    window.addEventListener('keydown', (e) => {
      if (['Space','KeyW','ArrowUp'].includes(e.code)) { e.preventDefault(); flap(); }
      else if (e.code === 'KeyP') togglePause();
      else if (e.code === 'KeyR') restart();
      else if (e.code === 'KeyM') toggleMute();
    });

    ['pointerdown','touchstart'].forEach(ev => {
      canvas.addEventListener(ev, () => flap());
      overlay.addEventListener(ev, () => flap());
    });

    btnStart.onclick = () => startGame();
    btnReplay.onclick = () => restart();
    btnShare.onclick = async () => {
      const text = `Мой счёт в Flappy JS: ${world.score}!`; 
      try {
        if (navigator.share) await navigator.share({ text, title: 'Flappy JS' });
        else await navigator.clipboard.writeText(text);
        btnShare.textContent = 'Скопировано!';
        setTimeout(() => btnShare.textContent = 'Поделиться', 1200);
      } catch {}
    };
    btnRestart.onclick = () => restart();
    btnPause.onclick = () => togglePause();
    btnMute.onclick = () => toggleMute();

    function toggleMute(){ world.muted = !world.muted; btnMute.textContent = 'Звук: ' + (world.muted ? 'выкл' : 'вкл'); }

    function togglePause(){
      if (world.state === 'play') { world.state = 'pause'; btnPause.textContent = 'Продолжить'; }
      else if (world.state === 'pause') { world.state = 'play'; btnPause.textContent = 'Пауза'; lastTime = performance.now(); requestAnimationFrame(loop); }
    }

    function startGame(){
      overlay.style.display = 'none';
      startCard.style.display = 'none';
      gameOverCard.style.display = 'none';
      world.state = 'play';
      world.t = 0; world.score = 0; world.speed = 2.1;
      bird.x = 140; bird.y = canvas.clientHeight/2; bird.vy = 0; bird.tilt = 0;
      resetPipes();
      sfx.start();
      lastTime = performance.now();
      requestAnimationFrame(loop);
    }

    function gameOver(){
      world.state = 'over';
      sfx.hit();
      overlay.style.display = '';
      startCard.style.display = 'none';
      gameOverCard.style.display = '';
      finalScoreEl.textContent = world.score;
      const isBest = world.score > world.best;
      if (isBest){ world.best = world.score; localStorage.setItem('flappy-best', String(world.best)); }
      bestEl.textContent = world.best;
      bestNote.style.display = isBest ? '' : 'none';
    }

    function restart(){
      overlay.style.display = 'none';
      gameOverCard.style.display = 'none';
      startGame();
    }

    // --- SFX (tiny WebAudio beeps) ---
    const sfx = (() => {
      const ctx = new (window.AudioContext || window.webkitAudioContext || function(){})();
      const can = !!ctx.createOscillator;
      function play(freq=440, dur=0.08, type='sine', gain=0.04){
        if (!can || world.muted) return;
        const o = ctx.createOscillator();
        const g = ctx.createGain();
        o.type = type; o.frequency.value = freq;
        g.gain.value = gain;
        o.connect(g).connect(ctx.destination);
        o.start();
        o.stop(ctx.currentTime + dur);
      }
      return {
        flap(){ play(740, .06, 'square', .05); },
        score(){ play(520, .1, 'triangle', .05); },
        hit(){ play(180, .25, 'sawtooth', .06); },
        start(){ play(420, .12, 'triangle', .05); }
      };
    })();

    // --- Loop ---
    let lastTime = 0;
    function loop(now){
      if (world.state !== 'play') return;
      const dt = Math.min(32, now - lastTime); // clamp frame delta
      lastTime = now;
      update(dt);
      render();
      requestAnimationFrame(loop);
    }

    function update(dt){
      world.t += dt;

      // difficulty scales slowly with time/score
      const speedBase = 2.1 + Math.min(3.0, world.score * 0.02);
      world.speed = speedBase;

      // bird physics
      bird.vy += G; // gravity each ms chunk (tuned)
      bird.y += bird.vy;
      bird.tilt = Math.max(-0.6, Math.min(0.9, bird.vy / 8));
      if (bird.y < 0) { bird.y = 0; bird.vy = 0; }

      // pipes move
      for (const p of pipes){ p.x -= world.speed; }

      // recycle pipes & scoring
      const w = canvas.clientWidth;
      if (pipes.length && pipes[0].x + PIPE_W < -10){ pipes.shift(); const lastX = pipes.at(-1).x; pipes.push(makePipe(lastX + PIPE_SPACING)); }

      for (const p of pipes){
        // score when passing center of pipe
        if (!p.scored && bird.x > p.x + PIPE_W){
          p.scored = true; world.score++; scoreEl.textContent = world.score; sfx.score();
        }
      }

      // ground collision
      const groundY = canvas.clientHeight - GROUND_H - bird.r;
      if (bird.y > groundY) { gameOver(); }

      // pipe collision: circle-rect
      const bx = bird.x, by = bird.y, r = bird.r;
      for (const p of pipes){
        const inX = bx + r > p.x && bx - r < p.x + PIPE_W;
        if (!inX) continue;
        const gapTop = p.top;
        const gapBot = p.top + p.gap;
        if (by - r < gapTop || by + r > gapBot){ gameOver(); break; }
      }

      // HUD speed
      speedEl.textContent = (world.speed).toFixed(1) + 'x';
    }

    function render(){
      const w = canvas.clientWidth, h = canvas.clientHeight;
      ctx.clearRect(0,0,w,h);

      // sky gradient already in CSS, but draw parallax clouds
      drawClouds();

      // pipes
      for (const p of pipes){ drawPipe(p); }

      // bird
      drawBird();

      // ground
      drawGround();

      // floating score (big)
      ctx.save();
      ctx.globalAlpha = 0.2;
      ctx.font = '700 80px Inter, Arial, sans-serif';
      ctx.textAlign = 'center';
      ctx.fillStyle = '#000';
      ctx.fillText(world.score, w/2 + 2, 120 + 2);
      ctx.globalAlpha = 1;
      ctx.fillStyle = 'white';
      ctx.fillText(world.score, w/2, 120);
      ctx.restore();
    }

    function drawClouds(){
      const w = canvas.clientWidth, h = canvas.clientHeight;
      const t = world.t * 0.02;
      ctx.save();
      for (let i=0;i<5;i++){
        const y = 40 + i*20;
        const x = (w - ((t + i*120) % (w+200))) - 100;
        cloud(x, y + (i%2?8:-8), 22 + i*4);
      }
      ctx.restore();
    }
    function cloud(x,y,r){
      ctx.fillStyle = 'rgba(255,255,255,.85)';
      ellipse(x, y, r*1.2, r*.8);
      ellipse(x+22, y-4, r*.9, r*.7);
      ellipse(x-18, y-6, r*.7, r*.6);
      ctx.fill();
    }

    function drawPipe(p){
      const h = canvas.clientHeight;
      const topH = p.top;
      const gap = p.gap;
      ctx.save();
      ctx.translate(p.x, 0);
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--pipe');
      ctx.strokeStyle = 'rgba(0,0,0,.25)';

      // top pipe
      ctx.fillRect(0, 0, PIPE_W, topH);
      // lip cap
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--pipeDark');
      ctx.fillRect(-4, topH-18, PIPE_W+8, 18);

      // bottom pipe
      const bottomY = topH + gap;
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--pipe');
      ctx.fillRect(0, bottomY, PIPE_W, h - bottomY - GROUND_H);
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--pipeDark');
      ctx.fillRect(-4, bottomY, PIPE_W+8, 18);

      // subtle shading
      ctx.fillStyle = 'rgba(255,255,255,.15)';
      ctx.fillRect(6, 0, 8, topH);
      ctx.fillRect(6, bottomY, 8, h - bottomY - GROUND_H);

      ctx.restore();
    }

    function drawBird(){
      ctx.save();
      ctx.translate(bird.x, bird.y);
      ctx.rotate(bird.tilt);

      // body
      ctx.fillStyle = '#f59e0b';
      ellipse(0, 0, bird.r*1.1, bird.r*0.9);
      ctx.fill();

      // belly
      ctx.fillStyle = '#fde68a';
      ellipse(-3, 4, bird.r*.8, bird.r*.55); ctx.fill();

      // eye
      ctx.fillStyle = 'white'; ellipse(6, -6, 7, 7); ctx.fill();
      ctx.fillStyle = '#0f172a';
      ellipse(8, -6, 3, 3); ctx.fill();

      // beak
      ctx.fillStyle = '#fb923c';
      tri(16, -2, 28, 2, 16, 6);

      // wing (flap)
      const flapT = Math.max(0, bird.flapCooldown -= 16);
      const wingUp = flapT > 0;
      ctx.save();
      ctx.rotate(wingUp ? -0.8 : -0.2);
      ctx.fillStyle = '#fbbf24';
      ellipse(-8, -4, 12, 8); ctx.fill();
      ctx.restore();

      ctx.restore();
    }

    function drawGround(){
      const w = canvas.clientWidth, h = canvas.clientHeight;
      const y = h - GROUND_H;
      ctx.save();
      // dirt
      const grd = ctx.createLinearGradient(0, y, 0, h);
      grd.addColorStop(0, '#16a34a');
      grd.addColorStop(1, '#166534');
      ctx.fillStyle = grd;
      ctx.fillRect(0, y, w, GROUND_H);

      // scrolling stripes
      const t = world.t * 0.3;
      ctx.fillStyle = 'rgba(255,255,255,.15)';
      for (let i=0;i<10;i++){
        const x = (i*120 - (t % 120));
        ctx.fillRect(x, y+8, 60, 6);
      }
      ctx.restore();
    }

    // helpers
    function ellipse(x,y,rx,ry){
      ctx.beginPath(); ctx.ellipse(x,y,rx,ry,0,0,Math.PI*2); }
    function tri(x1,y1,x2,y2,x3,y3){ ctx.beginPath(); ctx.moveTo(x1,y1); ctx.lineTo(x2,y2); ctx.lineTo(x3,y3); ctx.closePath(); ctx.fill(); }

    // show start screen initially
    overlay.style.display = '';
  })();
  </script>
</body>
</html>
