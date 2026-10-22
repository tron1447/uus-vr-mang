[Uploading index (16).html…]()
<!doctype html>
<html lang="et">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">
<title>Neoontunnel VR</title>
<script>
window.GAME_VERSION = 'nt2';
(function(){
  try {
    fetch(location.pathname + '?nocache=' + Date.now(), { cache: 'no-store' }).then(function(r){ return r.text(); }).then(function(t){
      var m = t.match(/window\.GAME_VERSION = '([^']+)'/);
      if (m && m[1] !== window.GAME_VERSION) { var k = 'nt_reload_' + m[1]; if (!sessionStorage.getItem(k)) { sessionStorage.setItem(k, '1'); location.replace(location.pathname + '?v=' + m[1] + '_' + Date.now()); } }
    }).catch(function(){});
  } catch (e) {}
})();
</script>
<style>
  html,body{margin:0;height:100%;background:#05010d;overflow:hidden;font-family:system-ui,"Segoe UI",sans-serif}
  #info{position:fixed;top:10px;left:10px;color:#e8e0ff;background:rgba(10,2,30,.7);border:1px solid rgba(0,229,255,.4);padding:10px 14px;border-radius:12px;font-size:14px;line-height:1.5;max-width:420px;z-index:5;box-shadow:0 0 20px rgba(255,43,214,.25)}
  #info b{color:#00e5ff}
</style>
<script type="importmap">{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js","three/addons/":"https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"}}</script>
</head>
<body>
<div id="info">
  <b>🌈 NEOONTUNNEL VR</b> <span style="opacity:.6">(nt2)</span><br>
  Questis: vajuta all <b>ENTER VR</b> – mängi pultide või paljaste kätega.<br>
  Arvutis: <b>hiir</b> = käsi (löö palle), <b>A/D</b> = kaldu, <b>C</b> = kükita, <b>tühik</b> = hüppa, <b>Esc</b> = paus.
</div>
<script type="module">
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';
import { XRHandModelFactory } from 'three/addons/webxr/XRHandModelFactory.js';

/* =========================================================
   ABI
   ========================================================= */
const V3 = (x = 0, y = 0, z = 0) => new THREE.Vector3(x, y, z);
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const lerp = (a, b, t) => a + (b - a) * t;
const store = (k, v) => { try { if (v === undefined) return JSON.parse(localStorage.getItem(k)); localStorage.setItem(k, JSON.stringify(v)); } catch (e) { return null; } };
function mulberry32(a){ return function(){ a |= 0; a = a + 0x6D2B79F5 | 0; let t = Math.imul(a ^ a >>> 15, 1 | a); t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t; return ((t ^ t >>> 14) >>> 0) / 4294967296; }; }
const FONT = '"Segoe UI", system-ui, sans-serif';
const tmpV = V3(), tmpV2 = V3(), tmpD = V3(), tmpQ = new THREE.Quaternion(), head = V3();
function segDist(p, a, b){ tmpD.copy(b).sub(a); const l2 = tmpD.lengthSq(); let k = l2 ? tmpV2.copy(p).sub(a).dot(tmpD) / l2 : 0; k = clamp(k, 0, 1); return tmpV2.copy(a).addScaledVector(tmpD, k).distanceTo(p); }
function roundRect(c, x, y, w, h, r){ c.beginPath(); c.moveTo(x + r, y); c.arcTo(x + w, y, x + w, y + h, r); c.arcTo(x + w, y + h, x, y + h, r); c.arcTo(x, y + h, x, y, r); c.arcTo(x, y, x + w, y, r); c.closePath(); }
const hex = c => '#' + new THREE.Color(c).getHexString();

/* =========================================================
   RENDERER, STSEEN
   ========================================================= */
const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
renderer.xr.enabled = true;
renderer.xr.setReferenceSpaceType('local-floor');
try { renderer.xr.setFoveation(0.25); } catch (e) {}
document.body.appendChild(renderer.domElement);
const vrButton = VRButton.createButton(renderer);
document.body.appendChild(vrButton);
vrButton.addEventListener('click', () => audioInit());

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x05010d);
scene.fog = new THREE.FogExp2(0x0d0322, 0.016);
const camera = new THREE.PerspectiveCamera(72, innerWidth / innerHeight, 0.03, 900);
camera.position.set(0, 1.6, 0);
const rig = new THREE.Group(); rig.add(camera); scene.add(rig);
addEventListener('resize', () => { camera.aspect = innerWidth / innerHeight; camera.updateProjectionMatrix(); renderer.setSize(innerWidth, innerHeight); });

const U = { uT: { value: 0 }, uDist: { value: 0 }, uBeat: { value: 0 }, uFever: { value: 0 }, uC1: { value: new THREE.Color(0x00e5ff) }, uC2: { value: new THREE.Color(0xff2bd6) } };

/* ---------- taevas ---------- */
const sky = new THREE.Mesh(new THREE.SphereGeometry(500, 32, 16), new THREE.ShaderMaterial({
  side: THREE.BackSide, depthWrite: false, fog: false, uniforms: U,
  vertexShader: `varying vec3 vP; void main(){ vP = normalize(position); gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
  fragmentShader: `uniform vec3 uC2; varying vec3 vP;
    void main(){ float h = vP.y;
      vec3 top = vec3(0.008, 0.0, 0.03), hor = vec3(0.16, 0.02, 0.28);
      vec3 c = mix(hor, top, smoothstep(0.0, 0.45, h));
      c += uC2 * pow(max(0.0, 1.0 - abs(h) * 5.0), 3.0) * 0.35;
      if (h < 0.0) c = mix(hor * 0.5, vec3(0.01, 0.0, 0.02), smoothstep(0.0, -0.25, h));
      gl_FragColor = vec4(c, 1.0); }`
}));
sky.renderOrder = -10; scene.add(sky);
{ // tähed
  const n = 1400, p = new Float32Array(n * 3);
  for (let i = 0; i < n; i++) { const th = Math.random() * Math.PI * 2, ph = Math.acos(Math.random() * 0.95), r = 450; p[i * 3] = Math.sin(ph) * Math.cos(th) * r; p[i * 3 + 1] = Math.cos(ph) * r * 0.9 + 10; p[i * 3 + 2] = Math.sin(ph) * Math.sin(th) * r; }
  const g = new THREE.BufferGeometry(); g.setAttribute('position', new THREE.BufferAttribute(p, 3));
  const stars = new THREE.Points(g, new THREE.PointsMaterial({ color: 0xffffff, size: 1.6, sizeAttenuation: false, fog: false, transparent: true, opacity: 0.85 }));
  sky.add(stars);
}
/* ---------- synthwave päike ---------- */
const sun = new THREE.Mesh(new THREE.PlaneGeometry(110, 110), new THREE.ShaderMaterial({
  transparent: true, depthWrite: false, fog: false, uniforms: U,
  vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
  fragmentShader: `uniform float uT, uBeat; varying vec2 vUv;
    void main(){ vec2 p = vUv - 0.5; float d = length(p);
      vec3 col = mix(vec3(1.0, 0.08, 0.55), vec3(1.0, 0.85, 0.25), smoothstep(-0.25, 0.25, p.y));
      float disk = smoothstep(0.255, 0.25, d);
      float y = 0.25 - p.y; float stripes = 1.0;
      if (p.y < 0.05) { float k = fract(y * 18.0 - uT * 0.4); float w = clamp((0.05 - p.y) * 2.2, 0.0, 0.55); stripes = step(w, k); }
      float glow = exp(-max(d - 0.25, 0.0) * 14.0) * 0.55 * (1.0 + uBeat * 0.3);
      vec3 c = col * disk * stripes + vec3(1.0, 0.3, 0.6) * glow * (1.0 - disk * stripes);
      float a = max(disk * stripes, glow);
      gl_FragColor = vec4(c, a); }`
}));
sun.position.set(0, 22, -330); sun.renderOrder = -9; scene.add(sun);

/* ---------- mäed külgedel (traatvõrk) ---------- */
const mountains = [];
for (const side of [-1, 1]) {
  const g = new THREE.PlaneGeometry(80, 400, 26, 130).rotateX(-Math.PI / 2), pos = g.attributes.position;
  for (let i = 0; i < pos.count; i++) {
    const lx = pos.getX(i), z = pos.getZ(i), d = lx + 40, W = Math.PI * 2 / 100;
    const n = 0.5 + 0.25 * Math.sin(z * W * 2 + d * 0.31) + 0.15 * Math.sin(z * W * 5 + d * 0.73) + 0.1 * Math.sin(z * W * 9 + d * 1.3);
    pos.setY(i, d < 1 ? -0.6 : Math.pow(d / 80, 0.9) * 26 * n - 0.6);
    pos.setX(i, side * (12 + d));
  }
  g.computeVertexNormals();
  const solid = new THREE.Mesh(g, new THREE.MeshBasicMaterial({ color: 0x07010f, polygonOffset: true, polygonOffsetFactor: 1, polygonOffsetUnits: 1 }));
  const wire = new THREE.Mesh(g, new THREE.MeshBasicMaterial({ color: 0xff2bd6, wireframe: true, transparent: true, opacity: 0.45 }));
  const grp = new THREE.Group(); grp.add(solid, wire); scene.add(grp); mountains.push({ grp, wire });
}

/* ---------- põrand ---------- */
const TUN_R = 3.0, TUN_Y = 1.2, TUN_L = 130, TUN_Z = -TUN_L / 2 + 4;
const floor = new THREE.Mesh(new THREE.PlaneGeometry(5.8, TUN_L).rotateX(-Math.PI / 2), new THREE.ShaderMaterial({
  uniforms: U, fog: false,
  vertexShader: `varying vec3 vW; void main(){ vec4 w = modelMatrix * vec4(position, 1.0); vW = w.xyz; gl_Position = projectionMatrix * viewMatrix * w; }`,
  fragmentShader: `uniform float uDist, uBeat, uFever, uT; uniform vec3 uC1, uC2; varying vec3 vW;
    float line(float v, float w){ float f = fract(v); return smoothstep(w, 0.0, min(f, 1.0 - f)); }
    void main(){
      float z = vW.z - uDist, x = vW.x;
      float fade = exp(-max(0.0, -vW.z) * 0.028);
      float lz = line(z * 0.5, 0.035);
      float lx = line(x / 0.7 + 0.5, 0.03);
      float edge = smoothstep(0.1, 0.0, abs(abs(x) - 2.75));
      float lane = smoothstep(0.05, 0.0, abs(x)) * 0.6;
      vec3 base = vec3(0.018, 0.004, 0.045);
      vec3 col = mix(uC2, uC1, edge + lane);
      float g = (lz * 0.9 + lx * 0.55 + edge * 1.4 + lane) * fade * (0.75 + 0.6 * uBeat);
      float halo = (smoothstep(0.2, 0.0, min(fract(z * 0.5), 1.0 - fract(z * 0.5))) * 0.12) * fade;
      vec3 c = base + col * (g + halo) * (1.0 + uFever * 0.6);
      c += uC2 * 0.05 * smoothstep(-60.0, -110.0, vW.z);
      gl_FragColor = vec4(c, 1.0); }`
}));
floor.position.set(0, 0, TUN_Z); scene.add(floor);

/* ---------- tunnel (helendav võrk) ---------- */
const tunnel = new THREE.Mesh(new THREE.CylinderGeometry(TUN_R, TUN_R, TUN_L, 72, 1, true).rotateX(Math.PI / 2), new THREE.ShaderMaterial({
  uniforms: U, side: THREE.BackSide, transparent: true, depthWrite: false, blending: THREE.AdditiveBlending, fog: false,
  vertexShader: `varying vec2 vUv; varying vec3 vW; void main(){ vUv = uv; vec4 w = modelMatrix * vec4(position, 1.0); vW = w.xyz; gl_Position = projectionMatrix * viewMatrix * w; }`,
  fragmentShader: `uniform float uDist, uBeat, uT, uFever; uniform vec3 uC1, uC2; varying vec2 vUv; varying vec3 vW;
    float hash(float n){ return fract(sin(n * 12.9898) * 43758.5453); }
    void main(){
      float a = vUv.x, z = vW.z - uDist;
      float fade = exp(-max(0.0, -vW.z) * 0.03) * smoothstep(-0.3, 0.6, vW.y);
      float rz = fract(z / 3.0); float dz = min(rz, 1.0 - rz) * 3.0;
      float ra = fract(a * 28.0); float da = min(ra, 1.0 - ra) / 28.0 * 6.2831 * 3.0;
      float ring = smoothstep(0.035, 0.0, dz), rad = smoothstep(0.025, 0.0, da);
      float halo = smoothstep(0.5, 0.0, dz) * 0.18 + smoothstep(0.25, 0.0, da) * 0.08;
      float li = floor(a * 28.0 + 0.5);
      float dash = smoothstep(0.92, 1.0, fract(z * 0.035 + hash(li) * 9.0)) * smoothstep(0.08, 0.0, da) * 1.5;
      float hue = sin(a * 12.566 + uT * 0.6 + z * 0.02) * 0.5 + 0.5;
      vec3 col = mix(uC1, uC2, hue);
      vec3 rainbow = 0.5 + 0.5 * cos(6.2831 * (a + z * 0.01 + uT * 0.2 + vec3(0.0, 0.33, 0.67)));
      col = mix(col, rainbow, uFever * 0.8);
      float g = (ring * 1.0 + rad * 0.55 + dash + halo) * (0.7 + 0.7 * uBeat);
      vec3 c = col * g * fade + col * 0.025 * fade;
      gl_FragColor = vec4(c, 1.0); }`
}));
tunnel.position.set(0, TUN_Y, TUN_Z); tunnel.renderOrder = 2; scene.add(tunnel);

/* ---------- neoonrõngad ---------- */
const RING_N = 16, RING_GAP = 7;
const ringCoreGeo = new THREE.TorusGeometry(TUN_R - 0.03, 0.022, 8, 120), ringGlowGeo = new THREE.TorusGeometry(TUN_R - 0.03, 0.13, 8, 120);
const rings = Array.from({ length: RING_N }, (_, i) => {
  const g = new THREE.Group();
  const core = new THREE.Mesh(ringCoreGeo, new THREE.MeshBasicMaterial({ color: 0xffffff }));
  const glow = new THREE.Mesh(ringGlowGeo, new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.16, blending: THREE.AdditiveBlending, depthWrite: false }));
  g.add(core, glow); g.position.set(0, TUN_Y, -i * RING_GAP); scene.add(g);
  return { g, core, glow, k: i };
});

/* ---------- kiirusjooned ---------- */
const STREAK_N = 160, streakPos = new Float32Array(STREAK_N * 6), streakData = [];
for (let i = 0; i < STREAK_N; i++) { const a = Math.random() * Math.PI * 2, r = 1.4 + Math.random() * 1.45; streakData.push({ x: Math.cos(a) * r, y: Math.sin(a) * r, z: -Math.random() * 110, l: 1 + Math.random() * 3 }); }
const streakGeo = new THREE.BufferGeometry(); streakGeo.setAttribute('position', new THREE.BufferAttribute(streakPos, 3));
const streaks = new THREE.LineSegments(streakGeo, new THREE.LineBasicMaterial({ color: 0x9ff6ff, transparent: true, opacity: 0.55, blending: THREE.AdditiveBlending, depthWrite: false }));
streaks.frustumCulled = false; const streakG = new THREE.Group(); streakG.position.y = TUN_Y; streakG.add(streaks); scene.add(streakG);

/* =========================================================
   OSAKESED (üks joonistuskutse)
   ========================================================= */
class Particles {
  constructor(n){
    this.n = n; this.i = 0;
    this.pos = new Float32Array(n * 3); this.col = new Float32Array(n * 3); this.size = new Float32Array(n); this.alpha = new Float32Array(n);
    this.vel = Array.from({ length: n }, () => V3()); this.life = new Float32Array(n); this.max = new Float32Array(n).fill(1); this.drag = new Float32Array(n);
    const g = new THREE.BufferGeometry();
    g.setAttribute('position', new THREE.BufferAttribute(this.pos, 3)); g.setAttribute('pcolor', new THREE.BufferAttribute(this.col, 3));
    g.setAttribute('psize', new THREE.BufferAttribute(this.size, 1)); g.setAttribute('palpha', new THREE.BufferAttribute(this.alpha, 1));
    this.geo = g;
    this.mat = new THREE.ShaderMaterial({
      uniforms: { uScale: { value: 600 } }, transparent: true, depthWrite: false, blending: THREE.AdditiveBlending,
      vertexShader: `attribute vec3 pcolor; attribute float psize; attribute float palpha; uniform float uScale; varying vec3 vC; varying float vA;
        void main(){ vC = pcolor; vA = palpha; vec4 mv = modelViewMatrix * vec4(position, 1.0); gl_PointSize = psize * uScale / max(0.05, -mv.z); gl_Position = projectionMatrix * mv; }`,
      fragmentShader: `varying vec3 vC; varying float vA; void main(){ vec2 c = gl_PointCoord - 0.5; float d = length(c); float a = smoothstep(0.5, 0.0, d); a = a * a * (0.6 + 0.4 * smoothstep(0.18, 0.0, d) * 2.0); gl_FragColor = vec4(vC + vec3(smoothstep(0.15, 0.0, d)) * 0.6, a * vA); }`
    });
    this.pts = new THREE.Points(g, this.mat); this.pts.frustumCulled = false; this.pts.renderOrder = 20; scene.add(this.pts);
  }
  emit(p, v, col, size, life, drag = 1.5){
    const i = this.i = (this.i + 1) % this.n;
    this.pos[i * 3] = p.x; this.pos[i * 3 + 1] = p.y; this.pos[i * 3 + 2] = p.z;
    this.vel[i].copy(v); const c = col.isColor ? col : tmpCol.set(col); this.col[i * 3] = c.r; this.col[i * 3 + 1] = c.g; this.col[i * 3 + 2] = c.b;
    this.size[i] = size; this.life[i] = life; this.max[i] = life; this.drag[i] = drag;
  }
  update(dt){
    for (let i = 0; i < this.n; i++) {
      if (this.life[i] <= 0) { this.alpha[i] = 0; continue; }
      this.life[i] -= dt; const v = this.vel[i];
      v.multiplyScalar(Math.exp(-this.drag[i] * dt));
      this.pos[i * 3] += v.x * dt; this.pos[i * 3 + 1] += v.y * dt; this.pos[i * 3 + 2] += v.z * dt;
      this.alpha[i] = Math.max(0, this.life[i] / this.max[i]);
    }
    for (const k of ['position', 'pcolor', 'psize', 'palpha']) this.geo.attributes[k].needsUpdate = true;
  }
}
const tmpCol = new THREE.Color();
const sparks = new Particles(700);
// püsivad helendused (pallid, käed) – täidetakse igas kaadris uuesti
class Glows {
  constructor(n){
    this.n = n; this.c = 0;
    this.pos = new Float32Array(n * 3); this.col = new Float32Array(n * 3); this.size = new Float32Array(n); this.alpha = new Float32Array(n);
    const g = new THREE.BufferGeometry();
    g.setAttribute('position', new THREE.BufferAttribute(this.pos, 3)); g.setAttribute('pcolor', new THREE.BufferAttribute(this.col, 3));
    g.setAttribute('psize', new THREE.BufferAttribute(this.size, 1)); g.setAttribute('palpha', new THREE.BufferAttribute(this.alpha, 1));
    this.geo = g; this.pts = new THREE.Points(g, sparks.mat); this.pts.frustumCulled = false; this.pts.renderOrder = 19; scene.add(this.pts);
  }
  begin(){ this.c = 0; }
  add(p, col, size, a = 1){ if (this.c >= this.n) return; const i = this.c++; this.pos[i * 3] = p.x; this.pos[i * 3 + 1] = p.y; this.pos[i * 3 + 2] = p.z; const c = col.isColor ? col : tmpCol.set(col); this.col[i * 3] = c.r; this.col[i * 3 + 1] = c.g; this.col[i * 3 + 2] = c.b; this.size[i] = size; this.alpha[i] = a; }
  end(){ this.geo.setDrawRange(0, this.c); for (const k of ['position', 'pcolor', 'psize', 'palpha']) this.geo.attributes[k].needsUpdate = true; }
}
const glows = new Glows(700);
function burst(p, col, n = 26, speed = 3, size = 0.05){
  for (let i = 0; i < n; i++) { const v = V3(Math.random() - 0.5, Math.random() - 0.5, Math.random() - 0.5).normalize().multiplyScalar(speed * (0.4 + Math.random() * 0.8)); sparks.emit(p, v, col, size * (0.6 + Math.random() * 0.8), 0.5 + Math.random() * 0.5, 2.2); }
}

/* =========================================================
   AUDIO: genereeritud synthwave
   ========================================================= */
let actx = null, master, musicBus, sfxBus, delayIn, noiseBuf;
function audioInit(){
  if (actx) { if (actx.state === 'suspended' && mode !== 'pause') actx.resume(); return; }
  actx = new (window.AudioContext || window.webkitAudioContext)();
  const comp = actx.createDynamicsCompressor(); comp.threshold.value = -14; comp.ratio.value = 4;
  master = actx.createGain(); master.gain.value = 0.85; master.connect(comp); comp.connect(actx.destination);
  musicBus = actx.createGain(); musicBus.gain.value = 0.6; musicBus.connect(master);
  sfxBus = actx.createGain(); sfxBus.gain.value = 0.7; sfxBus.connect(master);
  delayIn = actx.createGain(); const dl = actx.createDelay(1.5), fb = actx.createGain(), lp = actx.createBiquadFilter();
  dl.delayTime.value = 0.36; fb.gain.value = 0.38; lp.type = 'lowpass'; lp.frequency.value = 2600;
  delayIn.connect(dl); dl.connect(lp); lp.connect(fb); fb.connect(dl); lp.connect(musicBus);
  delayIn.userData = { dl };
  noiseBuf = actx.createBuffer(1, actx.sampleRate, actx.sampleRate); const d = noiseBuf.getChannelData(0); for (let i = 0; i < d.length; i++) d[i] = Math.random() * 2 - 1;
}
const mtof = m => 440 * Math.pow(2, (m - 69) / 12);
function env(g, t, a, peak, dec, sus = 0, rel = 0.05, dur = 0){
  g.gain.setValueAtTime(0.0001, t); g.gain.linearRampToValueAtTime(peak, t + a);
  if (dur) { g.gain.setTargetAtTime(peak * sus, t + a, dec / 3); g.gain.setTargetAtTime(0.0001, t + dur, rel / 3); }
  else g.gain.exponentialRampToValueAtTime(0.0001, t + a + dec);
}
function kick(t, v = 1){ const o = actx.createOscillator(), g = actx.createGain(); o.frequency.setValueAtTime(150, t); o.frequency.exponentialRampToValueAtTime(42, t + 0.12); env(g, t, 0.003, 0.9 * v, 0.38); o.connect(g); g.connect(musicBus); o.start(t); o.stop(t + 0.45); }
function noise(t, dur, type, freq, peak, bus = musicBus, q = 0.7){ const s = actx.createBufferSource(); s.buffer = noiseBuf; const f = actx.createBiquadFilter(); f.type = type; f.frequency.value = freq; f.Q.value = q; const g = actx.createGain(); env(g, t, 0.002, peak, dur); s.connect(f); f.connect(g); g.connect(bus); s.start(t, Math.random() * 0.5); s.stop(t + dur + 0.05); }
function snare(t, v = 1){ noise(t, 0.2, 'highpass', 1400, 0.42 * v); const o = actx.createOscillator(), g = actx.createGain(); o.type = 'triangle'; o.frequency.setValueAtTime(220, t); o.frequency.exponentialRampToValueAtTime(140, t + 0.08); env(g, t, 0.002, 0.25 * v, 0.1); o.connect(g); g.connect(musicBus); o.start(t); o.stop(t + 0.15); }
function hat(t, open = false, v = 1){ noise(t, open ? 0.22 : 0.035, 'highpass', 7500, (open ? 0.12 : 0.09) * v); }
function bass(t, m, dur, wave = 'sawtooth'){ const o = actx.createOscillator(), o2 = actx.createOscillator(), f = actx.createBiquadFilter(), g = actx.createGain(); o.type = wave; o2.type = 'square'; o.frequency.value = mtof(m); o2.frequency.value = mtof(m) * 0.5; f.type = 'lowpass'; f.Q.value = 7; f.frequency.setValueAtTime(260, t); f.frequency.linearRampToValueAtTime(1100, t + 0.02); f.frequency.exponentialRampToValueAtTime(240, t + dur * 0.9); env(g, t, 0.005, 0.32, dur * 0.9); const g2 = actx.createGain(); g2.gain.value = 0.35; o.connect(f); o2.connect(g2); g2.connect(f); f.connect(g); g.connect(musicBus); o.start(t); o2.start(t); o.stop(t + dur + 0.05); o2.stop(t + dur + 0.05); }
function pad(t, notes, dur){ const f = actx.createBiquadFilter(), g = actx.createGain(); f.type = 'lowpass'; f.frequency.setValueAtTime(500, t); f.frequency.linearRampToValueAtTime(1500, t + dur * 0.5); f.frequency.linearRampToValueAtTime(700, t + dur); g.gain.setValueAtTime(0.0001, t); g.gain.linearRampToValueAtTime(0.055, t + 0.35); g.gain.setValueAtTime(0.055, t + dur - 0.2); g.gain.linearRampToValueAtTime(0.0001, t + dur + 0.3); f.connect(g); g.connect(musicBus); g.connect(delayIn);
  for (const m of notes) for (const det of [-9, 9]) { const o = actx.createOscillator(); o.type = 'sawtooth'; o.frequency.value = mtof(m); o.detune.value = det; o.connect(f); o.start(t); o.stop(t + dur + 0.4); } }
function arp(t, m, dur, v = 1, wave = 'square'){ const o = actx.createOscillator(), f = actx.createBiquadFilter(), g = actx.createGain(); o.type = wave; o.frequency.value = mtof(m); f.type = 'lowpass'; f.frequency.value = wave === 'sawtooth' ? 1800 : 2600; const pk = wave === 'sine' ? 0.14 : wave === 'triangle' ? 0.1 : 0.06; env(g, t, 0.004, pk * v, wave === 'sine' ? dur * 4 : dur * 1.4); o.connect(f); f.connect(g); g.connect(musicBus); g.connect(delayIn); o.start(t); o.stop(t + dur * (wave === 'sine' ? 4.2 : 1.6) + 0.05); }
// heliefektid
function pluck(m, v = 1){ if (!actx) return; const t = actx.currentTime; for (const [type, mul, pk] of [['triangle', 1, 0.22], ['square', 2, 0.05]]) { const o = actx.createOscillator(), g = actx.createGain(); o.type = type; o.frequency.value = mtof(m) * mul; env(g, t, 0.003, pk * v, 0.35); o.connect(g); g.connect(sfxBus); g.connect(delayIn); o.start(t); o.stop(t + 0.4); } }
function crunch(){ if (!actx) return; const t = actx.currentTime; noise(t, 0.35, 'lowpass', 900, 0.6, sfxBus, 3); const o = actx.createOscillator(), g = actx.createGain(); o.type = 'sawtooth'; o.frequency.setValueAtTime(120, t); o.frequency.exponentialRampToValueAtTime(35, t + 0.3); env(g, t, 0.005, 0.4, 0.35); o.connect(g); g.connect(sfxBus); o.start(t); o.stop(t + 0.4); }
function blip(f0, f1, dur = 0.12, type = 'sine', pk = 0.15){ if (!actx) return; const t = actx.currentTime, o = actx.createOscillator(), g = actx.createGain(); o.type = type; o.frequency.setValueAtTime(f0, t); o.frequency.exponentialRampToValueAtTime(f1, t + dur); env(g, t, 0.004, pk, dur); o.connect(g); g.connect(sfxBus); o.start(t); o.stop(t + dur + 0.05); }
function whoosh(){ if (!actx) return; const t = actx.currentTime, s = actx.createBufferSource(); s.buffer = noiseBuf; const f = actx.createBiquadFilter(); f.type = 'bandpass'; f.Q.value = 1.5; f.frequency.setValueAtTime(400, t); f.frequency.exponentialRampToValueAtTime(2500, t + 0.25); const g = actx.createGain(); g.gain.setValueAtTime(0.0001, t); g.gain.linearRampToValueAtTime(0.25, t + 0.12); g.gain.exponentialRampToValueAtTime(0.0001, t + 0.4); s.connect(f); f.connect(g); g.connect(sfxBus); s.start(t); s.stop(t + 0.45); }

/* =========================================================
   LOOD JA RASKUSED
   ========================================================= */
const SONGS = [
  { name: '🌙 Neoonöö', bpm: 100, seed: 11, c1: 0x00e5ff, c2: 0xff2bd6, chords: [[57, 60, 64], [53, 57, 60], [55, 60, 64], [55, 59, 62]], bass: [45, 41, 48, 43], arpW: 'square', arpO: 12, bassW: 'sawtooth', drum: 'four' },
  { name: '🏎️ Kiirtee', bpm: 124, seed: 23, c1: 0xffa000, c2: 0x7c4dff, chords: [[52, 55, 59], [52, 55, 60], [50, 55, 59], [50, 54, 57]], bass: [40, 36, 43, 38], arpW: 'sawtooth', arpO: 12, bassW: 'sawtooth', drum: 'four' },
  { name: '🚀 Hüperruum', bpm: 148, seed: 37, c1: 0x00ff9c, c2: 0x2979ff, chords: [[50, 53, 57], [50, 53, 58], [48, 53, 57], [48, 52, 55]], bass: [38, 34, 41, 36], arpW: 'square', arpO: 12, bassW: 'square', drum: 'four' },
  { name: '🌊 Lainetus', bpm: 90, seed: 41, c1: 0x18ffff, c2: 0x2962ff, chords: [[53, 57, 60], [57, 60, 64], [50, 53, 57], [48, 52, 55]], bass: [41, 45, 38, 36], arpW: 'triangle', arpO: 24, bassW: 'triangle', drum: 'half' },
  { name: '🎮 Arkaad', bpm: 136, seed: 53, c1: 0xffee58, c2: 0x00e676, chords: [[48, 52, 55], [50, 55, 59], [52, 57, 60], [53, 57, 60]], bass: [36, 43, 45, 41], arpW: 'square', arpO: 24, bassW: 'square', drum: 'four' },
  { name: '🔥 Laava', bpm: 130, seed: 67, c1: 0xff3d00, c2: 0xffab00, chords: [[48, 51, 55], [48, 51, 56], [46, 51, 55], [46, 50, 53]], bass: [36, 32, 39, 34], arpW: 'sawtooth', arpO: 12, bassW: 'sawtooth', drum: 'rock' },
  { name: '❄️ Jää', bpm: 116, seed: 71, c1: 0xe0f7ff, c2: 0x40c4ff, chords: [[47, 50, 54], [47, 50, 55], [45, 50, 54], [45, 49, 52]], bass: [35, 31, 38, 33], arpW: 'sine', arpO: 24, bassW: 'triangle', drum: 'four' },
  { name: '👾 Turbo', bpm: 168, seed: 83, c1: 0xd500f9, c2: 0x76ff03, chords: [[55, 58, 62], [55, 58, 63], [53, 58, 62], [53, 57, 60]], bass: [43, 39, 46, 41], arpW: 'square', arpO: 12, bassW: 'sawtooth', drum: 'dnb' }
];
const KICK = { four: [0, 4, 8, 12], dnb: [0, 10], half: [0, 10], rock: [0, 6, 8] }, SNARE = { four: [4, 12], dnb: [4, 12], half: [8], rock: [4, 12] };
const SECTIONS = [
  { n: 'intro', bars: 4, kick: 0, snare: 0, hat: 1, bass: 0, arp: 0, I: 0.25 },
  { n: 'build', bars: 8, kick: 1, snare: 0, hat: 1, bass: 1, arp: 0, I: 0.55 },
  { n: 'drop', bars: 16, kick: 1, snare: 1, hat: 2, bass: 2, arp: 1, I: 1 },
  { n: 'break', bars: 8, kick: 0, snare: 0, hat: 1, bass: 0, arp: 1, I: 0.45 },
  { n: 'drop2', bars: 16, kick: 1, snare: 1, hat: 2, bass: 2, arp: 2, I: 1.1 },
  { n: 'outro', bars: 4, kick: 1, snare: 0, hat: 1, bass: 0, arp: 0, I: 0.15 }
];
const TOTAL_BARS = SECTIONS.reduce((a, s) => a + s.bars, 0);
const SEC_AT = []; { let b = 0; for (const s of SECTIONS) for (let i = 0; i < s.bars; i++) SEC_AT[b++] = { s, i }; }
const LOOP = [[SECTIONS[1], 8], [SECTIONS[2], 16], [SECTIONS[3], 8], [SECTIONS[4], 16]];
function secOf(bar){
  if (!run || !run.endless) return SEC_AT[clamp(bar, 0, TOTAL_BARS - 1)];
  if (bar < 4) return { s: SECTIONS[0], i: bar };
  let b = (bar - 4) % 48; for (const [s, n] of LOOP) { if (b < n) return { s, i: b }; b -= n; }
  return { s: SECTIONS[2], i: 0 };
}
const DIFFS = [
  { id: 0, name: '🧒 Laps', speed: 5.5, orb: 0.42, half: 0, dual: 0, obs: 0.16, obsGap: 6, mine: 0, types: ['bar', 'wall'], colorRule: false, rad: 0.25, loss: 0, barF: 0.76, help: true },
  { id: 1, name: '🙂 Kerge', speed: 7, orb: 0.55, half: 0, dual: 0.05, obs: 0.22, obsGap: 4, mine: 0, types: ['bar', 'wall'], colorRule: false, rad: 0.2, loss: 12, barF: 0.78, help: true },
  { id: 2, name: '😎 Tavaline', speed: 9, orb: 0.7, half: 0.25, dual: 0.15, obs: 0.3, obsGap: 3, mine: 0.07, types: ['bar', 'wall', 'wall'], colorRule: true, rad: 0.17, loss: 18, barF: 0.8, help: false },
  { id: 3, name: '🔥 Raske', speed: 12, orb: 0.85, half: 0.5, dual: 0.3, obs: 0.4, obsGap: 2, mine: 0.14, types: ['bar', 'wall', 'wall'], colorRule: true, rad: 0.15, loss: 25, barF: 0.82, help: false }
];
const WALL_EDGE = [0.1, 0.04, -0.02, -0.05], WALL_PIVOT = 0.8;
const AVATARS = ['🦊 Rebane', '🐼 Panda', '🦄 Ükssarv', '🐯 Tiiger', '🐸 Konn', '🤖 Robot', '🐲 Draakon', '🐧 Pingviin'];
const COL = { L: new THREE.Color(0x00e5ff), R: new THREE.Color(0xff2bd6), any: new THREE.Color(0xffd23f), mine: new THREE.Color(0xff1744), bar: new THREE.Color(0xff8a00), jump: new THREE.Color(0x76ff03), wall: new THREE.Color(0xb388ff), good: new THREE.Color(0x76ff03) };
const mult = c => c >= 32 ? 8 : c >= 16 ? 4 : c >= 8 ? 2 : 1;

/* ---------- tasemete genereerimine (ka lõputu režiimi jaoks tükkhaaval) ---------- */
function initChart(r){
  r.chart = []; r.rnd = mulberry32(r.song.seed * 10 + r.D.id + (r.endless ? 1000 : 0) + (r.flags.jump ? 100 : 0) + (r.flags.spin ? 200 : 0));
  r.g = { lastObs: -99, alt: 0, railEnd: -99, railSide: 0, bar: 2 };
}
function genBars(r, upto){
  const D = r.D, rnd = r.rnd, g = r.g, bd = r.bd, start = r.chart.length;
  const endBar = r.endless ? upto : Math.min(upto, TOTAL_BARS - 2);
  for (; g.bar < endBar; g.bar++) {
    const bar = g.bar, sec = secOf(bar).s, ramp = r.endless ? Math.min(1.5, 1 + bar / 120) : 1, I = sec.I * ramp;
    const steps = I >= 1 && D.half > 0 ? 8 : 4;
    for (let k = 0; k < steps; k++) {
      const beat = bar * 4 + k * (4 / steps), half = steps === 8 && k % 2 === 1, t = beat * bd;
      if (half && rnd() > D.half * Math.min(1.3, ramp)) continue;
      if (!half && I >= 0.5 && beat - g.lastObs >= D.obsGap / Math.min(1.5, ramp) && beat > g.railEnd && rnd() < D.obs * I) {
        const types = D.types.slice(); if (r.flags.jump) types.push('jump', 'jump');
        const ty = types[Math.floor(rnd() * types.length)];
        if (ty === 'wall') {
          const side = rnd() < 0.5 ? -1 : 1; let phi = side > 0 ? 0 : Math.PI;
          if (r.flags.spin && D.id >= 1 && rnd() < 0.5) { const a = 0.35 + rnd() * 0.45; phi = side > 0 ? a : Math.PI - a; }
          r.chart.push({ t, beat, type: 'wall', side, phi });
        } else r.chart.push({ t, beat, type: ty });
        g.lastObs = beat; continue;
      }
      if (!half && I >= 0.5 && beat > g.railEnd + 1 && beat - g.lastObs > 1 && rnd() < 0.07 * I) {
        const len = 2 + Math.floor(rnd() * 3), side = rnd() < 0.5 ? -1 : 1, dur = len * bd;
        r.chart.push({ t, beat, type: 'rail', side, dur, x0: side * (0.3 + rnd() * 0.15), ax: 0.1 + rnd() * 0.1, f0: 0.66 + rnd() * 0.12, af: 0.07 + rnd() * 0.08, w: Math.PI * (1 + rnd()) / dur, ph: rnd() * 6 });
        g.railEnd = beat + len; g.railSide = side; continue;
      }
      if (rnd() > D.orb * Math.max(0.35, Math.min(1.2, I))) continue;
      let side = (g.alt++ % 2) ? 1 : -1; if (rnd() < 0.3) side = -side;
      const inRail = beat <= g.railEnd; if (inRail) side = -g.railSide;
      if (D.mine && I >= 1 && !inRail && rnd() < D.mine) { r.chart.push({ t, beat, type: 'mine', x: side * (0.25 + rnd() * 0.3), f: 0.6 + rnd() * 0.3 }); continue; }
      r.chart.push({ t, beat, type: 'orb', side, x: side * (0.22 + rnd() * 0.33), f: 0.55 + rnd() * 0.4 });
      if (I >= 1 && !inRail && rnd() < D.dual) r.chart.push({ t, beat, type: 'orb', side: -side, x: -side * (0.22 + rnd() * 0.33), f: 0.55 + rnd() * 0.4 });
    }
  }
  fixConflicts(r, Math.max(0, start - 16));
}
function fixConflicts(r, from){
  const ev = r.chart, H = r.H, Hc = H * WALL_PIVOT, e0 = WALL_EDGE[r.D.id];
  const obs = ev.filter((e, i) => i >= from - 30 && (e.type === 'wall' || e.type === 'bar'));
  for (let i = from; i < ev.length; i++) {
    const e = ev[i]; if (e.type !== 'orb' && e.type !== 'mine') continue;
    for (const o of obs) {
      if (Math.abs(e.beat - o.beat) > 1.01) continue;
      if (o.type === 'wall') {
        const nx = Math.cos(o.phi), ny = Math.sin(o.phi), blocked = () => nx * e.x + ny * (e.f * H - Hc) > e0 - 0.2;
        if (blocked()) { e.x = -e.x; e.side = -e.side; }
        if (blocked()) e.f = Math.min(e.f, 0.55);
        if (blocked()) { e.x = -o.side * 0.45; e.side = -o.side; }
      }
      if (o.type === 'bar') e.f = Math.min(e.f, 0.55 - Math.abs(e.beat - o.beat) * 0.05);
    }
  }
  for (let i = ev.length - 1; i > from; i--) { const a = ev[i], b = ev[i - 1]; if (a.type === 'orb' && b.type === 'orb' && Math.abs(a.t - b.t) < 0.01 && Math.sign(a.x) === Math.sign(b.x)) ev.splice(i, 1); }
}
function chartMax(r){
  let combo = 0, max = 0;
  for (const e of r.chart) { if (e.type === 'orb') { combo++; max += 150 * mult(combo); } else if (e.type === 'rail') { combo++; max += Math.round(e.dur / 0.1) * 10 * mult(combo); } else if (e.type !== 'mine') max += 50; }
  return max;
}

/* =========================================================
   OBJEKTID: pallid, seinad, talad, hüppelaserid, miinid, rajad
   ========================================================= */
const orbCoreGeo = new THREE.IcosahedronGeometry(0.075, 2), orbWireGeo = new THREE.IcosahedronGeometry(0.125, 1);
const wallTex = (() => { const cv = document.createElement('canvas'); cv.width = 256; cv.height = 256; const c = cv.getContext('2d'); c.fillStyle = 'rgba(179,136,255,0.10)'; c.fillRect(0, 0, 256, 256); c.strokeStyle = 'rgba(200,160,255,0.55)'; c.lineWidth = 10; for (let i = -256; i < 512; i += 48) { c.beginPath(); c.moveTo(i, 0); c.lineTo(i + 256, 256); c.stroke(); } const t = new THREE.CanvasTexture(cv); t.wrapS = t.wrapT = THREE.RepeatWrapping; t.repeat.set(1 / 1.2, 1 / 1.2); return t; })();
const pool = { orb: [], wall: [], bar: [], jump: [], mine: [], rail: [] };
function beam(col, len = 5.6){
  const g = new THREE.Group();
  g.add(new THREE.Mesh(new THREE.CylinderGeometry(0.035, 0.035, len, 12).rotateZ(Math.PI / 2), new THREE.MeshBasicMaterial({ color: new THREE.Color(col).lerp(new THREE.Color(0xffffff), 0.6) })));
  g.add(new THREE.Mesh(new THREE.CylinderGeometry(0.14, 0.14, len, 12).rotateZ(Math.PI / 2), new THREE.MeshBasicMaterial({ color: col, transparent: true, opacity: 0.35, blending: THREE.AdditiveBlending, depthWrite: false })));
  return g;
}
function makeObj(type){
  const g = new THREE.Group(); g.userData.type = type; scene.add(g);
  if (type === 'orb') {
    g.userData.core = new THREE.Mesh(orbCoreGeo, new THREE.MeshBasicMaterial({ color: 0xffffff }));
    g.userData.wire = new THREE.Mesh(orbWireGeo, new THREE.MeshBasicMaterial({ color: 0xffffff, wireframe: true, transparent: true, opacity: 0.7, blending: THREE.AdditiveBlending, depthWrite: false }));
    g.add(g.userData.core, g.userData.wire);
  } else if (type === 'mine') {
    g.add(new THREE.Mesh(new THREE.OctahedronGeometry(0.09), new THREE.MeshBasicMaterial({ color: 0xff1744 })));
    for (const d of [[1, 0, 0], [-1, 0, 0], [0, 1, 0], [0, -1, 0], [0, 0, 1], [0, 0, -1]]) { const s = new THREE.Mesh(new THREE.ConeGeometry(0.022, 0.09, 6), new THREE.MeshBasicMaterial({ color: 0xff5252 })); s.position.set(d[0] * 0.1, d[1] * 0.1, d[2] * 0.1); s.quaternion.setFromUnitVectors(V3(0, 1, 0), V3(...d)); g.add(s); }
    g.userData.wire = new THREE.Mesh(new THREE.IcosahedronGeometry(0.16, 0), new THREE.MeshBasicMaterial({ color: 0xff1744, wireframe: true, transparent: true, opacity: 0.6, blending: THREE.AdditiveBlending, depthWrite: false })); g.add(g.userData.wire);
  } else if (type === 'wall') {
    const t = wallTex.clone(); t.needsUpdate = true;
    const fill = new THREE.Mesh(new THREE.BufferGeometry(), new THREE.MeshBasicMaterial({ map: t, transparent: true, blending: THREE.AdditiveBlending, depthWrite: false, side: THREE.DoubleSide }));
    const edgeCore = new THREE.Mesh(new THREE.BoxGeometry(1, 0.035, 0.035), new THREE.MeshBasicMaterial({ color: 0xffffff }));
    const edgeGlow = new THREE.Mesh(new THREE.BoxGeometry(1, 0.2, 0.2), new THREE.MeshBasicMaterial({ color: 0xb388ff, transparent: true, opacity: 0.3, blending: THREE.AdditiveBlending, depthWrite: false }));
    g.add(fill, edgeCore, edgeGlow); Object.assign(g.userData, { fill, edgeCore, edgeGlow });
  } else if (type === 'bar') g.add(beam(0xff8a00));
  else if (type === 'jump') { g.add(beam(0x76ff03)); for (const x of [-2.6, 2.6]) { const p = new THREE.Mesh(new THREE.BoxGeometry(0.08, 0.3, 0.08), new THREE.MeshBasicMaterial({ color: 0x76ff03 })); p.position.set(x, -0.08, 0); g.add(p); } }
  else if (type === 'rail') {
    const geo = new THREE.BufferGeometry(); geo.setAttribute('position', new THREE.BufferAttribute(new Float32Array(260 * 3), 3));
    const line = new THREE.Line(geo, new THREE.LineBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.9, blending: THREE.AdditiveBlending, depthWrite: false })); line.frustumCulled = false;
    const headM = new THREE.Mesh(new THREE.TorusGeometry(0.11, 0.018, 8, 32), new THREE.MeshBasicMaterial({ color: 0xffffff }));
    g.add(line, headM); Object.assign(g.userData, { line, geo, headM });
  }
  return g;
}
function getObj(type){ const p = pool[type]; const o = p.find(x => !x.visible) || (p.push(makeObj(type)), p[p.length - 1]); o.visible = true; o.position.set(0, 0, 0); o.rotation.set(0, 0, 0); return o; }
// viltuse seina kuju: tunneli ring lõigatuna pooltasandiga
function clipPoly(pts, f){ const out = []; for (let i = 0; i < pts.length; i++) { const a = pts[i], b = pts[(i + 1) % pts.length], fa = f(a), fb = f(b); if (fa >= 0) out.push(a); if ((fa >= 0) !== (fb >= 0)) { const k = fa / (fa - fb); out.push([a[0] + (b[0] - a[0]) * k, a[1] + (b[1] - a[1]) * k]); } } return out; }
function buildWall(o, e){
  const R = TUN_R - 0.05, Hc = run.H * WALL_PIVOT, e0 = WALL_EDGE[run.D.id], nx = Math.cos(e.phi), ny = Math.sin(e.phi);
  let poly = []; for (let i = 0; i < 72; i++) { const a = i / 72 * Math.PI * 2; poly.push([Math.cos(a) * R, TUN_Y + Math.sin(a) * R]); }
  poly = clipPoly(poly, p => p[1]);
  poly = clipPoly(poly, p => nx * p[0] + ny * (p[1] - Hc) - e0);
  if (poly.length >= 3) { const sh = new THREE.Shape(poly.map(p => new THREE.Vector2(p[0], p[1]))); o.userData.fill.geometry.dispose(); o.userData.fill.geometry = new THREE.ShapeGeometry(sh); }
  // serv: joone ja ringi lõikepunktid
  const ox = nx * e0, oy = Hc + ny * e0, tx = -ny, ty = nx, dx = ox, dy = oy - TUN_Y, b = dx * tx + dy * ty, c = dx * dx + dy * dy - R * R, disc = Math.sqrt(Math.max(0, b * b - c));
  let p0 = [ox + tx * (-b - disc), oy + ty * (-b - disc)], p1 = [ox + tx * (-b + disc), oy + ty * (-b + disc)];
  const toFloor = (p, q) => { if (p[1] >= 0) return p; const k = q[1] / (q[1] - p[1]); return [q[0] + (p[0] - q[0]) * k, 0]; };
  p0 = toFloor(p0, p1); p1 = toFloor(p1, p0);
  const L = Math.hypot(p1[0] - p0[0], p1[1] - p0[1]), ang = Math.atan2(p1[1] - p0[1], p1[0] - p0[0]);
  for (const m of [o.userData.edgeCore, o.userData.edgeGlow]) { m.scale.set(L, 1, 1); m.position.set((p0[0] + p1[0]) / 2, (p0[1] + p1[1]) / 2, 0); m.rotation.z = ang; }
  e.nx = nx; e.ny = ny; e.e0 = e0; e.Hc = Hc;
}

/* =========================================================
   KÄED: pultidega või paljaste kätega
   ========================================================= */
const handFactory = new XRHandModelFactory();
const raycaster = new THREE.Raycaster();
const hands = [0, 1].map(i => {
  const ctrl = renderer.xr.getController(i), grip = renderer.xr.getControllerGrip(i), hand = renderer.xr.getHand(i);
  rig.add(ctrl); rig.add(grip); rig.add(hand);
  const hm = handFactory.createHandModel(hand, 'spheres'); hand.add(hm);
  const core = new THREE.Mesh(new THREE.SphereGeometry(0.03, 16, 12), new THREE.MeshBasicMaterial({ color: 0xffffff })); grip.add(core);
  const ray = new THREE.Mesh(new THREE.CylinderGeometry(0.003, 0.003, 1, 6).translate(0, -0.5, 0).rotateX(Math.PI / 2), new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.7, blending: THREE.AdditiveBlending, depthWrite: false }));
  ray.visible = false; ctrl.add(ray);
  const h = { i, kind: 'xr', ctrl, grip, hand, hm, core, ray, side: null, isHand: false, connected: false, pos: V3(), prev: V3(), vel: V3(), hasPrev: false, hover: null, poke: null, colored: false, gp: null };
  ctrl.addEventListener('connected', e => { h.connected = true; h.side = e.data.handedness; h.isHand = !!e.data.hand; h.gp = e.data.gamepad; core.visible = !h.isHand; h.colored = false; h.hasPrev = false; });
  ctrl.addEventListener('disconnected', () => { h.connected = false; h.hasPrev = false; });
  ctrl.addEventListener('selectstart', () => { audioInit(); if (h.hover) clickBtn(h.hover, h); });
  return h;
});
const mouseHand = { kind: 'mouse', side: 'any', pos: V3(0, 1.3, -0.45), prev: V3(), vel: V3(), hasPrev: false, connected: true, hover: null };
const handCol = h => h.side === 'left' ? COL.L : h.side === 'right' ? COL.R : COL.any;
function handPoint(h, out){
  if (h.kind === 'mouse') return out.copy(h.pos);
  if (h.isHand && h.hand.joints) { const j = h.hand.joints['middle-finger-metacarpal'] || h.hand.joints['wrist']; if (j) return j.getWorldPosition(out); }
  return h.grip.getWorldPosition(out);
}
function pulse(h, s = 0.5, ms = 50){ try { const gp = h.gp; if (gp && gp.hapticActuators && gp.hapticActuators[0]) gp.hapticActuators[0].pulse(s, ms); } catch (e) {} }
function colorHands(){
  for (const h of hands) {
    if (!h.connected) continue;
    const c = handCol(h); h.core.material.color.copy(c).lerp(new THREE.Color(0xffffff), 0.35);
    if (h.isHand && !h.colored) { const mc = h.hm.motionController, mesh = mc && mc.handMesh; if (mesh && mesh.material) { mesh.material.color.copy(c).multiplyScalar(0.3); if (mesh.material.emissive) mesh.material.emissive.copy(c); mesh.material.needsUpdate = true; h.colored = true; } }
  }
}
const mouseNDC = new THREE.Vector2(0, -0.1);
addEventListener('mousemove', e => { mouseNDC.set(e.clientX / innerWidth * 2 - 1, -(e.clientY / innerHeight) * 2 + 1); });
renderer.domElement.addEventListener('mousedown', () => { audioInit(); if (mouseHand.hover) clickBtn(mouseHand.hover, mouseHand); });
const keys = {};
let deskJump = 0;
addEventListener('keydown', e => {
  keys[e.code] = true;
  if (e.code === 'Escape') togglePause();
  if (e.code === 'Space') { e.preventDefault(); if (mode === 'menu') startRun(true); else if (mode === 'play' && deskJump <= 0) deskJump = 0.6; }
});
addEventListener('keyup', e => { keys[e.code] = false; });

/* =========================================================
   KASUTAJALIIDES (VR paneelid)
   ========================================================= */
class Panel {
  constructor(w, h, ppm = 520){
    this.w = w; this.h = h; this.cv = document.createElement('canvas'); this.cv.width = Math.round(w * ppm); this.cv.height = Math.round(h * ppm); this.ctx = this.cv.getContext('2d');
    this.tex = new THREE.CanvasTexture(this.cv); this.tex.colorSpace = THREE.SRGBColorSpace; this.tex.anisotropy = 4;
    this.mesh = new THREE.Mesh(new THREE.PlaneGeometry(w, h), new THREE.MeshBasicMaterial({ map: this.tex, transparent: true, fog: false, depthWrite: false })); this.mesh.renderOrder = 30;
  }
  draw(fn){ const c = this.ctx; c.clearRect(0, 0, this.cv.width, this.cv.height); fn(c, this.cv.width, this.cv.height); this.tex.needsUpdate = true; }
}
function neonText(c, txt, x, y, size, col, blur = 18, weight = 'bold'){
  c.font = `${weight} ${size}px ${FONT}`; c.textAlign = 'center'; c.textBaseline = 'middle';
  c.shadowColor = col; c.shadowBlur = blur; c.fillStyle = col; c.fillText(txt, x, y); c.shadowBlur = blur * 0.4; c.fillStyle = '#ffffff'; c.globalAlpha = 0.85; c.fillText(txt, x, y); c.globalAlpha = 1; c.shadowBlur = 0;
}
function panelBg(c, W, H, col = '#00e5ff'){
  const g = c.createLinearGradient(0, 0, 0, H); g.addColorStop(0, 'rgba(18,4,40,0.93)'); g.addColorStop(1, 'rgba(6,1,18,0.93)');
  roundRect(c, 14, 14, W - 28, H - 28, 40); c.fillStyle = g; c.fill(); c.shadowColor = col; c.shadowBlur = 12; c.lineWidth = 6; c.strokeStyle = col; c.stroke(); c.shadowBlur = 0;
}
const hexA = (h, a) => { const c = new THREE.Color(h); return `rgba(${c.r * 255 | 0},${c.g * 255 | 0},${c.b * 255 | 0},${a})`; };
const buttons = [];
function makeButton(parent, w, h, x, y, label, onClick, col = '#00e5ff'){
  const p = new Panel(w, h, 600); p.mesh.position.set(x, y, 0.01); p.mesh.renderOrder = 31; parent.add(p.mesh);
  const b = { p, w, h, label, col, onClick, sel: false, hover: false, sub: '' };
  b.draw = () => p.draw((c, W, H) => {
    const m = Math.min(W, H) * 0.14; roundRect(c, m, m, W - m * 2, H - m * 2, Math.min(W, H) * 0.25);
    c.fillStyle = b.sel ? hexA(b.col, 0.45) : b.hover ? hexA(b.col, 0.25) : 'rgba(20,6,45,0.85)'; c.fill();
    c.shadowColor = b.col; c.shadowBlur = Math.min(m * 0.8, b.sel || b.hover ? 22 : 10); c.lineWidth = b.sel ? 10 : 6; c.strokeStyle = b.col; c.stroke(); c.shadowBlur = 0;
    let fs = H * (b.sub ? 0.32 : 0.42); c.font = `bold ${fs}px ${FONT}`; while (c.measureText(b.label).width > W * 0.86 && fs > 10) { fs--; c.font = `bold ${fs}px ${FONT}`; }
    c.textAlign = 'center'; c.textBaseline = 'middle'; c.fillStyle = '#fff'; c.shadowColor = b.col; c.shadowBlur = 12;
    c.fillText(b.label, W / 2, b.sub ? H * 0.38 : H * 0.53); c.shadowBlur = 0;
    if (b.sub) { let f2 = H * 0.21; c.font = `${f2}px ${FONT}`; while (c.measureText(b.sub).width > W * 0.86 && f2 > 8) { f2--; c.font = `${f2}px ${FONT}`; } c.fillStyle = 'rgba(230,220,255,0.85)'; c.fillText(b.sub, W / 2, H * 0.72); }
  });
  b.draw(); p.mesh.userData.btn = b; buttons.push(b); return b;
}
function clickBtn(b, h){ if (!isShown(b.p.mesh)) return; blip(900, 1300, 0.07, 'triangle', 0.12); if (h && h.kind === 'xr') pulse(h, 0.4, 30); b.onClick(); }
function isShown(o){ while (o) { if (!o.visible) return false; o = o.parent; } return true; }
function placeInFront(g, dist, yOff = 0){
  camera.getWorldPosition(head); camera.getWorldDirection(tmpD); tmpD.y = 0; if (tmpD.lengthSq() < 1e-4) tmpD.set(0, 0, -1); tmpD.normalize();
  g.position.copy(head).addScaledVector(tmpD, dist); g.position.y = head.y + yOff; g.rotation.set(0, Math.atan2(-tmpD.x, -tmpD.z), 0);
}

/* ---------- menüü ---------- */
const settings = Object.assign({ song: 0, diff: 1, mode: 'song', jump: false, spin: true, duel: [{ av: 0, diff: 2 }, { av: 2, diff: 0 }] }, store('nt_settings') || {});
if (settings.song >= SONGS.length) settings.song = 0;
const best = store('nt_best') || {};
const wins = store('nt_wins') || {};
const menuG = new THREE.Group(); scene.add(menuG);
const MW = 1.62, MH = 1.42;
const menuP = new Panel(MW, MH, 500); menuG.add(menuP.mesh);
const my = y => (MH / 2 - y) / MH;
function drawMenu(){
  menuP.draw((c, W, H) => {
    panelBg(c, W, H, '#ff2bd6');
    const g = c.createLinearGradient(W * 0.2, 0, W * 0.8, 0); g.addColorStop(0, '#00e5ff'); g.addColorStop(1, '#ff2bd6');
    c.font = `900 ${H * 0.085}px ${FONT}`; c.textAlign = 'center'; c.textBaseline = 'middle'; c.shadowColor = '#ff2bd6'; c.shadowBlur = 30; c.fillStyle = g; c.fillText('NEOONTUNNEL', W / 2, H * my(0.6)); c.shadowBlur = 0;
    neonText(c, 'LUGU', W / 2, H * my(0.335), H * 0.026, '#00e5ff', 8);
    neonText(c, settings.mode === 'duel' ? '⚔️ MÄNGIJAD (klõpsa, et vahetada)' : 'RASKUS', W / 2, H * my(0.015), H * 0.026, '#00e5ff', 8);
    c.fillStyle = 'rgba(230,220,255,0.82)'; c.font = `${H * 0.024}px ${FONT}`;
    const txt = settings.mode === 'duel' ? '⚔️ Mängite kordamööda samade prillidega. Võidab suurem % – raskusaste on arvestatud!'
      : settings.mode === 'endless' ? '♾️ Lõputu: läheb aina kiiremaks, kestab kuni energia saab otsa'
      : DIFFS[settings.diff].colorRule ? '🔵 Sinine pall = VASAK käsi   •   🩷 Roosa pall = PAREM käsi   •   🌀 rada = hoia kätt sees' : '🟡 Palle võib lüüa ükskõik kumma käega   •   🌀 rada = hoia kätt sees';
    c.fillText(txt, W / 2, H * my(-0.355));
    c.fillStyle = 'rgba(200,190,240,0.6)'; c.font = `${H * 0.02}px ${FONT}`;
    c.fillText('Pultidega: päästik = vali • B/Y = paus • A/X = hüppa     Kätega: näpista = vali, sõrmega vajuta • vasaku randme ⏸ = paus', W / 2, H * my(-0.65));
  });
}
const modeBtns = [['song', '🎵 Lugu'], ['endless', '♾️ Lõputu'], ['duel', '⚔️ Duell']].map(([m, l], i) => makeButton(menuG, 0.44, 0.1, -0.47 + i * 0.47, 0.465, l, () => { settings.mode = m; refreshMenu(); }, ['#00e5ff', '#76ff03', '#ff1744'][i]));
const songBtns = SONGS.map((s, i) => makeButton(menuG, 0.37, 0.125, -0.585 + (i % 4) * 0.39, 0.25 - Math.floor(i / 4) * 0.135, s.name, () => { settings.song = i; refreshMenu(); }, hex(s.c1)));
const diffG = new THREE.Group(); menuG.add(diffG);
const diffBtns = DIFFS.map((d, i) => makeButton(diffG, 0.36, 0.1, -0.555 + i * 0.37, -0.075, d.name, () => { settings.diff = i; refreshMenu(); }, ['#76ff03', '#00e5ff', '#ffd23f', '#ff1744'][i]));
const duelG = new THREE.Group(); menuG.add(duelG);
const duelBtns = [];
for (let p = 0; p < 2; p++) {
  const x0 = p === 0 ? -0.4 : 0.4;
  duelBtns.push(makeButton(duelG, 0.38, 0.13, x0 - 0.19, -0.085, '', () => { settings.duel[p].av = (settings.duel[p].av + 1) % AVATARS.length; if (settings.duel[p].av === settings.duel[1 - p].av) settings.duel[p].av = (settings.duel[p].av + 1) % AVATARS.length; refreshMenu(); }, p ? '#ff2bd6' : '#00e5ff'));
  duelBtns.push(makeButton(duelG, 0.36, 0.13, x0 + 0.19, -0.085, '', () => { settings.duel[p].diff = (settings.duel[p].diff + 1) % DIFFS.length; refreshMenu(); }, p ? '#ff2bd6' : '#00e5ff'));
}
const jumpBtn = makeButton(menuG, 0.44, 0.09, -0.24, -0.245, '', () => { settings.jump = !settings.jump; refreshMenu(); }, '#76ff03');
const spinBtn = makeButton(menuG, 0.44, 0.09, 0.24, -0.245, '', () => { settings.spin = !settings.spin; refreshMenu(); }, '#b388ff');
const startBtn = makeButton(menuG, 0.56, 0.14, 0, -0.5, '▶  ALUSTA', () => startRun(true), '#ff2bd6');
function bestKey(i = settings.song, d = settings.diff){ return (settings.mode === 'endless' ? 'e' : '') + `${i}_${d}`; }
function refreshMenu(){
  store('nt_settings', settings);
  modeBtns.forEach((b, i) => { b.sel = settings.mode === ['song', 'endless', 'duel'][i]; b.draw(); });
  songBtns.forEach((b, i) => { b.sel = settings.song === i; const r = settings.mode === 'duel' ? null : best[bestKey(i)]; b.sub = `${SONGS[i].bpm} BPM${r ? ' • 🏆 ' + r.score : ''}`; b.draw(); });
  diffG.visible = settings.mode !== 'duel'; duelG.visible = settings.mode === 'duel';
  diffBtns.forEach((b, i) => { b.sel = settings.diff === i; b.draw(); });
  for (let p = 0; p < 2; p++) {
    const d = settings.duel[p], a = duelBtns[p * 2], df = duelBtns[p * 2 + 1];
    a.label = AVATARS[d.av]; a.sub = `Mängija ${p + 1} • 🏆 ${wins[d.av] || 0} võitu`; a.draw();
    df.label = DIFFS[d.diff].name; df.sub = 'raskus'; df.draw();
  }
  jumpBtn.label = settings.jump ? '🦘 Hüpped: SEES' : '🦘 Hüpped: väljas'; jumpBtn.sel = settings.jump; jumpBtn.draw();
  spinBtn.label = settings.spin ? '🔄 Pöörlemine: SEES' : '🔄 Pöörlemine: väljas'; spinBtn.sel = settings.spin; spinBtn.draw();
  const s = SONGS[settings.song]; U.uC1.value.set(s.c1); U.uC2.value.set(s.c2); mountains.forEach(m => m.wire.material.color.set(s.c2));
  drawMenu();
}
/* ---------- paus ja lõpp ---------- */
const pauseG = new THREE.Group(); pauseG.visible = false; scene.add(pauseG);
const pauseP = new Panel(0.9, 0.62); pauseG.add(pauseP.mesh);
pauseP.draw((c, W, H) => { panelBg(c, W, H, '#00e5ff'); neonText(c, '⏸ PAUS', W / 2, H * 0.14, H * 0.12, '#00e5ff'); });
makeButton(pauseG, 0.6, 0.12, 0, 0.08, '▶ Jätka', () => resume(), '#76ff03');
makeButton(pauseG, 0.6, 0.12, 0, -0.07, '🔁 Alusta uuesti', () => startRun(false), '#ffd23f');
makeButton(pauseG, 0.6, 0.12, 0, -0.22, '🏠 Menüü', () => goMenu(), '#ff2bd6');
const endG = new THREE.Group(); endG.visible = false; scene.add(endG);
const endP = new Panel(1.2, 0.9); endG.add(endP.mesh);
const endBtns = [makeButton(endG, 0.46, 0.13, -0.25, -0.33, '', () => {}, '#ffd23f'), makeButton(endG, 0.42, 0.13, 0.25, -0.33, '🏠 Menüü', () => goMenu(), '#ff2bd6')];
function setEndBtn(i, label, fn, col){ const b = endBtns[i]; b.label = label; b.onClick = fn; if (col) b.col = col; b.draw(); }
/* ---------- HUD ---------- */
const hud = new Panel(1.9, 0.42, 420); hud.mesh.renderOrder = 25; scene.add(hud.mesh); hud.mesh.visible = false;
const bigMsg = new Panel(2.4, 0.5, 300); bigMsg.mesh.renderOrder = 26; scene.add(bigMsg.mesh); bigMsg.mesh.visible = false;
let bigMsgT = 0;
function showBig(txt, col = '#ffffff', dur = 0.9){ bigMsg.draw((c, W, H) => { let fs = H * 0.55; c.font = `900 ${fs}px ${FONT}`; while (c.measureText(txt).width > W * 0.92 && fs > 20) { fs -= 2; c.font = `900 ${fs}px ${FONT}`; } neonText(c, txt, W / 2, H / 2, fs, col, 30, '900'); }); bigMsg.mesh.visible = true; bigMsgT = dur; }
const pops = Array.from({ length: 10 }, () => { const p = new Panel(0.5, 0.16, 400); p.mesh.renderOrder = 27; p.mesh.visible = false; scene.add(p.mesh); return { p, t: 0 }; });
let popI = 0;
function popText(pos, txt, col){ const o = pops[popI = (popI + 1) % pops.length]; o.p.draw((c, W, H) => neonText(c, txt, W / 2, H / 2, H * 0.55, col, 16, '900')); o.p.mesh.position.copy(pos); o.p.mesh.visible = true; o.t = 0.8; camera.getWorldPosition(head); o.p.mesh.lookAt(head); }
const hurt = new THREE.Mesh(new THREE.SphereGeometry(0.25, 16, 12), new THREE.MeshBasicMaterial({ color: 0xff0033, transparent: true, opacity: 0, side: THREE.BackSide, depthTest: false, depthWrite: false, fog: false }));
hurt.renderOrder = 999; camera.add(hurt);
const wristBtnG = new THREE.Group(); scene.add(wristBtnG); wristBtnG.visible = false;
const wristBtn = makeButton(wristBtnG, 0.07, 0.07, 0, 0, '⏸', () => togglePause(), '#00e5ff');

/* =========================================================
   MÄNGU OLEK
   ========================================================= */
let mode = 'menu', run = null, songStart = 0, menuDist = 0, lastHud = '', duel = null, tunnelRoll = 0, spinSpeed = 0.03;
function songTime(){ return actx ? actx.currentTime - songStart : 0; }
function hideUI(){ menuG.visible = pauseG.visible = endG.visible = false; }
function goMenu(){
  if (actx && actx.state === 'suspended') actx.resume();
  stopMusic(); clearObjs(); hideUI(); run = null; duel = null; mode = 'menu'; hud.mesh.visible = false; bigMsg.mesh.visible = false;
  refreshMenu(); placeInFront(menuG, 1.1, -0.15); menuG.visible = true;
}
function clearObjs(){ for (const k in pool) for (const o of pool[k]) o.visible = false; }
function startRun(fresh){
  audioInit(); if (actx.state === 'suspended') actx.resume();
  stopMusic(); clearObjs(); hideUI();
  if (settings.mode === 'duel' && (fresh || !duel)) duel = { cur: 0, res: [] };
  if (settings.mode !== 'duel') duel = null;
  const song = SONGS[settings.song], D = DIFFS[duel ? settings.duel[duel.cur].diff : settings.diff], endless = settings.mode === 'endless';
  camera.getWorldPosition(head);
  if (renderer.xr.isPresenting) { rig.position.x -= head.x; rig.position.z -= head.z; }
  const H = renderer.xr.isPresenting ? clamp(head.y, 0.95, 2.05) : 1.6;
  run = { song, D, H, endless, flags: { jump: settings.jump, spin: settings.spin }, idx: 0, active: [], score: 0, combo: 0, maxCombo: 0, hits: 0, miss: 0, wrong: 0, obsHit: 0, dodged: 0, rails: 0, railsGood: 0,
    energy: 100, loss: D.loss || (endless ? 8 : 0), bd: 60 / song.bpm, len: endless ? Infinity : TOTAL_BARS * 4 * 60 / song.bpm, fever: 0, helped: {}, airT: -9, speed: D.speed };
  initChart(run); genBars(run, endless ? 14 : TOTAL_BARS);
  if (!endless) { run.max = chartMax(run); run.orbs = run.chart.filter(e => e.type === 'orb').length; }
  delayIn.userData.dl.delayTime.value = run.bd * 0.75;
  songStart = actx.currentTime + 3.2; schedStep = 0;
  for (let i = 0; i < 3; i++) { const t = songStart - 3 + i; const o = actx.createOscillator(), g = actx.createGain(); o.type = 'square'; o.frequency.value = 660; env(g, t, 0.005, 0.1, 0.15); o.connect(g); g.connect(sfxBus); o.start(t); o.stop(t + 0.2); }
  { const o = actx.createOscillator(), g = actx.createGain(); o.type = 'square'; o.frequency.value = 1320; env(g, songStart, 0.005, 0.12, 0.35); o.connect(g); g.connect(sfxBus); o.start(songStart); o.stop(songStart + 0.4); }
  mode = 'play'; hud.mesh.visible = true;
  hud.mesh.position.set(0, run.H + 0.55, -3.3); hud.mesh.rotation.set(0.12, 0, 0);
  bigMsg.mesh.position.set(0, run.H + 0.05, -2.6);
  U.uC1.value.set(song.c1); U.uC2.value.set(song.c2); mountains.forEach(m => m.wire.material.color.set(song.c2));
  lastHud = '';
  if (duel) setTimeout(() => { if (run && mode === 'play') showBig(`${AVATARS[settings.duel[duel.cur].av]} – SINU KORD!`, duel.cur ? '#ff2bd6' : '#00e5ff', 1.6); }, 50);
}
function togglePause(){ if (mode === 'play') pauseGame(); else if (mode === 'pause') resume(); }
function pauseGame(){ if (mode !== 'play') return; mode = 'pause'; if (actx) actx.suspend(); placeInFront(pauseG, 0.85, -0.15); pauseG.visible = true; }
function resume(){ if (mode !== 'pause') return; pauseG.visible = false; mode = 'play'; if (actx) actx.resume(); }
function stopMusic(){ schedStep = 1e9; }

/* ---------- muusika ajastaja ---------- */
let schedStep = 1e9;
function scheduleMusic(){
  if (!actx || !run || mode === 'menu') return;
  const st = run.bd / 4, ahead = actx.currentTime + 0.15, song = run.song, maxStep = run.endless ? Infinity : TOTAL_BARS * 16;
  while (schedStep < maxStep && songStart + schedStep * st < ahead) {
    const s = schedStep++, t = songStart + s * st; if (t < actx.currentTime - 0.05) continue;
    const bar = Math.floor(s / 16), pos = s % 16, info = secOf(bar), sec = info.s, ch = song.chords[bar % 4], root = song.bass[bar % 4];
    const lastBuild = sec.n === 'build' && info.i >= sec.bars - 2;
    if (sec.kick && KICK[song.drum].includes(pos)) kick(t);
    if (sec.snare && SNARE[song.drum].includes(pos)) snare(t);
    if (lastBuild && (info.i === sec.bars - 1 ? true : pos % 2 === 0)) snare(t, 0.35 + pos / 16 * 0.6);
    if (sec.hat === 1 && pos % 4 === 2) hat(t);
    if (sec.hat === 2) { const every = song.drum === 'dnb' ? 1 : 2; if (pos % every === 0) hat(t, false, pos % 4 === 2 ? 1.2 : 0.6); if (pos === 14) hat(t, true); }
    if (sec.bass === 1 && pos % 2 === 0) bass(t, root, st * 1.8, song.bassW);
    if (sec.bass === 2) bass(t, pos % 4 === 2 ? root + 12 : root, st * 0.95, song.bassW);
    if (pos === 0) pad(t, ch, st * 16);
    if (sec.arp) { const step = sec.arp === 2 ? 1 : 2; if (pos % step === 0) { const seq = [0, 1, 2, 3, 2, 1, 0, 2]; const k = seq[(pos / step) % 8]; const m = k === 3 ? ch[0] + 12 : ch[k]; arp(t, m + song.arpO, st * step * 0.9, sec.arp === 2 ? 1 : 0.8, song.arpW); } }
  }
}
setInterval(() => { if (mode === 'play') scheduleMusic(); }, 30);

/* ---------- objektide loogika ---------- */
const SPAWN = 48, ORB_Z = -0.45;
function spawn(e){
  const o = getObj(e.type); o.userData.e = e; e.o = o; e.done = false; e.prevZ = null;
  const D = run.D, H = run.H;
  if (e.type === 'orb' || e.type === 'rail') {
    const c = D.colorRule ? (e.side < 0 ? COL.L : COL.R) : COL.any; e.col = c; e.need = D.colorRule ? (e.side < 0 ? 'left' : 'right') : 'any';
  }
  if (e.type === 'orb') { o.userData.core.material.color.copy(e.col).lerp(new THREE.Color(0xffffff), 0.5); o.userData.wire.material.color.copy(e.col); o.scale.setScalar(D.id === 0 ? 1.35 : 1); e.y = e.f * H; }
  else if (e.type === 'mine') e.y = e.f * H;
  else if (e.type === 'wall') buildWall(o, e);
  else if (e.type === 'bar') e.y = D.barF * H;
  else if (e.type === 'jump') e.y = 0.2;
  else if (e.type === 'rail') { o.userData.line.material.color.copy(e.col); o.userData.headM.material.color.copy(e.col).lerp(new THREE.Color(0xffffff), 0.4); e.held = 0; e.acc = 0; }
}
const objZ = (e, target) => target - (e.t - songTime()) * run.speed;
const railX = (e, tau) => e.x0 + e.ax * Math.sin((tau - e.t) * e.w + e.ph);
const railY = (e, tau) => run.H * (e.f0 + e.af * Math.sin((tau - e.t) * e.w * 0.7 + e.ph * 1.3));
function playHands(){ const list = []; if (renderer.xr.isPresenting) { for (const h of hands) if (h.connected) list.push(h); } else list.push(mouseHand); return list; }
function chordNote(){ const ch = run.song.chords[Math.floor(Math.max(0, songTime()) / (run.bd * 4)) % 4]; return ch[Math.floor(Math.random() * 3)] + 24; }
const okHand = (e, h) => e.need === 'any' || h.kind !== 'xr' || h.side === e.need;
function hitOrb(e, h, pos){
  e.done = true; e.o.visible = false;
  if (!okHand(e, h)) { run.wrong++; run.combo = 0; popText(pos, '✋ Vale käsi', '#ffffff'); blip(300, 180, 0.15, 'square', 0.1); burst(pos, new THREE.Color(0x888888), 14, 2); pulse(h, 0.3, 40); return; }
  run.hits++; comboUp(e.col);
  const sp = h.vel.length(), power = sp > 1.6 ? 50 : Math.round(clamp((sp - 0.3) / 1.3, 0, 1) * 50);
  const pts = (100 + power) * mult(run.combo); run.score += pts;
  run.energy = Math.min(100, run.energy + 1.5);
  popText(pos, power >= 50 ? `💥 +${pts}` : `+${pts}`, hex(e.col));
  burst(pos, e.col, power >= 50 ? 40 : 26, power >= 50 ? 4.5 : 3, 0.05); burst(pos, new THREE.Color(0xffffff), 8, 1.5, 0.035);
  pluck(chordNote(), 0.9 + power / 100); pulse(h, 0.6 + power / 150, 45);
}
function comboUp(col){
  run.combo++; run.maxCombo = Math.max(run.maxCombo, run.combo);
  const m = mult(run.combo); if (m > 1 && mult(run.combo - 1) < m) { showBig(`COMBO x${m}!`, hex(col), 0.9); blip(600, 1800, 0.3, 'sawtooth', 0.08); }
}
function missOrb(e){ e.done = true; run.miss++; if (run.combo >= 8) blip(400, 150, 0.25, 'sawtooth', 0.06); run.combo = 0; if (run.loss) run.energy -= 2; }
function hitObstacle(e, txt){
  run.obsHit++; run.combo = 0; run.energy -= e.type === 'mine' ? run.loss * 0.6 : run.loss;
  hurt.material.opacity = 0.45; crunch(); for (const h of hands) if (h.connected) pulse(h, 1, 220);
  showBig(txt, '#ff1744', 0.8);
}
function dodge(e, pos){ run.dodged++; run.score += 50; whoosh(); popText(pos, '✔ +50', '#76ff03'); }
const HELP = { bar: ['⬇ KÜKITA!', '#ff8a00'], jump: ['🦘 HÜPPA!', '#76ff03'] };
function updateRun(dt){
  const t = songTime(), D = run.D;
  if (run.endless) { run.speed = D.speed * (1 + Math.min(0.7, Math.max(0, t) / 300)); const curBar = Math.floor(Math.max(0, t) / (run.bd * 4)); if (run.g.bar < curBar + 10) genBars(run, run.g.bar + 8); }
  const speed = run.speed;
  while (run.idx < run.chart.length && (run.chart[run.idx].t - t) * speed <= SPAWN) { const e = run.chart[run.idx++]; spawn(e); run.active.push(e); }
  camera.getWorldPosition(head);
  // õhus? (päris hüpe või arvutis tühik)
  if (renderer.xr.isPresenting ? head.y > run.H + 0.06 : deskJump > 0.1) run.airT = t;
  const hs = playHands(); for (const h of hs) handPoint(h, h.pos);
  for (let i = run.active.length - 1; i >= 0; i--) {
    const e = run.active[i], o = e.o;
    if (e.type === 'orb' || e.type === 'mine') {
      const z = objZ(e, ORB_Z); o.position.set(e.x, e.y, z);
      if (e.type === 'orb') { o.userData.wire.rotation.set(t * 1.3, t * 1.7, 0); o.rotation.y = t * 2; } else { o.rotation.set(t * 2, t * 3, 0); o.userData.wire.scale.setScalar(1 + Math.sin(t * 10) * 0.12); }
      if (!e.done) {
        const a = V3(e.x, e.y, e.prevZ ?? z), b = V3(e.x, e.y, z);
        if (e.type === 'orb') {
          glows.add(o.position, e.col, D.id === 0 ? 0.65 : 0.5, 0.9);
          for (const h of hs) { const r = D.rad + (h.isHand ? 0.04 : 0); if (z > ORB_Z - 1.0 && segDist(h.pos, a, b) < r) { hitOrb(e, h, b); break; } }
          if (!e.done && z > ORB_Z + 0.9) missOrb(e);
        } else {
          glows.add(o.position, COL.mine, 0.45, 0.8 + Math.sin(t * 12) * 0.2);
          for (const h of hs) if (segDist(h.pos, a, b) < 0.15) { e.done = true; o.visible = false; burst(b, COL.mine, 50, 5, 0.07); hitObstacle(e, '💥 MIIN!'); break; }
          if (!e.done && z > ORB_Z + 0.9) e.done = true;
        }
      }
      e.prevZ = z;
      if (z > 3 || !o.visible) { o.visible = false; run.active.splice(i, 1); }
      continue;
    }
    if (e.type === 'rail') { if (updateRail(e, t, hs, dt)) { o.visible = false; run.active.splice(i, 1); } continue; }
    // sein, tala või hüppelaser – kontroll, kui see jõuab pea kohale
    const z = objZ(e, head.z); o.position.z = z;
    if (e.type === 'bar' || e.type === 'jump') { o.position.y = e.y; glows.add(V3(0, e.y, z), e.type === 'bar' ? COL.bar : COL.jump, 0.9, 0.5); }
    const tl = e.t - t;
    if (!run.helped[e.t] && tl < 1.3 && tl > 0 && (D.help || run.helpCount < 3 || e.type === 'jump' && run.jumpHelp < 3)) {
      run.helped[e.t] = 1; run.helpCount = (run.helpCount || 0) + 1; if (e.type === 'jump') run.jumpHelp = (run.jumpHelp || 0) + 1;
      if (e.type === 'wall') { const tilt = Math.abs(Math.sin(e.phi)) > 0.2; showBig(e.side > 0 ? (tilt ? '↙ KALDU VASAKULE JA KÜKITA' : '⬅ KALDU VASAKULE') : (tilt ? 'KALDU PAREMALE JA KÜKITA ↘' : 'KALDU PAREMALE ➡'), '#b388ff', 1.0); }
      else showBig(HELP[e.type][0], HELP[e.type][1], 1.0);
    }
    if (!e.done && e.prevZ != null && e.prevZ < head.z && z >= head.z) {
      e.done = true; let hit = false;
      if (e.type === 'bar') hit = head.y > e.y - 0.07;
      else if (e.type === 'jump') hit = t - run.airT > 0.28;
      else hit = e.nx * head.x + e.ny * (head.y - e.Hc) > e.e0 - 0.1;
      if (hit) hitObstacle(e, e.type === 'bar' ? '❌ KÜKITA!' : e.type === 'jump' ? '❌ HÜPPA!' : '❌ SEIN!'); else dodge(e, V3(head.x, head.y + 0.25, head.z - 0.6));
    }
    e.prevZ = z;
    if (z > 4) { o.visible = false; run.active.splice(i, 1); }
  }
  if (run.loss && run.energy <= 0) { run.energy = 0; finish(run.endless); return; }
  if (t > run.len + 0.3) { finish(true); return; }
  const fv = run.combo >= 32 ? 1 : 0; run.fever += (fv - run.fever) * Math.min(1, dt * 2); U.uFever.value = run.fever;
  drawHud(t);
}
// 🌀 hoia-rada: käsi peab rajal püsima
function updateRail(e, t, hs, dt){
  const o = e.o, sp = run.speed, t1 = e.t + e.dur, pos = o.userData.geo.attributes.position.array;
  const tau0 = Math.max(e.t, t), step = 0.22 / sp; let n = 0;
  for (let tau = tau0; tau <= t1 + 1e-6 && n < 259; tau += step) {
    const z = ORB_Z - (tau - t) * sp; if (z < -SPAWN) break;
    const x = railX(e, tau), y = railY(e, tau); pos[n * 3] = x; pos[n * 3 + 1] = y; pos[n * 3 + 2] = z; n++;
    if (n % 2 === 0) glows.add(tmpV.set(x, y, z), e.col, 0.13, 0.55);
  }
  o.userData.geo.setDrawRange(0, n); o.userData.geo.attributes.position.needsUpdate = true;
  const hz = ORB_Z - (tau0 - t) * sp, hp = tmpV2.set(railX(e, tau0), railY(e, tau0), hz);
  o.userData.headM.position.copy(hp); o.userData.headM.rotation.z = t * 4; o.userData.headM.visible = t <= t1;
  if (t >= e.t && t <= t1) {
    let holder = null; for (const h of hs) if (okHand(e, h) && h.pos.distanceTo(hp) < run.D.rad + 0.03 + (h.isHand ? 0.04 : 0)) { holder = h; break; }
    glows.add(hp, holder ? new THREE.Color(0xffffff) : e.col, holder ? 0.75 : 0.5, 1);
    if (holder) {
      e.held += dt; e.acc += dt;
      if (e.acc >= 0.1) { e.acc -= 0.1; run.score += 10 * mult(Math.max(1, run.combo)); sparks.emit(hp, V3((Math.random() - 0.5) * 1.5, Math.random() * 1.5, 0.5), e.col, 0.05, 0.4, 2); pulse(holder, 0.25, 25); }
      if (!e.hum) { e.hum = 1; blip(500, 900, 0.15, 'triangle', 0.06); }
    } else e.hum = 0;
  } else if (t < e.t) glows.add(hp, e.col, 0.45, 0.9);
  if (t > t1 + 0.05 && !e.done) {
    e.done = true; run.rails++; const pct = Math.round(e.held / e.dur * 100);
    if (pct >= 70) { run.railsGood++; comboUp(e.col); popText(V3(hp.x, hp.y + 0.15, hp.z), `🌀 ${pct}%`, hex(e.col)); pluck(chordNote(), 1.1); burst(hp, e.col, 30, 3); }
    else { run.combo = 0; popText(V3(hp.x, hp.y + 0.15, hp.z), `🌀 ${pct}%`, '#888888'); }
  }
  return t > t1 + 0.4;
}
function drawHud(t){
  const r = run, cd = t < 0 ? Math.ceil(-t) : 0;
  const key = `${r.score}|${r.combo}|${Math.round(r.energy)}|${r.endless ? Math.floor(t) : Math.floor(t / r.len * 50)}|${cd}`;
  if (key === lastHud) return; lastHud = key;
  hud.draw((c, W, H) => {
    roundRect(c, 6, 6, W - 12, H - 12, 30); c.fillStyle = 'rgba(8,2,22,0.55)'; c.fill(); c.lineWidth = 4; c.strokeStyle = hexA(r.song.c1, 0.6); c.stroke();
    neonText(c, r.score.toLocaleString('et-EE'), W * 0.22, H * 0.4, H * 0.32, hex(r.song.c1), 20, '900');
    c.fillStyle = 'rgba(230,220,255,0.7)'; c.font = `${H * 0.11}px ${FONT}`; c.textAlign = 'center'; c.fillText(duel ? AVATARS[settings.duel[duel.cur].av] : 'PUNKTID', W * 0.22, H * 0.7);
    const m = mult(r.combo);
    neonText(c, `x${m}`, W * 0.5, H * 0.38, H * 0.34, m >= 8 ? '#ffd23f' : hex(r.song.c2), 24, '900');
    c.fillStyle = 'rgba(230,220,255,0.7)'; c.font = `${H * 0.11}px ${FONT}`; c.fillText(`COMBO ${r.combo}`, W * 0.5, H * 0.7);
    if (r.loss) {
      const ex = W * 0.66, ew = W * 0.28, ey = H * 0.3, eh = H * 0.16;
      roundRect(c, ex, ey, ew, eh, eh / 2); c.fillStyle = 'rgba(255,255,255,0.08)'; c.fill();
      const ec = r.energy > 50 ? '#76ff03' : r.energy > 25 ? '#ffd23f' : '#ff1744';
      roundRect(c, ex, ey, Math.max(eh, ew * r.energy / 100), eh, eh / 2); c.fillStyle = ec; c.shadowColor = ec; c.shadowBlur = 16; c.fill(); c.shadowBlur = 0;
      c.fillStyle = 'rgba(230,220,255,0.7)'; c.font = `${H * 0.11}px ${FONT}`; c.fillText('ENERGIA', ex + ew / 2, H * 0.7);
    } else neonText(c, '🧒 ⭐', W * 0.8, H * 0.42, H * 0.25, '#76ff03', 10);
    if (r.endless) { c.fillStyle = 'rgba(230,220,255,0.85)'; c.font = `bold ${H * 0.1}px ${FONT}`; const tt = Math.max(0, t); c.fillText(`♾️ ${Math.floor(tt / 60)}:${String(Math.floor(tt % 60)).padStart(2, '0')}   🚀 x${(r.speed / r.D.speed).toFixed(2)}`, W / 2, H * 0.9); }
    else { const p = clamp(t / r.len, 0, 1); c.fillStyle = 'rgba(255,255,255,0.1)'; c.fillRect(W * 0.05, H * 0.86, W * 0.9, H * 0.04); c.fillStyle = hex(r.song.c2); c.fillRect(W * 0.05, H * 0.86, W * 0.9 * p, H * 0.04); }
  });
  if (cd) { if (!duel || cd < 3) showBig(String(cd), '#ffffff', 0.3); } else if (t > 0 && t < 0.6 && !r.goShown) { r.goShown = true; showBig('MINE! 🚀', hex(r.song.c1), 0.8); }
}
function celebrate(n = 6){ for (let i = 0; i < n; i++) setTimeout(() => { camera.getWorldPosition(head); burst(V3(head.x + (Math.random() - 0.5) * 3, head.y + 1 + Math.random(), head.z - 3), new THREE.Color().setHSL(Math.random(), 1, 0.6), 50, 4, 0.08); }, i * 250); }
function finish(ok){
  mode = 'end'; stopMusic(); hud.mesh.visible = false;
  const r = run, t = Math.max(0, songTime());
  const pct = r.endless ? 0 : r.max ? r.score / r.max : 0;
  const rank = !ok ? '—' : pct >= 0.9 ? 'S' : pct >= 0.75 ? 'A' : pct >= 0.55 ? 'B' : pct >= 0.35 ? 'C' : 'D';
  const stars = r.endless ? (t >= 180 ? 3 : t >= 90 ? 2 : 1) : !ok ? 0 : pct >= 0.75 ? 3 : pct >= 0.5 ? 2 : 1;
  const totalOrbs = r.endless ? r.hits + r.miss + r.wrong : r.orbs, acc = totalOrbs ? Math.round(r.hits / totalOrbs * 100) : 0;
  // ⚔️ duell
  if (duel) {
    duel.res[duel.cur] = { score: r.score, pct: Math.round(pct * 100), ok, acc, combo: r.maxCombo, diff: r.D.name };
    const me = settings.duel[duel.cur], meName = AVATARS[me.av];
    if (duel.cur === 0) {
      const next = AVATARS[settings.duel[1].av];
      endP.draw((c, W, H) => {
        panelBg(c, W, H, '#00e5ff');
        neonText(c, `${meName}: ${r.score.toLocaleString('et-EE')} p`, W / 2, H * 0.12, H * 0.075, '#00e5ff', 22, '900');
        c.fillStyle = '#e8e0ff'; c.font = `${H * 0.045}px ${FONT}`; c.textAlign = 'center'; c.fillText(`${Math.round(pct * 100)}% maksimumist • tabamusi ${acc}% • ${r.D.name}`, W / 2, H * 0.22);
        neonText(c, '🥽 ANNA PRILLID EDASI!', W / 2, H * 0.4, H * 0.08, '#ffd23f', 26, '900');
        neonText(c, `Järgmisena: ${next}`, W / 2, H * 0.53, H * 0.07, '#ff2bd6', 22, '900');
        c.fillStyle = 'rgba(230,220,255,0.65)'; c.font = `${H * 0.038}px ${FONT}`; c.fillText('Pane prillid pähe, seisa sirgelt ja vajuta nuppu', W / 2, H * 0.63);
      });
      setEndBtn(0, `▶ ${next} on valmis`, () => { duel.cur = 1; startRun(false); }, '#76ff03');
    } else {
      const [a, b] = duel.res, A = AVATARS[settings.duel[0].av], B = AVATARS[settings.duel[1].av];
      const w = a.pct === b.pct ? (a.score === b.score ? -1 : a.score > b.score ? 0 : 1) : a.pct > b.pct ? 0 : 1;
      if (w >= 0) { const av = settings.duel[w].av; wins[av] = (wins[av] || 0) + 1; store('nt_wins', wins); }
      endP.draw((c, W, H) => {
        panelBg(c, W, H, '#ffd23f');
        neonText(c, w < 0 ? '🤝 VIIK!' : `🏆 ${w ? B : A} VÕITIS!`, W / 2, H * 0.12, H * 0.09, '#ffd23f', 30, '900');
        const row = (i, res, name, y, col) => {
          neonText(c, `${i === w ? '👑 ' : ''}${name}`, W * 0.27, H * y, H * 0.06, col, 16, '900');
          neonText(c, `${res.pct}%`, W * 0.58, H * y, H * 0.075, col, 16, '900');
          c.fillStyle = '#e8e0ff'; c.font = `${H * 0.036}px ${FONT}`; c.textAlign = 'center'; c.fillText(`${res.score.toLocaleString('et-EE')} p • ${res.diff}`, W * 0.82, H * y);
        };
        row(0, a, A, 0.32, '#00e5ff'); row(1, b, B, 0.46, '#ff2bd6');
        c.fillStyle = 'rgba(230,220,255,0.65)'; c.font = `${H * 0.036}px ${FONT}`; c.textAlign = 'center';
        c.fillText(`Võite kokku: ${A} ${wins[settings.duel[0].av] || 0}  •  ${B} ${wins[settings.duel[1].av] || 0}`, W / 2, H * 0.6);
      });
      setEndBtn(0, '🔁 Kordusmatš', () => startRun(true), '#ffd23f');
      celebrate(8); blip(523, 1046, 0.4, 'triangle', 0.15);
    }
    placeInFront(endG, 1.0, -0.12); endG.visible = true; return;
  }
  // 🎵 tavaline ja ♾️ lõputu
  const k = bestKey(), prev = best[k]; let rec = false;
  if ((ok || r.endless) && (!prev || r.score > prev.score)) { best[k] = { score: r.score, rank, stars }; store('nt_best', best); rec = true; }
  endP.draw((c, W, H) => {
    panelBg(c, W, H, ok ? '#76ff03' : '#ff1744');
    neonText(c, rec ? '🏆 UUS REKORD!' : r.endless ? '♾️ MÄNG LÄBI' : ok ? 'LÕPETATUD!' : '⚡ ENERGIA OTSAS', W / 2, H * 0.1, H * 0.085, rec || ok ? '#76ff03' : '#ff1744', 26, '900');
    neonText(c, r.score.toLocaleString('et-EE'), W / 2, H * 0.24, H * 0.12, hex(r.song.c1), 30, '900');
    neonText(c, '★'.repeat(stars) + '☆'.repeat(3 - stars), W * 0.3, H * 0.38, H * 0.08, '#ffd23f', 18);
    neonText(c, r.endless ? `⏱ ${Math.floor(t / 60)}:${String(Math.floor(t % 60)).padStart(2, '0')}` : ok ? `Hinne ${rank}` : '', W * 0.7, H * 0.38, H * 0.08, '#ffd23f', 18);
    c.fillStyle = '#e8e0ff'; c.font = `${H * 0.042}px ${FONT}`; c.textAlign = 'center';
    c.fillText(`🎯 Tabamusi ${r.hits}/${totalOrbs} (${acc}%)   •   🔥 Suurim combo ${r.maxCombo}${r.rails ? `   •   🌀 ${r.railsGood}/${r.rails}` : ''}`, W / 2, H * 0.5);
    c.fillText(`✔ Põikeid ${r.dodged}   •   ❌ Pihta said ${r.obsHit}${r.wrong ? `   •   ✋ Vale käsi ${r.wrong}` : ''}`, W / 2, H * 0.57);
    c.fillStyle = 'rgba(230,220,255,0.6)'; c.font = `${H * 0.038}px ${FONT}`;
    c.fillText(`${r.song.name} • ${r.D.name}${r.endless ? ' • ♾️ Lõputu' : ''}${prev ? ` • eelmine rekord ${prev.score}` : ''}`, W / 2, H * 0.65);
  });
  setEndBtn(0, '🔁 Uuesti', () => startRun(true), '#ffd23f');
  placeInFront(endG, 1.0, -0.12); endG.visible = true;
  if (ok || rec) { blip(523, 1046, 0.4, 'triangle', 0.15); setTimeout(() => blip(659, 1318, 0.4, 'triangle', 0.15), 150); celebrate(); } else crunch();
}

/* =========================================================
   KÄED, KIIRED JA PUUDUTUS
   ========================================================= */
const uiMode = () => mode !== 'play';
function updatePointers(dt){
  const btnMeshes = buttons.filter(b => isShown(b.p.mesh)).map(b => b.p.mesh);
  const setHover = (h, b) => { if (h.hover === b) return; if (h.hover) { h.hover.hover = false; h.hover.draw(); } h.hover = b; if (b) { b.hover = true; b.draw(); if (h.kind === 'xr') pulse(h, 0.15, 15); } };
  for (const h of hands) {
    h.ray.visible = false;
    if (!h.connected) { setHover(h, null); continue; }
    handPoint(h, tmpV); if (h.hasPrev) h.vel.lerp(tmpD.copy(tmpV).sub(h.prev).divideScalar(Math.max(dt, 1e-3)), 0.5); h.prev.copy(tmpV); h.hasPrev = true;
    const c = handCol(h); glows.add(tmpV, c, h.isHand ? 0.35 : 0.28, 0.7);
    if (h.vel.length() > 0.8) sparks.emit(tmpV, tmpD.copy(h.vel).multiplyScalar(-0.1), c, 0.035, 0.35, 3);
    if (uiMode() && btnMeshes.length) {
      h.ctrl.getWorldPosition(tmpV); h.ctrl.getWorldQuaternion(tmpQ); tmpD.set(0, 0, -1).applyQuaternion(tmpQ);
      raycaster.set(tmpV, tmpD); raycaster.far = 6;
      const hit = raycaster.intersectObjects(btnMeshes, false)[0];
      setHover(h, hit ? hit.object.userData.btn : null);
      h.ray.visible = true; h.ray.scale.set(1, 1, hit ? hit.distance : 2.5); h.ray.material.color.copy(c);
    } else setHover(h, null);
    if (h.isHand && h.hand.joints && h.hand.joints['index-finger-tip']) {
      const tip = h.hand.joints['index-finger-tip'].getWorldPosition(V3());
      let touched = null;
      for (const m of btnMeshes) { if (m === wristBtn.p.mesh && h.side === 'left') continue; const p = m.worldToLocal(tip.clone()), b = m.userData.btn; if (Math.abs(p.x) < b.w / 2 && Math.abs(p.y) < b.h / 2 && Math.abs(p.z) < 0.025) { touched = b; break; } }
      if (touched && h.poke !== touched) clickBtn(touched, h);
      h.poke = touched;
    }
  }
  if (!renderer.xr.isPresenting) {
    raycaster.setFromCamera(mouseNDC, camera); raycaster.far = 10;
    if (uiMode()) { const hit = raycaster.intersectObjects(btnMeshes, false)[0]; setHover(mouseHand, hit ? hit.object.userData.btn : null); }
    else setHover(mouseHand, null);
    const r = raycaster.ray, tz = (ORB_Z - r.origin.z) / (r.direction.z || -1e-3);
    if (tz > 0) { const p = r.origin.clone().addScaledVector(r.direction, tz); if (mouseHand.hasPrev) mouseHand.vel.lerp(tmpD.copy(p).sub(mouseHand.pos).divideScalar(Math.max(dt, 1e-3)), 0.5); mouseHand.pos.copy(p); mouseHand.hasPrev = true; }
    if (window.__botHand) window.__botHand(mouseHand);
    if (mode === 'play') { glows.add(mouseHand.pos, COL.any, 0.3, 0.8); if (mouseHand.vel.length() > 0.8) sparks.emit(mouseHand.pos, V3(), COL.any, 0.03, 0.3, 3); }
  }
  const L = hands.find(h => h.connected && h.isHand && h.side === 'left');
  wristBtnG.visible = !!L && renderer.xr.isPresenting && (mode === 'play' || mode === 'pause');
  if (wristBtnG.visible) { const w = L.hand.joints && L.hand.joints['wrist']; if (w) { w.getWorldPosition(tmpV); w.getWorldQuaternion(tmpQ); wristBtnG.position.copy(tmpV).add(tmpD.set(0, 0.035, 0.05).applyQuaternion(tmpQ)); wristBtnG.quaternion.copy(tmpQ); wristBtnG.rotateX(-Math.PI / 2); } }
}
// pultide nupud: B/Y = paus, A/X = hüppa
const btnPrev = {};
function readButtons(){
  const s = renderer.xr.getSession(); if (!s) return;
  for (const src of s.inputSources) {
    const gp = src.gamepad; if (!gp) continue; const k = src.handedness;
    const b = gp.buttons[5] && gp.buttons[5].pressed; if (b && !btnPrev[k]) togglePause(); btnPrev[k] = b;
    const a = gp.buttons[4] && gp.buttons[4].pressed; if (a && !btnPrev[k + 'A'] && run && mode === 'play') run.airT = songTime() + 0.1; btnPrev[k + 'A'] = a;
  }
}

/* =========================================================
   VR SESSIOON
   ========================================================= */
const info = document.getElementById('info');
renderer.xr.addEventListener('sessionstart', () => {
  info.style.display = 'none'; audioInit();
  const s = renderer.xr.getSession(); s.addEventListener('visibilitychange', () => { if (s.visibilityState !== 'visible' && mode === 'play') pauseGame(); });
  setTimeout(() => { if (mode === 'menu') placeInFront(menuG, 1.1, -0.15); else if (mode === 'pause') placeInFront(pauseG, 0.85, -0.15); else if (mode === 'end') placeInFront(endG, 1.0, -0.12); }, 400);
});
renderer.xr.addEventListener('sessionend', () => { info.style.display = ''; camera.position.set(0, 1.6, 0); camera.rotation.set(0, 0, 0); rig.position.set(0, 0, 0); if (mode === 'play') pauseGame(); });

/* =========================================================
   PÕHITSÜKKEL
   ========================================================= */
refreshMenu(); placeInFront(menuG, 1.1, -0.15);
const clock = new THREE.Clock();
renderer.setAnimationLoop(() => {
  const dt = Math.min(clock.getDelta(), 0.05), T = clock.elapsedTime;
  U.uT.value = T;
  readButtons();
  if (!renderer.xr.isPresenting) {
    deskJump = Math.max(0, deskJump - dt);
    const jy = deskJump > 0 ? Math.sin(Math.PI * (1 - deskJump / 0.6)) * 0.45 : 0;
    const lx = (keys.KeyA || keys.ArrowLeft ? -0.35 : 0) + (keys.KeyD || keys.ArrowRight ? 0.35 : 0), ly = (keys.KeyC || keys.ControlLeft || keys.ArrowDown ? 0.95 : 1.6) + jy;
    camera.position.x += (lx - camera.position.x) * Math.min(1, dt * 12); camera.position.y += (ly - camera.position.y) * Math.min(1, dt * (deskJump > 0 ? 30 : 12));
  }
  glows.begin();
  let speed = 2.2, beat = 0, intense = 0;
  if (run && (mode === 'play' || mode === 'pause' || mode === 'end')) {
    if (mode === 'play') scheduleMusic();
    const t = songTime(); speed = run.speed;
    if (mode === 'play' && t > 0) run.dist = (run.dist || 0) + speed * dt;
    U.uDist.value = (run.dist || 0) + menuDist;
    const sec = t >= 0 && t < run.len ? secOf(Math.floor(t / (run.bd * 4))).s : null, ph = (((t / run.bd) % 1) + 1) % 1;
    beat = sec && sec.kick ? Math.exp(-ph * run.bd * 7) : sec ? 0.35 * Math.exp(-ph * run.bd * 4) : 0;
    intense = sec ? sec.I : 0;
    if (mode === 'play') updateRun(dt);
  } else { menuDist += dt * speed; U.uDist.value = menuDist; beat = 0.15 + 0.1 * Math.sin(T * 2); U.uFever.value *= 0.95; }
  U.uBeat.value = beat;
  // 🔄 pöörlev tunnel
  const spinT = run && run.flags.spin && mode === 'play' ? (intense >= 1 ? 0.4 : 0.12) * (run.D.id === 0 ? 0.5 : 1) * (1 + U.uFever.value) : mode === 'menu' ? 0.05 : 0;
  spinSpeed += (spinT - spinSpeed) * Math.min(1, dt * 0.8); tunnelRoll += spinSpeed * dt;
  tunnel.rotation.z = tunnelRoll; streakG.rotation.z = tunnelRoll * 1.3;
  const dist = U.uDist.value, span = RING_N * RING_GAP;
  for (const r of rings) {
    const z = ((-r.k * RING_GAP + dist) % span + span) % span - span + 4; r.g.position.z = z;
    const hue = (r.k / RING_N + T * 0.05) % 1, c = tmpCol.copy(U.uC1.value).lerp(U.uC2.value, 0.5 + 0.5 * Math.sin(r.k * 0.8 + T));
    if (U.uFever.value > 0.05) c.lerp(new THREE.Color().setHSL(hue, 1, 0.6), U.uFever.value);
    r.core.material.color.copy(c).lerp(new THREE.Color(0xffffff), 0.3); r.glow.material.color.copy(c);
    const fade = Math.exp(Math.min(0, z) * 0.03); r.glow.material.opacity = (0.12 + beat * 0.2) * fade; r.g.scale.setScalar(1 + beat * 0.015);
  }
  for (let i = 0; i < STREAK_N; i++) { const s = streakData[i]; s.z += speed * 1.7 * dt; if (s.z > 4) s.z = -105 - Math.random() * 10; const k = i * 6, l = s.l * (0.5 + speed / 8); streakPos[k] = s.x; streakPos[k + 1] = s.y; streakPos[k + 2] = s.z; streakPos[k + 3] = s.x; streakPos[k + 4] = s.y; streakPos[k + 5] = s.z - l; }
  streakGeo.attributes.position.needsUpdate = true; streaks.material.color.copy(U.uC1.value).lerp(new THREE.Color(0xffffff), 0.5);
  for (const m of mountains) m.grp.position.z = -150 + (dist * 0.6) % 100;
  camera.getWorldPosition(head); sky.position.copy(head);
  colorHands();
  updatePointers(dt);
  for (const p of pops) if (p.p.mesh.visible) { p.t -= dt; p.p.mesh.position.y += dt * 0.5; p.p.mesh.material.opacity = clamp(p.t / 0.5, 0, 1); if (p.t <= 0) p.p.mesh.visible = false; }
  hurt.material.opacity = Math.max(0, hurt.material.opacity - dt * 1.2);
  if (bigMsg.mesh.visible) { bigMsgT -= dt; bigMsg.mesh.material.opacity = clamp(bigMsgT / 0.3, 0, 1); if (bigMsgT <= 0) bigMsg.mesh.visible = false; }
  sparks.update(dt);
  glows.end();
  sparks.mat.uniforms.uScale.value = renderer.xr.isPresenting ? 900 : renderer.domElement.height / 2 * camera.projectionMatrix.elements[5];
  renderer.render(scene, camera);
});
window.__nt = { startRun, goMenu, settings, get run(){ return run; }, mouseHand, songTime, refreshMenu, finish, get duel(){ return duel; } };
</script>
</body>
</html>
