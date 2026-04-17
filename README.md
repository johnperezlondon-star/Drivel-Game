# Drivel-Game
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Drivel Dodge</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { background: #0d0d1a; overflow: hidden; font-family: 'Courier New', monospace; }
canvas { display: block; }
#ui { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; flex-direction: column; pointer-events: none; }
#menu, #gameover { background: rgba(10,10,30,0.93); border: 2px solid #7f77dd; border-radius: 16px; padding: 32px 40px; text-align: center; color: #fff; max-width: 420px; width: 90%; pointer-events: all; }
#gameover { display: none; }
h1 { font-size: 30px; color: #afa9ec; margin-bottom: 4px; letter-spacing: 2px; }
.sub { font-size: 12px; color: #777; margin-bottom: 24px; }
.chars { display: flex; gap: 10px; flex-wrap: wrap; justify-content: center; margin-bottom: 20px; }
.char-btn { background: #26215c; border: 2px solid #534ab7; border-radius: 10px; padding: 10px 14px; cursor: pointer; color: #fff; font-size: 13px; font-family: monospace; transition: all 0.15s; }
.char-btn:hover, .char-btn.sel { background: #534ab7; border-color: #afa9ec; transform: scale(1.06); }
.char-icon { font-size: 24px; display: block; margin-bottom: 4px; }
.start-btn { background: #534ab7; border: none; border-radius: 10px; padding: 13px 32px; color: #fff; font-size: 15px; font-family: monospace; cursor: pointer; width: 100%; margin-top: 8px; letter-spacing: 1px; }
.start-btn:hover { background: #7f77dd; }
#hud { position: absolute; top: 14px; left: 0; width: 100%; display: none; justify-content: space-between; padding: 0 20px; color: #afa9ec; font-family: monospace; font-size: 14px; pointer-events: none; }
#controls-hint { position: absolute; bottom: 12px; left: 50%; transform: translateX(-50%); color: #444; font-family: monospace; font-size: 11px; pointer-events: none; display: none; white-space: nowrap; }
#score-go { font-size: 36px; color: #afa9ec; margin: 10px 0 4px; font-weight: bold; }
#go-name { color: #afa9ec; }
#go-msg { margin-top: 12px; font-size: 12px; color: #7f77dd; font-style: italic; }
#play-again { background: #534ab7; border: none; border-radius: 10px; padding: 11px 28px; color: #fff; font-size: 15px; font-family: monospace; cursor: pointer; margin-top: 16px; }
#play-again:hover { background: #7f77dd; }
</style>
</head>
<body>
<canvas id="c"></canvas>

<div id="ui">
  <div id="menu">
    <h1>DRIVEL DODGE</h1>
    <p class="sub">Escape the discourse. Survive the chat.</p>
    <div class="chars">
      <div class="char-btn sel" data-id="0"><span class="char-icon">🧢</span>Jaime</div>
      <div class="char-btn" data-id="1"><span class="char-icon">🎧</span>Vik</div>
      <div class="char-btn" data-id="2"><span class="char-icon">🍺</span>Adam</div>
      <div class="char-btn" data-id="3"><span class="char-icon">☕</span>John</div>
      <div class="char-btn" data-id="4"><span class="char-icon">🕶️</span>Vim</div>
    </div>
    <button class="start-btn" id="startBtn">🎧 PLUG IN & PLAY</button>
  </div>
  <div id="gameover">
    <p style="font-size:13px;color:#888;">The Drivel got <span id="go-name"></span>...</p>
    <div id="score-go">0</div>
    <p style="font-size:13px;color:#aaa;">points of sanity maintained</p>
    <p id="go-msg"></p>
    <button id="play-again">TRY AGAIN</button>
  </div>
</div>

<div id="hud">
  <span>SCORE: <span id="sc">0</span></span>
  <span id="cur-player">JAIME</span>
  <span>LIVES: <span id="lv">3</span></span>
</div>
<div id="controls-hint">⬅ ➡ to move &bull; SPACE / ⬆ to jump (x2) &bull; jump ON Dr Ivel to hurt him</div>

<script>
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');

const CHARS = [
  {name:'Jaime', icon:'🧢', color:'#7f77dd'},
  {name:'Vik',   icon:'🎧', color:'#1d9e75'},
  {name:'Adam',  icon:'🍺', color:'#d85a30'},
  {name:'John',  icon:'☕', color:'#ba7517'},
  {name:'Vim',   icon:'🕶️', color:'#d4537e'},
];

const DRIVEL_WORDS = [
  'DRIVEL','DRIVEL','DRIVEL','PUTIN','TRUMP','NUCLEAR WAR',
  'NUKES','MAGIC STARMER','PERRO SANCHEZ','DRIVEL',
  'POLITICS','CRISIS','NATO','BREXIT','REGIME CHANGE',
  'THE ECONOMY','THIRD WORLD WAR','GLOBALISM','DEEP STATE'
];

const GO_MSGS = [
  "The group chat wins again.",
  "Next time, use Do Not Disturb.",
  "You almost escaped the discourse...",
  "Dr Ivel sends his regards.",
  "Magic Starmer did this.",
  "Perro Sanchez was probably involved.",
  "Should've stayed off WhatsApp.",
  "The nukes got you in the end.",
];

let W, H, selChar = 0, score = 0, lives = 3, gameRunning = false;
let player, platforms, drivels, drIvel;
let frameCount = 0, speed = 2, spawnTimer = 0, drIvelTimer = 0;
let keys = {};

function resize() {
  W = canvas.width = window.innerWidth;
  H = canvas.height = window.innerHeight;
}
resize();
window.addEventListener('resize', () => { resize(); if(!gameRunning) {} });

// Character select
document.querySelectorAll('.char-btn').forEach(b => {
  b.addEventListener('click', () => {
    document.querySelectorAll('.char-btn').forEach(x => x.classList.remove('sel'));
    b.classList.add('sel');
    selChar = +b.dataset.id;
  });
});

// Keyboard
document.addEventListener('keydown', e => {
  keys[e.code] = true;
  if(['Space','ArrowUp','ArrowLeft','ArrowRight','ArrowDown','KeyW','KeyA','KeyD','KeyS'].includes(e.code)) e.preventDefault();
});
document.addEventListener('keyup', e => { keys[e.code] = false; });

// Touch controls
let touchStartX = 0, touchStartY = 0;
canvas.addEventListener('touchstart', e => {
  touchStartX = e.touches[0].clientX;
  touchStartY = e.touches[0].clientY;
}, {passive: true});
canvas.addEventListener('touchmove', e => {
  const dx = e.touches[0].clientX - touchStartX;
  const dy = e.touches[0].clientY - touchStartY;
  keys['ArrowLeft']  = dx < -15;
  keys['ArrowRight'] = dx > 15;
  keys['Space']      = dy < -30;
}, {passive: true});
canvas.addEventListener('touchend', () => {
  keys['ArrowLeft'] = keys['ArrowRight'] = keys['Space'] = false;
});

function buildPlatforms() {
  const g = H - 60;
  return [
    {x: 0,       y: g,       w: W,    h: 60,  color: '#1e1a4a'},
    {x: 80,      y: g - 130, w: 160,  h: 14,  color: '#534ab7'},
    {x: 320,     y: g - 175, w: 140,  h: 14,  color: '#534ab7'},
    {x: W-300,   y: g - 130, w: 160,  h: 14,  color: '#534ab7'},
    {x: W/2 - 80,y: g - 240, w: 160,  h: 14,  color: '#3c3489'},
    {x: 60,      y: g - 300, w: 110,  h: 14,  color: '#534ab7'},
    {x: W - 200, y: g - 280, w: 130,  h: 14,  color: '#534ab7'},
    {x: W/2 - 50,y: g - 380, w: 100,  h: 14,  color: '#26215c'},
  ];
}

function initGame() {
  score = 0; lives = 3; speed = 2; frameCount = 0; spawnTimer = 0; drIvelTimer = 0;
  const g = H - 60;
  player = {
    x: W/2 - 20, y: g - 60,
    w: 40, h: 56,
    vx: 0, vy: 0,
    onGround: false,
    char: CHARS[selChar],
    invincible: 0,
    jumpCount: 0,
  };
  platforms = buildPlatforms();
  drivels = [];
  drIvel = null;
  document.getElementById('hud').style.display = 'flex';
  document.getElementById('controls-hint').style.display = 'block';
  document.getElementById('cur-player').textContent = CHARS[selChar].name.toUpperCase();
  updateHUD();
}

function updateHUD() {
  document.getElementById('sc').textContent = score;
  document.getElementById('lv').textContent = lives;
}

function rectOverlap(a, b) {
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
}

function spawnDrivel() {
  const word = DRIVEL_WORDS[Math.floor(Math.random() * DRIVEL_WORDS.length)];
  const fromLeft = Math.random() < 0.5;
  const y = Math.random() * (H * 0.7) + H * 0.05;
  const spd = speed + Math.random() * 1.8;
  drivels.push({
    x: fromLeft ? -220 : W + 220,
    y,
    vx: fromLeft ? spd : -spd,
    vy: (Math.random() - 0.5) * 0.9,
    word,
    size: word === 'DRIVEL' ? 20 : 13,
    angle: (Math.random() - 0.5) * 0.35,
    isBig: word === 'DRIVEL',
  });
}

const DR_QUIPS = [
  "ACTUALLY...", "WELL IN FACT...", "THE REAL ISSUE IS...",
  "ACCORDING TO MY SOURCES...", "HISTORICALLY SPEAKING...",
  "LET ME EXPLAIN WHY...", "IT'S MORE COMPLICATED...",
  "THE WEST IS TO BLAME!", "HAVE YOU READ CHOMSKY?",
  "THIS IS IMPERIALISM.", "THINK ABOUT THE NUKES!"
];

function spawnDrIvel() {
  const fromLeft = Math.random() < 0.5;
  drIvel = {
    x: fromLeft ? -70 : W + 70,
    y: H - 130,
    vx: fromLeft ? speed * 1.3 + 1.5 : -(speed * 1.3 + 1.5),
    vy: -2,
    phase: 0,
    health: 3,
    hitFlash: 0,
    bubble: DR_QUIPS[Math.floor(Math.random() * DR_QUIPS.length)],
    bubbleTimer: 100,
  };
}

function loseLife() {
  lives--;
  updateHUD();
  if (lives <= 0) { endGame(); return; }
  player.invincible = 100;
  player.x = W / 2 - 20;
  player.y = H - 120;
  player.vx = 0; player.vy = 0;
}

function endGame() {
  gameRunning = false;
  document.getElementById('hud').style.display = 'none';
  document.getElementById('controls-hint').style.display = 'none';
  document.getElementById('gameover').style.display = 'block';
  document.getElementById('go-name').textContent = CHARS[selChar].name;
  document.getElementById('score-go').textContent = score;
  document.getElementById('go-msg').textContent = GO_MSGS[Math.floor(Math.random() * GO_MSGS.length)];
}

function update() {
  if (!gameRunning) return;
  frameCount++;
  score = Math.floor(score + 1 + frameCount / 600);
  speed = 2 + frameCount / 2000;

  // Input
  if (keys['ArrowLeft']  || keys['KeyA']) player.vx = -5;
  else if (keys['ArrowRight'] || keys['KeyD']) player.vx = 5;
  else player.vx *= 0.8;

  if ((keys['Space'] || keys['ArrowUp'] || keys['KeyW']) && player.jumpCount < 2) {
    player.vy = -13.5;
    player.onGround = false;
    player.jumpCount++;
    keys['Space'] = keys['ArrowUp'] = keys['KeyW'] = false;
  }

  player.vy += 0.55;
  player.x += player.vx;
  player.y += player.vy;
  player.x = Math.max(0, Math.min(W - player.w, player.x));

  // Platform collisions
  player.onGround = false;
  for (let p of platforms) {
    if (player.x + player.w > p.x && player.x < p.x + p.w &&
        player.y + player.h > p.y && player.y + player.h < p.y + p.h + 18 && player.vy >= 0) {
      player.y = p.y - player.h;
      player.vy = 0;
      player.onGround = true;
      player.jumpCount = 0;
    }
  }

  if (player.y > H + 120) { loseLife(); return; }
  if (player.invincible > 0) player.invincible--;

  // Spawn drivel words
  spawnTimer++;
  const spawnRate = Math.max(25, 85 - frameCount / 70);
  if (spawnTimer > spawnRate) { spawnDrivel(); spawnTimer = 0; }

  // Drivel word collisions + cleanup
  for (let i = drivels.length - 1; i >= 0; i--) {
    let d = drivels[i];
    d.x += d.vx; d.y += d.vy; d.angle += 0.008;
    if (d.x < -350 || d.x > W + 350 || d.y > H + 150 || d.y < -200) { drivels.splice(i, 1); continue; }
    if (player.invincible === 0) {
      const pr = {x: player.x + 5, y: player.y + 5, w: player.w - 10, h: player.h - 10};
      const dr = {x: d.x - 50, y: d.y - 16, w: 100, h: 32};
      if (rectOverlap(pr, dr)) { drivels.splice(i, 1); loseLife(); return; }
    }
  }

  // Dr Ivel
  drIvelTimer++;
  if (!drIvel && drIvelTimer > 500) { spawnDrIvel(); drIvelTimer = 0; }

  if (drIvel) {
    drIvel.phase++;
    drIvel.vy += 0.5;
    drIvel.x += drIvel.vx;
    drIvel.y += drIvel.vy;
    if (drIvel.bubbleTimer > 0) drIvel.bubbleTimer--;
    if (drIvel.hitFlash > 0) drIvel.hitFlash--;

    // Dr Ivel platform collision
    for (let p of platforms) {
      if (drIvel.x + 50 > p.x && drIvel.x < p.x + p.w &&
          drIvel.y + 70 > p.y && drIvel.y + 70 < p.y + p.h + 18 && drIvel.vy >= 0) {
        drIvel.y = p.y - 70;
        drIvel.vy = 0;
        if (Math.random() < 0.015) drIvel.vy = -11;
      }
    }

    if (drIvel.y > H + 150) { drIvel = null; }

    if (drIvel) {
      const pr = {x: player.x + 5, y: player.y + 5, w: player.w - 10, h: player.h - 10};
      const dr2 = {x: drIvel.x, y: drIvel.y, w: 50, h: 70};

      // Player stomps Dr Ivel
      if (player.vy > 0 && pr.y + pr.h - player.vy <= drIvel.y + 10 && rectOverlap(pr, dr2)) {
        drIvel.health--;
        drIvel.hitFlash = 22;
        drIvel.bubble = DR_QUIPS[Math.floor(Math.random() * DR_QUIPS.length)];
        drIvel.bubbleTimer = 80;
        player.vy = -12;
        score += 150;
        if (drIvel.health <= 0) { score += 500; drIvel = null; }
        return;
      }
      // Dr Ivel touches player
      if (player.invincible === 0 && rectOverlap(pr, dr2)) { loseLife(); return; }
    }
  }

  updateHUD();
}

function drawStar(x, y, r, col) {
  ctx.fillStyle = col;
  ctx.beginPath();
  for (let i = 0; i < 5; i++) {
    const a = -Math.PI / 2 + i * Math.PI * 2 / 5;
    const b = a + Math.PI / 5;
    ctx.lineTo(x + Math.cos(a) * r, y + Math.sin(a) * r);
    ctx.lineTo(x + Math.cos(b) * r * 0.42, y + Math.sin(b) * r * 0.42);
  }
  ctx.closePath(); ctx.fill();
}

function draw() {
  // Background
  ctx.fillStyle = '#0d0d1a';
  ctx.fillRect(0, 0, W, H);

  // Scrolling stars
  ctx.fillStyle = 'rgba(200,200,255,0.35)';
  for (let i = 0; i < 70; i++) {
    const sx = (i * 139 + frameCount * 0.04) % W;
    const sy = (i * 101 + frameCount * 0.015) % H;
    ctx.fillRect(sx, sy, i % 5 === 0 ? 2 : 1, i % 5 === 0 ? 2 : 1);
  }

  // Platforms
  for (let p of platforms) {
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.roundRect(p.x, p.y, p.w, p.h, 5);
    ctx.fill();
    // Top shine
    ctx.fillStyle = 'rgba(175,169,236,0.25)';
    ctx.fillRect(p.x + 4, p.y, p.w - 8, 2);
  }

  // Drivel words
  for (let d of drivels) {
    ctx.save();
    ctx.translate(d.x, d.y);
    ctx.rotate(d.angle);
    const col = d.isBig ? '#e24b4a' : '#7f77dd';
    ctx.shadowColor = col;
    ctx.shadowBlur = 10;
    ctx.fillStyle = col;
    ctx.font = `bold ${d.size}px monospace`;
    ctx.textAlign = 'center';
    ctx.fillText(d.word, 0, 0);
    ctx.shadowBlur = 0;
    ctx.restore();
  }

  // Dr Ivel
  if (drIvel) {
    ctx.save();
    ctx.translate(drIvel.x + 25, drIvel.y);
    if (drIvel.hitFlash > 0 && Math.floor(drIvel.hitFlash / 3) % 2 === 0) ctx.globalAlpha = 0.25;
    const bob = Math.sin(drIvel.phase * 0.09) * 3;

    // Body (lab coat)
    ctx.fillStyle = '#e8e8f4';
    ctx.beginPath(); ctx.roundRect(-18, 22 + bob, 36, 48, 5); ctx.fill();
    // Tie
    ctx.fillStyle = '#a32d2d';
    ctx.fillRect(-3, 28 + bob, 6, 22);

    // Head
    ctx.fillStyle = '#f09595';
    ctx.beginPath(); ctx.arc(0, 10 + bob, 20, 0, Math.PI * 2); ctx.fill();

    // Evil glasses
    ctx.strokeStyle = '#534ab7'; ctx.lineWidth = 2.5;
    ctx.strokeRect(-14, 4 + bob, 11, 9);
    ctx.strokeRect(3, 4 + bob, 11, 9);
    ctx.beginPath(); ctx.moveTo(-3, 8 + bob); ctx.lineTo(3, 8 + bob); ctx.stroke();

    // Eyes
    ctx.fillStyle = '#2c2c2a';
    ctx.beginPath(); ctx.arc(-8, 8 + bob, 3, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.arc(8, 8 + bob, 3, 0, Math.PI * 2); ctx.fill();

    // Mouth (smug)
    ctx.strokeStyle = '#555'; ctx.lineWidth = 1.5;
    ctx.beginPath(); ctx.arc(4, 17 + bob, 5, Math.PI, 0); ctx.stroke();

    // Health stars
    for (let h = 0; h < drIvel.health; h++) {
      drawStar(-16 + h * 16, -18 + bob, 7, '#e24b4a');
    }

    // Name label
    ctx.fillStyle = '#a32d2d';
    ctx.font = 'bold 9px monospace';
    ctx.textAlign = 'center';
    ctx.fillText('DR IVEL', 0, 78 + bob);

    // Speech bubble
    if (drIvel.bubbleTimer > 0) {
      const bw = Math.min(drIvel.bubble.length * 6 + 18, 170);
      ctx.fillStyle = 'rgba(255,255,255,0.93)';
      ctx.beginPath(); ctx.roundRect(-bw / 2, -58 + bob, bw, 26, 6); ctx.fill();
      ctx.fillStyle = '#a32d2d';
      ctx.font = 'bold 9px monospace';
      ctx.textAlign = 'center';
      ctx.fillText(drIvel.bubble, 0, -40 + bob);
    }
    ctx.restore();
  }

  // Player
  if (player) {
    ctx.save();
    if (player.invincible > 0 && Math.floor(player.invincible / 6) % 2 === 0) ctx.globalAlpha = 0.15;
    ctx.translate(player.x + player.w / 2, player.y);
    const bob = player.onGround ? Math.sin(frameCount * 0.22) * 2 : 0;

    // Body
    ctx.fillStyle = player.char.color;
    ctx.beginPath(); ctx.roundRect(-15, 22 + bob, 30, 34, 6); ctx.fill();

    // Head
    ctx.fillStyle = '#f4c0d1';
    ctx.beginPath(); ctx.arc(0, 10 + bob, 18, 0, Math.PI * 2); ctx.fill();

    // Headphones arc
    ctx.strokeStyle = '#1a1a2e'; ctx.lineWidth = 4;
    ctx.beginPath(); ctx.arc(0, 6 + bob, 17, Math.PI, 0); ctx.stroke();
    // Ear cups
    ctx.fillStyle = '#111';
    ctx.beginPath(); ctx.arc(-17, 10 + bob, 5, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.arc(17, 10 + bob, 5, 0, Math.PI * 2); ctx.fill();

    // Closed eyes (blissfully ignoring the drivel)
    ctx.strokeStyle = '#555'; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.arc(-6, 12 + bob, 3.5, 0, Math.PI); ctx.stroke();
    ctx.beginPath(); ctx.arc(6, 12 + bob, 3.5, 0, Math.PI); ctx.stroke();

    // Smile
    ctx.beginPath(); ctx.arc(0, 17 + bob, 5, 0, Math.PI); ctx.stroke();

    // Name
    ctx.fillStyle = 'rgba(255,255,255,0.8)';
    ctx.font = 'bold 10px monospace';
    ctx.textAlign = 'center';
    ctx.fillText(player.char.name.toUpperCase(), 0, 65 + bob);

    ctx.restore();
  }
}

function loop() {
  update();
  draw();
  requestAnimationFrame(loop);
}

document.getElementById('startBtn').addEventListener('click', () => {
  document.getElementById('menu').style.display = 'none';
  gameRunning = true;
  initGame();
});

document.getElementById('play-again').addEventListener('click', () => {
  document.getElementById('gameover').style.display = 'none';
  gameRunning = true;
  initGame();
});

loop();
</script>
</body>
</html>
