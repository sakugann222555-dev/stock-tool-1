<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
  <title>Dodge — Web Game</title>
  <style>
    :root { color-scheme: dark; }
    html, body { margin:0; height:100%; background:#0b0f19; font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
    #wrap { height:100%; display:flex; align-items:center; justify-content:center; }
    canvas { background: radial-gradient(1200px 800px at 50% 30%, #141b2d, #070a12); border:1px solid #26304b; border-radius:14px; touch-action:none; }
    .hud {
      position: fixed; inset: 12px 12px auto 12px;
      display:flex; gap:12px; align-items:center; justify-content:space-between;
      pointer-events:none;
    }
    .pill{
      padding:10px 12px; border-radius:999px; border:1px solid #26304b;
      background: rgba(10,14,24,.6); backdrop-filter: blur(6px);
      color:#dbe6ff; font-weight:600; letter-spacing:.2px;
      display:flex; gap:10px; align-items:center;
    }
    .sub { opacity:.8; font-weight:500; }
    .help {
      position: fixed; inset: auto 12px 12px 12px;
      color:#bcd0ff; opacity:.85; font-size:12px; line-height:1.35;
      text-align:center;
    }
    .help kbd { border:1px solid #26304b; border-bottom-width:2px; padding:1px 6px; border-radius:6px; background: rgba(10,14,24,.6); }
  </style>
</head>
<body>
  <div class="hud">
    <div class="pill"><span>Score</span><span id="score">0</span></div>
    <div class="pill sub" id="status">Playing</div>
  </div>

  <div id="wrap">
    <canvas id="game" width="420" height="720" aria-label="Dodge game"></canvas>
  </div>

  <div class="help">
    PC: <kbd>←</kbd><kbd>→</kbd><kbd>↑</kbd><kbd>↓</kbd> / <kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd>　
    Mobile: タッチしてドラッグ　
    Restart: <kbd>R</kbd> または 画面タップ
  </div>

<script>
(() => {
  const canvas = document.getElementById("game");
  const ctx = canvas.getContext("2d");
  const scoreEl = document.getElementById("score");
  const statusEl = document.getElementById("status");

  // --- Responsive canvas (keep aspect) ---
  function fitCanvas() {
    const maxW = Math.min(window.innerWidth - 24, 520);
    const maxH = Math.min(window.innerHeight - 120, 860);
    const aspect = 420 / 720;
    let w = maxW, h = w / aspect;
    if (h > maxH) { h = maxH; w = h * aspect; }
    canvas.style.width = w + "px";
    canvas.style.height = h + "px";
  }
  window.addEventListener("resize", fitCanvas, { passive:true });
  fitCanvas();

  // --- Game state ---
  const W = canvas.width, H = canvas.height;
  const rng = (a,b)=> a + Math.random()*(b-a);
  const clamp = (v,a,b)=> Math.max(a, Math.min(b, v));

  const player = { x: W/2, y: H*0.78, r: 16, vx: 0, vy: 0, speed: 420 };
  let enemies = [];
  let particles = [];
  let score = 0;
  let best = Number(localStorage.getItem("dodge_best") || 0);
  let alive = true;
  let last = performance.now();
  let spawnT = 0;
  let difficulty = 1;

  // --- Input ---
  const keys = new Set();
  window.addEventListener("keydown", (e) => {
    if (["ArrowLeft","ArrowRight","ArrowUp","ArrowDown","KeyW","KeyA","KeyS","KeyD","KeyR","Space"].includes(e.code)) e.preventDefault();
    keys.add(e.code);
    if ((e.code === "KeyR" || e.code === "Space") && !alive) reset();
  }, { passive:false });

  window.addEventListener("keyup", (e) => keys.delete(e.code), { passive:true });

  let dragging = false;
  let dragOffset = { x:0, y:0 };
  canvas.addEventListener("pointerdown", (e) => {
    canvas.setPointerCapture(e.pointerId);
    const p = toGameCoords(e);
    // tap to restart when dead
    if (!alive) { reset(); return; }
    // start drag if near player
    const dx = p.x - player.x, dy = p.y - player.y;
    if (dx*dx + dy*dy <= (player.r*3)**2) {
      dragging = true;
      dragOffset.x = dx; dragOffset.y = dy;
    } else {
      // quick move target: nudge toward tap
      player.x = clamp(p.x, player.r, W-player.r);
      player.y = clamp(p.y, player.r, H-player.r);
    }
  }, { passive:true });

  canvas.addEventListener("pointermove", (e) => {
    if (!dragging || !alive) return;
    const p = toGameCoords(e);
    player.x = clamp(p.x - dragOffset.x, player.r, W-player.r);
    player.y = clamp(p.y - dragOffset.y, player.r, H-player.r);
  }, { passive:true });

  canvas.addEventListener("pointerup", () => dragging = false, { passive:true });
  canvas.addEventListener("pointercancel", () => dragging = false, { passive:true });

  function toGameCoords(e) {
    const rect = canvas.getBoundingClientRect();
    const sx = W / rect.width;
    const sy = H / rect.height;
    return { x: (e.clientX - rect.left) * sx, y: (e.clientY - rect.top) * sy };
  }

  // --- Helpers ---
  function addEnemy() {
    const size = rng(18, 34);
    const x = rng(size, W - size);
    const y = -size - 10;
    const spd = rng(160, 260) * (1 + difficulty*0.12);
    const drift = rng(-40, 40) * (1 + difficulty*0.08);
    enemies.push({ x, y, r: size, vy: spd, vx: drift });
  }

  function explode(x, y, n=22) {
    for (let i=0;i<n;i++){
      const a = rng(0, Math.PI*2);
      const s = rng(60, 280);
      particles.push({ x, y, vx: Math.cos(a)*s, vy: Math.sin(a)*s, life:rng(0.35,0.8) });
    }
  }

  function reset() {
    enemies = [];
    particles = [];
    score = 0;
    alive = true;
    last = performance.now();
    spawnT = 0;
    difficulty = 1;
    player.x = W/2; player.y = H*0.78;
    statusEl.textContent = "Playing";
  }

  // --- Game loop ---
  function step(now) {
    const dt = Math.min(0.033, (now - last) / 1000);
    last = now;

    // Difficulty ramps with time
    if (alive) {
      score += dt * 10;
      difficulty = 1 + score / 120; // slowly increases
      scoreEl.textContent = Math.floor(score);
    }

    update(dt);
    render();

    requestAnimationFrame(step);
  }

  function update(dt) {
    // Player keyboard move (only when not dragging)
    if (alive && !dragging) {
      let ax = 0, ay = 0;
      if (keys.has("ArrowLeft") || keys.has("KeyA")) ax -= 1;
      if (keys.has("ArrowRight") || keys.has("KeyD")) ax += 1;
      if (keys.has("ArrowUp") || keys.has("KeyW")) ay -= 1;
      if (keys.has("ArrowDown") || keys.has("KeyS")) ay += 1;

      const len = Math.hypot(ax, ay) || 1;
      ax /= len; ay /= len;

      player.x = clamp(player.x + ax * player.speed * dt, player.r, W-player.r);
      player.y = clamp(player.y + ay * player.speed * dt, player.r, H-player.r);
    }

    // Spawn enemies
    if (alive) {
      spawnT -= dt;
      const interval = clamp(0.75 - difficulty*0.06, 0.22, 0.75);
      while (spawnT <= 0) {
        addEnemy();
        spawnT += interval * rng(0.75, 1.25);
      }
    }

    // Move enemies
    for (const e of enemies) {
      e.x += e.vx * dt;
      e.y += e.vy * dt;
      // bounce off walls a bit
      if (e.x < e.r) { e.x = e.r; e.vx *= -1; }
      if (e.x > W - e.r) { e.x = W - e.r; e.vx *= -1; }
    }
    enemies = enemies.filter(e => e.y < H + e.r + 60);

    // Collisions
    if (alive) {
      for (const e of enemies) {
        const dx = e.x - player.x, dy = e.y - player.y;
        const rr = e.r + player.r;
        if (dx*dx + dy*dy < rr*rr) {
          alive = false;
          statusEl.textContent = "Game Over — tap or R to restart";
          explode(player.x, player.y, 34);
          best = Math.max(best, Math.floor(score));
          localStorage.setItem("dodge_best", String(best));
          break;
        }
      }
    }

    // Particles
    for (const p of particles) {
      p.x += p.vx * dt;
      p.y += p.vy * dt;
      p.vx *= 0.98;
      p.vy *= 0.98;
      p.life -= dt;
    }
    particles = particles.filter(p => p.life > 0);
  }

  function render() {
    ctx.clearRect(0,0,W,H);

    // subtle stars
    ctx.save();
    ctx.globalAlpha = 0.25;
    for (let i=0;i<40;i++){
      const x = (i*97 % W);
      const y = (i*151 % H);
      ctx.fillStyle = "#dbe6ff";
      ctx.fillRect(x, y, 1, 1);
    }
    ctx.restore();

    // Arena border glow
    ctx.save();
    ctx.strokeStyle = "rgba(120,160,255,0.25)";
    ctx.lineWidth = 2;
    ctx.strokeRect(10,10,W-20,H-20);
    ctx.restore();

    // Enemies
    for (const e of enemies) {
      ctx.save();
      ctx.fillStyle = "rgba(255,80,90,0.92)";
      ctx.beginPath();
      ctx.arc(e.x, e.y, e.r, 0, Math.PI*2);
      ctx.fill();

      // highlight
      ctx.globalAlpha = 0.25;
      ctx.fillStyle = "#ffffff";
      ctx.beginPath();
      ctx.arc(e.x - e.r*0.25, e.y - e.r*0.25, e.r*0.35, 0, Math.PI*2);
      ctx.fill();
      ctx.restore();
    }

    // Player
    ctx.save();
    ctx.fillStyle = alive ? "rgba(60,160,255,0.95)" : "rgba(120,160,255,0.35)";
    ctx.beginPath();
    ctx.arc(player.x, player.y, player.r, 0, Math.PI*2);
    ctx.fill();

    // player ring
    ctx.strokeStyle = "rgba(200,230,255,0.55)";
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.arc(player.x, player.y, player.r+6, 0, Math.PI*2);
    ctx.stroke();
    ctx.restore();

    // Particles
    for (const p of particles) {
      ctx.save();
      ctx.globalAlpha = Math.max(0, Math.min(1, p.life / 0.8));
      ctx.fillStyle = "rgba(220,240,255,0.9)";
      ctx.fillRect(p.x, p.y, 2, 2);
      ctx.restore();
    }

    // Text
    ctx.save();
    ctx.fillStyle = "rgba(219,230,255,0.85)";
    ctx.font = "600 18px system-ui, -apple-system, Segoe UI, Roboto, sans-serif";
    ctx.fillText("BEST: " + best, 18, 40);
    ctx.font = "500 14px system-ui, -apple-system, Segoe UI, Roboto, sans-serif";
    ctx.fillStyle = "rgba(219,230,255,0.65)";
    ctx.fillText("Difficulty: " + difficulty.toFixed(2), 18, 62);
    ctx.restore();
  }

  // Start
  requestAnimationFrame(step);
})();
</script>
</body>
</html>
