/*
 * Excalidraw Layer Manager Pro
 */

const panelId = "layer-manager-pro-panel";
const view = app.workspace.getActiveViewOfType(customElements.get("excalidraw-view")?.constructor || Object);
if (!view || !view.excalidrawAPI) { new Notice("Open an Excalidraw drawing first"); return; }

if (typeof ea !== "undefined") ea.setView(view);

const existingPanel = view.contentEl.querySelector(`#${panelId}`);
if (existingPanel) existingPanel.remove();

const STORAGE_KEY = "excalidraw-layers-cache";
if (!window[STORAGE_KEY]) window[STORAGE_KEY] = {};

const getSettings = () => {
    const file = view.file;
    if (!file) return {};
    const memory = window[STORAGE_KEY][file.path];
    if (memory) return memory;
    
    const cache = app.metadataCache.getFileCache(file);
    return cache?.frontmatter?.["excalidraw-layers"] ?? {};
};

const saveSettings = async (s) => {
    const file = view.file;
    if (!file) return;
    window[STORAGE_KEY][file.path] = s;
    const dataToSave = JSON.parse(JSON.stringify(s));
    try {
        await app.fileManager.processFrontMatter(file, (frontmatter) => {
            frontmatter["excalidraw-layers"] = dataToSave;
        });
    } catch (e) {
        console.error("Layer Manager Pro: Save failed", e);
    }
};

let saved = getSettings();
let config = {
  layers: Array.isArray(saved.layers) ? saved.layers : [{ id: "layer-root", name: "Layer 1", visible: true, parentId: null, expanded: true }],
  activeLayerId: saved.activeLayerId ?? "layer-root",
  panelPos: saved.panelPos ?? { top: "110px", right: "33px" }
};

let state = {
    editingLayerId: null,
    draggedLayerId: null,
    isUpdating: false,
    lastElementsJson: ""
};

// --- Styles ---
const styleTag = document.createElement("style");
styleTag.id = "layer-manager-pro-styles";
styleTag.innerHTML = `
    .layer-item:hover { background: var(--background-modifier-hover) !important; }
    .layer-item.is-active { background: var(--interactive-accent) !important; color: white !important; }
    .layer-item.drag-over { border-top: 2px solid var(--interactive-accent); }
    .layer-action-btn:hover { opacity: 1 !important; transform: scale(1.1); }
`;
document.head.appendChild(styleTag);

// --- UI Construction ---
const panel = document.createElement("div");
panel.id = panelId;
const pos = config.panelPos;
panel.style.cssText = `position:absolute; top:${pos.top}; right:${pos.right || "auto"}; left:${pos.left || "auto"}; width:250px; background:var(--background-secondary); border:1px solid var(--divider-color); box-shadow:0 4px 12px rgba(0,0,0,0.15); border-radius:8px; padding:10px; z-index:100; font-size:12px; display:flex; flex-direction:column; gap:8px; max-height: 80vh; overflow-y:auto; font-family: var(--font-ui); color: var(--text-normal);`;

const renderLayer = (layer, depth = 0) => {
    const isActive = layer.id === config.activeLayerId;
    const isVisible = layer.visible !== false;
    const hasChildren = config.layers.some(l => l.parentId === layer.id);
    
    let html = `
    <div class="layer-item ${isActive ? 'is-active' : ''}" 
         data-id="${layer.id}" 
         draggable="true"
         style="display:flex; align-items:center; gap:4px; padding:4px 6px; border-radius:4px; margin-left:${depth * 12}px; cursor:pointer; transition: all 0.1s; position:relative;">
        <span class="layer-toggle-expand" style="width:12px; display:inline-block; opacity:0.5; font-size:9px;">${hasChildren ? (layer.expanded ? '▼' : '▶') : ''}</span>
        <span class="layer-visibility layer-action-btn" style="cursor:pointer; opacity:${isVisible ? 1 : 0.3}; margin-right:4px;" title="Toggle Visibility">${isVisible ? '👁️' : '🕶️'}</span>
        <div class="layer-name-container" style="flex:1; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;">
            ${state.editingLayerId === layer.id 
                ? `<input type="text" class="layer-rename-input" value="${layer.name}" style="width:100%; height:18px; font-size:11px; padding:0 2px; border:none; outline:none; background:white; color:black; border-radius:2px;">`
                : `<span class="layer-name">${layer.name}</span>`
            }
        </div>
        <div class="layer-actions" style="display:none; gap:6px; align-items:center;">
            <span class="layer-select-elements layer-action-btn" title="Select Elements" style="opacity:0.6; font-size:10px;">🎯</span>
            <span class="layer-add-sub layer-action-btn" title="Add Sub-layer" style="opacity:0.6; font-size:10px;">➕</span>
            <span class="layer-delete layer-action-btn" title="Delete Layer" style="opacity:0.6; font-size:10px;">🗑️</span>
        </div>
    </div>`;

    if (layer.expanded !== false) {
        const children = config.layers.filter(l => l.parentId === layer.id);
        children.forEach(child => {
            html += renderLayer(child, depth + 1);
        });
    }
    return html;
};

const updateUI = () => {
    panel.innerHTML = `
    <div style="display:flex; justify-content:space-between; align-items:center; font-weight:bold; margin-bottom:4px; padding-bottom:4px; border-bottom:1px solid var(--divider-color);">
      <span>📂 Layers</span>
      <div style="display:flex; gap:8px; align-items:center;">
        <div id="btn-drag-panel" style="cursor:grab; padding:2px 4px; font-size: 14px; color: var(--text-muted);">⠿</div>
        <div id="btn-close" style="cursor:pointer; padding:2px 6px;">✕</div>
      </div>
    </div>
    <div id="layer-list" style="display:flex; flex-direction:column; gap:1px; flex:1; overflow-y:auto; border:1px solid var(--divider-color); border-radius:4px; padding:2px; background:var(--background-primary); min-height:100px;">
        ${config.layers.filter(l => !l.parentId).map(l => renderLayer(l)).join('')}
    </div>
    <div style="display:flex; gap:4px; margin-top:4px;">
        <button id="btn-new-layer" style="flex:1; height:24px; font-size:11px; cursor:pointer;">New Root Layer</button>
        <button id="btn-assign" style="flex:1; height:24px; font-size:11px; cursor:pointer;" title="Assign Selection to Active Layer">Move Selection</button>
    </div>
    <div style="display:flex; gap:4px; margin-top:2px; font-size:10px; color:var(--text-muted); justify-content:center;">
        Double-click name to rename • Drag to reorder
    </div>
    `;

    // Re-attach event listeners
    panel.querySelectorAll('.layer-item').forEach(item => {
        item.addEventListener('mouseenter', () => item.querySelector('.layer-actions').style.display = 'flex');
        item.addEventListener('mouseleave', () => item.querySelector('.layer-actions').style.display = 'none');
        
        item.addEventListener('click', (e) => {
            const id = item.dataset.id;
            if (e.target.classList.contains('layer-toggle-expand')) {
                const l = config.layers.find(l => l.id === id);
                l.expanded = !l.expanded;
                updateUI();
            } else if (e.target.classList.contains('layer-visibility')) {
                toggleLayerVisibility(id);
            } else if (e.target.classList.contains('layer-add-sub')) {
                addNewLayer(id);
            } else if (e.target.classList.contains('layer-delete')) {
                deleteLayer(id);
            } else if (e.target.classList.contains('layer-select-elements')) {
                selectLayerElements(id);
            } else {
                config.activeLayerId = id;
                saveSettings(config);
                updateUI();
            }
        });

        item.addEventListener('dblclick', (e) => {
            if (e.target.classList.contains('layer-name')) {
                state.editingLayerId = item.dataset.id;
                updateUI();
                const input = panel.querySelector('.layer-rename-input');
                input.focus();
                input.select();
                input.onblur = () => commitRename(item.dataset.id, input.value);
                input.onkeydown = (e) => { if (e.key === 'Enter') commitRename(item.dataset.id, input.value); };
            }
        });

        // Drag and Drop for Layer Arrangement
        item.addEventListener('dragstart', (e) => {
            state.draggedLayerId = item.dataset.id;
            e.dataTransfer.setData('text/plain', item.dataset.id);
            item.style.opacity = '0.5';
        });
        item.addEventListener('dragend', () => {
            item.style.opacity = '1';
            panel.querySelectorAll('.layer-item').forEach(i => i.classList.remove('drag-over'));
        });
        item.addEventListener('dragover', (e) => {
            e.preventDefault();
            item.classList.add('drag-over');
        });
        item.addEventListener('dragleave', () => {
            item.classList.remove('drag-over');
        });
        item.addEventListener('drop', (e) => {
            e.preventDefault();
            const targetId = item.dataset.id;
            const draggedId = state.draggedLayerId;
            if (draggedId && draggedId !== targetId) {
                handleLayerDrop(draggedId, targetId);
            }
        });
    });

    panel.querySelector('#btn-new-layer').onclick = () => addNewLayer();
    panel.querySelector('#btn-assign').onclick = () => moveSelectionToActiveLayer();
    panel.querySelector('#btn-close').onclick = () => {
        if (window.layerManagerProListener) window.layerManagerProListener();
        panel.remove();
        styleTag.remove();
    };
    
    initDraggable(panel);
};

// --- Actions ---

const addNewLayer = async (parentId = null) => {
    const id = "layer-" + Math.random().toString(36).substring(2, 9);
    const newLayer = { id, name: parentId ? "Sub-layer" : "New Layer", visible: true, parentId, expanded: true };
    config.layers.push(newLayer);
    config.activeLayerId = id;
    await saveSettings(config);
    updateUI();
};

const commitRename = async (id, newName) => {
    const layer = config.layers.find(l => l.id === id);
    if (layer) layer.name = newName;
    state.editingLayerId = null;
    await saveSettings(config);
    updateUI();
};

const deleteLayer = async (id) => {
    if (config.layers.length <= 1) return;
    const confirmDelete = confirm("Delete this layer? Elements will be reassigned to the default layer.");
    if (!confirmDelete) return;

    config.layers = config.layers.filter(l => l.id !== id && l.parentId !== id);
    if (config.activeLayerId === id) config.activeLayerId = config.layers[0].id;
    await saveSettings(config);
    updateUI();
};

const toggleLayerVisibility = async (id) => {
    const layer = config.layers.find(l => l.id === id);
    if (layer) {
        layer.visible = (layer.visible === false) ? true : false;
        await syncSceneVisibility();
        await saveSettings(config);
        updateUI();
    }
};

const selectLayerElements = (layerId) => {
    const elements = ea.getViewElements().filter(el => el.customData?.layerId === layerId);
    ea.selectElementsInView(elements);
};

const moveSelectionToActiveLayer = async () => {
    const selected = ea.getViewSelectedElements();
    if (selected.length === 0) return;
    
    state.isUpdating = true;
    await ea.copyViewElementsToEAforEditing(selected);
    ea.getElements().forEach(el => {
        el.customData = { ...el.customData, layerId: config.activeLayerId };
    });
    await ea.addElementsToView(false, false);
    await syncSceneVisibility();
    state.isUpdating = false;
};

const handleLayerDrop = async (draggedId, targetId) => {
    const draggedIdx = config.layers.findIndex(l => l.id === draggedId);
    const targetIdx = config.layers.findIndex(l => l.id === targetId);
    
    const [moved] = config.layers.splice(draggedIdx, 1);
    config.layers.splice(targetIdx, 0, moved);
    
    // Optional: Update parentId if dropped on/near? 
    // For now, simple reordering in the list.
    
    await saveSettings(config);
    updateUI();
    await syncSceneZIndex();
};

// --- Logic ---

const tagNewElements = async () => {
    if (state.isUpdating) return;
    
    const elements = ea.getViewElements();
    const updates = [];
    
    for (const el of elements) {
        if (!el.isDeleted && !el.customData?.layerId) {
            updates.push(el);
        }
    }
    
    if (updates.length > 0) {
        state.isUpdating = true;
        try {
            ea.clear();
            await ea.copyViewElementsToEAforEditing(updates);
            ea.getElements().forEach(el => {
                el.customData = { ...el.customData, layerId: config.activeLayerId };
                const visible = getLayerVisibility(config.activeLayerId);
                if (!visible) {
                    if (el.customData.originalOpacity === undefined) el.customData.originalOpacity = el.opacity;
                    el.opacity = 0;
                }
            });
            await ea.addElementsToView(false, false);
        } catch (e) {
            console.error("Layer Manager: Tagging failed", e);
        } finally {
            state.isUpdating = false;
        }
    }
};

const getLayerVisibility = (layerId) => {
    const layer = config.layers.find(l => l.id === layerId);
    if (!layer) return true;
    if (layer.visible === false) return false;
    if (layer.parentId) return getLayerVisibility(layer.parentId);
    return true;
};

const syncSceneVisibility = async () => {
    if (state.isUpdating) return;
    state.isUpdating = true;
    
    try {
        const elements = ea.getViewElements();
        const elementsToUpdate = [];
        
        for (const el of elements) {
            if (el.isDeleted) continue;
            const layerId = el.customData?.layerId || config.activeLayerId;
            const shouldBeVisible = getLayerVisibility(layerId);
            const targetOpacity = shouldBeVisible ? (el.customData?.originalOpacity ?? 100) : 0;
            
            if (el.opacity !== targetOpacity) {
                elementsToUpdate.push(el);
            }
        }
        
        if (elementsToUpdate.length > 0) {
            ea.clear();
            await ea.copyViewElementsToEAforEditing(elementsToUpdate);
            ea.getElements().forEach(el => {
                const layerId = el.customData?.layerId || config.activeLayerId;
                const shouldBeVisible = getLayerVisibility(layerId);
                const targetOpacity = shouldBeVisible ? (el.customData?.originalOpacity ?? 100) : 0;
                
                if (!shouldBeVisible && el.customData.originalOpacity === undefined) {
                    el.customData.originalOpacity = el.opacity;
                }
                el.opacity = targetOpacity;
                el.customData.layerId = layerId;
            });
            await ea.addElementsToView(false, false);
        }
    } catch (e) {
        console.error("Layer Manager: Sync visibility failed", e);
    } finally {
        state.isUpdating = false;
    }
};

const syncSceneZIndex = async () => {
    if (state.isUpdating) return;
    
    const elements = ea.getViewElements();
    if (elements.length === 0) return;

    const sortedElements = [...elements].sort((a, b) => {
        const layerA = a.customData?.layerId || config.activeLayerId;
        const layerB = b.customData?.layerId || config.activeLayerId;
        const idxA = config.layers.findIndex(l => l.id === layerA);
        const idxB = config.layers.findIndex(l => l.id === layerB);
        return idxA - idxB;
    });
    
    let changed = false;
    for (let i = 0; i < elements.length; i++) {
        if (elements[i].id !== sortedElements[i].id) {
            changed = true;
            break;
        }
    }
    
    if (changed) {
        state.isUpdating = true;
        try {
            ea.clear();
            await ea.copyViewElementsToEAforEditing(sortedElements);
            if (ea.getElements().length > 0) {
                await ea.addElementsToView(true, false);
            }
        } catch (e) {
            console.error("Layer Manager: Sync Z-Index failed", e);
        } finally {
            state.isUpdating = false;
        }
    }
};

// --- Change Listener ---

window.layerManagerProListener = view.excalidrawAPI.onChange(async (elements, appState) => {
    if (state.isUpdating) return;
    
    const elementsJson = JSON.stringify(elements.map(el => ({ id: el.id, version: el.version, layerId: el.customData?.layerId, groupIds: el.groupIds })));
    if (elementsJson === state.lastElementsJson) return;
    state.lastElementsJson = elementsJson;

    // Interaction check for group logic
    if (appState.pointerDown || appState.editingElement || appState.resizingElement || appState.draggingElement) return;

    const updates = [];
    for (const el of elements) {
        if (el.isDeleted) continue;
        if (el.groupIds && el.groupIds.length > 0) {
            const topGroupId = el.groupIds[el.groupIds.length - 1];
            const groupElements = elements.filter(e => e.groupIds && e.groupIds.includes(topGroupId));
            
            let currentLayerId = el.customData?.layerId || config.activeLayerId;
            let bestLayerId = currentLayerId;
            let bestLayerIdx = config.layers.findIndex(l => l.id === bestLayerId);
            
            for (const ge of groupElements) {
                const gelId = ge.customData?.layerId;
                if (gelId) {
                    const idx = config.layers.findIndex(l => l.id === gelId);
                    if (idx > bestLayerIdx) {
                        bestLayerIdx = idx;
                        bestLayerId = gelId;
                    }
                }
            }
            if (currentLayerId !== bestLayerId) {
                if (!updates.includes(el)) updates.push(el);
                el._targetLayerId = bestLayerId;
            }
        }
    }

    if (updates.length > 0) {
        state.isUpdating = true;
        try {
            ea.clear();
            await ea.copyViewElementsToEAforEditing(updates);
            ea.getElements().forEach(el => {
                const targetId = el._targetLayerId || config.activeLayerId;
                el.customData = { ...el.customData, layerId: targetId };
                delete el._targetLayerId;
            });
            await ea.addElementsToView(false, false);
        } catch (e) { console.error(e); } finally { state.isUpdating = false; }
    }
});

// --- Event Handlers ---
// Using capture phase or a safer event attachment
const excalContainer = view.contentEl.querySelector(".excalidraw");
if (excalContainer) {
    excalContainer.addEventListener("pointerup", () => {
        setTimeout(tagNewElements, 100);
    }, true);
}

// --- Draggable Panel ---
const initDraggable = (p) => {
  let isDragging = false;
  let offset = { x: 0, y: 0 };

  const finalizeDrag = async () => {
    if (!isDragging) return;
    isDragging = false;
    p.style.cursor = "default";
    config.panelPos = { top: p.style.top, right: p.style.right, left: p.style.left };
    await saveSettings(config);
  };

  const dragBtn = p.querySelector("#btn-drag-panel");
  dragBtn.addEventListener("mousedown", (e) => {
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

  window.addEventListener("mouseup", finalizeDrag);
};

// --- Init ---
view.contentEl.appendChild(panel);
updateUI();
tagNewElements().then(() => {
    syncSceneVisibility();
});
