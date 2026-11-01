<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
<title>Snake</title>
<style>
  :root {
    --bg: #16222f;
    --board: #1f3347;
    --grid: #25405a;
    --snake: #ffd166;
    --food: #ef476f;
    --text: #e8eef4;
    --muted: #8fa5b8;
  }
  * { box-sizing: border-box; }
  html, body { height: 100%; margin: 0; }
  body {
    background: var(--bg);
    color: var(--text);
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    touch-action: none;
    user-select: none;
  }
  header { display: flex; gap: 28px; align-items: baseline; }
  h1 { margin: 0; font-size: 1.6rem; letter-spacing: .02em; }
  .stat { color: var(--muted); font-size: .95rem; }
  .stat b { color: var(--text); font-size: 1.1rem; margin-left: 4px; }
  #wrap { position: relative; }
  canvas {
    display: block;
    width: min(92vw, 60vh, 420px);
    height: auto;
    aspect-ratio: 1;
    background: var(--board);
    border-radius: 10px;
  }
  #overlay {
    position: absolute; inset: 0;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    gap: 6px; text-align: center;
    background: rgba(22, 34, 47, .82);
    border-radius: 10px;
    cursor: pointer;
  }
  #overlay h2 { margin: 0; font-size: 1.5rem; }
  #overlay p { margin: 0; color: var(--muted); }
  footer { color: var(--muted); font-size: .85rem; }
  @media (prefers-reduced-motion: reduce) { * { transition: none !important; } }
</style>
</head>
<body>
<header>
  <h1>Snake</h1>
  <span class="stat">Score<b id="score">0</b></span>
  <span class="stat">Best<b id="best">0</b></span>
</header>

<div id="wrap">
  <canvas id="game" width="420" height="420"></canvas>
  <div id="overlay">
    <h2 id="title">Snake</h2>
    <p id="msg">Tap or press any key to start</p>
  </div>
</div>

<footer>Arrow keys / WASD, or swipe. Space pauses.</footer>

<script>
(() => {
  const COLS = 21, CELL = 20;
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const scoreEl = document.getElementById('score');
  const bestEl = document.getElementById('best');
  const overlay = document.getElementById('overlay');
  const titleEl = document.getElementById('title');
  const msgEl = document.getElementById('msg');
  const css = n => getComputedStyle(document.documentElement).getPropertyValue(n).trim();

  let snake, dir, nextDir, food, score, timer, state = 'idle';
  let best = 0;
  try { best = +localStorage.getItem('snake-best') || 0; } catch (e) {}
  bestEl.textContent = best;

  function reset() {
    const mid = Math.floor(COLS / 2);
    snake = [{x: mid, y: mid}, {x: mid - 1, y: mid}, {x: mid - 2, y: mid}];
    dir = nextDir = {x: 1, y: 0};
    score = 0;
    scoreEl.textContent = 0;
    placeFood();
    draw();
  }

  function placeFood() {
    do {
      food = {x: Math.floor(Math.random() * COLS), y: Math.floor(Math.random() * COLS)};
    } while (snake.some(s => s.x === food.x && s.y === food.y));
  }

  function speed() { return Math.max(65, 140 - score * 3); }

  function start() {
    reset();
    state = 'running';
    overlay.style.display = 'none';
    schedule();
  }

  function schedule() {
    clearTimeout(timer);
    timer = setTimeout(tick, speed());
  }

  function tick() {
    if (state !== 'running') return;
    dir = nextDir;
    const head = {x: snake[0].x + dir.x, y: snake[0].y + dir.y};
    const hitWall = head.x < 0 || head.y < 0 || head.x >= COLS || head.y >= COLS;
    const hitSelf = snake.some(s => s.x === head.x && s.y === head.y);
    if (hitWall || hitSelf) return gameOver();

    snake.unshift(head);
    if (head.x === food.x && head.y === food.y) {
      score++;
      scoreEl.textContent = score;
      placeFood();
    } else {
      snake.pop();
    }
    draw();
    schedule();
  }

  function gameOver() {
    state = 'over';
    if (score > best) {
      best = score;
      bestEl.textContent = best;
      try { localStorage.setItem('snake-best', best); } catch (e) {}
    }
    titleEl.textContent = 'Game over';
    msgEl.textContent = `Score ${score}. Tap or press any key to retry`;
    overlay.style.display = 'flex';
  }

  function togglePause() {
    if (state === 'running') {
      state = 'paused';
      clearTimeout(timer);
      titleEl.textContent = 'Paused';
      msgEl.textContent = 'Tap or press Space to resume';
      overlay.style.display = 'flex';
    } else if (state === 'paused') {
      state = 'running';
      overlay.style.display = 'none';
      schedule();
    }
  }

  function draw() {
    ctx.fillStyle = css('--board');
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = css('--grid');
    for (let x = 0; x < COLS; x++)
      for (let y = 0; y < COLS; y++)
        if ((x + y) % 2) ctx.fillRect(x * CELL, y * CELL, CELL, CELL);

    ctx.fillStyle = css('--food');
    ctx.beginPath();
    ctx.arc(food.x * CELL + CELL / 2, food.y * CELL + CELL / 2, CELL / 2 - 3, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = css('--snake');
    snake.forEach((s, i) => {
      const pad = i === 0 ? 1 : 2;
      ctx.globalAlpha = i === 0 ? 1 : 0.9;
      ctx.fillRect(s.x * CELL + pad, s.y * CELL + pad, CELL - pad * 2, CELL - pad * 2);
    });
    ctx.globalAlpha = 1;
  }

  function turn(x, y) {
    if (x === -dir.x && y === -dir.y) return; // no reversing
    nextDir = {x, y};
  }

  const keys = {
    ArrowUp: [0, -1], w: [0, -1], ArrowDown: [0, 1], s: [0, 1],
    ArrowLeft: [-1, 0], a: [-1, 0], ArrowRight: [1, 0], d: [1, 0]
  };

  document.addEventListener('keydown', e => {
    if (e.key === ' ') { e.preventDefault(); return state === 'running' || state === 'paused' ? togglePause() : start(); }
    const k = keys[e.key] || keys[e.key.toLowerCase()];
    if (k) {
      e.preventDefault();
      if (state === 'idle' || state === 'over') start();
      turn(k[0], k[1]);
    } else if (state === 'idle' || state === 'over') start();
  });

  overlay.addEventListener('click', () => state === 'paused' ? togglePause() : start());

  let t0 = null;
  document.addEventListener('touchstart', e => {
    const t = e.changedTouches[0];
    t0 = {x: t.clientX, y: t.clientY};
  }, {passive: true});
  document.addEventListener('touchend', e => {
    if (!t0 || state !== 'running') return;
    const t = e.changedTouches[0];
    const dx = t.clientX - t0.x, dy = t.clientY - t0.y;
    if (Math.max(Math.abs(dx), Math.abs(dy)) < 20) return;
    if (Math.abs(dx) > Math.abs(dy)) turn(dx > 0 ? 1 : -1, 0);
    else turn(0, dy > 0 ? 1 : -1);
  }, {passive: true});
  document.addEventListener('touchmove', e => e.preventDefault(), {passive: false});

  reset();
})();
</script>
</body>
</html>
