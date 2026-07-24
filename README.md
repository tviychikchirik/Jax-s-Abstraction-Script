<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Click and Hold</title>
<style>
html, body {
  margin: 0;
  height: 100%;
  overflow: hidden;
  background: #000;
  color: #0f0;
}
body {
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: Arial, sans-serif;
}
#hint {
  position: absolute;
  top: 24px;
  width: 100%;
  text-align: center;
  font-size: 24px;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: #0f0;
  user-select: none;
  z-index: 2;
  pointer-events: none;
}
#glitch {
  position: absolute;
  bottom: 32px;
  width: 100%;
  text-align: center;
  font-size: 20px;
  letter-spacing: 0.05em;
  color: #0f0;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s ease;
}
#canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
</style>
</head>
<body>
<div id="hint">CLICK AND HOLD.</div>
<canvas id="canvas"></canvas>
<div id="glitch">When will i get out...</div>
<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const hint = document.getElementById('hint');
const glitch = document.getElementById('glitch');
let width = canvas.width = window.innerWidth;
let height = canvas.height = window.innerHeight;
let pointer = { x: width / 2, y: height / 2, active: false };
let showHint = true;
let particles = [];
let eyeGlitches = [];
let absorbed = 0;
let completed = false;
const absorptionThreshold = 130;
const center = { x: width / 2, y: height / 2, r: 110 };
const spawnRate = 3;
const eyeGlitchColors = ['rgba(255,255,0,0.18)', 'rgba(0,255,255,0.18)', 'rgba(255,105,225,0.18)'];

window.addEventListener('resize', () => {
  width = canvas.width = window.innerWidth;
  height = canvas.height = window.innerHeight;
  center.x = width / 2;
  center.y = height / 2;
});

function hideHint() {
  if (!showHint) return;
  showHint = false;
  hint.style.opacity = '0';
  hint.style.transition = 'opacity 0.25s ease';
}

function setPointer(event, active) {
  const touch = event.changedTouches ? event.changedTouches[0] : event;
  pointer.x = touch.clientX;
  pointer.y = touch.clientY;
  pointer.active = active;
  if (active) hideHint();
}

canvas.addEventListener('mousedown', event => setPointer(event, true));
window.addEventListener('mouseup', () => { pointer.active = false; });
canvas.addEventListener('mousemove', event => {
  pointer.x = event.clientX;
  pointer.y = event.clientY;
});
canvas.addEventListener('touchstart', event => {
  event.preventDefault();
  setPointer(event, true);
}, { passive: false });
window.addEventListener('touchend', () => { pointer.active = false; });
canvas.addEventListener('touchmove', event => {
  event.preventDefault();
  setPointer(event, true);
}, { passive: false });

function random(min, max) {
  return Math.random() * (max - min) + min;
}

function spawnParticle() {
  const side = Math.floor(random(0, 4));
  let x = 0;
  let y = 0;

  if (side === 0) {
    x = -20;
    y = random(-20, height + 20);
  } else if (side === 1) {
    x = width + 20;
    y = random(-20, height + 20);
  } else if (side === 2) {
    x = random(-20, width + 20);
    y = -20;
  } else {
    x = random(-20, width + 20);
    y = height + 20;
  }

  const color = Math.random() < 0.5 ? 'rgba(255,110,215,0.92)' : 'rgba(110,190,255,0.92)';
  const angle = Math.atan2(center.y - y, center.x - x);
  const speed = random(1.4, 3.2);
  const spread = random(-0.18, 0.18);

  particles.push({
    x,
    y,
    vx: Math.cos(angle + spread) * speed,
    vy: Math.sin(angle + spread) * speed,
    color,
    radius: random(3.5, 5.5),
    life: 0
  });
}

function drawCore() {
  const corruption = Math.min(1, absorbed / absorptionThreshold);
  const lightness = 58 - corruption * 38;
  const coreColor = `hsl(278, 92%, ${Math.max(12, lightness)}%)`;
  const glow = ctx.createRadialGradient(center.x, center.y, 0, center.x, center.y, center.r * 1.4);
  glow.addColorStop(0, 'rgba(158, 80, 255, 0.35)');
  glow.addColorStop(0.8, 'rgba(0, 0, 0, 0.01)');
  glow.addColorStop(1, 'rgba(0, 0, 0, 0)');

  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, width, height);

  ctx.beginPath();
  ctx.arc(center.x, center.y, center.r * 1.35, 0, Math.PI * 2);
  ctx.fillStyle = glow;
  ctx.fill();

  ctx.beginPath();
  ctx.arc(center.x, center.y, center.r, 0, Math.PI * 2);
  ctx.fillStyle = coreColor;
  ctx.fill();

  ctx.lineWidth = 5;
  ctx.strokeStyle = `rgba(255, 255, 255, ${0.15 + corruption * 0.15})`;
  ctx.stroke();
}

function playGlitchSound() {
  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  const duration = 5.0;
  const now = audioCtx.currentTime;
  const buffer = audioCtx.createBuffer(1, audioCtx.sampleRate * duration, audioCtx.sampleRate);
  const data = buffer.getChannelData(0);

  for (let i = 0; i < data.length; i++) {
    const t = i / data.length;
    const envelope = 0.75 * (1 - t) + 0.18;
    const noise = (Math.random() * 2 - 1) * envelope;
    const modulation = Math.sin(50 * t * Math.PI) * 0.15 + Math.sin(110 * t * Math.PI) * 0.08;
    data[i] = noise * (1 + modulation);
  }

  const source = audioCtx.createBufferSource();
  source.buffer = buffer;

  const bandpass = audioCtx.createBiquadFilter();
  bandpass.type = 'bandpass';
  bandpass.frequency.setValueAtTime(1200, now);
  bandpass.Q.value = 0.8;
  bandpass.frequency.linearRampToValueAtTime(2600, now + duration);

  const highpass = audioCtx.createBiquadFilter();
  highpass.type = 'highpass';
  highpass.frequency.setValueAtTime(180, now);
  highpass.frequency.linearRampToValueAtTime(420, now + duration);

  const gain = audioCtx.createGain();
  gain.gain.setValueAtTime(0.0001, now);
  gain.gain.linearRampToValueAtTime(0.7, now + 0.1);
  gain.gain.setValueAtTime(0.65, now + 0.5);
  gain.gain.setValueAtTime(0.55, now + 1.5);
  gain.gain.exponentialRampToValueAtTime(0.001, now + duration);

  source.connect(bandpass);
  bandpass.connect(highpass);
  highpass.connect(gain);
  gain.connect(audioCtx.destination);

  source.start(now);
  source.stop(now + duration);
}

function spawnEyeGlitch() {
  const angle = Math.random() * Math.PI * 2;
  const distance = center.r * (1.05 + Math.random() * 0.38);
  const size = random(8, 18);
  eyeGlitches.push({
    x: Math.cos(angle) * distance + random(-10, 10),
    y: Math.sin(angle) * distance + random(-10, 10),
    w: size,
    h: random(3, 7),
    color: eyeGlitchColors[Math.floor(Math.random() * eyeGlitchColors.length)],
    life: random(18, 34),
    vx: random(-1.2, 1.2),
    vy: random(-1.2, 1.2)
  });
}

function updateEyeGlitches() {
  for (let i = eyeGlitches.length - 1; i >= 0; i--) {
    const glitchPixel = eyeGlitches[i];
    glitchPixel.life -= 1;
    glitchPixel.x += glitchPixel.vx;
    glitchPixel.y += glitchPixel.vy;
    if (glitchPixel.life <= 0) {
      eyeGlitches.splice(i, 1);
    }
  }
}

function drawEyeGlitches() {
  for (const glitchPixel of eyeGlitches) {
    ctx.fillStyle = glitchPixel.color;
    ctx.globalAlpha = Math.max(0, glitchPixel.life / 28) * 0.78;
    ctx.fillRect(glitchPixel.x, glitchPixel.y, glitchPixel.w, glitchPixel.h);
  }
  ctx.globalAlpha = 1;
}

function drawEye() {
  ctx.save();
  ctx.translate(center.x, center.y);
  const wobble = 1 + Math.sin(Date.now() / 120) * 0.03;
  ctx.scale(wobble, wobble);

  const scleraGradient = ctx.createRadialGradient(0, 0, center.r * 0.1, 0, 0, center.r * 1.05);
  scleraGradient.addColorStop(0, '#f6f6f8');
  scleraGradient.addColorStop(0.6, '#dadbe4');
  scleraGradient.addColorStop(1, '#111214');
  ctx.beginPath();
  ctx.ellipse(0, 0, center.r * 1.45, center.r * 1.05, 0, 0, Math.PI * 2);
  ctx.fillStyle = scleraGradient;
  ctx.fill();

  ctx.lineWidth = 5;
  ctx.strokeStyle = '#333';
  ctx.stroke();

  const irisRadius = center.r * 0.55;
  const irisGradient = ctx.createRadialGradient(0, 0, irisRadius * 0.16, 0, 0, irisRadius);
  irisGradient.addColorStop(0, '#0f3d8f');
  irisGradient.addColorStop(0.35, '#3666ff');
  irisGradient.addColorStop(0.7, '#f0d24a');
  irisGradient.addColorStop(1, '#ff5dd3');

  ctx.beginPath();
  ctx.arc(0, 0, irisRadius, 0, Math.PI * 2);
  ctx.fillStyle = irisGradient;
  ctx.fill();

  ctx.beginPath();
  ctx.arc(0, 0, irisRadius * 1.02, 0, Math.PI * 2);
  ctx.strokeStyle = 'rgba(0,0,0,0.18)';
  ctx.lineWidth = 6;
  ctx.stroke();

  for (let i = 0; i < 22; i++) {
    const angle = (Math.PI * 2 / 22) * i;
    const length = irisRadius * (0.62 + Math.random() * 0.14);
    const lineWidth = 1 + Math.random() * 1.2;
    ctx.strokeStyle = `rgba(255,255,255,${0.08 + Math.random() * 0.14})`;
    ctx.lineWidth = lineWidth;
    ctx.beginPath();
    ctx.moveTo(Math.cos(angle) * irisRadius * 0.2, Math.sin(angle) * irisRadius * 0.2);
    ctx.lineTo(Math.cos(angle) * length, Math.sin(angle) * length);
    ctx.stroke();
  }

  ctx.beginPath();
  ctx.arc(0, 0, irisRadius * 0.32, 0, Math.PI * 2);
  ctx.fillStyle = '#060606';
  ctx.fill();

  ctx.beginPath();
  ctx.arc(-irisRadius * 0.12, -irisRadius * 0.06, irisRadius * 0.05, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255,255,255,0.22)';
  ctx.fill();

  ctx.beginPath();
  ctx.arc(-irisRadius * 0.16, -irisRadius * 0.08, irisRadius * 0.02, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255,255,255,0.88)';
  ctx.fill();

  const glare = ctx.createRadialGradient(-irisRadius * 0.16, -irisRadius * 0.08, 0, -irisRadius * 0.16, -irisRadius * 0.08, irisRadius * 0.4);
  glare.addColorStop(0, 'rgba(255,255,255,0.5)');
  glare.addColorStop(0.7, 'rgba(255,255,255,0.0)');
  ctx.fillStyle = glare;
  ctx.beginPath();
  ctx.arc(-irisRadius * 0.16, -irisRadius * 0.08, irisRadius * 0.42, 0, Math.PI * 2);
  ctx.fill();

  if (Math.random() < 0.25) spawnEyeGlitch();
  updateEyeGlitches();
  drawEyeGlitches();

  ctx.restore();

  glitch.style.opacity = 0.96;
  glitch.style.transform = `translateX(${Math.sin(Date.now() / 30) * 5}px)`;
  glitch.style.filter = `blur(${0.8 + Math.abs(Math.sin(Date.now() / 60))}px)`;
}

function updateParticles() {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    const dx = center.x - p.x;
    const dy = center.y - p.y;
    const dist = Math.hypot(dx, dy) || 1;
    const pull = 0.24;

    p.vx = p.vx * 0.94 + (dx / dist) * pull + random(-0.02, 0.02);
    p.vy = p.vy * 0.94 + (dy / dist) * pull + random(-0.02, 0.02);
    p.x += p.vx;
    p.y += p.vy;
    p.life += 1;

    const targetRadius = center.r - p.radius * 0.5;
    if (dist <= targetRadius) {
      if (!completed) absorbed += 1;
      particles.splice(i, 1);
      continue;
    }

    if (p.life > 260 || p.x < -80 || p.x > width + 80 || p.y < -80 || p.y > height + 80) {
      particles.splice(i, 1);
      continue;
    }

    ctx.beginPath();
    ctx.fillStyle = p.color;
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();
  }
}

let hasPlayedGlitch = false;

function loop() {
  if (pointer.active && !completed) {
    for (let i = 0; i < spawnRate; i++) {
      if (Math.random() < 0.8) spawnParticle();
    }
  }

  if (!completed && absorbed >= absorptionThreshold) {
    completed = true;
    hint.textContent = 'ABSORPTION COMPLETE';
    particles = [];
    if (!hasPlayedGlitch) {
      playGlitchSound();
      hasPlayedGlitch = true;
    }
  }

  drawCore();
  updateParticles();

  if (completed) {
    drawEye();
  }

  requestAnimationFrame(loop);
}

loop();
</script>
</body>
</html>
