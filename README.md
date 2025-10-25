# site077
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>MouseAttractReptile — Interactive Reptile</title>
  <style>
    :root{--bg:#07121a;--panel:#0b2530;--accent:#7ee6a6}
    html,body{height:100%;margin:0;background:linear-gradient(180deg,var(--bg),#03101a);color:#e6f3ef;font-family:Inter,system-ui,Segoe UI,Arial}
    .topbar{position:fixed;left:12px;top:12px;z-index:40;background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));padding:10px 12px;border-radius:10px;backdrop-filter:blur(6px);box-shadow:0 6px 18px rgba(0,0,0,0.6)}
    .topbar h1{margin:0;font-size:14px;font-weight:600}
    .topbar p{margin:4px 0 0;font-size:12px;color:#9fb6a8}
    canvas{display:block;position:fixed;inset:0;width:100%;height:100%;touch-action:none}
    .controls{position:fixed;right:12px;top:12px;background:var(--panel);padding:10px;border-radius:10px;box-shadow:0 6px 18px rgba(0,0,0,0.6)}
    .controls label{display:flex;align-items:center;gap:8px;font-size:13px;color:#cfeee0}
    .controls input[type=range]{width:140px}
    .hint{position:fixed;left:50%;transform:translateX(-50%);bottom:18px;color:#9fb6a8;font-size:13px}
    @media (max-width:520px){.topbar{left:8px;top:8px;padding:8px}.controls{right:8px;top:8px;padding:8px}}
  </style>
</head>
<body>
  <div class="topbar">
    <h1>MouseAttractReptile</h1>
    <p>Move the mouse — the reptile's head will chase the cursor and the body will follow smoothly.</p>
  </div>

  <div class="controls" aria-hidden="false">
    <label>Segments: <input id="segRange" type="range" min="6" max="48" value="18"></label>
    <label>Smoothness: <input id="smoothRange" type="range" min="0.01" max="0.8" step="0.01" value="0.3"></label>
    <label>Size: <input id="sizeRange" type="range" min="6" max="28" value="12"></label>
  </div>

  <div class="hint">Tip: Click to respawn the reptile; press Space to toggle follow strength.</div>
  <canvas id="c"></canvas>

  <script>
  // MouseAttractReptile — single-file demo
  // Features:
  // - Canvas full-screen
  // - Head seeks mouse (steering behavior)
  // - Body segments follow head positions with smoothing
  // - Eyes/pupils follow mouse independently
  // - Controls: segments, smoothness, size

  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  let DPR = Math.max(1, window.devicePixelRatio || 1);

  function resize(){
    DPR = Math.max(1, window.devicePixelRatio || 1);
    canvas.width = Math.floor(window.innerWidth * DPR);
    canvas.height = Math.floor(window.innerHeight * DPR);
    canvas.style.width = window.innerWidth + 'px';
    canvas.style.height = window.innerHeight + 'px';
    ctx.setTransform(DPR,0,0,DPR,0,0);
  }
  window.addEventListener('resize', resize);
  resize();

  // state
  let mouse = {x: window.innerWidth/2, y: window.innerHeight/2};
  let attract = true; // follow or not

  canvas.addEventListener('mousemove', (e)=>{ mouse.x = e.clientX; mouse.y = e.clientY; });
  canvas.addEventListener('touchmove', (e)=>{ e.preventDefault(); const t = e.touches[0]; mouse.x = t.clientX; mouse.y = t.clientY; }, {passive:false});

  window.addEventListener('click', ()=>{ initReptile(); });
  window.addEventListener('keydown', (e)=>{ if(e.code === 'Space'){ attract = !attract } });

  // controls
  const segRange = document.getElementById('segRange');
  const smoothRange = document.getElementById('smoothRange');
  const sizeRange = document.getElementById('sizeRange');

  // reptile model
  let reptile = null;

  function initReptile(){
    const segCount = parseInt(segRange.value,10);
    const segSize = parseFloat(sizeRange.value);
    const smoothness = parseFloat(smoothRange.value);

    // The body is represented as an array of points. head is index 0.
    const points = [];
    const startX = canvas.width/(2*DPR);
    const startY = canvas.height/(2*DPR);
    for(let i=0;i<segCount;i++){
      points.push({x: startX - i * (segSize*0.9), y: startY, vx:0, vy:0});
    }

    reptile = {
      points,
      segSize,
      smoothness,
      colorA: '#2b8a3e',
      colorB: '#7be495',
      headAngle:0,
      tailWiggle:0
    };
  }

  // steering behavior: head moves toward mouse with max speed
  function updateReptile(dt){
    if(!reptile) return;
    const p0 = reptile.points[0];
    // desired velocity towards mouse
    const dx = (mouse.x - p0.x);
    const dy = (mouse.y - p0.y);
    const dist = Math.hypot(dx,dy) || 0.0001;
    const desiredSpeed = attract ? Math.min(600, dist*2) : Math.max(40, dist*0.3);
    const desiredVx = (dx / dist) * desiredSpeed;
    const desiredVy = (dy / dist) * desiredSpeed;

    // simple steering with damping
    const steerStrength = 12; // responsiveness
    p0.vx += (desiredVx - p0.vx) * Math.min(1, steerStrength * dt);
    p0.vy += (desiredVy - p0.vy) * Math.min(1, steerStrength * dt);

    // integrate head
    p0.x += p0.vx * dt;
    p0.y += p0.vy * dt;

    // store head angle for drawing eyes
    reptile.headAngle = Math.atan2(p0.vy, p0.vx);

    // follow behavior: each segment chases previous segment's previous position
    for(let i=1;i<reptile.points.length;i++){
      const prev = reptile.points[i-1];
      const cur = reptile.points[i];
      const dx2 = prev.x - cur.x;
      const dy2 = prev.y - cur.y;
      const d = Math.hypot(dx2,dy2) || 0.0001;
      // desired position = at a distance equal to segSize*0.95 from prev along line
      const targetX = prev.x - (dx2 / d) * (reptile.segSize*0.95);
      const targetY = prev.y - (dy2 / d) * (reptile.segSize*0.95);
      // smooth move towards target
      cur.x += (targetX - cur.x) * reptile.smoothness;
      cur.y += (targetY - cur.y) * reptile.smoothness;
    }

    // tail wiggle driven by time and head speed
    reptile.tailWiggle += dt * (1 + Math.hypot(p0.vx,p0.vy) * 0.005);
  }

  function drawReptile(){
    if(!reptile) return;
    const pts = reptile.points;
    ctx.save();
    ctx.lineJoin = 'round';
    ctx.lineCap = 'round';

    // draw body as gradient-stroked chain
    for(let i=pts.length-1;i>=0;i--){
      const p = pts[i];
      const t = i / (pts.length - 1);
      // size tapering
      const size = reptile.segSize * (0.85 + 0.7 * (1 - t));
      // color blend
      ctx.beginPath();
      ctx.fillStyle = mixColor(reptile.colorA, reptile.colorB, t * 0.9);
      // tail wobble offset
      const wobble = Math.sin(reptile.tailWiggle - i * 0.5) * (1 - t) * 6;
      ctx.ellipse(p.x + wobble, p.y, size, size*0.9, 0, 0, Math.PI*2);
      ctx.fill();
    }

    // draw head with eyes on top
    const head = pts[0];
    const headSize = reptile.segSize * 1.6;
    ctx.save();
    ctx.translate(head.x, head.y);
    ctx.rotate(reptile.headAngle + Math.PI/2);

    // head gradient
    const g = ctx.createLinearGradient(-headSize, -headSize, headSize, headSize);
    g.addColorStop(0, reptile.colorB);
    g.addColorStop(1, reptile.colorA);
    ctx.fillStyle = g;
    ctx.beginPath();
    ctx.ellipse(0,0, headSize*1.1, headSize, 0, 0, Math.PI*2);
    ctx.fill();

    // eye parameters
    const eyeOffsetX = headSize * 0.45;
    const eyeOffsetY = -headSize * 0.05;
    const eyeRadius = Math.max(4, headSize * 0.28);
    const pupilRadius = Math.max(2, eyeRadius * 0.36);

    // draw eyes (left & right)
    for(const sign of [-1,1]){
      const ex = sign * eyeOffsetX;
      const ey = eyeOffsetY;
      // eye white
      ctx.beginPath(); ctx.fillStyle = '#fff'; ctx.ellipse(ex, ey, eyeRadius, eyeRadius*0.9, 0, 0, Math.PI*2); ctx.fill();

      // pupil looks at global mouse: compute relative direction in world space
      const worldEyeX = head.x + ex * Math.cos(reptile.headAngle) - ey * Math.sin(reptile.headAngle);
      const worldEyeY = head.y + ex * Math.sin(reptile.headAngle) + ey * Math.cos(reptile.headAngle);
      const mx = mouse.x, my = mouse.y;
      const pdx = mx - worldEyeX; const pdy = my - worldEyeY;
      const pdist = Math.hypot(pdx,pdy) || 0.0001;
      const maxOffset = eyeRadius * 0.38;
      const px = ex + (pdx / pdist) * maxOffset;
      const py = ey + (pdy / pdist) * maxOffset;

      // pupil
      ctx.beginPath(); ctx.fillStyle = '#0b2b16'; ctx.ellipse(px, py, pupilRadius, pupilRadius, 0, 0, Math.PI*2); ctx.fill();

      // small glossy shine
      ctx.beginPath(); ctx.fillStyle = 'rgba(255,255,255,0.6)'; ctx.ellipse(px - pupilRadius*0.4, py - pupilRadius*0.6, pupilRadius*0.35, pupilRadius*0.25, 0, 0, Math.PI*2); ctx.fill();
    }

    // nostrils / mouth subtle line
    ctx.beginPath(); ctx.strokeStyle = 'rgba(0,0,0,0.18)'; ctx.lineWidth = Math.max(1, headSize*0.07); ctx.moveTo(-headSize*0.25, headSize*0.4); ctx.quadraticCurveTo(0, headSize*0.6, headSize*0.25, headSize*0.4); ctx.stroke();

    ctx.restore();
    ctx.restore();
  }

  // simple color mix helper (hex to rgb and back)
  function mixColor(a,b,t){
    const aa = hexToRgb(a), bb = hexToRgb(b);
    const r = Math.round(aa.r + (bb.r - aa.r) * t);
    const g = Math.round(aa.g + (bb.g - aa.g) * t);
    const bl = Math.round(aa.b + (bb.b - aa.b) * t);
    return `rgb(${r},${g},${bl})`;
  }
  function hexToRgb(hex){
    const h = hex.replace('#','');
    return { r: parseInt(h.substring(0,2),16), g: parseInt(h.substring(2,4),16), b: parseInt(h.substring(4,6),16) };
  }

  // animation loop
  let last = performance.now();
  function frame(now){
    const dt = Math.min(0.04, (now - last) / 1000); // clamp dt
    last = now;

    // update reptile based on controls
    if(reptile){
      reptile.smoothness = parseFloat(smoothRange.value);
      reptile.segSize = parseFloat(sizeRange.value);
      // if seg count has changed, rebuild but keep head pos
      const desired = parseInt(segRange.value,10);
      if(reptile.points.length !== desired){
        const head = reptile.points[0];
        const newPts = [{x:head.x, y:head.y, vx:head.vx||0, vy:head.vy||0}];
        for(let i=1;i<desired;i++) newPts.push({x: head.x - i * reptile.segSize*0.95, y: head.y, vx:0, vy:0});
        reptile.points = newPts;
      }

      updateReptile(dt);
    }

    // draw
    ctx.clearRect(0,0, canvas.width, canvas.height);
    // subtle background vignette
    const w = canvas.width/DPR, h = canvas.height/DPR;
    const bgGrad = ctx.createLinearGradient(0,0,0,h);
    bgGrad.addColorStop(0, 'rgba(4,12,18,0.9)'); bgGrad.addColorStop(1, 'rgba(1,6,10,0.9)');
    ctx.fillStyle = bgGrad; ctx.fillRect(0,0,w,h);

    drawReptile();

    requestAnimationFrame(frame);
  }

  // init
  initReptile();
  requestAnimationFrame(frame);

  // utility: ensure crispness on high-dpi
  // (already handled by transform + DPR)

  // accessibility: provide keyboard control to nudge mouse target
  window.addEventListener('keydown', (e)=>{
    const step = 20;
    if(e.key === 'ArrowLeft') mouse.x -= step;
    if(e.key === 'ArrowRight') mouse.x += step;
    if(e.key === 'ArrowUp') mouse.y -= step;
    if(e.key === 'ArrowDown') mouse.y += step;
  });

  </script>
</body>
</html>
