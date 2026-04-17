<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Drivel Dodge</title>
<style>
html, body { margin: 0; padding: 0; background: #0d0d1a; overflow: hidden; height: 100%; width: 100%; }
canvas { position: absolute; top: 0; left: 0; }
#overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; }
.panel { background: rgba(10,10,40,0.95); border: 2px solid #7f77dd; border-radius: 14px; padding: 28px 36px; text-align: center; color: #fff; font-family: monospace; max-width: 400px; width: 88%; }
h1 { color: #afa9ec; font-size: 26px; margin: 0 0 4px; letter-spacing: 2px; }
.sub { color: #666; font-size: 11px; margin: 0 0 20px; }
.chars { display: flex; flex-wrap: wrap; gap: 8px; justify-content: center; margin-bottom: 18px; }
.cb { background: #1e1a4a; border: 2px solid #534ab7; border-radius: 8px; padding: 8px 12px; cursor: pointer; color: #ccc; font-size: 12px; font-family: monospace; }
.cb.on { background: #534ab7; color: #fff; border-color: #afa9ec; }
.cb span { display: block; font-size: 20px; margin-bottom: 3px; }
.btn { background: #534ab7; border: none; border-radius: 8px; padding: 12px; color: #fff; font-size: 14px; font-family: monospace; cursor: pointer; width: 100%; margin-top: 6px; letter-spacing: 1px; }
.btn:hover { background: #7f77dd; }
#hud { position: absolute; top: 12px; left: 0; width: 100%; display: none; justify-content: space-between; padding: 0 18px; color: #afa9ec; font-family: monospace; font-size: 13px; pointer-events: none; box-sizing: border-box; }
#hint { position: absolute; bottom: 8px; left: 50%; transform: translateX(-50%); color: #333; font-family: monospace; font-size: 10px; white-space: nowrap; pointer-events: none; display: none; }
</style>
</head>
<body>
<canvas id="c"></canvas>
<div id="hud"><span>SCORE: <b id="sc">0</b></span><span id="pname"></span><span>LIVES: <b id="lv">3</b></span></div>
<div id="hint">Arrows/WASD = move &nbsp;|&nbsp; Space/Up = jump (x2) &nbsp;|&nbsp; Stomp Dr Ivel 3x to beat him</div>
<div id="overlay">
  <div class="panel" id="menuPanel">
    <h1>DRIVEL DODGE</h1>
    <p class="sub">Escape the discourse. Survive the chat.</p>
    <div class="chars">
      <div class="cb on" data-i="0"><span>🧢</span>Jaime</div>
      <div class="cb" data-i="1"><span>🎧</span>Vik</div>
      <div class="cb" data-i="2"><span>🍺</span>Adam</div>
      <div class="cb" data-i="3"><span>☕</span>John</div>
      <div class="cb" data-i="4"><span>🕶️</span>Vim</div>
    </div>
    <button class="btn" id="startBtn">🎧 PLUG IN & PLAY</button>
  </div>
  <div class="panel" id="goPanel" style="display:none">
    <h1 style="font-size:18px;margin-bottom:8px">DRIVEL GOT <span id="goName"></span>...</h1>
    <div id="goScore" style="font-size:40px;color:#afa9ec;font-weight:bold;margin:8px 0"></div>
    <div style="color:#888;font-size:12px">points of sanity maintained</div>
    <div id="goMsg" style="color:#7f77dd;font-size:12px;font-style:italic;margin-top:10px"></div>
    <button class="btn" id="retryBtn" style="margin-top:14px">TRY AGAIN</button>
  </div>
</div>

<script>
var C = document.getElementById('c');
var X = C.getContext('2d');
var W, H;

function resize() {
  W = C.width = window.innerWidth;
  H = C.height = window.innerHeight;
}
resize();
window.addEventListener('resize', resize);

var CHARS = [
  {name:'Jaime', col:'#7f77dd'},
  {name:'Vik',   col:'#1d9e75'},
  {name:'Adam',  col:'#d85a30'},
  {name:'John',  col:'#ba7517'},
  {name:'Vim',   col:'#d4537e'}
];
var WORDS = ['DRIVEL','DRIVEL','DRIVEL','PUTIN','TRUMP','NUCLEAR WAR','NUKES','MAGIC STARMER','PERRO SANCHEZ','POLITICS','CRISIS','NATO','BREXIT','REGIME CHANGE','DEEP STATE','THIRD WORLD WAR'];
var MSGS = ['The group chat wins again.','Next time, use Do Not Disturb.','Dr Ivel sends his regards.','Magic Starmer did this.','Perro Sanchez was probably involved.','Should\'ve stayed off WhatsApp.'];
var QUIPS = ['ACTUALLY...','WELL IN FACT...','THE REAL ISSUE IS...','HISTORICALLY SPEAKING...','HAVE YOU READ CHOMSKY?','THE WEST IS TO BLAME!','THINK ABOUT THE NUKES!','IT\'S IMPERIALISM.'];

var sel = 0, keys = {}, running = false;
var P, plats, words, boss, frame, spd, wTimer, bTimer, score, lives;

document.querySelectorAll('.cb').forEach(function(b) {
  b.addEventListener('click', function() {
    document.querySelectorAll('.cb').forEach(function(x){ x.classList.remove('on'); });
    b.classList.add('on');
    sel = +b.dataset.i;
  });
});

window.addEventListener('keydown', function(e) {
  keys[e.code] = true;
  if('Space ArrowUp ArrowDown ArrowLeft ArrowRight KeyW KeyA KeyS KeyD'.indexOf(e.code) > -1) e.preventDefault();
});
window.addEventListener('keyup', function(e) { keys[e.code] = false; });

var tx = 0, ty = 0;
C.addEventListener('touchstart', function(e){ tx = e.touches[0].clientX; ty = e.touches[0].clientY; }, {passive:true});
C.addEventListener('touchmove', function(e){
  var dx = e.touches[0].clientX - tx, dy = e.touches[0].clientY - ty;
  keys['ArrowLeft'] = dx < -15; keys['ArrowRight'] = dx > 15; keys['Space'] = dy < -30;
}, {passive:true});
C.addEventListener('touchend', function(){ keys['ArrowLeft'] = keys['ArrowRight'] = keys['Space'] = false; });

function makePlats() {
  var g = H - 55;
  return [
    {x:0,       y:g,      w:W,    h:55},
    {x:70,      y:g-125,  w:150,  h:12},
    {x:300,     y:g-170,  w:130,  h:12},
    {x:W-280,   y:g-125,  w:150,  h:12},
    {x:W/2-75,  y:g-235,  w:150,  h:12},
    {x:50,      y:g-295,  w:110,  h:12},
    {x:W-195,   y:g-270,  w:125,  h:12},
    {x:W/2-55,  y:g-370,  w:110,  h:12}
  ];
}

function startGame() {
  frame = 0; spd = 2; wTimer = 0; bTimer = 0; score = 0; lives = 3;
  plats = makePlats();
  words = []; boss = null;
  var g = H - 55;
  P = {x:W/2-18, y:g-60, w:36, h:54, vx:0, vy:0, grnd:false, inv:0, jc:0, char:CHARS[sel]};
  document.getElementById('hud').style.display = 'flex';
  document.getElementById('hint').style.display = 'block';
  document.getElementById('pname').textContent = CHARS[sel].name.toUpperCase();
  document.getElementById('sc').textContent = '0';
  document.getElementById('lv').textContent = '3';
  running = true;
}

function hit() {
  lives--;
  document.getElementById('lv').textContent = lives;
  if(lives <= 0) { gameOver(); return; }
  P.inv = 100; P.x = W/2-18; P.y = H-120; P.vx = 0; P.vy = 0;
}

function gameOver() {
  running = false;
  document.getElementById('hud').style.display = 'none';
  document.getElementById('hint').style.display = 'none';
  document.getElementById('overlay').style.display = 'flex';
  document.getElementById('menuPanel').style.display = 'none';
  document.getElementById('goPanel').style.display = 'block';
  document.getElementById('goName').textContent = CHARS[sel].name;
  document.getElementById('goScore').textContent = score;
  document.getElementById('goMsg').textContent = MSGS[Math.floor(Math.random()*MSGS.length)];
}

function ox(a,b){ return a.x < b.x+b.w && a.x+a.w > b.x && a.y < b.y+b.h && a.y+a.h > b.y; }

function update() {
  if(!running) return;
  frame++; score += 1 + Math.floor(frame/600); spd = 2 + frame/2000;

  if(keys['ArrowLeft']||keys['KeyA']) P.vx = -5;
  else if(keys['ArrowRight']||keys['KeyD']) P.vx = 5;
  else P.vx *= 0.78;

  if((keys['Space']||keys['ArrowUp']||keys['KeyW']) && P.jc < 2) {
    P.vy = -13; P.grnd = false; P.jc++;
    keys['Space'] = keys['ArrowUp'] = keys['KeyW'] = false;
  }

  P.vy = Math.min(P.vy + 0.55, 18);
  P.x += P.vx; P.y += P.vy;
  P.x = Math.max(0, Math.min(W-P.w, P.x));

  P.grnd = false;
  for(var i=0;i<plats.length;i++){
    var pl=plats[i];
    if(P.x+P.w>pl.x && P.x<pl.x+pl.w && P.vy>=0 && P.y+P.h>pl.y && P.y+P.h<pl.y+pl.h+20){
      P.y=pl.y-P.h; P.vy=0; P.grnd=true; P.jc=0;
    }
  }
  if(P.y > H+100){ hit(); return; }
  if(P.inv>0) P.inv--;

  wTimer++;
  if(wTimer > Math.max(22, 80-frame/65)){ wTimer=0; spawnWord(); }

  for(var i=words.length-1;i>=0;i--){
    var w=words[i];
    w.x+=w.vx; w.y+=w.vy; w.a+=0.008;
    if(w.x<-300||w.x>W+300||w.y>H+100||w.y<-150){ words.splice(i,1); continue; }
    if(P.inv===0){
      var pr={x:P.x+5,y:P.y+5,w:P.w-10,h:P.h-10};
      var wr={x:w.x-55,y:w.y-18,w:110,h:36};
      if(ox(pr,wr)){ words.splice(i,1); hit(); return; }
    }
  }

  bTimer++;
  if(!boss && bTimer>480){ bTimer=0; spawnBoss(); }

  if(boss){
    boss.phase++; boss.vy=Math.min(boss.vy+0.5,18); boss.x+=boss.vx; boss.y+=boss.vy;
    if(boss.bt>0) boss.bt--;
    if(boss.hf>0) boss.hf--;
    for(var i=0;i<plats.length;i++){
      var pl=plats[i];
      if(boss.x+50>pl.x && boss.x<pl.x+pl.w && boss.vy>=0 && boss.y+70>pl.y && boss.y+70<pl.y+pl.h+20){
        boss.y=pl.y-70; boss.vy=0;
        if(Math.random()<0.012) boss.vy=-11;
      }
    }
    if(boss.y>H+150){ boss=null; return; }
    var pr={x:P.x+5,y:P.y+5,w:P.w-10,h:P.h-10};
    var br={x:boss.x,y:boss.y,w:50,h:70};
    if(P.vy>0 && pr.y+pr.h-P.vy<=boss.y+12 && ox(pr,br)){
      boss.hp--; boss.hf=24; P.vy=-12; score+=150;
      boss.bubble=QUIPS[Math.floor(Math.random()*QUIPS.length)]; boss.bt=90;
      if(boss.hp<=0){ score+=500; boss=null; return; }
      return;
    }
    if(P.inv===0 && ox(pr,br)){ hit(); return; }
  }

  document.getElementById('sc').textContent = score;
}

function spawnWord(){
  var word=WORDS[Math.floor(Math.random()*WORDS.length)];
  var left=Math.random()<0.5, s=spd+Math.random()*1.8;
  words.push({x:left?-200:W+200, y:Math.random()*(H*0.72)+H*0.05, vx:left?s:-s, vy:(Math.random()-0.5)*0.8, a:(Math.random()-0.5)*0.3, word:word, big:word==='DRIVEL'});
}

function spawnBoss(){
  var left=Math.random()<0.5;
  boss={x:left?-70:W+70, y:H-140, vx:left?spd*1.4+1.5:-(spd*1.4+1.5), vy:-2, phase:0, hp:3, hf:0, bubble:QUIPS[0], bt:100};
}

function drawRect(x,y,w,h,r,col){
  X.fillStyle=col;
  X.beginPath();
  r=Math.min(r,w/2,h/2);
  X.moveTo(x+r,y); X.lineTo(x+w-r,y); X.quadraticCurveTo(x+w,y,x+w,y+r);
  X.lineTo(x+w,y+h-r); X.quadraticCurveTo(x+w,y+h,x+w-r,y+h);
  X.lineTo(x+r,y+h); X.quadraticCurveTo(x,y+h,x,y+h-r);
  X.lineTo(x,y+r); X.quadraticCurveTo(x,y,x+r,y);
  X.closePath(); X.fill();
}

function star(x,y,r,c){
  X.fillStyle=c; X.beginPath();
  for(var i=0;i<5;i++){
    var a=-Math.PI/2+i*Math.PI*2/5, b=a+Math.PI/5;
    X.lineTo(x+Math.cos(a)*r,y+Math.sin(a)*r);
    X.lineTo(x+Math.cos(b)*r*0.42,y+Math.sin(b)*r*0.42);
  }
  X.closePath(); X.fill();
}

function draw(){
  X.fillStyle='#0d0d1a'; X.fillRect(0,0,W,H);

  X.fillStyle='rgba(180,180,255,0.3)';
  for(var i=0;i<65;i++){
    X.fillRect((i*137+frame*0.04)%W,(i*103+frame*0.012)%H,i%6===0?2:1,i%6===0?2:1);
  }

  for(var i=0;i<plats.length;i++){
    var pl=plats[i], isg=i===0;
    drawRect(pl.x,pl.y,pl.w,pl.h,5, isg?'#1a1636':'#3a3080');
    if(!isg){ X.fillStyle='rgba(160,150,220,0.3)'; X.fillRect(pl.x+4,pl.y,pl.w-8,2); }
  }

  for(var i=0;i<words.length;i++){
    var w=words[i];
    X.save(); X.translate(w.x,w.y); X.rotate(w.a);
    var c=w.big?'#e24b4a':'#8877ee';
    X.shadowColor=c; X.shadowBlur=8;
    X.fillStyle=c; X.font='bold '+(w.big?20:13)+'px monospace'; X.textAlign='center';
    X.fillText(w.word,0,0); X.shadowBlur=0; X.restore();
  }

  if(boss){
    X.save(); X.translate(boss.x+25,boss.y);
    if(boss.hf>0 && Math.floor(boss.hf/3)%2===0) X.globalAlpha=0.2;
    var b=Math.sin(boss.phase*0.09)*3;
    drawRect(-18,22+b,36,48,5,'#e8e8f4');
    X.fillStyle='#a32d2d'; X.fillRect(-3,28+b,6,22);
    X.fillStyle='#f09595'; X.beginPath(); X.arc(0,10+b,20,0,Math.PI*2); X.fill();
    X.strokeStyle='#534ab7'; X.lineWidth=2.5;
    X.strokeRect(-14,4+b,11,9); X.strokeRect(3,4+b,11,9);
    X.beginPath(); X.moveTo(-3,8+b); X.lineTo(3,8+b); X.stroke();
    X.fillStyle='#111';
    X.beginPath(); X.arc(-8,8+b,3,0,Math.PI*2); X.fill();
    X.beginPath(); X.arc(8,8+b,3,0,Math.PI*2); X.fill();
    for(var h=0;h<boss.hp;h++) star(-16+h*16,-18+b,7,'#e24b4a');
    X.fillStyle='#a32d2d'; X.font='bold 9px monospace'; X.textAlign='center';
    X.fillText('DR IVEL',0,78+b);
    if(boss.bt>0){
      var bw=Math.min(boss.bubble.length*6+16,165);
      drawRect(-bw/2,-58+b,bw,24,5,'rgba(255,255,255,0.93)');
      X.fillStyle='#a32d2d'; X.font='bold 9px monospace'; X.textAlign='center';
      X.fillText(boss.bubble,0,-40+b);
    }
    X.restore();
  }

  if(P){
    X.save(); X.translate(P.x+P.w/2,P.y);
    if(P.inv>0 && Math.floor(P.inv/6)%2===0) X.globalAlpha=0.1;
    var b=P.grnd?Math.sin(frame*0.22)*2:0;
    drawRect(-14,22+b,28,32,6,P.char.col);
    X.fillStyle='#f4c0d1'; X.beginPath(); X.arc(0,10+b,17,0,Math.PI*2); X.fill();
    X.strokeStyle='#111'; X.lineWidth=4;
    X.beginPath(); X.arc(0,6+b,16,Math.PI,0); X.stroke();
    X.fillStyle='#111';
    X.beginPath(); X.arc(-16,10+b,5,0,Math.PI*2); X.fill();
    X.beginPath(); X.arc(16,10+b,5,0,Math.PI*2); X.fill();
    X.strokeStyle='#555'; X.lineWidth=2;
    X.beginPath(); X.arc(-6,12+b,3,0,Math.PI); X.stroke();
    X.beginPath(); X.arc(6,12+b,3,0,Math.PI); X.stroke();
    X.beginPath(); X.arc(0,17+b,4,0,Math.PI); X.stroke();
    X.fillStyle='rgba(255,255,255,0.75)'; X.font='bold 9px monospace'; X.textAlign='center';
    X.fillText(P.char.name.toUpperCase(),0,62+b);
    X.restore();
  }
}

function loop(){ update(); draw(); requestAnimationFrame(loop); }

document.getElementById('startBtn').addEventListener('click',function(){
  document.getElementById('overlay').style.display='none';
  startGame();
});
document.getElementById('retryBtn').addEventListener('click',function(){
  document.getElementById('goPanel').style.display='none';
  document.getElementById('menuPanel').style.display='block';
  document.getElementById('overlay').style.display='flex';
});

loop();
</script>
</body>
</html>
