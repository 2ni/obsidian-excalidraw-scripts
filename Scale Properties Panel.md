/*
 * Excalidraw Geometry Pro (v29)
 *
 * :setf javascript
 * :iunmap <buffer> <Tab
 */

const panelId = "geometry-pro-panel";
const view = app.workspace.getActiveViewOfType(customElements.get("excalidraw-view")?.constructor || Object);
if (!view || !view.excalidrawAPI) { new Notice("Open an Excalidraw drawing first"); return; }

const existingPanel = view.contentEl.querySelector(`#${panelId}`);
if (existingPanel) existingPanel.remove();

const getSettings = () => ea.getScriptSettings() ?? {};
const saveSettings = (s) => ea.setScriptSettings({ ...getSettings(), [view.file.path]: s });

let saved = getSettings()[view.file.path] || {};
let config = {
  scale: saved.scale ?? 100,
  origins: Array.isArray(saved.origins) ? saved.origins : [{ id: "00", x: 0, y: 0, persistent: true, visualIds: [] }],
  activeOriginId: saved.activeOriginId ?? "00"
};

let state = { isAdding: false, editingField: null, timer: null, capturedSelection: [], snapPoint: null };

// --- Snapping UI Overlay ---
const getOverlay = () => {
  let svg = view.contentEl.querySelector("#geo-snap-overlay");
  if (!svg) {
    svg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
    svg.id = "geo-snap-overlay";
    svg.style.cssText = "position:absolute; top:0; left:0; width:100%; height:100%; pointer-events:none; z-index:9;";
    view.contentEl.querySelector(".excalidraw").appendChild(svg);
  }
  return svg;
};

const drawSnapIndicator = (p) => {
  const svg = getOverlay();
  svg.innerHTML = p ? `<circle cx="${p.vpX}" cy="${p.vpY}" r="6" fill="none" stroke="#e03131" stroke-width="2" />
                       <circle cx="${p.vpX}" cy="${p.vpY}" r="1.5" fill="#e03131" />` : "";
};

// --- Conversion Helpers ---
const toMm = (px) => px * 25.4 / 96;
const toPx = (mm) => mm * 96 / 25.4;
const toMeters = (px) => toMm(px) * config.scale / 1000;
const getOrigin = () => config.origins.find(o => o.id === config.activeOriginId) || config.origins[0];
const toLocal = (val, axis) => val - getOrigin()[axis];

// --- Snapping Logic ---
const getSnapPoint = (pointer) => {
  const api = view.excalidrawAPI;
  const appState = api.getAppState();
  const zoom = appState.zoom.value;
  const threshold = 20 / zoom;
  const sceneEls = api.getSceneElements();
  const midpoint = (a, b) => ({ x: (a.x + b.x) / 2, y: (a.y + b.y) / 2, });

  let best = null, minD = threshold;
  for (const el of sceneEls) {
    if (el.isDeleted || el.type === "selection") continue;
    const points = [];
    if (Array.isArray(el.points)) {
      const segments = el.points.map(([px, py]) => ({ x: el.x + px, y: el.y + py }));
      points.push(...segments);

      // midpoints of each line
      for (let i = 0; i < segments.length - 1; i++) {
        points.push(midpoint(segments[i], segments[i + 1]));
      }
    } else {
      points.push(...[
        { x: el.x, y: el.y },
        { x: el.x + el.width, y: el.y }, { x: el.x, y: el.y + el.height },
        { x: el.x + el.width, y: el.y + el.height },
        { x: el.x + el.width / 2, y: el.y + el.height / 2 }
      ]);
    }

    for (const p of points) {
      const d = Math.hypot(pointer.x - p.x, pointer.y - p.y);
      if (d < minD) { minD = d; best = p; }
    }
  }
  if (best) return { x: best.x, y: best.y, vpX: (best.x + appState.scrollX) * zoom, vpY: (best.y + appState.scrollY) * zoom };
  return null;
};

// --- UI Construction ---
const panel = document.createElement("div");
panel.id = panelId;
panel.style.cssText = `position:absolute; top:110px; right:33px; width:240px; background:var(--background-secondary); border:1px solid var(--divider-color); box-shadow:0 4px 12px rgba(0,0,0,0.15); border-radius:8px; padding:10px; z-index:100; font-size:11px; display:flex; flex-direction:column; gap:6px; max-height: 85vh; overflow-y:auto; font-family: var(--font-ui); color: var(--text-normal);`;

const buildInputRow = (label, id) => `<div style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">${label}</span><input type="number" step="any" data-id="${id}" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>`;
const buildSection = (title) => `<div style="font-weight:600; opacity:0.8; margin-top:4px; border-bottom:1px solid var(--divider-color); padding-bottom:2px;">${title}</div>`;

const buildInputRow2 = (label1, id1, label2, id2) => `
<div style="display:flex; justify-content:space-between; gap:6px; align-items:center;">
  <div style="display:flex; justify-content:space-between; align-items:center; flex:1;">
    <span style="opacity:0.8">${label1}</span>
    <input type="number" step="any" data-id="${id1}" class="geo-input"
      style="width:70px; text-align:right; padding:2px 4px;
             background:var(--background-primary);
             border:1px solid var(--divider-color);
             color:var(--text-normal); border-radius:4px;">
  </div>
  <div style="display:flex; justify-content:space-between; align-items:center; flex:1;">
    <span style="opacity:0.8">${label2}</span>
    <input type="number" step="any" data-id="${id2}" class="geo-input"
      style="width:70px; text-align:right; padding:2px 4px;
             background:var(--background-primary);
             border:1px solid var(--divider-color);
             color:var(--text-normal); border-radius:4px;">
  </div>
</div>`;

panel.innerHTML = `
<div style="display:flex; justify-content:space-between; align-items:center; font-weight:bold; margin-bottom:4px;">
  <span>📏 Geometry Pro</span>
  <div id="btn-close" style="cursor:pointer; padding:2px 6px;">✕</div>
</div>

${buildSection("Element (px)")}
${buildInputRow2("X px", "x_px", "Y px", "y_px")}
${buildInputRow2("W px", "w_px", "H px", "h_px")}

${buildSection("Element (mm)")}
${buildInputRow2("X mm", "x_mm", "Y mm", "y_mm")}
${buildInputRow2("W mm", "w_mm", "H mm", "h_mm")}

${buildSection("Element (scaled)")}
${buildInputRow2("X m", "x_m", "Y m", "y_m")}
${buildInputRow2("W m", "w_m", "H m", "h_m")}

${buildSection("Origins")}
${buildInputRow2("X px", "ox_px", "Y px", "oy_px")}
${buildInputRow2("X mm", "ox_mm", "Y mm", "oy_mm")}
${buildInputRow2("X m", "ox_m", "Y m", "oy_m")}

${buildSection("Configuration")}
${buildInputRow("Scale 1:", "scale")}

<div style="display:flex; gap:5px; margin-top:8px;">
  <button id="btn-add" style="flex:1;">Add Origin</button>
  <button id="btn-del" style="flex:1;">Delete Origin</button>
</div>

<button id="btn-front" style="width:100%; margin-top:2px;">Origins to front</button>

<select id="origin-select" style="width:100%; margin-top:5px;
  background:var(--background-primary);
  border:1px solid var(--divider-color);
  color: var(--text-normal);">
</select>
`;

view.contentEl.appendChild(panel);

// --- Core API Logic ---
const createVisuals = async (x, y, id) => {
  ea.clear();
  ea.style.strokeColor = "#e03131";
  ea.style.backgroundColor = "#ffc9c9";
  ea.style.fillStyle = "solid";
  ea.style.strokeWidth = 1;
  ea.style.roughness = 0;
  ea.style.fontSize = 16;
  const visualIds = [
    ea.addArrow([[x, y], [x + 50, y]], { startArrowHead: null, endArrowHead: "triangle" }),
    ea.addArrow([[x, y], [x, y + 50]], { startArrowHead: null, endArrowHead: "triangle" }),
    ea.addText(x + 3, y + 3, id.toUpperCase())
  ];
  ea.addToGroup(visualIds);
  await ea.addElementsToView(false, false);

  // Lock them manually
  const sceneEls = view.excalidrawAPI.getSceneElements();
  const createdEls = sceneEls.filter(el => visualIds.includes(el.id));
  ea.clear();
  ea.copyViewElementsToEAforEditing(createdEls);
  ea.getElements().forEach(el => el.locked = true);
  await ea.addElementsToView(false, false);

  if (state.capturedSelection?.length > 0) ea.selectElementsInView(state.capturedSelection);
  return visualIds;
};

const bringOriginsToFront = async () => {
  const allOriginIds = config.origins.flatMap(o => o.visualIds);

  // 1. Fetch elements from the scene
  const sceneEls = view.excalidrawAPI.getSceneElements();
  const toFront = sceneEls.filter(el => allOriginIds.includes(el.id));

  if (toFront.length > 0) {
    // 2. Prepare the Workbench (EA Buffer)
    ea.clear();

    // 3. Copy to workbench for editing
    await ea.copyViewElementsToEAforEditing(toFront);

    // 4. Perform ALL modifications on the workbench
    // Set them to locked here so they are added to the front already locked
    // already locked ea.getElements().forEach(el => { console.log(el.id); el.locked = true; });

    // needed or the index will not be updated
    await ea.deleteViewElements(toFront);

    // 5. Push to view once with addAtFront = true
    // Parameters: (shouldRestoreElements: false, shouldRestoreSelection: false, addAtFront: true)
    await ea.addElementsToView(false, false, true);

    new Notice("Coordinate markers moved to front and locked");
  }
};

// --- Interaction Handlers ---
const startOriginCreation = () => {
  state.capturedSelection = ea.getViewSelectedElements();
  state.isAdding = true;
  panel.querySelector("#btn-add").innerText = "Click (Cmd=Snap)";
  const excalContainer = view.contentEl.querySelector(".excalidraw");
  if(excalContainer) excalContainer.style.cursor = "crosshair";

  const onMove = (e) => {
    let pos = ea.getViewLastPointerPosition();
    if (!pos) return;
    state.snapPoint = (e.metaKey || e.ctrlKey) ? getSnapPoint(pos) : null;
    const target = state.snapPoint || pos;
    drawSnapIndicator(state.snapPoint);

    const setV = (id, v, p=2) => { const i = panel.querySelector(`[data-id="${id}"]`); if(i && state.editingField !== id) i.value = v.toFixed(p); };
    setV("ox_px", target.x); setV("oy_px", target.y);
    setV("ox_mm", toMm(target.x)); setV("oy_mm", toMm(target.y));
    setV("ox_m", toMeters(target.x), 3); setV("oy_m", toMeters(target.y), 3);

    const el = ea.getViewSelectedElement();
    if (el) {
      const lx = el.x - target.x, ly = el.y - target.y;
      setV("x_px", lx); setV("y_px", ly);
      setV("x_mm", toMm(lx)); setV("y_mm", toMm(ly));
      setV("x_m", toMeters(lx), 3); setV("y_m", toMeters(ly), 3);
    }
  };

  const onClick = async (e) => {
    if (panel.contains(e.target)) return;
    e.stopPropagation();
    const pos = ea.getViewLastPointerPosition();
    const target = state.snapPoint || pos;
    if(target) {
      const id = Math.random().toString(36).substring(2, 4);
      const vIds = await createVisuals(target.x, target.y, id);
      config.origins.push({ id, x: target.x, y: target.y, visualIds: vIds });
      config.activeOriginId = id;
      saveSettings(config); refreshDropdown(); updateUI(true);
    }
    stop();
  };

  const stop = () => {
    state.isAdding = false; drawSnapIndicator(null);
    panel.querySelector("#btn-add").innerText = "Add Origin";
    if(excalContainer) excalContainer.style.cursor = "";
    window.removeEventListener("mousemove", onMove); window.removeEventListener("mousedown", onClick, true); window.removeEventListener("keydown", onEsc, true);
  };
  const onEsc = (e) => { if (e.key === "Escape") stop(); };
  window.addEventListener("mousemove", onMove); window.addEventListener("mousedown", onClick, true); window.addEventListener("keydown", onEsc, true);
};

const updateUI = (force = false) => {
  if (state.isAdding && !force) return;
  const o = getOrigin();
  const setV = (id, v, p=2) => { if (state.editingField !== id || force) { const i = panel.querySelector(`[data-id="${id}"]`); if (i) i.value = v.toFixed(p); }};
  setV("ox_px", o.x); setV("oy_px", o.y); setV("ox_mm", toMm(o.x)); setV("oy_mm", toMm(o.y));
  setV("ox_m", toMeters(o.x), 3); setV("oy_m", toMeters(o.y), 3);
  setV("scale", config.scale, 0);

  const el = ea.getViewSelectedElement();
  if (!el) { panel.querySelectorAll('input:not([data-id^="o"]):not([data-id="scale"])').forEach(i => i.value = ""); return; }
  const lx = toLocal(el.x, 'x'), ly = toLocal(el.y, 'y');
  setV("x_px", lx); setV("y_px", ly); setV("w_px", el.width); setV("h_px", el.height);
  setV("x_mm", toMm(lx)); setV("y_mm", toMm(ly)); setV("w_mm", toMm(el.width)); setV("h_mm", toMm(el.height));
  setV("x_m", toMeters(lx), 3); setV("y_m", toMeters(ly), 3); setV("w_m", toMeters(el.width), 3); setV("h_m", toMeters(el.height), 3);
};

const refreshDropdown = () => {
  const select = panel.querySelector("#origin-select");
  if (select && Array.isArray(config.origins)) {
    select.innerHTML = config.origins.map(o => `<option value="${o.id}" ${o.id === config.activeOriginId ? "selected" : ""}>LCS: ${o.id.toUpperCase()}</option>`).join("");
  }
};

panel.addEventListener("click", async (e) => {
  if (e.target.id === "btn-close") { clearInterval(state.timer); panel.remove(); document.querySelector("#geo-snap-overlay")?.remove(); }
  if (e.target.id === "btn-add") startOriginCreation();
  if (e.target.id === "btn-front") bringOriginsToFront();
  if (e.target.id === "btn-del") {
    const active = getOrigin();
    if (active.persistent) return;
    const sceneEls = view.excalidrawAPI.getSceneElements();
    const toDelete = sceneEls.filter(el => active.visualIds.includes(el.id));
    await ea.deleteViewElements(toDelete);
    config.origins = config.origins.filter(o => o.id !== active.id);
    config.activeOriginId = "00";
    saveSettings(config); refreshDropdown(); updateUI(true);
  }
});

panel.addEventListener("change", (e) => { if (e.target.id === "origin-select") { config.activeOriginId = e.target.value; saveSettings(config); updateUI(true); } });
panel.addEventListener("focusin", (e) => state.editingField = e.target.dataset.id);
panel.addEventListener("focusout", () => state.editingField = null);

refreshDropdown();
state.timer = setInterval(updateUI, 200);
