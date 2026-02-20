/*
 * ╔══════════════════════════════════════════════╗
 * ║         LAYER MANAGER — Excalidraw Script    ║
 * ║  Floating, draggable panel — stays open      ║
 * ║  Inspired by Excalidraw PR #10447            ║
 * ╚══════════════════════════════════════════════╝
 *
 * INSTALL: Drop into your Excalidraw scripts folder.
 * USE:     Run from command palette or assign a hotkey.
 *          Run again to CLOSE the panel (toggle).
 *
 * FEATURES:
 *  • Draggable floating panel (position saved in frontmatter)
 *  • Create / rename (dblclick name) / delete layers
 *  • Toggle visibility  👁 / 🙈  (opacity trick, non-destructive)
 *  • Toggle lock  🔒 / 🔓  (deselects elements on lock)
 *  • Set active layer — click layer name to activate
 *  • Reorder layers  ↑ ↓
 *  • Assign selected canvas elements to the active layer
 *  • Select all elements on the active layer
 *  • Merge all layers into the active one
 *  • Element count badge per layer
 *  • Full layer state persisted in file frontmatter
 */

// ─── Bootstrap ────────────────────────────────────────────────────────────────
const PANEL_ID   = "layer-manager-panel";
const FM_KEY     = "layer-manager-state";
const FM_POS_KEY = "layer-manager-pos";

// Resolve the active Excalidraw view
const activeView = (() => {
  const ExcalidrawView = customElements.get("excalidraw-view")?.constructor;
  if (ExcalidrawView) {
    const v = app.workspace.getActiveViewOfType(ExcalidrawView);
    if (v?.excalidrawAPI) return v;
  }
  for (const leaf of app.workspace.getLeavesOfType("excalidraw")) {
    if (leaf.view?.excalidrawAPI) return leaf.view;
  }
  return null;
})();

if (!activeView?.excalidrawAPI) {
  new Notice("⚠️  Open an Excalidraw drawing first.");
  return;
}

// Toggle: running again closes the panel
const existing = activeView.contentEl.querySelector(`#${PANEL_ID}`);
if (existing) {
  existing.querySelector("button[title='Close panel']")?.click() ?? existing.remove();
  return;
}

const api = activeView.excalidrawAPI;

// ─── Frontmatter helpers ───────────────────────────────────────────────────────
const getFM = () => {
  const cache = app.metadataCache.getFileCache(activeView.file);
  return cache?.frontmatter ?? {};
};

const saveState = (layers, activeLayerId) =>
  app.fileManager.processFrontMatter(activeView.file, fm => {
    fm[FM_KEY] = { layers, activeLayerId };
  });

const getPos = () => {
  const pos = getFM()[FM_POS_KEY];
  return pos ?? { x: 24, y: 80 };
};

const savePos = (x, y) =>
  app.fileManager.processFrontMatter(activeView.file, fm => {
    fm[FM_POS_KEY] = { x, y };
  });

// ─── Utilities ─────────────────────────────────────────────────────────────────
const uid = () => "lyr_" + Math.random().toString(36).slice(2, 9);

const COLORS = [
  "#4f86f7","#f74f4f","#3ecf8e","#f5c518","#b04ff7",
  "#f74fb8","#4ff0f7","#f7824f","#9af74f","#a78bfa"
];

// ─── Layer state ───────────────────────────────────────────────────────────────
let layers, activeLayerId;
const stored = getFM()[FM_KEY];

if (stored?.layers?.length) {
  layers        = stored.layers;
  // Migrate old saves that have no isDefault flag — mark the first layer
  if (!layers.some(l => l.isDefault)) layers[0].isDefault = true;
  activeLayerId = stored.activeLayerId ?? layers[0].id;
} else {
  const defId   = "default";
  layers        = [{ id: defId, name: "Layer 1", visible: true, locked: false, color: COLORS[0], isDefault: true }];
  activeLayerId = defId;
}

// The "default" layer is identified by the stable isDefault flag, NOT by position.
// This means reordering layers never changes which layer unassigned elements belong to.
const getDefaultLayerId = () => (layers.find(l => l.isDefault) ?? layers[0])?.id ?? "default";

// ─── Scene helpers ─────────────────────────────────────────────────────────────

// Unassigned elements (no customData.layerId) belong to the default layer.
// We never write customData onto elements unless the user explicitly assigns them.
const getDisplayLayerId = el => el.customData?.layerId ?? getDefaultLayerId();

const liveElements = () => api.getSceneElements(); // non-deleted, visible to user

const elementCounts = () => {
  const counts = {};
  for (const el of liveElements()) {
    const lid = getDisplayLayerId(el);
    counts[lid] = (counts[lid] || 0) + 1;
  }
  return counts;
};

// ─── Opacity side-map ─────────────────────────────────────────────────────────
// Stores original opacity before we hide an element.
// Lives in script memory only — never written to element customData.
const origOpacityMap = new Map();

// ─── Core scene mutator ───────────────────────────────────────────────────────
// Uses the EA workbench: copyViewElementsToEAforEditing loads all live elements
// into ea.elementsDict as spread copies. We mutate those copies (returning new
// objects when changed), then addElementsToView commits them back.
// Passing ALL live elements (not a subset) is required — Excalidraw needs the
// full picture to reconcile correctly.
//
// isMutating: set during our own scene writes so the onChange handler doesn't
// re-run patchElements and undo our changes before they settle.
let isMutating = false;

const patchElements = (mutateFn) => {
  const els = liveElements();
  ea.copyViewElementsToEAforEditing(els);

  let anyChanged = false;
  for (const id of Object.keys(ea.elementsDict)) {
    const el = ea.elementsDict[id];
    const patched = mutateFn(el);
    if (patched !== el) {
      ea.elementsDict[id] = patched;
      anyChanged = true;
    }
  }

  if (anyChanged) {
    isMutating = true;
    ea.addElementsToView(false, false, false);
    setTimeout(() => { isMutating = false; }, 500);
  }
  ea.elementsDict = {};
};

// ─── Visibility ───────────────────────────────────────────────────────────────
const applyVisibility = () => {
  const lm = Object.fromEntries(layers.map(l => [l.id, l]));

  patchElements(el => {
    const effectiveLid = el.customData?.layerId ?? getDefaultLayerId();
    const layer = lm[effectiveLid];
    if (!layer) return el;

    if (!layer.visible) {
      if (!origOpacityMap.has(el.id)) origOpacityMap.set(el.id, el.opacity ?? 100);
      return el.opacity === 0 ? el : { ...el, opacity: 0 };
    } else {
      if (origOpacityMap.has(el.id)) {
        const orig = origOpacityMap.get(el.id);
        origOpacityMap.delete(el.id);
        return el.opacity === orig ? el : { ...el, opacity: orig };
      }
      return el;
    }
  });
};

// ─── Assign selected elements to a layer ──────────────────────────────────────
const assignSelected = (targetId) => {
  const selIds = api.getAppState().selectedElementIds ?? {};
  const count  = Object.keys(selIds).length;
  if (!count) return 0;

  patchElements(el => {
    if (!selIds[el.id]) return el;
    const restoredOpacity = origOpacityMap.get(el.id) ?? el.opacity ?? 100;
    origOpacityMap.delete(el.id);
    return { ...el, opacity: restoredOpacity, customData: { ...el.customData, layerId: targetId } };
  });

  applyVisibility();
  return count;
};

// ─── Select all elements on a layer ───────────────────────────────────────────
const selectLayerElements = (layerId) => {
  const ids = {};
  liveElements().forEach(el => {
    if (getDisplayLayerId(el) === layerId) ids[el.id] = true;
  });
  api.updateScene({ appState: { selectedElementIds: ids }, commitToHistory: false });
  return Object.keys(ids).length;
};

// ─── Merge all layers into one ────────────────────────────────────────────────
const mergeAll = (targetId) => {
  patchElements(el => {
    if ((el.customData?.layerId ?? getDefaultLayerId()) === targetId) return el;
    return { ...el, customData: { ...el.customData, layerId: targetId } };
  });
  layers        = layers.filter(l => l.id === targetId);
  activeLayerId = targetId;
  origOpacityMap.clear();
};

// ─── Delete a layer (elements migrate to fallback) ────────────────────────────
const deleteLayer = (layerId) => {
  const fallback = layers.find(l => l.id !== layerId)?.id ?? getDefaultLayerId();
  patchElements(el => {
    if (getDisplayLayerId(el) !== layerId) return el;
    const restoredOpacity = origOpacityMap.get(el.id) ?? el.opacity ?? 100;
    origOpacityMap.delete(el.id);
    return { ...el, opacity: restoredOpacity, customData: { ...el.customData, layerId: fallback } };
  });
  layers = layers.filter(l => l.id !== layerId);
  if (activeLayerId === layerId) activeLayerId = fallback;
};

const persist = () => saveState(layers, activeLayerId);

// ─── Startup migration ─────────────────────────────────────────────────────────
// Previous version stored origOpacity inside element customData. Strip it and restore.
(() => {
  const sceneElements = liveElements();
  if (!sceneElements.some(el => el.customData?.origOpacity !== undefined)) return;

  const lm = Object.fromEntries(layers.map(l => [l.id, l]));
  ea.copyViewElementsToEAforEditing(sceneElements);
  let anyChanged = false;
  for (const id of Object.keys(ea.elementsDict)) {
    const el = ea.elementsDict[id];
    if (el.customData?.origOpacity === undefined) continue;
    const effectiveLid = el.customData?.layerId ?? getDefaultLayerId();
    const layer = lm[effectiveLid];
    const isVisible = layer ? layer.visible : true;
    const { origOpacity, ...restCustomData } = el.customData;
    ea.elementsDict[id] = { ...el, opacity: isVisible ? (origOpacity ?? 100) : 0, customData: restCustomData };
    anyChanged = true;
  }
  if (anyChanged) ea.addElementsToView(false, false, false);
  ea.elementsDict = {};
})();

// ─── Build Panel DOM ────────────────────────────────────────────────────────────
const pos = getPos();

const panel = activeView.contentEl.createDiv({ attr: { id: PANEL_ID } });
Object.assign(panel.style, {
  position:  "absolute",
  left:      pos.x + "px",
  top:       pos.y + "px",
  width:     "270px",
  minWidth:  "200px",
  maxHeight: "72vh",
  background: "var(--background-primary)",
  border:    "1px solid var(--background-modifier-border)",
  borderRadius: "10px",
  boxShadow: "0 8px 32px rgba(0,0,0,.30)",
  display:   "flex",
  flexDirection: "column",
  zIndex:    "9999",
  fontFamily: "var(--font-interface)",
  overflow:  "hidden",
  resize:    "both",
  userSelect: "none",
});

// ── Title bar ────────────────────────────────────────────────────────────────
const titleBar = panel.createDiv();
Object.assign(titleBar.style, {
  display:        "flex",
  alignItems:     "center",
  justifyContent: "space-between",
  padding:        "8px 10px",
  background:     "var(--background-secondary)",
  borderBottom:   "1px solid var(--background-modifier-border)",
  cursor:         "grab",
  borderRadius:   "10px 10px 0 0",
  flexShrink:     "0",
});

const titleText = titleBar.createEl("span", { text: "⊞  Layers" });
Object.assign(titleText.style, { fontWeight: "700", fontSize: "13px", pointerEvents: "none" });

const tbRight = titleBar.createDiv();
Object.assign(tbRight.style, { display: "flex", gap: "2px", alignItems: "center" });

const mkTbBtn = (label, title, color) => {
  const b = tbRight.createEl("button", { text: label });
  b.title = title;
  Object.assign(b.style, {
    fontSize: "15px", padding: "0 5px", border: "none",
    background: "none", cursor: "pointer",
    color: color ?? "var(--text-muted)",
  });
  return b;
};

const addBtn   = mkTbBtn("＋", "New layer", "var(--interactive-accent)");
const closeBtn = mkTbBtn("✕", "Close panel");

// ── Active layer info strip ──────────────────────────────────────────────────
const infoBar = panel.createDiv();
Object.assign(infoBar.style, {
  padding:      "3px 10px",
  fontSize:     "10px",
  color:        "var(--text-muted)",
  background:   "var(--background-secondary-alt)",
  borderBottom: "1px solid var(--background-modifier-border)",
  flexShrink:   "0",
});

// ── Layer list ───────────────────────────────────────────────────────────────
const listEl = panel.createDiv();
Object.assign(listEl.style, {
  flex:       "1",
  overflowY:  "auto",
  padding:    "4px 0",
  background: "var(--background-primary)",
});

// ── Footer ───────────────────────────────────────────────────────────────────
const footer = panel.createDiv();
Object.assign(footer.style, {
  display:      "flex",
  flexWrap:     "wrap",
  gap:          "4px",
  padding:      "6px 8px",
  borderTop:    "1px solid var(--background-modifier-border)",
  background:   "var(--background-secondary)",
  borderRadius: "0 0 10px 10px",
  flexShrink:   "0",
});

const mkFooterBtn = (label, title) => {
  const b = footer.createEl("button", { text: label });
  b.title = title;
  Object.assign(b.style, {
    flex: "1", minWidth: "60px", fontSize: "11px", padding: "4px 5px",
    borderRadius: "5px", cursor: "pointer",
    background: "var(--background-primary)",
    color: "var(--text-normal)",
    border: "1px solid var(--background-modifier-border)",
  });
  return b;
};

const assignBtn = mkFooterBtn("↳ Assign", "Assign selected elements to the active layer");
const selectBtn = mkFooterBtn("⊡ Select", "Select all elements on the active layer");
const mergeBtn  = mkFooterBtn("⊕ Merge All", "Merge all layers into the active layer");

// ─── Render function ────────────────────────────────────────────────────────────
const render = () => {
  const active = layers.find(l => l.id === activeLayerId);
  infoBar.setText(`Active layer: ${active?.name ?? "—"}`);

  listEl.empty();
  // Clear incremental maps — will be repopulated as rows are built below
  for (const k of Object.keys(rowMap))   delete rowMap[k];
  for (const k of Object.keys(badgeMap)) delete badgeMap[k];
  const counts = elementCounts();

  // Determine which single layer all currently selected elements share (if any).
  // api.getAppState().selectedElementIds can lag — read directly from scene elements.
  // An element is "selected" if it appears in the appState selectedElementIds map.
  const selIds   = api.getAppState()?.selectedElementIds ?? {};
  const selElems = liveElements().filter(el => selIds[el.id]);
  let selLayerId = null;
  if (selElems.length > 0) {
    const firstLid = getDisplayLayerId(selElems[0]);
    if (selElems.every(el => getDisplayLayerId(el) === firstLid)) {
      selLayerId = firstLid;
    }
  }

  // Default layer is always rendered first regardless of its position in the array.
  // For reordering, we work on the nonDefaultLayers sub-array so the default layer
  // never participates in swaps.
  const nonDefaultLayers = layers.filter(l => !l.isDefault);
  const sorted = [
    ...layers.filter(l => l.isDefault),
    ...nonDefaultLayers,
  ];

  sorted.forEach((layer) => {
    const isActive   = layer.id === activeLayerId;
    const isSelected = selLayerId !== null && layer.id === selLayerId;
    const isDefault  = !!layer.isDefault;

    // Position within the non-default sub-array (used for ↑↓)
    const ndIdx   = nonDefaultLayers.indexOf(layer);
    const canUp   = !isDefault && ndIdx > 0;
    const canDown = !isDefault && ndIdx < nonDefaultLayers.length - 1;

    const row = listEl.createDiv();
    Object.assign(row.style, {
      display:      "flex",
      alignItems:   "center",
      gap:          "4px",
      padding:      "5px 8px",
      borderLeft:   `3px solid ${isActive ? layer.color : "transparent"}`,
      borderRight:  `3px solid ${isSelected ? layer.color : "transparent"}`,
      background:   isActive ? "var(--background-secondary)" : "transparent",
      borderRadius: "3px",
      margin:       "1px 4px",
      cursor:       "pointer",
      transition:   "background .1s",
    });
    row.onmouseenter = () => { if (!isActive) row.style.background = "var(--background-secondary-alt)"; };
    row.onmouseleave = () => { if (!isActive) row.style.background = "transparent"; };

    // Color swatch
    const swatch = row.createDiv();
    Object.assign(swatch.style, {
      width: "9px", height: "9px", borderRadius: "50%",
      background: layer.color, flexShrink: "0",
    });

    // Default badge (small pill, not interactive label)
    if (isDefault) {
      const pill = row.createEl("span", { text: "default" });
      Object.assign(pill.style, {
        fontSize: "9px", padding: "0 4px", borderRadius: "4px",
        background: "var(--background-modifier-border)",
        color: "var(--text-faint)", flexShrink: "0",
      });
    }

    // Layer name — single click = activate, double click = rename inline.
    // We use a short timer to distinguish the two so dblclick doesn't also
    // fire the single-click handler.
    const nameEl = row.createEl("span");
    Object.assign(nameEl.style, {
      flex: "1", fontSize: "12px", overflow: "hidden",
      textOverflow: "ellipsis", whiteSpace: "nowrap",
      color:          layer.locked ? "var(--text-faint)" : "var(--text-normal)",
      textDecoration: layer.locked ? "line-through" : "none",
    });
    nameEl.setText(layer.name);
    nameEl.title = "Click to activate · Double-click to rename";

    let clickTimer = null;
    nameEl.onclick = () => {
      if (clickTimer) return; // will be handled by dblclick
      clickTimer = setTimeout(() => {
        clickTimer = null;
        if (layer.id === activeLayerId) return; // already active
        const prevId = activeLayerId;
        activeLayerId = layer.id;
        persist();
        // Update borders in-place — no full re-render needed
        if (rowMap[prevId]) {
          rowMap[prevId].style.borderLeft = "3px solid transparent";
          rowMap[prevId].style.background = "transparent";
        }
        if (rowMap[activeLayerId]) {
          rowMap[activeLayerId].style.borderLeft = `3px solid ${layer.color}`;
          rowMap[activeLayerId].style.background = "var(--background-secondary)";
        }
        infoBar.setText(`Active layer: ${layer.name}`);
      }, 220);
    };

    nameEl.ondblclick = (e) => {
      e.stopPropagation();
      if (clickTimer) { clearTimeout(clickTimer); clickTimer = null; }

      // Enter inline edit mode
      nameEl.contentEditable = "true";
      Object.assign(nameEl.style, {
        background: "var(--background-modifier-form-field)",
        borderRadius: "3px", padding: "0 3px",
        outline: "1px solid var(--interactive-accent)",
      });
      nameEl.focus();
      document.execCommand("selectAll", false, null);

      const done = () => {
        nameEl.contentEditable = "false";
        Object.assign(nameEl.style, { background: "", padding: "", outline: "" });
        const v = nameEl.textContent.trim();
        if (v && v !== layer.name) { layer.name = v; persist(); }
        else { nameEl.setText(layer.name); } // restore if empty or unchanged
        render();
      };
      nameEl.onblur = done;
      nameEl.onkeydown = ev => {
        if (ev.key === "Enter")  { ev.preventDefault(); nameEl.blur(); }
        if (ev.key === "Escape") { nameEl.textContent = layer.name; nameEl.blur(); }
      };
    };

    // Element count badge
    const cnt = counts[layer.id] ?? 0;
    const badge = row.createEl("span", { text: String(cnt) });
    Object.assign(badge.style, {
      fontSize: "10px", minWidth: "18px", textAlign: "center",
      padding: "0 4px", borderRadius: "9px",
      background: "var(--background-modifier-border)",
      color: "var(--text-muted)", flexShrink: "0",
    });
    badge.title = `${cnt} element(s) on this layer`;

    // Register for incremental updates
    rowMap[layer.id]   = row;
    badgeMap[layer.id] = badge;

    // Mini icon buttons helper
    const mkIcon = (text, title, disabled = false) => {
      const b = row.createEl("button", { text });
      b.title = title;
      b.disabled = disabled;
      Object.assign(b.style, {
        fontSize: "11px", padding: "0 3px",
        border: "none", background: "none",
        cursor: disabled ? "default" : "pointer",
        color: disabled ? "var(--text-faint)" : "var(--text-muted)",
        flexShrink: "0",
        opacity: disabled ? "0.3" : "1",
      });
      return b;
    };

    // ↑ ↓ reorder — not shown for default layer (it always stays on top)
    if (!isDefault) {
      const reorder = (fromIdx, toIdx) => {
        // Swap within nonDefaultLayers, then rebuild layers = [default, ...nonDefault]
        [nonDefaultLayers[fromIdx], nonDefaultLayers[toIdx]] = [nonDefaultLayers[toIdx], nonDefaultLayers[fromIdx]];
        layers = [...layers.filter(l => l.isDefault), ...nonDefaultLayers];
        persist(); render();
      };

      const upBtn = mkIcon("↑", "Move layer up", !canUp);
      upBtn.onclick = (e) => { e.stopPropagation(); if (canUp) reorder(ndIdx, ndIdx - 1); };

      const dnBtn = mkIcon("↓", "Move layer down", !canDown);
      dnBtn.onclick = (e) => { e.stopPropagation(); if (canDown) reorder(ndIdx, ndIdx + 1); };
    } else {
      // Placeholder spacer so alignment stays consistent
      const sp = row.createEl("span");
      sp.style.cssText = "width:28px;flex-shrink:0;";
    }

    const visBtn = mkIcon(layer.visible ? "👁" : "🙈", layer.visible ? "Hide layer" : "Show layer");
    visBtn.style.fontSize = "13px";
    visBtn.onclick = (e) => {
      e.stopPropagation();
      layer.visible = !layer.visible;
      // Update icon and tooltip in-place — no full re-render needed
      visBtn.setText(layer.visible ? "👁" : "🙈");
      visBtn.title = layer.visible ? "Hide layer" : "Show layer";
      applyVisibility();
      persist();
    };

    const lockBtn = mkIcon(layer.locked ? "🔒" : "🔓", layer.locked ? "Unlock layer" : "Lock layer");
    lockBtn.onclick = (e) => {
      e.stopPropagation();
      layer.locked = !layer.locked;
      // Update icon and name style in-place — no full re-render needed
      lockBtn.setText(layer.locked ? "🔒" : "🔓");
      lockBtn.title = layer.locked ? "Unlock layer" : "Lock layer";
      nameEl.style.color          = layer.locked ? "var(--text-faint)" : "var(--text-normal)";
      nameEl.style.textDecoration = layer.locked ? "line-through" : "none";
      if (layer.locked) {
        const sel = { ...api.getAppState().selectedElementIds };
        api.getSceneElements().forEach(el => { if (getDisplayLayerId(el) === layer.id) delete sel[el.id]; });
        api.updateScene({ appState: { selectedElementIds: sel }, commitToHistory: false });
      }
      persist();
    };

    const delBtn = mkIcon("🗑", isDefault ? "Cannot delete the default layer" : "Delete layer", isDefault);
    delBtn.style.opacity = isDefault ? "0.2" : "0.45";
    delBtn.onclick = (e) => {
      e.stopPropagation();
      if (isDefault) { new Notice("Cannot delete the default layer."); return; }
      if (layers.length === 1) { new Notice("Cannot delete the last layer."); return; }
      if (!confirm(`Delete "${layer.name}"? Its elements will move to the active layer.`)) return;
      deleteLayer(layer.id);
      persist();
      applyVisibility();
      render();
    };
  });
};

// ─── Footer button actions ─────────────────────────────────────────────────────
closeBtn.onclick = () => panel.remove();

addBtn.onclick = () => {
  const newLayer = {
    id:      uid(),
    name:    `Layer ${layers.length + 1}`,
    visible: true,
    locked:  false,
    color:   COLORS[layers.length % COLORS.length],
  };
  layers.push(newLayer);
  activeLayerId = newLayer.id;
  persist();
  render();
};

assignBtn.onclick = () => {
  const count = assignSelected(activeLayerId);
  const name  = layers.find(l => l.id === activeLayerId)?.name ?? activeLayerId;
  new Notice(count > 0 ? `↳ Assigned ${count} element(s) to "${name}"` : "No elements selected.");
  render();
};

selectBtn.onclick = () => {
  const count = selectLayerElements(activeLayerId);
  new Notice(`⊡ Selected ${count} element(s).`);
};

mergeBtn.onclick = () => {
  if (layers.length <= 1) { new Notice("Only one layer — nothing to merge."); return; }
  if (!confirm("Merge ALL layers into the active layer?")) return;
  mergeAll(activeLayerId);
  persist();
  applyVisibility();
  render();
  new Notice("⊕ All layers merged.");
};

// ─── Drag to reposition ────────────────────────────────────────────────────────
let dragging = false, dragOffX = 0, dragOffY = 0;

const onTitleMouseDown = (e) => {
  if (e.target === closeBtn || e.target === addBtn) return;
  dragging = true;
  const pr = activeView.contentEl.getBoundingClientRect();
  dragOffX = (e.clientX - pr.left) - parseFloat(panel.style.left || "0");
  dragOffY = (e.clientY - pr.top)  - parseFloat(panel.style.top  || "0");
  titleBar.style.cursor = "grabbing";
  e.preventDefault();
};
const onDocMouseMove = (e) => {
  if (!dragging) return;
  const pr = activeView.contentEl.getBoundingClientRect();
  panel.style.left = Math.max(0, Math.min(e.clientX - pr.left - dragOffX, pr.width  - panel.offsetWidth))  + "px";
  panel.style.top  = Math.max(0, Math.min(e.clientY - pr.top  - dragOffY, pr.height - panel.offsetHeight)) + "px";
};
const onDocMouseUp = () => {
  if (!dragging) return;
  dragging = false;
  titleBar.style.cursor = "grab";
  savePos(parseFloat(panel.style.left), parseFloat(panel.style.top));
};

titleBar.addEventListener("mousedown", onTitleMouseDown);
document.addEventListener("mousemove", onDocMouseMove);
document.addEventListener("mouseup",   onDocMouseUp);

// ─── Incremental UI updates ───────────────────────────────────────────────────
// Row elements keyed by layer id — lets us update badges and borders in-place
// without rebuilding the entire list DOM on every scene change.
const rowMap   = {};   // layerId → row div
const badgeMap = {};   // layerId → badge span

// Called only when counts change (element added/deleted)
const updateCounts = () => {
  const counts = elementCounts();
  for (const [lid, badge] of Object.entries(badgeMap)) {
    const cnt = counts[lid] ?? 0;
    badge.setText(String(cnt));
    badge.title = `${cnt} element(s) on this layer`;
  }
};

// Called only when selection changes
const updateSelection = () => {
  const selIds   = api.getAppState()?.selectedElementIds ?? {};
  const selElems = liveElements().filter(el => selIds[el.id]);
  let selLayerId = null;
  if (selElems.length > 0) {
    const firstLid = getDisplayLayerId(selElems[0]);
    if (selElems.every(el => getDisplayLayerId(el) === firstLid)) selLayerId = firstLid;
  }
  for (const [lid, row] of Object.entries(rowMap)) {
    const isActive   = lid === activeLayerId;
    const isSelected = lid === selLayerId;
    const layer      = layers.find(l => l.id === lid);
    if (!layer) continue;
    row.style.borderRight = `3px solid ${isSelected ? layer.color : "transparent"}`;
  }
};

// ─── Scene change: new elements + incremental count update ───────────────────
let knownElementIds = new Set(api.getSceneElements().map(el => el.id));
let sceneChangeTimer = null;

// pendingNewIds: elements seen mid-draw that need stamping once drawing finishes
const pendingNewIds = new Set();

const sceneChangeHandler = () => {
  if (isMutating) {
    knownElementIds = new Set(api.getSceneElements().map(el => el.id));
    return;
  }

  const current    = api.getSceneElements();
  const appState   = api.getAppState();
  const isDrawing  = appState?.activeTool?.type !== "selection" &&
                     appState?.activeTool?.type !== "hand" &&
                     appState?.activeTool?.type != null;
  const prevSize   = knownElementIds.size;

  // Collect genuinely new elements (not yet in knownElementIds, not yet stamped)
  const newEls = current.filter(el => !knownElementIds.has(el.id) && !el.customData?.layerId);
  knownElementIds = new Set(current.map(el => el.id));

  if (newEls.length > 0) {
    for (const el of newEls) pendingNewIds.add(el.id);
  }

  // Only stamp when user is back on selection tool (drawing finished)
  // and there are pending elements to assign
  if (!isDrawing && pendingNewIds.size > 0) {
    // Remove any ids that have since been deleted or already got a layerId
    const currentMap = new Map(current.map(el => [el.id, el]));
    for (const id of [...pendingNewIds]) {
      const el = currentMap.get(id);
      if (!el || el.customData?.layerId) pendingNewIds.delete(id);
    }

    if (pendingNewIds.size > 0) {
      const activeLayer = layers.find(l => l.id === activeLayerId);
      if (activeLayer && activeLayer.visible && !activeLayer.locked) {
        const all = api.getSceneElementsIncludingDeleted();
        const updated = all.map(el =>
          pendingNewIds.has(el.id)
            ? { ...el, customData: { ...el.customData, layerId: activeLayerId } }
            : el
        );
        isMutating = true;
        api.updateScene({ elements: updated, commitToHistory: false });
        setTimeout(() => { isMutating = false; }, 200);
      }
      pendingNewIds.clear();
      updateCounts();
    }
  }

  if (current.length !== prevSize && prevSize > 0) {
    updateCounts();
  }
};

const throttledSceneChange = () => {
  if (sceneChangeTimer) clearTimeout(sceneChangeTimer);
  sceneChangeTimer = setTimeout(() => {
    sceneChangeTimer = null;
    sceneChangeHandler();
  }, 300);
};

// ─── onChange registration ───────────────────────────────────────────────────
// ea.setOnChangeHook may not fire reliably (confirmed by user testing).
// Instead we poll with setInterval — simple, always works, low overhead at 500ms.
// We also listen to pointer/key events for immediate response to user actions.
const excalidrawContainer = activeView.contentEl.querySelector(".excalidraw");

const onPointerUp = () => {
  setTimeout(updateSelection, 150);  // selection state settles after pointer release
  setTimeout(updateCounts,   150);   // catch deletions via click+delete
};
const onKeyUp = (e) => {
  if (e.key === "Delete" || e.key === "Backspace") setTimeout(updateCounts, 150);
};

let pollInterval = setInterval(sceneChangeHandler, 500);

if (excalidrawContainer) {
  excalidrawContainer.addEventListener("pointerup", onPointerUp);
  excalidrawContainer.addEventListener("keyup",     onKeyUp);
}

// ─── Close: restore opacities and remove ALL listeners ───────────────────────
const cleanup = () => {
  clearInterval(pollInterval);
  pollInterval = null;
  if (sceneChangeTimer) { clearTimeout(sceneChangeTimer); sceneChangeTimer = null; }
  document.removeEventListener("mousemove", onDocMouseMove);
  document.removeEventListener("mouseup",   onDocMouseUp);
  if (excalidrawContainer) {
    excalidrawContainer.removeEventListener("pointerup", onPointerUp);
    excalidrawContainer.removeEventListener("keyup",     onKeyUp);
  }
  // Restore all hidden elements to their original opacity
  if (origOpacityMap.size > 0) {
    const all = api.getSceneElementsIncludingDeleted();
    const restored = all.map(el =>
      origOpacityMap.has(el.id) ? { ...el, opacity: origOpacityMap.get(el.id) } : el
    );
    api.updateScene({ elements: restored, commitToHistory: false });
    origOpacityMap.clear();
  }
  persist();
};

closeBtn.onclick = () => { cleanup(); panel.remove(); };

// ─── Initial render ───────────────────────────────────────────────────────────
render();
applyVisibility();
