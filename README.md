# 🌌 Spiral Galaxy

An interactive spiral galaxy made of **12,000 colorful glowing particles** against a deep-black background.

A single self-contained HTML file — no dependencies, no build step. Just open it in a browser.

## Features

- **Smooth rotation** — particles orbit on four logarithmic spiral arms with differential rotation (inner stars orbit faster, like a real galaxy)
- **Mouse attraction** — move the mouse and nearby particles are gently pulled toward the cursor; release and they drift back into the arms
- **Click ripples** — click anywhere to spawn an expanding luminous ripple that pushes particles outward as it passes
- **Three color palettes** — Nebula, Ember, and Aurora, switchable live
- **Compact HUD** — palette switcher and particle-count display
- **Glow rendering** — pre-rendered radial-gradient sprites with additive blending and soft motion trails on a faint starfield

## How to run

Open `index.html` in any modern browser (Edge, Chrome, Firefox), or serve the folder:

```
npx serve .
```

## Controls

| Action | Effect |
| --- | --- |
| Move mouse | Gently attracts nearby particles |
| Release / leave | Particles drift back into the galaxy |
| Click | Expanding luminous ripple |
| Palette buttons (top-left) | Switch color palette |

## Palettes

| Palette | Colors |
| --- | --- |
| Nebula | pink · violet · blue · cyan |
| Ember | gold · orange · red |
| Aurora | green · teal · blue · violet |

## Full source code

Everything lives in [`index.html`](index.html):

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Spiral Galaxy</title>
<style>
  html, body { margin: 0; padding: 0; height: 100%; overflow: hidden; background: #000; }
  #galaxy { position: fixed; inset: 0; display: block; cursor: crosshair; }

  .hud {
    position: fixed; top: 16px; left: 16px; z-index: 10;
    font-family: 'Segoe UI', system-ui, sans-serif; color: #cfd6ff;
    background: rgba(10, 12, 24, 0.55);
    border: 1px solid rgba(120, 140, 255, 0.25);
    border-radius: 12px; padding: 10px 14px;
    backdrop-filter: blur(8px); user-select: none;
  }
  .hud h1 {
    margin: 0 0 8px; font-size: 13px; font-weight: 600;
    letter-spacing: 0.12em; text-transform: uppercase; color: #eef1ff;
  }
  .row { display: flex; align-items: center; gap: 8px; }
  .swatch-btn {
    display: flex; align-items: center; gap: 6px;
    padding: 4px 10px; border-radius: 999px;
    border: 1px solid rgba(255, 255, 255, 0.18);
    background: rgba(255, 255, 255, 0.06); color: #dfe4ff;
    font-size: 12px; font-family: inherit; cursor: pointer;
    transition: background 0.15s, border-color 0.15s;
  }
  .swatch-btn:hover { background: rgba(255, 255, 255, 0.14); }
  .swatch-btn.active { border-color: rgba(255, 255, 255, 0.7); background: rgba(255, 255, 255, 0.16); }
  .dot { width: 10px; height: 10px; border-radius: 50%; flex: none; }
  #count { margin-top: 8px; font-size: 11px; color: #9aa3c9; font-variant-numeric: tabular-nums; }

  .hint {
    position: fixed; bottom: 14px; left: 50%; transform: translateX(-50%);
    font-family: 'Segoe UI', system-ui, sans-serif; font-size: 12px;
    color: rgba(200, 210, 255, 0.55);
    background: rgba(10, 12, 24, 0.45);
    border: 1px solid rgba(120, 140, 255, 0.18);
    border-radius: 999px; padding: 6px 14px;
    backdrop-filter: blur(8px); user-select: none; pointer-events: none;
    white-space: nowrap;
  }
</style>
</head>
<body>
<canvas id="galaxy"></canvas>

<div class="hud">
  <h1>Spiral Galaxy</h1>
  <div class="row" id="palettes"></div>
  <div id="count"></div>
</div>
<div class="hint">Move the mouse to attract particles &middot; Click for a ripple</div>

<script>
'use strict';

const canvas = document.getElementById('galaxy');
const ctx = canvas.getContext('2d');

/* ---------------- palettes ---------------- */
const PALETTES = [
  { name: 'Nebula', colors: ['#ff5ec8', '#c06bff', '#6f8bff', '#43d6ff', '#ffffff'] },
  { name: 'Ember',  colors: ['#ffd166', '#ff9f43', '#ff6b35', '#ff4d4d', '#fff3d6'] },
  { name: 'Aurora', colors: ['#5effa1', '#2ee6c8', '#38b6ff', '#8a6bff', '#eafff5'] },
];

/* ---------------- tuning ---------------- */
const COUNT = 12000;        // particle count
const ARMS = 4;             // spiral arms
const SPRITE = 64;        // glow sprite resolution
const SPRING = 5.0;       // pull-back strength toward the rotating home slot
const DAMP = 2.4;         // velocity decay per second
const MOUSE_R = 150;      // mouse attraction radius (px)
const MOUSE_F = 2400;     // mouse attraction strength (px/s^2)
const RIPPLE_SPEED = 460; // ripple expansion speed (px/s)
const RIPPLE_LIFE = 1.15; // ripple lifetime (s)
const RIPPLE_POWER = 3000; // ripple push strength (px/s^2)
const RIPPLE_BAND = 90;   // ripple band width (px)

/* ---------------- state ---------------- */
let W = 0, H = 0, DPR = 1, R = 0;
let t = 0, last = performance.now();
let palette = PALETTES[0];
let sprites = [], coreSprite = null, starCanvas = null;
const ripples = [];
const mouse = { x: -1e4, y: -1e4, active: false };

const x = new Float32Array(COUNT), y = new Float32Array(COUNT);
const vx = new Float32Array(COUNT), vy = new Float32Array(COUNT);
const homeR = new Float32Array(COUNT), homeA = new Float32Array(COUNT);
const omega = new Float32Array(COUNT);
const size = new Float32Array(COUNT), alpha = new Float32Array(COUNT);
const colorIdx = new Uint8Array(COUNT);

/* ---------------- helpers ---------------- */
function hexToRgba(hex, a) {
  const n = parseInt(hex.slice(1), 16);
  return 'rgba(' + ((n >> 16) & 255) + ',' + ((n >> 8) & 255) + ',' + (n & 255) + ',' + a + ')';
}

function makeSprite(color) {
  const c = document.createElement('canvas');
  c.width = c.height = SPRITE;
  const g = c.getContext('2d');
  const grad = g.createRadialGradient(32, 32, 0, 32, 32, 32);
  grad.addColorStop(0, 'rgba(255,255,255,0.95)');
  grad.addColorStop(0.22, hexToRgba(color, 0.9));
  grad.addColorStop(0.55, hexToRgba(color, 0.30));
  grad.addColorStop(1, hexToRgba(color, 0));
  g.fillStyle = grad;
  g.fillRect(0, 0, SPRITE, SPRITE);
  return c;
}

function buildSprites() {
  sprites = palette.colors.map(makeSprite);
  coreSprite = makeSprite(palette.colors[3]);
}

function gauss() { // cheap approx. normal in [-1, 1]
  return (Math.random() + Math.random() + Math.random() - 1.5) / 1.5;
}

/* ---------------- galaxy construction ---------------- */
function buildParticles() {
  R = Math.min(W, H) * 0.46;
  for (let i = 0; i < COUNT; i++) {
    const arm = i % ARMS;
    const r = R * Math.pow(Math.random(), 0.6); // denser toward the core
    const spread = 0.35 + 0.85 * (r / R);
    homeA[i] = (arm / ARMS) * Math.PI * 2
      + 2.6 * Math.log(1 + r / (R * 0.18))   // logarithmic-spiral twist
      + gauss() * spread;
    homeR[i] = r;
    // differential rotation: inner stars orbit faster
    omega[i] = (0.16 + 0.5 * Math.pow(1 - r / R, 1.15)) * (0.85 + 0.3 * Math.random());
    // white-hot core, colored arms
    colorIdx[i] = (r / R) < 0.16 ? 4 : (Math.random() * 4) | 0;
    size[i] = 2.5 + Math.random() * 4.5;
    alpha[i] = 0.45 + Math.random() * 0.55;
    const a = homeA[i];
    x[i] = W / 2 + Math.cos(a) * r;
    y[i] = H / 2 + Math.sin(a) * r;
    vx[i] = vy[i] = 0;
  }
}

function buildStars() {
  starCanvas = document.createElement('canvas');
  starCanvas.width = Math.max(1, Math.round(W * DPR));
  starCanvas.height = Math.max(1, Math.round(H * DPR));
  const g = starCanvas.getContext('2d');
  g.scale(DPR, DPR);
  const n = Math.round(W * H / 9000);
  for (let i = 0; i < n; i++) {
    g.fillStyle = 'rgba(255,255,255,' + (0.05 + Math.random() * 0.35).toFixed(3) + ')';
    const s = Math.random() < 0.9 ? 1 : 2;
    g.fillRect(Math.random() * W, Math.random() * H, s, s);
  }
}

function resize() {
  DPR = Math.min(2, window.devicePixelRatio || 1);
  W = window.innerWidth; H = window.innerHeight;
  canvas.width = Math.round(W * DPR);
  canvas.height = Math.round(H * DPR);
  canvas.style.width = W + 'px';
  canvas.style.height = H + 'px';
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, W, H);
  buildParticles();
  buildStars();
}

/* ---------------- main loop ---------------- */
function frame(now) {
  const dt = Math.min(0.05, (now - last) / 1000) || 0.016;
  last = now;
  t += dt;

  // fade previous frame -> soft motion trails on deep black
  ctx.globalCompositeOperation = 'source-over';
  ctx.globalAlpha = 1;
  ctx.fillStyle = 'rgba(0,0,0,0.32)';
  ctx.fillRect(0, 0, W, H);

  // static faint starfield for depth
  ctx.drawImage(starCanvas, 0, 0, W, H);

  // advance ripples
  for (let i = ripples.length - 1; i >= 0; i--) {
    const rp = ripples[i];
    rp.age += dt;
    rp.r += RIPPLE_SPEED * dt;
    if (rp.age >= RIPPLE_LIFE) ripples.splice(i, 1);
  }

  const cx = W / 2, cy = H / 2;
  const damp = Math.exp(-DAMP * dt);
  const mr2 = MOUSE_R * MOUSE_R;

  for (let i = 0; i < COUNT; i++) {
    // rotating home slot (smooth differential rotation)
    const a = homeA[i] + omega[i] * t;
    const hx = cx + Math.cos(a) * homeR[i];
    const hy = cy + Math.sin(a) * homeR[i];

    let ax = (hx - x[i]) * SPRING;
    let ay = (hy - y[i]) * SPRING;

    // gentle mouse attraction
    if (mouse.active) {
      const dx = mouse.x - x[i], dy = mouse.y - y[i];
      const d2 = dx * dx + dy * dy;
      if (d2 < mr2) {
        const d = Math.sqrt(d2) || 1;
        const f = MOUSE_F * (1 - d / MOUSE_R);
        ax += (dx / d) * f;
        ay += (dy / d) * f;
      }
    }

    // expanding ripple pushes particles outward as it passes
    for (let j = 0; j < ripples.length; j++) {
      const rp = ripples[j];
      const dx = x[i] - rp.x, dy = y[i] - rp.y;
      const d = Math.sqrt(dx * dx + dy * dy);
      const diff = d - rp.r;
      if (diff > -RIPPLE_BAND && diff < RIPPLE_BAND) {
        const f = RIPPLE_POWER * (1 - Math.abs(diff) / RIPPLE_BAND) * (1 - rp.age / RIPPLE_LIFE);
        ax += (dx / (d || 1)) * f;
        ay += (dy / (d || 1)) * f;
      }
    }

    vx[i] = (vx[i] + ax * dt) * damp;
    vy[i] = (vy[i] + ay * dt) * damp;
    x[i] += vx[i] * dt;
    y[i] += vy[i] * dt;
  }

  // glowing core
  ctx.globalCompositeOperation = 'lighter';
  const coreSize = R * 0.9;
  ctx.globalAlpha = 0.5;
  ctx.drawImage(coreSprite, cx - coreSize / 2, cy - coreSize / 2, coreSize, coreSize);

  // particles (additive glow)
  for (let i = 0; i < COUNT; i++) {
    ctx.globalAlpha = alpha[i];
    const s = size[i];
    ctx.drawImage(sprites[colorIdx[i]], x[i] - s / 2, y[i] - s / 2, s, s);
  }

  // luminous ripple rings
  for (let j = 0; j < ripples.length; j++) {
    const rp = ripples[j];
    const a = 1 - rp.age / RIPPLE_LIFE;
    const inner = Math.max(0, rp.r - 46);
    const grad = ctx.createRadialGradient(rp.x, rp.y, inner, rp.x, rp.y, rp.r + 46);
    grad.addColorStop(0, 'rgba(0,0,0,0)');
    grad.addColorStop(0.5, hexToRgba(palette.colors[3], 0.55 * a));
    grad.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.globalAlpha = 1;
    ctx.fillStyle = grad;
    ctx.beginPath();
    ctx.arc(rp.x, rp.y, rp.r + 46, 0, Math.PI * 2);
    ctx.fill();
  }
  ctx.globalAlpha = 1;

  requestAnimationFrame(frame);
}

/* ---------------- input ---------------- */
canvas.addEventListener('pointermove', e => {
  mouse.x = e.clientX; mouse.y = e.clientY; mouse.active = true;
});
canvas.addEventListener('pointerleave', () => { mouse.active = false; });
canvas.addEventListener('pointerdown', e => {
  mouse.x = e.clientX; mouse.y = e.clientY; mouse.active = true;
  ripples.push({ x: e.clientX, y: e.clientY, r: 0, age: 0 });
  if (ripples.length > 6) ripples.shift();
});

/* ---------------- HUD ---------------- */
const palRow = document.getElementById('palettes');
PALETTES.forEach((p, i) => {
  const b = document.createElement('button');
  b.className = 'swatch-btn' + (i === 0 ? ' active' : '');
  const dot = document.createElement('span');
  dot.className = 'dot';
  dot.style.background = 'linear-gradient(135deg,' + p.colors[0] + ',' + p.colors[2] + ')';
  b.appendChild(dot);
  b.appendChild(document.createTextNode(p.name));
  b.addEventListener('click', () => {
    palette = p;
    buildSprites();
    document.querySelectorAll('.swatch-btn').forEach(el => el.classList.remove('active'));
    b.classList.add('active');
  });
  palRow.appendChild(b);
});
document.getElementById('count').textContent = COUNT.toLocaleString() + ' particles';

/* ---------------- go ---------------- */
window.addEventListener('resize', resize);
resize();
buildSprites();
requestAnimationFrame(frame);
</script>
</body>
</html>
```
