/* ============================================================
   EDEN — main.js
   Structure:
     PART A — pure DOM/UI logic. No Three.js dependency. Runs
       immediately and unconditionally, so nav, reveals, cursor,
       and reading the page all work even if WebGL/the CDN fails.
     PART B — the procedural Three.js scene, loaded via dynamic
       import() inside a try/catch so a network or module failure
       degrades gracefully instead of silently killing the page.
   ============================================================ */

document.documentElement.classList.remove("no-js");

const prefersReduced = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
if (prefersReduced) document.body.classList.add("reduced-motion");

const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const lerp = (a, b, t) => a + (b - a) * t;

/* shared state read by the 3D scene if/when it loads */
const state = { scrollProgressRaw: 0, activeIndex: 0 };

/* ------------------------------------------------------------
   A1. On-screen scene status (visible diagnostic, no devtools needed)
   ------------------------------------------------------------ */
function showSceneStatus(message, isError) {
  let el = document.getElementById("scene-status");
  if (!el) {
    el = document.createElement("p");
    el.id = "scene-status";
    el.setAttribute("role", "status");
    el.style.cssText = "position:fixed;left:16px;bottom:16px;z-index:70;max-width:280px;" +
      "font-family:ui-monospace,monospace;font-size:11px;line-height:1.5;letter-spacing:.02em;" +
      "padding:10px 12px;border:1px solid rgba(236,230,216,0.18);" +
      "background:rgba(7,8,10,0.85);color:#ece6d8;backdrop-filter:blur(4px);";
    document.body.appendChild(el);
  }
  el.textContent = message;
  el.style.borderColor = isError ? "rgba(178,58,47,0.6)" : "rgba(216,163,95,0.5)";
}
function hideSceneStatus() {
  const el = document.getElementById("scene-status");
  if (el) el.remove();
}

/* ------------------------------------------------------------
   A2. Word-split heading reveal
   ------------------------------------------------------------ */
function splitWords(el) {
  const lines = el.innerHTML.split(/<br\s*\/?>/i);
  el.innerHTML = lines
    .map((line) =>
      line.trim().split(/\s+/).filter(Boolean)
        .map((w) => `<span class="word"><span>${w}</span></span>`)
        .join(" ")
    )
    .join("<br>");
  el.querySelectorAll(".word span").forEach((s, i) => {
    s.style.transitionDelay = `${Math.min(i * 45, 480)}ms`;
  });
}
document.querySelectorAll("[data-split]").forEach(splitWords);

requestAnimationFrame(() => {
  const hero = document.getElementById("hero");
  if (hero) hero.classList.add("is-active");
});

const revealObserver = ("IntersectionObserver" in window)
  ? new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            entry.target.classList.add("is-active");
            revealObserver.unobserve(entry.target);
          }
        });
      },
      { threshold: 0.001, rootMargin: "-18% 0px -18% 0px" }
    )
  : null;

document.querySelectorAll(".section").forEach((sec) => {
  if (revealObserver) revealObserver.observe(sec);
  else sec.classList.add("is-active");
});

/* ------------------------------------------------------------
   A3. Active-section tracking → nav, chapter rail, foreground layers
   ------------------------------------------------------------ */
const sectionIds = ["hero", "threshold", "gardens", "incense", "lanterns", "afterlight"];
const sectionEls = sectionIds.map((id) => document.getElementById(id)).filter(Boolean);
const navLinks = document.querySelectorAll(".primary-nav a");
const railButtons = document.querySelectorAll(".chapter-rail button");
const fgLayers = document.querySelectorAll(".fg-layer");
const mainEl = document.querySelector("main");

function setActiveSection(id) {
  navLinks.forEach((a) => {
    const match = a.getAttribute("href") === `#${id}`;
    if (match) a.setAttribute("aria-current", "true"); else a.removeAttribute("aria-current");
  });
  railButtons.forEach((b) => {
    b.setAttribute("aria-current", b.dataset.target === id ? "true" : "false");
  });
  fgLayers.forEach((layer) => {
    layer.classList.toggle("is-active", layer.dataset.section === id);
  });
}
setActiveSection("hero");

function computeScrollProgress() {
  if (!mainEl) return 0;
  const max = Math.max(1, mainEl.offsetHeight - window.innerHeight);
  return clamp(window.scrollY, 0, max) / max;
}
function findActiveByScroll() {
  const centerY = window.scrollY + window.innerHeight * 0.5;
  let idx = 0;
  for (let i = 0; i < sectionEls.length; i++) {
    if (sectionEls[i].offsetTop <= centerY) idx = i;
  }
  return idx;
}

let scrollDirty = true;
window.addEventListener("scroll", () => { scrollDirty = true; }, { passive: true });

function tickScrollState() {
  if (scrollDirty) {
    scrollDirty = false;
    state.scrollProgressRaw = computeScrollProgress();
    const newIdx = findActiveByScroll();
    if (newIdx !== state.activeIndex) {
      state.activeIndex = newIdx;
      setActiveSection(sectionIds[state.activeIndex]);
      window.dispatchEvent(new CustomEvent("eden:sectionchange", { detail: { index: newIdx, id: sectionIds[newIdx] } }));
    }
  }
  requestAnimationFrame(tickScrollState);
}
tickScrollState();

/* ------------------------------------------------------------
   A4. Chapter rail / nav click-to-scroll
   ------------------------------------------------------------ */
railButtons.forEach((btn) => {
  btn.addEventListener("click", () => {
    const target = document.getElementById(btn.dataset.target);
    if (target) target.scrollIntoView({ behavior: prefersReduced ? "auto" : "smooth" });
  });
});

/* ------------------------------------------------------------
   A5. Mobile nav
   ------------------------------------------------------------ */
const mobileNav = document.getElementById("mobile-nav");
const navToggleBtn = document.getElementById("nav-toggle-btn");
const mnCloseBtn = document.getElementById("mn-close-btn");
if (mobileNav && navToggleBtn && mnCloseBtn) {
  function openMobileNav() {
    mobileNav.classList.add("is-open");
    navToggleBtn.setAttribute("aria-expanded", "true");
    mnCloseBtn.focus();
  }
  function closeMobileNav() {
    mobileNav.classList.remove("is-open");
    navToggleBtn.setAttribute("aria-expanded", "false");
    navToggleBtn.focus();
  }
  navToggleBtn.addEventListener("click", openMobileNav);
  mnCloseBtn.addEventListener("click", closeMobileNav);
  mobileNav.querySelectorAll("a").forEach((a) => a.addEventListener("click", closeMobileNav));
  window.addEventListener("keydown", (e) => {
    if (e.key === "Escape" && mobileNav.classList.contains("is-open")) closeMobileNav();
  });
  function syncNavToggleVisibility() {
    const show = window.innerWidth <= 880;
    navToggleBtn.style.display = show ? "flex" : "none";
    const primary = document.querySelector(".primary-nav");
    if (primary) primary.style.display = show ? "none" : "flex";
  }
  syncNavToggleVisibility();
  window.addEventListener("resize", syncNavToggleVisibility);
}

/* ------------------------------------------------------------
   A6. Custom cursor (fine pointers only)
   ------------------------------------------------------------ */
const finePointer = window.matchMedia("(pointer: fine)").matches;
if (finePointer) {
  document.body.classList.add("has-cursor");
  const dot = document.querySelector(".cursor-dot");
  const ring = document.querySelector(".cursor-ring");
  let mx = -100, my = -100, rx = -100, ry = -100;
  window.addEventListener("mousemove", (e) => { mx = e.clientX; my = e.clientY; });
  function cursorLoop() {
    if (dot) dot.style.transform = `translate(${mx}px, ${my}px)`;
    rx = lerp(rx, mx, 0.18); ry = lerp(ry, my, 0.18);
    if (ring) ring.style.transform = `translate(${rx}px, ${ry}px)`;
    requestAnimationFrame(cursorLoop);
  }
  cursorLoop();
  document.querySelectorAll("a, button, .card").forEach((el) => {
    el.addEventListener("mouseenter", () => { dot && dot.classList.add("is-hover"); ring && ring.classList.add("is-hover"); });
    el.addEventListener("mouseleave", () => { dot && dot.classList.remove("is-hover"); ring && ring.classList.remove("is-hover"); });
  });
}

/* ------------------------------------------------------------
   PART B — the procedural 3D scene (loaded defensively)
   ------------------------------------------------------------ */
async function initScene() {
  const canvas = document.getElementById("scene-canvas");
  if (!canvas) return;
  if (!window.WebGLRenderingContext) {
    showSceneStatus("This browser doesn't support WebGL — the temple scene is unavailable, but every chapter below still reads in full.", true);
    return;
  }

  showSceneStatus("Loading the temple scene…", false);

  let THREE, EffectComposer, RenderPass, UnrealBloomPass;
  try {
    THREE = await import("three");
    ({ EffectComposer } = await import("three/addons/postprocessing/EffectComposer.js"));
    ({ RenderPass } = await import("three/addons/postprocessing/RenderPass.js"));
    ({ UnrealBloomPass } = await import("three/addons/postprocessing/UnrealBloomPass.js"));
  } catch (err) {
    console.error("EDEN: failed to load Three.js modules", err);
    showSceneStatus("The 3D scene couldn't load — likely a blocked network request to the module CDN in this preview. Every chapter below still reads in full; this will resolve on a normal hosted deploy.", true);
    return;
  }

  try {
    runScene(THREE, EffectComposer, RenderPass, UnrealBloomPass, canvas);
    hideSceneStatus();
  } catch (err) {
    console.error("EDEN: scene failed to initialize", err);
    showSceneStatus("The 3D scene hit a runtime error and stopped. Every chapter below still reads in full.", true);
  }
}

function runScene(THREE, EffectComposer, RenderPass, UnrealBloomPass, canvas) {
  function hash(x, y) {
    const s = Math.sin(x * 127.1 + y * 311.7) * 43758.5453123;
    return s - Math.floor(s);
  }
  function smoothstep(t) { return t * t * (3 - 2 * t); }
  function valueNoise(x, y) {
    const xi = Math.floor(x), yi = Math.floor(y);
    const xf = x - xi, yf = y - yi;
    const a = hash(xi, yi), b = hash(xi + 1, yi);
    const c = hash(xi, yi + 1), d = hash(xi + 1, yi + 1);
    const u = smoothstep(xf), v = smoothstep(yf);
    return lerp(lerp(a, b, u), lerp(c, d, u), v);
  }
  function fbm(x, y, oct = 4) {
    let total = 0, amp = 0.5, freq = 1, max = 0;
    for (let i = 0; i < oct; i++) {
      total += valueNoise(x * freq, y * freq) * amp;
      max += amp; amp *= 0.5; freq *= 2;
    }
    return total / max;
  }

  const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: false, powerPreference: "high-performance" });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.05;
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  const COLORS = {
    void: 0x07080a, charcoal: 0x151f2c, charcoal2: 0x0d1420,
    amber: 0xd8a35f, amberBright: 0xffd39a,
    vermilion: 0xb23a2f, vermilionBright: 0xd1503f,
    wood: 0x1c1410, stone: 0x2a3038,
  };

  const scene = new THREE.Scene();
  scene.background = new THREE.Color(COLORS.void);
  scene.fog = new THREE.FogExp2(COLORS.void, 0.028);

  const camera = new THREE.PerspectiveCamera(42, window.innerWidth / window.innerHeight, 0.1, 400);
  camera.position.set(0, 9, 42);

  const hemi = new THREE.HemisphereLight(0x2b3a52, 0x05070a, 0.55);
  scene.add(hemi);

  const moonLight = new THREE.DirectionalLight(0x9fb4d8, 0.9);
  moonLight.position.set(-30, 40, -20);
  moonLight.castShadow = true;
  moonLight.shadow.mapSize.set(1024, 1024);
  moonLight.shadow.camera.left = -40; moonLight.shadow.camera.right = 40;
  moonLight.shadow.camera.top = 40; moonLight.shadow.camera.bottom = -40;
  moonLight.shadow.camera.far = 120; moonLight.shadow.bias = -0.0015;
  scene.add(moonLight);

  const warmLights = [];
  function addWarmLight(x, y, z, intensity = 1.1, distance = 9) {
    const l = new THREE.PointLight(COLORS.amber, intensity, distance, 2);
    l.position.set(x, y, z);
    scene.add(l);
    warmLights.push(l);
    return l;
  }

  const terrainGeo = new THREE.PlaneGeometry(220, 220, 90, 90);
  terrainGeo.rotateX(-Math.PI / 2);
  const posAttr = terrainGeo.attributes.position;
  for (let i = 0; i < posAttr.count; i++) {
    const x = posAttr.getX(i), z = posAttr.getZ(i);
    const d = Math.sqrt(x * x + z * z);
    const path = Math.exp(-((x * x) / 40)) * 1.2;
    let h = fbm(x * 0.035, z * 0.035, 5) * 7.5 + fbm(x * 0.12, z * 0.12, 3) * 1.4;
    h -= path;
    h -= clamp((14 - d) / 14, 0, 1) * 1.5;
    posAttr.setY(i, h);
  }
  terrainGeo.computeVertexNormals();
  const terrain = new THREE.Mesh(terrainGeo, new THREE.MeshStandardMaterial({ color: COLORS.charcoal2, roughness: 1, metalness: 0.05 }));
  terrain.receiveShadow = true;
  terrain.position.y = -2.4;
  scene.add(terrain);

  function makeGlowTexture(hex) {
    const c = document.createElement("canvas");
    c.width = c.height = 256;
    const ctx = c.getContext("2d");
    const g = ctx.createRadialGradient(128, 128, 0, 128, 128, 128);
    const col = new THREE.Color(hex);
    const rgb = `${Math.floor(col.r * 255)},${Math.floor(col.g * 255)},${Math.floor(col.b * 255)}`;
    g.addColorStop(0, `rgba(${rgb},0.9)`);
    g.addColorStop(0.4, `rgba(${rgb},0.35)`);
    g.addColorStop(1, `rgba(${rgb},0)`);
    ctx.fillStyle = g; ctx.fillRect(0, 0, 256, 256);
    return new THREE.CanvasTexture(c);
  }

  const moonGroup = new THREE.Group();
  const moon = new THREE.Mesh(new THREE.SphereGeometry(6, 32, 32), new THREE.MeshBasicMaterial({ color: COLORS.vermilionBright }));
  moonGroup.add(moon);
  const moonGlow = new THREE.Sprite(new THREE.SpriteMaterial({ map: makeGlowTexture(COLORS.vermilion), transparent: true, depthWrite: false, blending: THREE.AdditiveBlending }));
  moonGlow.scale.set(46, 46, 1);
  moonGroup.add(moonGlow);
  moonGroup.position.set(-26, 34, -70);
  scene.add(moonGroup);

  function buildTorii() {
    const g = new THREE.Group();
    const pillarMat = new THREE.MeshStandardMaterial({ color: COLORS.vermilion, roughness: 0.7 });
    const beamMat = new THREE.MeshStandardMaterial({ color: 0x8a2c22, roughness: 0.6 });
    const pillarGeo = new THREE.CylinderGeometry(0.45, 0.55, 9, 12);
    [-3.4, 3.4].forEach((x) => {
      const p = new THREE.Mesh(pillarGeo, pillarMat);
      p.position.set(x, 4.5, 0); p.castShadow = true; p.receiveShadow = true;
      g.add(p);
    });
    const kasagi = new THREE.Mesh(new THREE.BoxGeometry(9.4, 0.7, 1.1), beamMat);
    kasagi.position.set(0, 8.9, 0); kasagi.castShadow = true; g.add(kasagi);
    const kasagiTop = new THREE.Mesh(new THREE.BoxGeometry(9.8, 0.35, 1.4), beamMat);
    kasagiTop.position.set(0, 9.32, 0); g.add(kasagiTop);
    const nuki = new THREE.Mesh(new THREE.BoxGeometry(8.6, 0.5, 0.5), pillarMat);
    nuki.position.set(0, 6.6, 0); nuki.castShadow = true; g.add(nuki);
    return g;
  }
  const torii = buildTorii();
  torii.position.set(2, -0.4, 14);
  scene.add(torii);

  function buildStairs(count, startY, startZ, endZ) {
    const g = new THREE.Group();
    const mat = new THREE.MeshStandardMaterial({ color: COLORS.stone, roughness: 0.95 });
    for (let i = 0; i < count; i++) {
      const t = i / count;
      const step = new THREE.Mesh(new THREE.BoxGeometry(6.4, 0.4, 0.7), mat);
      step.position.set(2, startY + t * 9, lerp(startZ, endZ, t));
      step.receiveShadow = true; step.castShadow = true;
      g.add(step);
    }
    return g;
  }
  scene.add(buildStairs(26, -2.2, 26, 6));

  function buildLantern(scale = 1) {
    const g = new THREE.Group();
    const stone = new THREE.MeshStandardMaterial({ color: COLORS.stone, roughness: 0.9 });
    const base = new THREE.Mesh(new THREE.CylinderGeometry(0.55, 0.7, 0.4, 8), stone);
    base.position.y = 0.2; g.add(base);
    const pole = new THREE.Mesh(new THREE.CylinderGeometry(0.22, 0.26, 1.6, 8), stone);
    pole.position.y = 1.2; g.add(pole);
    const chamber = new THREE.Mesh(new THREE.BoxGeometry(0.9, 0.9, 0.9), stone);
    chamber.position.y = 2.2; g.add(chamber);
    const glowMat = new THREE.MeshStandardMaterial({ color: COLORS.amber, emissive: COLORS.amber, emissiveIntensity: 1.6 });
    const inset = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.5, 0.5), glowMat);
    inset.position.y = 2.2; g.add(inset);
    const roof = new THREE.Mesh(new THREE.ConeGeometry(0.95, 0.6, 4), stone);
    roof.rotation.y = Math.PI / 4; roof.position.y = 3.0; g.add(roof);
    const cap = new THREE.Mesh(new THREE.SphereGeometry(0.14, 8, 8), stone);
    cap.position.y = 3.42; g.add(cap);
    g.traverse((o) => { if (o.isMesh) { o.castShadow = true; o.receiveShadow = true; } });
    g.scale.setScalar(scale);
    g.userData.glow = inset;
    return g;
  }
  const lanternPositions = [
    [-4.2, -1.6, 22], [4.6, -1, 18], [-4.4, -0.2, 12], [4.8, 0.6, 8],
    [-5, 1.4, 2], [5.4, 2.2, -4], [-5.6, 2.6, -10], [6, 2.8, -16],
    [-6.4, 3, -22], [6.8, 3, -28],
  ];
  const lanterns = lanternPositions.map(([x, y, z], i) => {
    const l = buildLantern(0.9 + (i % 3) * 0.06);
    l.position.set(x, y, z);
    scene.add(l);
    if (i % 2 === 0) addWarmLight(x, y + 2.4, z, 0.9, 8);
    return l;
  });

  function buildTemple() {
    const g = new THREE.Group();
    const wood = new THREE.MeshStandardMaterial({ color: COLORS.wood, roughness: 0.85 });
    const roofMat = new THREE.MeshStandardMaterial({ color: 0x0e1621, roughness: 0.65, metalness: 0.1 });
    const stone = new THREE.MeshStandardMaterial({ color: COLORS.stone, roughness: 0.95 });
    const platform = new THREE.Mesh(new THREE.BoxGeometry(20, 0.8, 14), stone);
    platform.position.y = 0.4; platform.receiveShadow = true; g.add(platform);
    const hall = new THREE.Mesh(new THREE.BoxGeometry(15, 5, 10), wood);
    hall.position.y = 3.3; hall.castShadow = true; hall.receiveShadow = true; g.add(hall);
    const roofLower = new THREE.Mesh(new THREE.ConeGeometry(13, 3, 4), roofMat);
    roofLower.rotation.y = Math.PI / 4; roofLower.position.y = 6.6; roofLower.scale.set(1, 0.55, 1);
    roofLower.castShadow = true; g.add(roofLower);
    const roofUpper = new THREE.Mesh(new THREE.ConeGeometry(8.4, 2.6, 4), roofMat);
    roofUpper.rotation.y = Math.PI / 4; roofUpper.position.y = 8.6; roofUpper.scale.set(1, 0.6, 1);
    roofUpper.castShadow = true; g.add(roofUpper);
    const finial = new THREE.Mesh(new THREE.CylinderGeometry(0.12, 0.12, 1.4, 6), stone);
    finial.position.y = 10.2; g.add(finial);
    const glowMat = new THREE.MeshStandardMaterial({ color: COLORS.amber, emissive: COLORS.amberBright, emissiveIntensity: 2.2 });
    for (let i = -1; i <= 1; i += 2) {
      const win = new THREE.Mesh(new THREE.PlaneGeometry(3.4, 2.6), glowMat);
      win.position.set(i * 4.4, 3.4, 5.01);
      g.add(win);
    }
    const doorGlow = new THREE.Mesh(new THREE.PlaneGeometry(2.6, 3), glowMat);
    doorGlow.position.set(0, 3.1, 5.01);
    g.add(doorGlow);
    const postGeo = new THREE.CylinderGeometry(0.28, 0.32, 5.2, 8);
    [[-7.2, -4.8], [7.2, -4.8], [-7.2, 4.8], [7.2, 4.8]].forEach(([x, z]) => {
      const p = new THREE.Mesh(postGeo, wood);
      p.position.set(x, 3.3, z); p.castShadow = true;
      g.add(p);
    });
    return g;
  }
  const temple = buildTemple();
  temple.position.set(0, -1.6, -6);
  scene.add(temple);
  addWarmLight(0, 3.6, -1, 1.4, 12);
  addWarmLight(-4, 3.4, -0.9, 1.0, 9);
  addWarmLight(4, 3.4, -0.9, 1.0, 9);

  function buildTrees() {
    const trunkMat = new THREE.MeshStandardMaterial({ color: 0x120d0a, roughness: 1 });
    const foliageMat = new THREE.MeshStandardMaterial({ color: 0x0e1c15, rough
