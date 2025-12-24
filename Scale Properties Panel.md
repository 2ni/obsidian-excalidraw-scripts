/*
 * Excalidraw Geometry Pro (v34 - Draggable Panel)
 *
 * :setf javascript
 * :iunmap <buffer> <Tab>
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
  useCenter: saved.useCenter ?? false,
  origins: Array.isArray(saved.origins) ? saved.origins : [{ id: "00", x: 0, y: 0, angle: 0, length: 100, persistent: true, visualIds: [] }],
  activeOriginId: saved.activeOriginId ?? "00",
  panelPos: saved.panelPos ?? { top: "110px", right: "33px" } // Store position
};

let state = {
  mode: "idle",
  editingField: null,
  originalValue: null,
  timer: null,
  snapPoint: null,
  tempStart: null,
  capturedSelection: []
};

// --- Math Helpers ---
const toRad = (deg) => deg * Math.PI / 180;
const toDeg = (rad) => rad * 180 / Math.PI;
const toMm = (px) => px * 25.4 / 96;
const toPx = (mm) => mm * 96 / 25.4;
const toMeters = (px) => toMm(px) * config.scale / 1000;
/**
 * Normalizes angle to (-180, 180] range
 * and prevents the -0.00 display artifact
 */
const formatAngle = (rad) => {
  let deg = toDeg(rad) % 360;
  if (deg > 180) deg -= 360;
  if (deg <= -180) deg += 360;
  return deg;
};

/**
 * Ensures value doesn't show -0.00 in UI
 */
const cleanValue = (val, precision) => {
  const fixed = val.toFixed(precision);
  // If rounded value is 0 (e.g. "0.00" or "-0.00"), force positive "0.00"
  if (parseFloat(fixed) === 0) return (0).toFixed(precision);
  return fixed;
};

const getOrigin = () => {
  const o = config.origins.find(o => o.id === config.activeOriginId) || config.origins[0];
  if (typeof o.angle === 'undefined') o.angle = 0;
  if (typeof o.length === 'undefined') o.length = 100;
  return o;
};

const toLocal = (p) => {
  const o = getOrigin();
  const dx = p.x - o.x; const dy = p.y - o.y;
  const cos = Math.cos(-o.angle); const sin = Math.sin(-o.angle);
  return { x: dx * cos - dy * sin, y: dx * sin + dy * cos };
};

const toGlobal = (p) => {
  const o = getOrigin();
  const cos = Math.cos(o.angle); const sin = Math.sin(o.angle);
  return { x: (p.x * cos - p.y * sin) + o.x, y: (p.x * sin + p.y * cos) + o.y };
};

// --- Snapping & Overlay ---
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

const drawOverlay = (snapP, startP, currP) => {
  const svg = getOverlay();
  let html = "";
  if (snapP) {
    html += `<circle cx="${snapP.vpX}" cy="${snapP.vpY}" r="6" fill="none" stroke="#e03131" stroke-width="2" />
             <circle cx="${snapP.vpX}" cy="${snapP.vpY}" r="1.5" fill="#e03131" />`;
  }
  if (startP && currP) {
    const api = view.excalidrawAPI;
    const appState = api.getAppState();
    const zoom = appState.zoom.value;
    const x1 = (startP.x + appState.scrollX) * zoom;
    const y1 = (startP.y + appState.scrollY) * zoom;
    const x2 = (currP.x + appState.scrollX) * zoom;
    const y2 = (currP.y + appState.scrollY) * zoom;
    html += `<line x1="${x1}" y1="${y1}" x2="${x2}" y2="${y2}" stroke="#e03131" stroke-width="1" stroke-dasharray="4" />`;
  }
  svg.innerHTML = html;
};

/*
 * TODO: do not calculate the points everytime the mouse moves
 *       but instead when the zoom or screen moves
 */
const getSnapPoint = (pointer) => {
  const api = view.excalidrawAPI;
  const appState = api.getAppState();
  const zoom = appState.zoom.value;
  const threshold = 20 / zoom;
  const sceneEls = api.getSceneElements();

  let best = null, minD = threshold;

  const rotatePoint = (px, py, cx, cy, angle) => {
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);
    const dx = px - cx;
    const dy = py - cy;
    return {
      x: cx + (dx * cos - dy * sin),
      y: cy + (dx * sin + dy * cos)
    };
  };

  for (const el of sceneEls) {
    if (el.isDeleted || el.type === "selection" || (getOrigin().visualIds || []).includes(el.id)) continue;

    const points = [];
    const cx = el.x + el.width / 2;
    const cy = el.y + el.height / 2;

    if (el.type === "line" || el.type === "arrow" || el.type === "freedraw") {
      // For multi-point elements
      const globalPoints = el.points.map(([px, py]) => {
        // Points are relative to el.x, el.y but rotation is around cx, cy
        return rotatePoint(el.x + px, el.y + py, cx, cy, el.angle);
      });

      points.push(...globalPoints);

      // Calculate midpoints of segments
      for (let i = 0; i < globalPoints.length - 1; i++) {
        points.push({
          x: (globalPoints[i].x + globalPoints[i + 1].x) / 2,
          y: (globalPoints[i].y + globalPoints[i + 1].y) / 2
        });
      }
    } else {
      // For rectangular elements (Rectangle, Diamond, Ellipse, Image, Text)
      const corners = [
        { x: el.x, y: el.y }, // Top-Left
        { x: el.x + el.width, y: el.y }, // Top-Right
        { x: el.x, y: el.y + el.height }, // Bottom-Left
        { x: el.x + el.width, y: el.y + el.height }, // Bottom-Right
        { x: cx, y: cy } // Center
      ];

      // Add mid-edges
      corners.push({ x: cx, y: el.y }); // Top-Mid
      corners.push({ x: cx, y: el.y + el.height }); // Bottom-Mid
      corners.push({ x: el.x, y: cy }); // Left-Mid
      corners.push({ x: el.x + el.width, y: cy }); // Right-Mid

      points.push(...corners.map(p => rotatePoint(p.x, p.y, cx, cy, el.angle)));
    }

    for (const p of points) {
      const d = Math.hypot(pointer.x - p.x, pointer.y - p.y);
      if (d < minD) {
        minD = d;
        best = p;
      }
    }
  }

  if (best) return {
    x: best.x,
    y: best.y,
    vpX: (best.x + appState.scrollX) * zoom,
    vpY: (best.y + appState.scrollY) * zoom
  };
  return null;
};

// --- UI Construction ---
const panel = document.createElement("div");
panel.id = panelId;
panel.style.cssText = `position:absolute; top:${config.panelPos.top}; right:${config.panelPos.right}; width:260px; background:var(--background-secondary); border:1px solid var(--divider-color); box-shadow:0 4px 12px rgba(0,0,0,0.15); border-radius:8px; padding:10px; z-index:100; font-size:11px; display:flex; flex-direction:column; gap:6px; max-height: 85vh; overflow-y:auto; font-family: var(--font-ui); color: var(--text-normal);`;

const buildInputRow = (label, id) => `<div style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">${label}</span><input type="number" step="any" data-id="${id}" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>`;
const buildSection = (title) => `<div style="font-weight:600; opacity:0.8; margin-top:4px; border-bottom:1px solid var(--divider-color); padding-bottom:2px;">${title}</div>`;
const buildInputRow2 = (label1, id1, label2, id2) => `
<div style="display:flex; justify-content:space-between; gap:6px; align-items:center;">
  <div style="display:flex; justify-content:space-between; align-items:center; flex:1;">
    <span style="opacity:0.8">${label1}</span>
    <input type="number" step="any" data-id="${id1}" class="geo-input" style="width:75px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;">
  </div>
  <div style="display:flex; justify-content:space-between; align-items:center; flex:1;">
    <span style="opacity:0.8">${label2}</span>
    <input type="number" step="any" data-id="${id2}" class="geo-input" style="width:75px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;">
  </div>
</div>`;

panel.innerHTML = `
<div style="display:flex; justify-content:space-between; align-items:center; font-weight:bold; margin-bottom:4px;">
  <span>📏 Geometry Pro</span>
  <div style="display:flex; gap:8px; align-items:center;">
    <div id="btn-drag" style="cursor:grab; padding:2px 4px; font-size: 14px; color: var(--text-muted);">⠿</div>
    <div id="btn-close" style="cursor:pointer; padding:2px 6px;">✕</div>
  </div>
</div>
${buildSection("Element (px)")}
<div style="display:flex; align-items:center; gap:6px; margin-bottom:4px;">
  <input type="checkbox" id="chk-center" ${config.useCenter ? "checked" : ""}>
  <label for="chk-center" style="opacity:0.9; cursor:pointer;">Use Center</label>
</div>
${buildInputRow2("X px", "x_px", "Y px", "y_px")}
${buildInputRow2("W px", "w_px", "H px", "h_px")}
${buildInputRow("Angle °", "el_angle")}
${buildSection("Element (mm)")}
${buildInputRow2("X mm", "x_mm", "Y mm", "y_mm")}
${buildInputRow2("W mm", "w_mm", "H mm", "h_mm")}
${buildSection("Element (m)")}
${buildInputRow2("X m", "x_m", "Y m", "y_m")}
${buildInputRow2("W m", "w_m", "H m", "h_m")}
${buildSection("Coordinate System")}
${buildInputRow("Scale 1:", "scale")}
${buildInputRow2("X0 px", "ox0_px", "Y0 px", "oy0_px")}
${buildInputRow2("X1 px", "ox1_px", "Y1 px", "oy1_px")}
${buildInputRow2("X0 mm", "ox0_mm", "Y0 mm", "oy0_mm")}
${buildInputRow2("X1 mm", "ox1_mm", "Y1 mm", "oy1_mm")}
${buildInputRow2("X0 m", "ox0_m", "Y0 m", "oy0_m")}
${buildInputRow2("X1 m", "ox1_m", "Y1 m", "oy1_m")}
${buildInputRow("Angle °", "o_angle")}
<div style="display:flex; gap:5px; margin-top:8px; width:100%;">
  <button id="btn-add" style="flex:1 1 0; width:0; min-width:0; height:24px; padding:0; display:flex; align-items:center; justify-content:center; box-sizing:border-box;">New</button>
  <button id="btn-del" style="flex:1 1 0; width:0; min-width:0; height:24px; padding:0; display:flex; align-items:center; justify-content:center; box-sizing:border-box;">Delete</button>
</div>
<div style="display:flex; gap:5px; margin-top:4px; width:100%;">
  <button id="btn-front" style="flex:1 1 0; width:0; min-width:0; height:24px; padding:0; display:flex; align-items:center; justify-content:center; box-sizing:border-box;">To front</button>
  <select id="origin-select" style="flex:1 1 0; width:0; min-width:0; height:24px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px; box-sizing:border-box; cursor:pointer; padding:0 2px; font-size:13px; text-align: center;"></select>
</div>
`;
view.contentEl.appendChild(panel);

// --- Draggable Logic ---
const dragBtn = panel.querySelector("#btn-drag");
dragBtn.onmousedown = (e) => {
    e.preventDefault();
    let startX = e.clientX, startY = e.clientY;
    document.onmousemove = (e) => {
        let dx = startX - e.clientX, dy = startY - e.clientY;
        startX = e.clientX; startY = e.clientY;
        panel.style.top = (panel.offsetTop - dy) + "px";
        panel.style.right = (parseInt(getComputedStyle(panel).right) + dx) + "px";
    };
    document.onmouseup = () => {
        document.onmousemove = null; document.onmouseup = null;
        config.panelPos = { top: panel.style.top, right: panel.style.right };
        saveSettings(config);
    };
};

// --- Rest of logic (from v33) ---
const updateOriginVisuals = async (o) => {
  if (o.visualIds.length > 0) {
    const sceneEls = view.excalidrawAPI.getSceneElements();
    const toDelete = sceneEls.filter(el => o.visualIds.includes(el.id));
    await ea.deleteViewElements(toDelete);
  }
  ea.clear();
  ea.style.strokeColor = "#e03131"; ea.style.strokeWidth = 1; ea.style.roughness = 0; ea.style.fontSize = 16; ea.style.fontFamily = 3;
  const p0 = { x: o.x, y: o.y };
  const pX = toGlobal({ x: o.length, y: 0 });
  const pY = toGlobal({ x: 0, y: o.length });
  const vIds = [
    ea.addArrow([[p0.x, p0.y], [pX.x, pX.y]], { endArrowHead: "triangle", strokeColor: "#ff0000" }),
    ea.addArrow([[p0.x, p0.y], [pY.x, pY.y]], { endArrowHead: "triangle", strokeColor: "#00aa00" }),
    ea.addText(pX.x + 5, pX.y + 5, `X (${o.id.toUpperCase()})`, { strokeColor: "#ff0000" }),
    ea.addText(pY.x + 5, pY.y + 5, `Y`, { strokeColor: "#00aa00" })
  ];
  ea.addToGroup(vIds);
  await ea.addElementsToView(false, false);
  const sceneEls = view.excalidrawAPI.getSceneElements();
  const createdEls = sceneEls.filter(el => vIds.includes(el.id));
  ea.clear();
  ea.copyViewElementsToEAforEditing(createdEls);
  ea.getElements().forEach(el => el.locked = true);
  await ea.addElementsToView(false, false);
  o.visualIds = vIds;
};

const applyInputToScene = async (id, val) => {
  if (isNaN(val)) return;
  if (id === "scale") { config.scale = val; saveSettings(config); return; }
  const o = getOrigin();
  if (id.startsWith("o") || id === "o_angle") {
    let p1 = { x: o.x + o.length * Math.cos(o.angle), y: o.y + o.length * Math.sin(o.angle) };
    if (id === "ox0_px") o.x = val; if (id === "oy0_px") o.y = val;
    if (id === "ox0_mm") o.x = toPx(val); if (id === "oy0_mm") o.y = toPx(val);
    if (id === "ox0_m") o.x = toPx((val * 1000) / (config.scale / 100)); if (id === "oy0_m") o.y = toPx((val * 1000) / (config.scale / 100));
    let updateP1 = false;
    if (id === "ox1_px") { p1.x = val; updateP1 = true; } if (id === "oy1_px") { p1.y = val; updateP1 = true; }
    if (id === "ox1_mm") { p1.x = toPx(val); updateP1 = true; } if (id === "oy1_mm") { p1.y = toPx(val); updateP1 = true; }
    if (id === "ox1_m") { p1.x = toPx((val * 1000) / (config.scale / 100)); updateP1 = true; } if (id === "oy1_m") { p1.y = toPx((val * 1000) / (config.scale / 100)); updateP1 = true; }
    if (updateP1) { o.angle = Math.atan2(p1.y - o.y, p1.x - o.x); o.length = Math.hypot(p1.x - o.x, p1.y - o.y); }
    if (id === "o_angle") o.angle = toRad(val);
    saveSettings(config); await updateOriginVisuals(o); return;
  }
  const el = ea.getViewSelectedElement();
  const uc = config.useCenter;
  if (el) {
    ea.clear(); await ea.copyViewElementsToEAforEditing([el]);
    const activeEl = ea.getElements()[0];
    let cx = activeEl.x + activeEl.width / 2; let cy = activeEl.y + activeEl.height / 2;
    let localC = toLocal({ x: cx, y: cy });
    const applyPos = (lx, ly) => {
        const globalAnchor = toGlobal({ x: lx, y: ly });
        if (uc) { activeEl.x = globalAnchor.x - activeEl.width / 2; activeEl.y = globalAnchor.y - activeEl.height / 2; }
        else { activeEl.x = globalAnchor.x; activeEl.y = globalAnchor.y; }
    };
    const setW = (v) => { const oldC = toLocal({x: activeEl.x + activeEl.width/2, y: activeEl.y + activeEl.height/2}); activeEl.width = v; if(uc) applyPos(oldC.x, oldC.y); };
    const setH = (v) => { const oldC = toLocal({x: activeEl.x + activeEl.width/2, y: activeEl.y + activeEl.height/2}); activeEl.height = v; if(uc) applyPos(oldC.x, oldC.y); };
    if (id === "x_px") applyPos(val, localC.y); if (id === "y_px") applyPos(localC.x, val);
    if (id === "w_px") setW(val); if (id === "h_px") setH(val);
    if (id === "x_mm") applyPos(toPx(val), localC.y); if (id === "y_mm") applyPos(localC.x, toPx(val));
    if (id === "w_mm") setW(toPx(val)); if (id === "h_mm") setH(toPx(val));
    const toPxS = (m) => toPx((m * 1000) / (config.scale / 100));
    if (id === "x_m") applyPos(toPxS(val), localC.y); if (id === "y_m") applyPos(localC.x, toPxS(val));
    if (id === "w_m") setW(toPxS(val)); if (id === "h_m") setH(toPxS(val));
    if (id === "el_angle") activeEl.angle = toRad(val) + o.angle;
    await ea.addElementsToView(false, false);
  }
};

const updateUI = (force = false) => {
  if (state.mode !== "idle" && !force) return;
  const o = getOrigin();
  const setV = (id, v, p=2) => {
    if (state.editingField === id && !force) return;
    const i = panel.querySelector(`[data-id="${id}"]`);
    if (i) i.value = cleanValue(v, p);
  };

  // Origin Section
  setV("ox0_px", o.x); setV("oy0_px", o.y);
  setV("ox0_mm", toMm(o.x)); setV("oy0_mm", toMm(o.y));
  setV("ox0_m", toMeters(o.x), 3); setV("oy0_m", toMeters(o.y), 3);

  const p1 = { x: o.x + o.length * Math.cos(o.angle), y: o.y + o.length * Math.sin(o.angle) };
  setV("ox1_px", p1.x); setV("oy1_px", p1.y);
  setV("ox1_mm", toMm(p1.x)); setV("oy1_mm", toMm(p1.y));
  setV("ox1_m", toMeters(p1.x), 3); setV("oy1_m", toMeters(p1.y), 3);

  // Apply normalized angle display
  setV("o_angle", formatAngle(o.angle));
  setV("scale", config.scale, 0);

  const el = ea.getViewSelectedElement();
  if (!el) {
    panel.querySelectorAll('input:not([data-id^="o"]):not([data-id="scale"])').forEach(i => {
      if (state.editingField !== i.dataset.id) i.value = "";
    });
    return;
  }

  let targetP = config.useCenter ? { x: el.x + el.width / 2, y: el.y + el.height / 2 } : { x: el.x, y: el.y };
  const localP = toLocal(targetP);

  setV("x_px", localP.x); setV("y_px", localP.y);
  setV("w_px", el.width); setV("h_px", el.height);
  setV("x_mm", toMm(localP.x)); setV("y_mm", toMm(localP.y));
  setV("w_mm", toMm(el.width)); setV("h_mm", toMm(el.height));
  setV("x_m", toMeters(localP.x), 3); setV("y_m", toMeters(localP.y), 3);
  setV("w_m", toMeters(el.width), 3); setV("h_m", toMeters(el.height), 3);

  // Element Angle relative to LCS (Local Coordinate System)
  setV("el_angle", formatAngle(el.angle - o.angle));
};

const startOriginCreation = () => {
  state.capturedSelection = ea.getViewSelectedElements();
  state.mode = "placing_start"; state.tempStart = null;
  panel.querySelector("#btn-add").innerText = "Click Start Point";
  const excalContainer = view.contentEl.querySelector(".excalidraw");
  if(excalContainer) excalContainer.style.cursor = "crosshair";

  const onMove = (e) => {
    let pos = ea.getViewLastPointerPosition(); if (!pos) return;
    state.snapPoint = (e.metaKey || e.ctrlKey) ? getSnapPoint(pos) : null;
    let target = state.snapPoint || pos;
    if (e.shiftKey && state.tempStart) {
        const dx = target.x - state.tempStart.x; const dy = target.y - state.tempStart.y;
        const dist = Math.hypot(dx, dy); const snappedAngle = Math.round(toDeg(Math.atan2(dy, dx)) / 15) * 15;
        target = { x: state.tempStart.x + dist * Math.cos(toRad(snappedAngle)), y: state.tempStart.y + dist * Math.sin(toRad(snappedAngle)) };
    }
    drawOverlay(state.snapPoint, state.tempStart, target);
  };

  const onClick = async (e) => {
    if (panel.contains(e.target)) return;
    e.stopPropagation(); e.preventDefault();
    let pos = ea.getViewLastPointerPosition();
    let target = state.snapPoint || pos;
    if (e.shiftKey && state.tempStart) {
        const dx = target.x - state.tempStart.x; const dy = target.y - state.tempStart.y;
        const dist = Math.hypot(dx, dy); const snappedAngle = Math.round(toDeg(Math.atan2(dy, dx)) / 15) * 15;
        target = { x: state.tempStart.x + dist * Math.cos(toRad(snappedAngle)), y: state.tempStart.y + dist * Math.sin(toRad(snappedAngle)) };
    }
    if (state.mode === "placing_start") { state.tempStart = target; state.mode = "placing_end"; panel.querySelector("#btn-add").innerText = "Click X End (Shift=15°)"; }
    else if (state.mode === "placing_end") {
        const dx = target.x - state.tempStart.x; const dy = target.y - state.tempStart.y;
        const id = Math.random().toString(36).substring(2, 4);
        const newOrigin = { id, x: state.tempStart.x, y: state.tempStart.y, angle: Math.atan2(dy, dx), length: Math.hypot(dx, dy) || 100, visualIds: [] };
        config.origins.push(newOrigin); config.activeOriginId = id;
        await updateOriginVisuals(newOrigin); saveSettings(config); refreshDropdown(); updateUI(true);
        if (state.capturedSelection?.length > 0) ea.selectElementsInView(state.capturedSelection);
        stop();
    }
  };
  const stop = () => {
    state.mode = "idle"; state.tempStart = null; drawOverlay(null, null, null);
    panel.querySelector("#btn-add").innerText = "New";
    if(excalContainer) excalContainer.style.cursor = "";
    window.removeEventListener("mousemove", onMove); window.removeEventListener("mousedown", onClick, true); window.removeEventListener("keydown", onEsc, true);
  };
  const onEsc = (e) => { if (e.key === "Escape") stop(); };
  window.addEventListener("mousemove", onMove); window.addEventListener("mousedown", onClick, true); window.addEventListener("keydown", onEsc, true);
};

const refreshDropdown = () => {
  const select = panel.querySelector("#origin-select");
  if (select && Array.isArray(config.origins)) {
    select.innerHTML = config.origins.map(o => `<option value="${o.id}" ${o.id === config.activeOriginId ? "selected" : ""}>${o.id.toUpperCase()}</option>`).join("");
  }
};
panel.addEventListener("change", (e) => {
  if (e.target.id === "chk-center") { config.useCenter = e.target.checked; saveSettings(config); updateUI(true); }
  if (e.target.id === "origin-select") { config.activeOriginId = e.target.value; saveSettings(config); updateUI(true); }
});
panel.addEventListener("focusin", (e) => { if (e.target.classList.contains("geo-input")) { state.editingField = e.target.dataset.id; state.originalValue = e.target.value; } });
panel.addEventListener("focusout", () => { state.editingField = null; });
panel.addEventListener("keydown", async (e) => {
  if (!e.target.classList.contains("geo-input")) return;
  if (e.key === "Escape") { e.target.value = state.originalValue; e.target.blur(); return; }
  if (e.key === "Enter" || e.key === "Tab") {
    await applyInputToScene(e.target.dataset.id, parseFloat(e.target.value));
    updateUI(true);
    if (e.key === "Enter") { e.preventDefault(); e.target.select(); }
  }
});
panel.addEventListener("click", async (e) => {
  if (e.target.id === "btn-close") { clearInterval(state.timer); panel.remove(); document.querySelector("#geo-snap-overlay")?.remove(); }
  if (e.target.id === "btn-add") startOriginCreation();
  if (e.target.id === "btn-front") {
    const allOriginIds = config.origins.flatMap(o => o.visualIds);
    const sceneEls = view.excalidrawAPI.getSceneElements();
    const toFront = sceneEls.filter(el => allOriginIds.includes(el.id));
    if (toFront.length > 0) { ea.clear(); await ea.copyViewElementsToEAforEditing(toFront); await ea.deleteViewElements(toFront); await ea.addElementsToView(false, false, true); }
  }
  if (e.target.id === "btn-del") {
    const active = getOrigin(); if (active.persistent) return;
    const sceneEls = view.excalidrawAPI.getSceneElements();
    const toDelete = sceneEls.filter(el => active.visualIds.includes(el.id));
    await ea.deleteViewElements(toDelete);
    config.origins = config.origins.filter(o => o.id !== active.id); config.activeOriginId = "00";
    saveSettings(config); refreshDropdown(); updateUI(true);
  }
});

refreshDropdown();
state.timer = setInterval(updateUI, 200);
