/*
 * Excalidraw Geometry Pro
 * TODO performance of snap points: use excalidrawCanvasChangeActive to handle which snapping points to use. When canvas is moved or zoomed -> calculate new snappingPoints
 */

const panelId = "geometry-pro-panel";
const view = app.workspace.getActiveViewOfType(customElements.get("excalidraw-view")?.constructor || Object);
if (!view || !view.excalidrawAPI) { new Notice("Open an Excalidraw drawing first"); return; }

const existingPanel = view.contentEl.querySelector(`#${panelId}`);
if (existingPanel) existingPanel.remove();

const getSettings = () => {
    const file = view.file;
    const cache = app.metadataCache.getFileCache(file);
    return cache?.frontmatter?.["excalidraw-coords"] ?? {};
};

const saveSettings = async (s) => {
    const file = view.file;
    await app.fileManager.processFrontMatter(file, (frontmatter) => {
        frontmatter["excalidraw-coords"] = s;
    });
};

let saved = getSettings();
let config = {
  scale: saved.scale ?? 100,
  useCenter: saved.useCenter ?? false,
  showPx: saved.showPx ?? true,
  showMm: saved.showMm ?? true,
  showM: saved.showM ?? false,
  origins: Array.isArray(saved.origins) ? saved.origins : [{ id: "00", x: 0, y: 0, angle: 0, length: 100, persistent: true, visualIds: [] }],
  activeOriginId: saved.activeOriginId ?? "00",
  panelPos: saved.panelPos ?? { top: "110px", right: "33px" }
};

let state = { mode: "idle", editingField: null, originalValue: null, snapPoint: null, tempStart: null, capturedSelection: [] };

// --- Transformation Math Helpers ---
const toRad = (deg) => deg * Math.PI / 180;
const toDeg = (rad) => rad * 180 / Math.PI;
const toMm = (px) => px * 25.4 / 96;
const toPx = (mm) => mm * 96 / 25.4;
const fromMetersToPx = (m) => toPx(m * 1000 / config.scale);
const toMeters = (px) => toMm(px) * config.scale / 1000;

const formatAngle = (rad) => {
  let deg = toDeg(rad) % 360;
  if (deg > 180) deg -= 360;
  if (deg <= -180) deg += 360;
  return deg;
};

const getOrigin = () => config.origins.find(o => o.id === config.activeOriginId) || config.origins[0];

const getCurrentAnchor = (element) => {
  const isLine = ["line", "arrow"].includes(element.type) && element.points.length === 2;

  if (config.useCenter) {
    if (isLine) {
      // 1. Start point (P1) is element.x, element.y
      const p1 = { x: element.x, y: element.y };

      // 2. End point (P2) is the relative point [1] rotated by element.angle
      const p2rel = element.points[1];
      const cos = Math.cos(element.angle), sin = Math.sin(element.angle);
      const p2 = {
        x: p1.x + (p2rel[0] * cos - p2rel[1] * sin),
        y: p1.y + (p2rel[0] * sin + p2rel[1] * cos)
      };

      // 3. Midpoint of the actual vector
      return {
        x: (p1.x + p2.x) / 2,
        y: (p1.y + p2.y) / 2
      };
    }

    // Default center for other shapes (Rect, Circle, etc.)
    return {
      x: element.x + element.width / 2,
      y: element.y + element.height / 2
    };
  }

  // Reference Point (Top-Left) logic
  const cx = element.x + element.width / 2;
  const cy = element.y + element.height / 2;
  const cos = Math.cos(element.angle), sin = Math.sin(element.angle);
  const dx = -element.width / 2, dy = -element.height / 2;
  return { x: cx + (dx * cos - dy * sin), y: cy + (dx * sin + dy * cos) };
};

/**
 * Transforms World (Excalidraw) to Local (Your CS)
 */
const toLocal = (worldP, worldAngle = 0) => {
  const o = getOrigin();
  const dx = worldP.x - o.x;
  const dy = worldP.y - o.y;
  const cos = Math.cos(-o.angle), sin = Math.sin(-o.angle);
  return {
    x: dx * cos - dy * sin,
    y: dx * sin + dy * cos,
    angle: worldAngle - o.angle
  };
};

/**
 * Transforms Local (UI) to World (Excalidraw)
 */
const toGlobal = (localP, localAngle = 0) => {
  const o = getOrigin();
  const cos = Math.cos(o.angle), sin = Math.sin(o.angle);
  const rx = localP.x * cos - localP.y * sin;
  const ry = localP.x * sin + localP.y * cos;
  return {
    x: rx + o.x,
    y: ry + o.y,
    angle: localAngle + o.angle
  };
};

const cleanValue = (val, precision) => {
  const fixed = val.toFixed(precision);
  return parseFloat(fixed) === 0 ? (0).toFixed(precision) : fixed;
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

// A simple throttle flag
let updateRequested = false;

const drawOverlay = (snapP, startP, currP) => {
  const svg = getOverlay();

  // Ensure the arrowhead definition exists once and persists
  if (!svg.querySelector("#geo-defs")) {
    svg.innerHTML = `
      <defs id="geo-defs">
        <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
          <polygon points="0 0, 10 3.5, 0 7" fill="context-stroke" />
        </marker>
      </defs>
      <g id="geo-content"></g>`;
  }

  const contentGroup = svg.querySelector("#geo-content");
  const api = view.excalidrawAPI;
  const appState = api.getAppState();
  const { scrollX, scrollY, zoom } = appState;

  const toVp = (x, y) => ({
    x: (x + scrollX) * zoom.value,
    y: (y + scrollY) * zoom.value
  });

  let html = "";

  // 1. Draw Origins
  config.origins.forEach(o => {
    const p0 = toVp(o.x, o.y);
    const cos = Math.cos(o.angle), sin = Math.sin(o.angle);
    const pX = toVp(o.x + o.length * cos, o.y + o.length * sin);
    const pY = toVp(o.x - o.length * sin, o.y + o.length * cos);

    const isActive = o.id === config.activeOriginId;
    const color = isActive ? "#e03131" : "#999999";
    const opacity = isActive ? "1.0" : "0.2";

    html += `
      <g opacity="${opacity}" stroke="${color}" stroke-width="2">
        <line x1="${p0.x}" y1="${p0.y}" x2="${pX.x}" y2="${pX.y}" marker-end="url(#arrowhead)" />
        <line x1="${p0.x}" y1="${p0.y}" x2="${pY.x}" y2="${pY.y}" marker-end="url(#arrowhead)" />
        <text x="${pX.x + 5}" y="${pX.y + 5}" fill="${color}" stroke="none" style="font: bold 12px sans-serif">X (${o.id.toUpperCase()})</text>
        <text x="${pY.x + 5}" y="${pY.y + 5}" fill="${color}" stroke="none" style="font: bold 12px sans-serif">Y</text>
      </g>`;
  });

  // 2. Snap Points & Construction Lines
  if (snapP) {
    html += `<circle cx="${snapP.vpX}" cy="${snapP.vpY}" r="6" fill="none" stroke="#e03131" stroke-width="2" />`;
  }
  if (startP && currP) {
    const v1 = toVp(startP.x, startP.y);
    const v2 = toVp(currP.x, currP.y);
    html += `<line x1="${v1.x}" y1="${v1.y}" x2="${v2.x}" y2="${v2.y}" stroke="#e03131" stroke-width="1" stroke-dasharray="4" />`;
  }

  contentGroup.innerHTML = html;
};

const getSnapPoint = (pointer) => {
  const api = view.excalidrawAPI;
  const appState = api.getAppState();
  const zoom = appState.zoom.value;
  const threshold = 20 / zoom;
  const sceneEls = api.getSceneElements();
  let best = null, minD = threshold;

  for (const el of sceneEls) {
    if (el.isDeleted || el.type === "selection" || (getOrigin().visualIds || []).includes(el.id)) continue;

    const points = [];
    const cx = el.x + el.width / 2;
    const cy = el.y + el.height / 2;

    const rotatePoint = (px, py) => {
      if (el.angle === 0) return { x: px, y: py };
      const cos = Math.cos(el.angle), sin = Math.sin(el.angle);
      const dx = px - cx, dy = py - cy;
      return {
        x: cx + (dx * cos - dy * sin),
        y: cy + (dx * sin + dy * cos)
      };
    };

    if (["line", "arrow"].includes(el.type)) {
      // For lines: snap to vertices AND midpoints of segments
      const worldPoints = el.points.map(([px, py]) => rotatePoint(el.x + px, el.y + py));

      worldPoints.forEach((p, i) => {
        points.push(p); // Vertex
        if (i < worldPoints.length - 1) { // Midpoint of segment
          const next = worldPoints[i + 1];
          points.push({ x: (p.x + next.x) / 2, y: (p.y + next.y) / 2 });
        }
      });
    } else {
      // For shapes: snap to corners, edge midpoints, AND the center
      const corners = [
        {x: el.x, y: el.y},                                  // Top-Left
        {x: el.x + el.width, y: el.y},                       // Top-Right
        {x: el.x, y: el.y + el.height},                      // Bottom-Left
        {x: el.x + el.width, y: el.y + el.height},           // Bottom-Right
        {x: cx, y: cy},                                      // Center
        {x: cx, y: el.y},                                    // Top-Middle
        {x: cx, y: el.y + el.height},                        // Bottom-Middle
        {x: el.x, y: cy},                                    // Left-Middle
        {x: el.x + el.width, y: cy}                          // Right-Middle
      ];
      points.push(...corners.map(p => rotatePoint(p.x, p.y)));
    }

    for (const p of points) {
      const d = Math.hypot(pointer.x - p.x, pointer.y - p.y);
      if (d < minD) { minD = d; best = p; }
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
const buildSection = (title) => `<div style="font-weight:600; opacity:0.8; margin-top:4px; border-bottom:1px solid var(--divider-color); padding-bottom:2px;">${title}</div>`;
const buildRow = (label1, id1, label2, id2, unitClass) => `
<div class="row-${unitClass}" style="display:flex; justify-content:space-between; gap:6px; align-items:center;">
  <div style="display:flex; justify-content:space-between; align-items:center; flex:1;">
    <span style="opacity:0.8">${label1}</span>
    <input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="${id1}" class="geo-input" style="width:75px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;">
  </div>
  <div style="display:flex; justify-content:space-between; align-items:center; flex:1;">
    <span style="opacity:0.8">${label2}</span>
    <input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="${id2}" class="geo-input" style="width:75px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;">
  </div>
</div>`;

const panel = document.createElement("div");
panel.id = panelId;
panel.style.cssText = `position:absolute; top:${config.panelPos.top}; right:${config.panelPos.right}; width:260px; background:var(--background-secondary); border:1px solid var(--divider-color); box-shadow:0 4px 12px rgba(0,0,0,0.15); border-radius:8px; padding:10px; z-index:100; font-size:11px; display:flex; flex-direction:column; gap:6px; max-height: 85vh; overflow-y:auto; font-family: var(--font-ui); color: var(--text-normal);`;

panel.innerHTML = `
<div style="display:flex; justify-content:space-between; align-items:center; font-weight:bold; margin-bottom:4px;">
  <span>📏 Geometry Pro</span>
  <div style="display:flex; gap:8px; align-items:center;">
    <div id="btn-drag" style="cursor:grab; padding:2px 4px; font-size: 14px; color: var(--text-muted);">⠿</div>
    <div id="btn-close" style="cursor:pointer; padding:2px 6px;">✕</div>
  </div>
</div>
<div style="display:flex; align-items:center; justify-content: space-between; gap:4px; margin-bottom:4px; border-bottom: 1px solid var(--divider-color); padding-bottom: 6px;">
  <label style="display:flex; align-items:center; gap:2px;"><input type="checkbox" id="chk-center" ${config.useCenter ? "checked" : ""}> Center</label>
  <label style="display:flex; align-items:center; gap:2px;"><input type="checkbox" id="chk-px" ${config.showPx ? "checked" : ""}> px</label>
  <label style="display:flex; align-items:center; gap:2px;"><input type="checkbox" id="chk-mm" ${config.showMm ? "checked" : ""}> mm</label>
  <label style="display:flex; align-items:center; gap:2px;"><input type="checkbox" id="chk-m" ${config.showM ? "checked" : ""}> m</label>
</div>
${buildSection("Element")}
<div style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">Local Rotation °</span><input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="el_angle" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>
${buildRow("X px", "x_px", "Y px", "y_px", "px")}
${buildRow("W px", "w_px", "H px", "h_px", "px")}
${buildRow("X mm", "x_mm", "Y mm", "y_mm", "mm")}
${buildRow("W mm", "w_mm", "H mm", "h_mm", "mm")}
${buildRow("X m", "x_m", "Y m", "y_m", "m")}
${buildRow("W m", "w_m", "H m", "h_m", "m")}
<div id="section-line" style="display:none;">
  ${buildSection("Line / Arrow")}
  <div style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">Vector Rot °</span><input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="line_angle" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>
  <div class="row-px" style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">Length px</span><input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="line_len_px" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>
  <div class="row-mm" style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">Length mm</span><input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="line_len_mm" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>
  <div class="row-m" style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">Length m</span><input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="line_len_m" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>
</div>
${buildSection("Coordinate System")}
<div style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">Scale 1:</span><input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="scale" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>
<div style="display:flex; justify-content:space-between; align-items:center;"><span style="opacity:0.8">Rotation °</span><input type="number" inputmode="decimal" enterkeyhint="done" step="any" data-id="o_angle" class="geo-input" style="width:90px; text-align:right; padding:2px 4px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px;"></div>
${buildRow("X0 px", "ox0_px", "Y0 px", "oy0_px", "px")}
${buildRow("X1 px", "ox1_px", "Y1 px", "oy1_px", "px")}
${buildRow("X0 mm", "ox0_mm", "Y0 mm", "oy0_mm", "mm")}
${buildRow("X1 mm", "ox1_mm", "Y1 mm", "oy1_mm", "mm")}
${buildRow("X0 m", "ox0_m", "Y0 m", "oy0_m", "m")}
${buildRow("X1 m", "ox1_m", "Y1 m", "oy1_m", "m")}
<div style="display:flex; gap:5px; margin-top:8px; width:100%;">
  <button id="btn-add" style="flex:1; height:24px;">New</button>
  <button id="btn-del" style="flex:1; height:24px;">Delete</button>
</div>
<div style="display:flex; gap:5px; margin-top:4px; width:100%;">
  <select id="origin-select" style="flex:1; height:24px; background:var(--background-primary); border:1px solid var(--divider-color); color:var(--text-normal); border-radius:4px; font-size:13px;"></select>
</div>
`;
view.contentEl.appendChild(panel);

const updateVisibility = () => {
  panel.querySelectorAll(".row-px").forEach(r => r.style.display = config.showPx ? "flex" : "none");
  panel.querySelectorAll(".row-mm").forEach(r => r.style.display = config.showMm ? "flex" : "none");
  panel.querySelectorAll(".row-m").forEach(r => r.style.display = config.showM ? "flex" : "none");
};
updateVisibility();

// --- Logic ---
const updateOriginVisuals = async (o) => {
  drawOverlay(state.snapPoint, state.tempStart, null);
};

const getLineData = (el) => {
  if (!el || !["line", "arrow"].includes(el.type) || el.points.length !== 2) return null;
  const o = getOrigin();

  // 1. Get Global P1 and P2
  const p1 = { x: el.x, y: el.y };
  const p2rel = el.points[1];
  const cos = Math.cos(el.angle), sin = Math.sin(el.angle);
  const p2 = {
    x: p1.x + (p2rel[0] * cos - p2rel[1] * sin),
    y: p1.y + (p2rel[0] * sin + p2rel[1] * cos)
  };

  // 2. Transform both points to Local Space
  const l1 = toLocal(p1);
  const l2 = toLocal(p2);

  // 3. Dimensions are the delta between local coordinates
  const localW = l2.x - l1.x;
  const localH = l2.y - l1.y;
  const globalVecAngle = Math.atan2(p2.y - p1.y, p2.x - p1.x);

  return {
    length: Math.hypot(localW, localH),
    localAngle: globalVecAngle - o.angle,
    localW: Math.abs(localW),
    localH: Math.abs(localH),
    rawLocalW: localW, // Keep sign for resizing logic
    rawLocalH: localH
  };
};

const applyInputToScene = async (id, val) => {
  if (isNaN(val)) return;
  const o = getOrigin();

  // 1. Origin/Scale handling (Stays same)
  if (id === "scale") { config.scale = val; saveSettings(config); return; }
  if (id.startsWith("o")) {
    let p1 = { x: o.x + o.length * Math.cos(o.angle), y: o.y + o.length * Math.sin(o.angle) };
    if (id === "ox0_px") o.x = val; if (id === "oy0_px") o.y = val;
    if (id === "ox0_mm") o.x = toPx(val); if (id === "oy0_mm") o.y = toPx(val);
    if (id === "ox0_m") o.x = fromMetersToPx(val); if (id === "oy0_m") o.y = fromMetersToPx(val);
    if (id === "ox1_px") p1.x = val; if (id === "oy1_px") p1.y = val;
    if (id === "ox1_mm") p1.x = toPx(val); if (id === "oy1_mm") p1.y = toPx(val);
    if (id === "ox1_m") p1.x = fromMetersToPx(val); if (id === "oy1_m") p1.y = fromMetersToPx(val);
    o.angle = (id === "o_angle") ? toRad(val) : Math.atan2(p1.y - o.y, p1.x - o.x);
    o.length = Math.hypot(p1.x - o.x, p1.y - o.y) || 100;
    saveSettings(config); await updateOriginVisuals(o); return;
  }

  const el = ea.getViewSelectedElement();
  if (!el) return;
  ea.clear(); await ea.copyViewElementsToEAforEditing([el]);
  const activeEl = ea.getElements()[0];
  const isLine = ["line", "arrow"].includes(activeEl.type) && activeEl.points.length === 2;

  // CAPTURE STATE
  const globalAnchorBefore = getCurrentAnchor(activeEl);
  const currentLocal = toLocal(globalAnchorBefore, activeEl.angle);
  let targetLocal = { ...currentLocal };
  const lData = getLineData(activeEl);

  const syncToGlobalAnchor = (element, targetG) => {
    const currentG = getCurrentAnchor(element);
    element.x += (targetG.x - currentG.x);
    element.y += (targetG.y - currentG.y);
  };

  // 2. Update Local State based on Input
  if (id === "el_angle") targetLocal.angle = toRad(val);
  else if (id.startsWith("x_")) targetLocal.x = (id.includes("_px") ? val : id.includes("_mm") ? toPx(val) : fromMetersToPx(val));
  else if (id.startsWith("y_")) targetLocal.y = (id.includes("_px") ? val : id.includes("_mm") ? toPx(val) : fromMetersToPx(val));

  // 3. Size Handling
  if (id.startsWith("w_") || id.startsWith("h_")) {
    let pxVal = (id.includes("_px") ? val : id.includes("_mm") ? toPx(val) : fromMetersToPx(val));

    if (isLine) {
      let lDX = (id.startsWith("w")) ? pxVal * Math.sign(lData.rawLocalW || 1) : lData.rawLocalW;
      let lDY = (id.startsWith("h")) ? pxVal * Math.sign(lData.rawLocalH || 1) : lData.rawLocalH;
      const cosO = Math.cos(o.angle), sinO = Math.sin(o.angle);
      const gDX = lDX * cosO - lDY * sinO;
      const gDY = lDX * sinO + lDY * cosO;
      const tLen = Math.hypot(gDX, gDY);
      const tRot = Math.atan2(gDY, gDX) - o.angle;
      const targetGlobalAngle = tRot + o.angle;
      const internalAngle = targetGlobalAngle - activeEl.angle;
      const nx2 = tLen * Math.cos(internalAngle), ny2 = tLen * Math.sin(internalAngle);
      activeEl.points = [[0, 0], [nx2, ny2]];
      activeEl.width = Math.max(0.01, Math.abs(nx2));
      activeEl.height = Math.max(0.01, Math.abs(ny2));
    } else {
      if (id.startsWith("w")) activeEl.width = pxVal;
      else activeEl.height = pxVal;
    }
  }

  // 4. Line Angle/Length Handling
  if (id.startsWith("line_") && isLine) {
    let tLen = lData.length, tRot = lData.localAngle;
    if (id === "line_len_px") tLen = val;
    if (id === "line_len_mm") tLen = toPx(val);
    if (id === "line_len_m") tLen = fromMetersToPx(val);
    if (id === "line_angle") tRot = toRad(val);
    const tGAngle = tRot + o.angle;
    const iAng = tGAngle - activeEl.angle;
    const nx2 = tLen * Math.cos(iAng), ny2 = tLen * Math.sin(iAng);
    activeEl.points = [[0, 0], [nx2, ny2]];
    activeEl.width = Math.max(0.01, Math.abs(nx2));
    activeEl.height = Math.max(0.01, Math.abs(ny2));
  }

  // 5. FINAL SYNC
  activeEl.angle = targetLocal.angle + o.angle;
  const finalGlobalTarget = toGlobal(targetLocal, targetLocal.angle);
  syncToGlobalAnchor(activeEl, finalGlobalTarget);

  await ea.addElementsToView(false, false);
};

const updateUI = (force = false) => {
  if (state.mode !== "idle" && !force) return;
  const o = getOrigin();

  const setV = (id, v, p = 2) => {
    if (state.editingField === id && !force) return;
    const i = panel.querySelector(`[data-id="${id}"]`);
    if (!i) return;

    // Logic: If origin is default (0,0,0), leave origin fields empty
    const isOriginField = id.startsWith("o");
    const isDefaultOrigin = o.x === 0 && o.y === 0 && o.angle === 0;

    if (isOriginField && isDefaultOrigin && id !== "scale") {
      i.value = "";
    } else {
      i.value = (v === null || v === undefined) ? "" : cleanValue(v, p);
    }
  };

  // Coordinate System / Origin Section
  setV("o_angle", formatAngle(o.angle));
  setV("scale", config.scale, 0);

  const p1 = { x: o.x + o.length * Math.cos(o.angle), y: o.y + o.length * Math.sin(o.angle) };

  // Origin Points in all units
  setV("ox0_px", o.x); setV("oy0_px", o.y);
  setV("ox1_px", p1.x); setV("oy1_px", p1.y);
  setV("ox0_mm", toMm(o.x)); setV("oy0_mm", toMm(o.y));
  setV("ox1_mm", toMm(p1.x)); setV("oy1_mm", toMm(p1.y));
  setV("ox0_m", toMeters(o.x), 3); setV("oy0_m", toMeters(o.y), 3);
  setV("ox1_m", toMeters(p1.x), 3); setV("oy1_m", toMeters(p1.y), 3);

  const el = ea.getViewSelectedElement();
  const lineSection = panel.querySelector("#section-line");
  const lData = getLineData(el);

  // Line specific fields
  if (lData) {
    lineSection.style.display = "block";
    setV("line_angle", formatAngle(lData.localAngle));
    setV("line_len_px", lData.length);
    setV("line_len_mm", toMm(lData.length));
    setV("line_len_m", toMeters(lData.length), 3);
  } else {
    lineSection.style.display = "none";
  }

  if (!el) {
    panel.querySelectorAll('input:not([data-id^="o"]):not([data-id="scale"])').forEach(i => {
      if (state.editingField !== i.dataset.id) i.value = "";
    });
    return;
  }

  // Selected Element Positioning
  const globalAnchor = getCurrentAnchor(el);
  const local = toLocal(globalAnchor, el.angle);
  setV("el_angle", formatAngle(local.angle));

  const displayW = lData ? lData.localW : el.width;
  const displayH = lData ? lData.localH : el.height;

  setV("x_px", local.x); setV("y_px", local.y);
  setV("w_px", displayW); setV("h_px", displayH);
  setV("x_mm", toMm(local.x)); setV("y_mm", toMm(local.y));
  setV("w_mm", toMm(displayW)); setV("h_mm", toMm(displayH));
  setV("x_m", toMeters(local.x), 3); setV("y_m", toMeters(local.y), 3);
  setV("w_m", toMeters(displayW), 3); setV("h_m", toMeters(displayH), 3);
};

const startOriginCreation = () => {
  state.capturedSelection = ea.getViewSelectedElements();
  state.mode = "placing_start"; state.tempStart = null;
  panel.querySelector("#btn-add").innerText = "Click Start Point";
  const excalContainer = view.contentEl.querySelector(".excalidraw");
  if(excalContainer) excalContainer.style.cursor = "crosshair";

  const setV = (id, v, p = 2) => {
    const i = panel.querySelector(`[data-id="${id}"]`);
    if (i) i.value = (v === null) ? "" : cleanValue(v, p);
  };

  const onMove = (e) => {
    let pos = ea.getViewLastPointerPosition(); if (!pos) return;
    state.snapPoint = (e.metaKey || e.ctrlKey) ? getSnapPoint(pos) : null;
    let target = state.snapPoint || pos;

    if (state.mode === "placing_start") {
      if(config.showPx) { setV("ox0_px", target.x); setV("oy0_px", target.y); }
      if(config.showMm) { setV("ox0_mm", toMm(target.x)); setV("oy0_mm", toMm(target.y)); }
      if(config.showM) { setV("ox0_m", toMeters(target.x), 3); setV("oy0_m", toMeters(target.y), 3); }
      ["ox1_px", "oy1_px", "ox1_mm", "oy1_mm", "ox1_m", "oy1_m", "o_angle"].forEach(id => setV(id, null));
    } else if (state.mode === "placing_end" && state.tempStart) {
      let ft = target;
      if (e.shiftKey) {
        const dx = target.x - state.tempStart.x, dy = target.y - state.tempStart.y;
        const dist = Math.hypot(dx, dy), sa = Math.round(toDeg(Math.atan2(dy, dx)) / 15) * 15;
        ft = { x: state.tempStart.x + dist * Math.cos(toRad(sa)), y: state.tempStart.y + dist * Math.sin(toRad(sa)) };
      }
      if(config.showPx) { setV("ox1_px", ft.x); setV("oy1_px", ft.y); }
      if(config.showMm) { setV("ox1_mm", toMm(ft.x)); setV("oy1_mm", toMm(ft.y)); }
      if(config.showM) { setV("ox1_m", toMeters(ft.x), 3); setV("oy1_m", toMeters(ft.y), 3); }
      setV("o_angle", formatAngle(Math.atan2(ft.y - state.tempStart.y, ft.x - state.tempStart.x)));
      target = ft;
    }
    drawOverlay(state.snapPoint, state.tempStart, target);
  };

  const onEsc = (e) => { if (e.key === "Escape") stop(); };
  const stop = () => {
    state.mode = "idle"; state.tempStart = null; drawOverlay(null, null, null);
    panel.querySelector("#btn-add").innerText = "New";
    if(excalContainer) excalContainer.style.cursor = "";
    window.removeEventListener("mousemove", onMove); window.removeEventListener("mousedown", onClick, true); window.removeEventListener("keydown", onEsc, true);
    updateUI(true);
  };

  const onClick = async (e) => {
    if (panel.contains(e.target)) return;
    e.stopPropagation(); e.preventDefault();
    let pos = ea.getViewLastPointerPosition();
    let target = state.snapPoint || pos;
    if (state.mode === "placing_start") { state.tempStart = target; state.mode = "placing_end"; panel.querySelector("#btn-add").innerText = "Click X End (Shift=15°)"; }
    else if (state.mode === "placing_end") {
        if (e.shiftKey) {
            const dx = target.x - state.tempStart.x, dy = target.y - state.tempStart.y;
            const dist = Math.hypot(dx, dy), sa = Math.round(toDeg(Math.atan2(dy, dx)) / 15) * 15;
            target = { x: state.tempStart.x + dist * Math.cos(toRad(sa)), y: state.tempStart.y + dist * Math.sin(toRad(sa)) };
        }
        const dx = target.x - state.tempStart.x, dy = target.y - state.tempStart.y;
        const id = Math.random().toString(36).substring(2, 4);
        const newOrigin = { id, x: state.tempStart.x, y: state.tempStart.y, angle: Math.atan2(dy, dx), length: Math.hypot(dx, dy) || 100, visualIds: [] };
        config.origins.push(newOrigin); config.activeOriginId = id;
        await updateOriginVisuals(newOrigin); saveSettings(config); refreshDropdown(); updateUI(true);
        if (state.capturedSelection?.length > 0) ea.selectElementsInView(state.capturedSelection);
        stop();
    }
  };
  window.addEventListener("mousemove", onMove); window.addEventListener("mousedown", onClick, true); window.addEventListener("keydown", onEsc, true);
};

const refreshDropdown = () => {
  const select = panel.querySelector("#origin-select");
  if (select && Array.isArray(config.origins)) {
    select.innerHTML = config.origins.map(o => `<option value="${o.id}" ${o.id === config.activeOriginId ? "selected" : ""}>${o.id.toUpperCase()}</option>`).join("");
  }
};

panel.addEventListener("change", async (e) => {
  if (e.target.id === "chk-center") config.useCenter = e.target.checked;
  if (e.target.id === "chk-px") config.showPx = e.target.checked;
  if (e.target.id === "chk-mm") config.showMm = e.target.checked;
  if (e.target.id === "chk-m") config.showM = e.target.checked;
  if (e.target.id === "origin-select") config.activeOriginId = e.target.value;
  if (e.target.id.startsWith("chk") || e.target.id === "origin-select") {
    saveSettings(config);
    updateVisibility();
    updateUI(true);
    drawOverlay(null, null, null);
    return;
  }

  // 2. Handle the numeric inputs (replaces focusout)
  if (e.target.classList.contains("geo-input")) {
    await handleInputCommit(e.target);
    e.target.blur(); // Dismisses keyboard on iPad after Enter/Change
  }
});

// Helper to apply and refresh
const handleInputCommit = async (inputEl) => {
  const val = parseFloat(inputEl.value);
  const id = inputEl.dataset.id;

  // Only apply if the value actually changed to avoid redundant file writes
  if (inputEl.value !== state.originalValue) {
    await applyInputToScene(id, val);
    updateUI(true);
  }
  state.editingField = null;
};

panel.addEventListener("focusin", (e) => {
  if (e.target.classList.contains("geo-input")) {
    e.target.select();
    state.editingField = e.target.dataset.id;
    state.originalValue = e.target.value;
  }
});

/*
panel.addEventListener("focusout", async (e) => {
  if (e.target.classList.contains("geo-input")) {
    await handleInputCommit(e.target);
  }
});
*/

panel.addEventListener("keydown", async (e) => {
  if (!e.target.classList.contains("geo-input")) return;

  if (e.key === "Escape") {
    e.target.value = state.originalValue;
    state.editingField = null; // Prevent focusout from saving the reverted value
    e.target.select();
    return;
  }

  if (e.key === "Enter" || e.key === "Tab") {
    // On Enter, we apply but keep focus so user can see the change
    await handleInputCommit(e.target);
    if (e.key === "Enter") {
      e.preventDefault();
      e.target.select();
    }
  }
});

panel.addEventListener("click", async (e) => {
  if (e.target.id === "btn-close") {
    if (window.geoProListener) {
      window.geoProListener(); // Unsubscribes from Excalidraw
      window.geoProListener = null;
    }
    panel.remove();
    document.querySelector("#geo-snap-overlay")?.remove();
  }
  if (e.target.id === "btn-add") startOriginCreation();
  if (e.target.id === "btn-del") {
    const active = getOrigin(); if (active.persistent) return;
    config.origins = config.origins.filter(o => o.id !== active.id); config.activeOriginId = "00";
    saveSettings(config); refreshDropdown(); updateUI(true); drawOverlay(null, null, null);
  }
});

if (window.geoProListener) window.geoProListener(); // Cleanup old one
let lastVersionNonce = 0;

// --- Throttled Listener ---
let lastZoom = 0;
let lastScrollX = 0;
let lastScrollY = 0;
let lastSelectionString = "";
window.geoProListener = view.excalidrawAPI.onChange((elements, appState) => {
  if (updateRequested) return;

  // Check if anything "spatial" actually changed
  const selectionIds = appState.selectedElementIds ? Object.keys(appState.selectedElementIds) : [];
  const selectionIdsString = selectionIds.join(",");
  const hasMoved = appState.scrollX !== lastScrollX || appState.scrollY !== lastScrollY || appState.zoom.value !== lastZoom;
  const selectionChanged = selectionIdsString !== lastSelectionString;

  // Check if the selected element has changed (only check the 1st element)
  let elementChanged = false;
  if (selectionIds.length > 0) {
    const activeEl = elements.find(el => el.id === selectionIds[0]);
    if (activeEl && activeEl.versionNonce !== lastVersionNonce) {
      elementChanged = true;
      lastVersionNonce = activeEl.versionNonce;
    }
  }


  if (hasMoved || selectionChanged || elementChanged || state.mode !== "idle") {
    updateRequested = true;
    requestAnimationFrame(() => {
      drawOverlay(state.snapPoint, state.tempStart, null);
      updateUI(); // Updates the input fields

      // Store current state to compare next time
      lastScrollX = appState.scrollX;
      lastScrollY = appState.scrollY;
      lastZoom = appState.zoom.value;
      lastSelectionString = selectionIdsString;

      updateRequested = false;
    });
  }
});

// Clean up the listener when the panel is closed
const originalClose = panel.querySelector("#btn-close").onclick;
panel.querySelector("#btn-close").addEventListener("click", () => {
  if (window.geoProListener) {
    window.geoProListener(); // Unsubscribe
    window.geoProListener = null;
  }
});

const initDraggable = (p) => {
  let isDragging = false;
  let offset = { x: 0, y: 0 };

  // --- MOUSE EVENTS (macOS/Desktop) ---
  p.addEventListener("mousedown", (e) => {
    if (["INPUT", "BUTTON", "SELECT"].includes(e.target.tagName)) return;
    isDragging = true;
    offset = { x: e.clientX - p.offsetLeft, y: e.clientY - p.offsetTop };
    p.style.cursor = "grabbing";
  });

  window.addEventListener("mousemove", (e) => {
    if (!isDragging) return;
    e.stopPropagation();
    p.style.left = `${e.clientX - offset.x}px`;
    p.style.top = `${e.clientY - offset.y}px`;
    p.style.right = "auto";
  });

  window.addEventListener("mouseup", () => {
    isDragging = false;
    p.style.cursor = "default";
  });

  // --- TOUCH EVENTS (iPad/Mobile) ---
  p.addEventListener("touchstart", (e) => {
    if (["INPUT", "BUTTON", "SELECT"].includes(e.target.tagName) || e.target.id === "btn-close") return;
    e.stopPropagation();

    // We use e.touches[0] to get the first finger
    const touch = e.touches[0];
    isDragging = true;
    offset = { x: touch.clientX - p.offsetLeft, y: touch.clientY - p.offsetTop };

    // Prevents iPad from scrolling the page while dragging the panel
    if (e.cancelable) e.preventDefault();
  }, { passive: false });

  window.addEventListener("touchmove", (e) => {
    if (!isDragging) return;
    const touch = e.touches[0];

    p.style.left = `${touch.clientX - offset.x}px`;
    p.style.top = `${touch.clientY - offset.y}px`;
    p.style.right = "auto";

    if (e.cancelable) e.preventDefault();
  }, { passive: false });

  window.addEventListener("touchend", () => {
    isDragging = false;
  });
};

refreshDropdown();
updateVisibility();
updateUI(true);
drawOverlay(null, null, null);
