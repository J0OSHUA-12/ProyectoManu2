# ProyectoManu2

<style>
  canvas { display: block; }
  #controls { position: absolute; top: 10px; right: 10px; display: flex; flex-direction: column; gap: 6px; }
  button { font-size: 12px; padding: 4px 10px; cursor: pointer; background: var(--color-background-secondary); border: 0.5px solid var(--color-border-secondary); border-radius: var(--border-radius-md); color: var(--color-text-primary); }
  button:hover { background: var(--color-background-primary); }
  #label { position: absolute; bottom: 10px; left: 12px; font-size: 12px; color: var(--color-text-secondary); }
</style>
<div style="position: relative; width: 100%; height: 500px;">
  <canvas id="c"></canvas>
  <div id="controls">
    <button onclick="toggleDrawers()">Abrir/cerrar cajones</button>
    <button onclick="resetView()">Restablecer vista</button>
  </div>
  <div id="label">Arrastrar para rotar · Scroll para zoom</div>
</div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
const canvas = document.getElementById('c');
const W = canvas.parentElement.clientWidth, H = 500;
canvas.width = W; canvas.height = H;

const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
renderer.setSize(W, H);
renderer.shadowMap.enabled = true;
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(38, W / H, 0.1, 100);

scene.add(new THREE.AmbientLight(0xffffff, 0.65));
const sun = new THREE.DirectionalLight(0xffffff, 0.85);
sun.position.set(3, 5, 4); sun.castShadow = true; scene.add(sun);
const fill = new THREE.DirectionalLight(0xffffff, 0.25);
fill.position.set(-3, 2, -2); scene.add(fill);

const wood     = new THREE.MeshStandardMaterial({ color: 0xC8A455, roughness: 0.8 });
const darkWood = new THREE.MeshStandardMaterial({ color: 0xB8943A, roughness: 0.8 });
const seatMat  = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.95 });
const metalMat = new THREE.MeshStandardMaterial({ color: 0x777777, roughness: 0.3, metalness: 0.9 });
const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.9 });
const slideMat = new THREE.MeshStandardMaterial({ color: 0x999999, roughness: 0.2, metalness: 1.0 });

const F = 0.05;
const f = v => v * F;

function box(w, h, d, mat, x, y, z, par) {
  const m = new THREE.Mesh(new THREE.BoxGeometry(f(w), f(h), f(d)), mat);
  m.position.set(f(x), f(y), f(z));
  m.castShadow = true; m.receiveShadow = true;
  (par || scene).add(m); return m;
}

const T       = 0.75;
const CART_W  = 18;
const CART_D  = 18;
const SIDE_H  = 13.5;
const SIDE_D  = 11;
const WHEEL_R = 1.0;
const WHEEL_T = 0.7;

const WHEEL_CY = WHEEL_R;
const BASE_BY  = WHEEL_R * 2;
const BASE_CY  = BASE_BY + T / 2;
const COL_BY   = BASE_BY + T;
const COL_CY   = COL_BY + SIDE_H / 2;
const TOP_BY   = COL_BY + SIDE_H;
const TOP_CY   = TOP_BY + T / 2;

// Floor shadow
const floorM = new THREE.Mesh(new THREE.PlaneGeometry(4,4), new THREE.ShadowMaterial({ opacity: 0.12 }));
floorM.rotation.x = -Math.PI/2; floorM.receiveShadow = true; scene.add(floorM);

// CASTERS
const COX = CART_W/2 - 1.5, COZ = CART_D/2 - 1.5;
[[-1,-1],[-1,1],[1,-1],[1,1]].forEach(([sx,sz]) => {
  const cx = sx*COX, cz = sz*COZ;
  box(0.7, WHEEL_R*1.5, 0.7, metalMat, cx, BASE_BY - WHEEL_R*0.75, cz);
  const wh = new THREE.Mesh(new THREE.CylinderGeometry(f(WHEEL_R),f(WHEEL_R),f(WHEEL_T),24), wheelMat);
  wh.rotation.x = Math.PI/2; wh.position.set(f(cx),f(WHEEL_CY),f(cz)); wh.castShadow=true; scene.add(wh);
  const hub = new THREE.Mesh(new THREE.CylinderGeometry(f(0.28),f(0.28),f(WHEEL_T+0.1),12), metalMat);
  hub.rotation.x = Math.PI/2; hub.position.set(f(cx),f(WHEEL_CY),f(cz)); scene.add(hub);
});

// BASE SLAB
box(CART_W, T, CART_D, wood, 0, BASE_CY, 0);

// COLUMNS
const COL_X = CART_W/2 - T/2;
box(T, SIDE_H, SIDE_D, darkWood, -COL_X, COL_CY, 0);
box(T, SIDE_H, SIDE_D, darkWood,  COL_X, COL_CY, 0);

// BACK PANEL
box(CART_W - T*2, SIDE_H, T, darkWood, 0, COL_CY, -SIDE_D/2 + T/2);

// TOP PANEL — flush on columns
box(13.5, T, CART_D, wood, 0, TOP_CY, 0);

// SEAT CUSHION
box(13.4, 0.55, CART_D - 0.1, seatMat, 0, TOP_CY + T/2 + 0.275, 0);

// DRAWER SLIDES
const DRW_H = 2.5, DRW_D = 12.5;
const D1Y = COL_BY + T/2 + 0.2;
const D2Y = D1Y + DRW_H + T + 0.5;
const RAIL_X = COL_X - T/2 - 0.15;
[D1Y, D2Y].forEach(dy => {
  [-RAIL_X, RAIL_X].forEach(rx => {
    box(0.4, 0.3, DRW_D, slideMat, rx, dy, 0);
  });
});

// DRAWERS
let d1 = new THREE.Group(), d2 = new THREE.Group();
let isOpen = false;

function buildDrawer(g) {
  const DW=14, DH=DRW_H, DD=DRW_D;
  box(DW, T,  DD,  wood,     0,         0,       0,          g);
  box(DW, DH, T,   darkWood, 0,         DH/2,    DD/2+T/2,   g);
  box(DW, DH, T,   wood,     0,         DH/2,   -DD/2-T/2,   g);
  box(T,  DH, DD,  wood,    -DW/2-T/2,  DH/2,   0,          g);
  box(T,  DH, DD,  wood,     DW/2+T/2,  DH/2,   0,          g);
  const arc = new THREE.Mesh(new THREE.TorusGeometry(f(1.1),f(0.2),8,24,Math.PI), metalMat);
  arc.rotation.x = Math.PI;
  arc.position.set(0, f(DH+0.1), f(DD/2+T/2));
  g.add(arc);
  scene.add(g);
}

buildDrawer(d1); buildDrawer(d2);
d1.position.set(0, f(D1Y), 0);
d2.position.set(0, f(D2Y), 0);

function toggleDrawers() {
  isOpen = !isOpen;
  const pull = f(isOpen ? 10 : 0);
  d1.position.z = pull;
  d2.position.z = pull;
}

// SHELF — lateral derecha
// Ancho en Z (frente-fondo del carro) = 13", poco saliente en X = 4"
const SHELF_STICK = 4;    // cuánto sale hacia la derecha (poco)
const SHELF_SPAN  = 13;   // ancho a lo largo del carro (Z axis)
const SHELF_LIP   = 1.5;
const SHELF_Y     = COL_BY + SIDE_H * 0.5;
const SHELF_X     = COL_X + T/2 + SHELF_STICK/2;  // pegado a columna derecha

// tablero
box(SHELF_STICK, T, SHELF_SPAN, wood, SHELF_X, SHELF_Y, 0);
// labio frontal (+Z)
box(SHELF_STICK, SHELF_LIP, T*0.5, darkWood, SHELF_X, SHELF_Y + T/2 + SHELF_LIP/2,  SHELF_SPAN/2);
// labio trasero (-Z)
box(SHELF_STICK, SHELF_LIP, T*0.5, darkWood, SHELF_X, SHELF_Y + T/2 + SHELF_LIP/2, -SHELF_SPAN/2);
// labio exterior (extremo derecho)
box(T*0.5, SHELF_LIP, SHELF_SPAN, darkWood, SHELF_X + SHELF_STICK/2, SHELF_Y + T/2 + SHELF_LIP/2, 0);

// CAMERA
let rotX = 0.32, rotY = -0.62, zoom = 1.0;
function updateCamera() {
  const r = 1.7 * zoom;
  camera.position.x = r * Math.sin(rotY) * Math.cos(rotX);
  camera.position.y = f(BASE_CY + SIDE_H*0.5) + r * Math.sin(rotX);
  camera.position.z = r * Math.cos(rotY) * Math.cos(rotX);
  camera.lookAt(f(4), f(COL_CY*0.8), 0);
}
updateCamera();

let drag=false, px=0, py=0;
canvas.addEventListener('mousedown', e=>{drag=true;px=e.clientX;py=e.clientY;});
window.addEventListener('mouseup', ()=>drag=false);
window.addEventListener('mousemove', e=>{
  if(!drag)return;
  rotY+=(e.clientX-px)*0.008; rotX+=(e.clientY-py)*0.008;
  rotX=Math.max(-0.9,Math.min(1.1,rotX));
  px=e.clientX;py=e.clientY;updateCamera();
});
canvas.addEventListener('wheel',e=>{
  zoom=Math.max(0.4,Math.min(3.5,zoom+e.deltaY*0.001));
  updateCamera();e.preventDefault();
},{passive:false});
canvas.addEventListener('touchstart',e=>{drag=true;px=e.touches[0].clientX;py=e.touches[0].clientY;});
window.addEventListener('touchend',()=>drag=false);
window.addEventListener('touchmove',e=>{
  if(!drag)return;
  rotY+=(e.touches[0].clientX-px)*0.008;rotX+=(e.touches[0].clientY-py)*0.008;
  rotX=Math.max(-0.9,Math.min(1.1,rotX));
  px=e.touches[0].clientX;py=e.touches[0].clientY;updateCamera();
},{passive:true});

function resetView(){rotX=0.32;rotY=-0.62;zoom=1.0;updateCamera();}
(function loop(){requestAnimationFrame(loop);renderer.render(scene,camera);})();
</script>
