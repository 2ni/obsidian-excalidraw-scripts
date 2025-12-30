/*
```javascript*/
// Excalidraw Layer Manager Script - v5 FINAL

// Verify the minimum plugin version to ensure compatibility.
if (!ea.verifyMinimumPluginVersion || !ea.verifyMinimumPluginVersion("2.0.0")) {
  new Notice("This script requires a newer version of Excalidraw. Please install the latest version.");
  return;
}

// --- SINGLETON PATTERN: Ensure only one Layer Manager exists ---
if (window.layerManager && window.layerManager.isOpen) {
  window.layerManager.open(); // Re-focus if already open
  return;
}

class LayerManagerModal extends ea.FloatingModal {
  constructor(app) {
    super(app);
    this.isLayerManager = true;
    this.currentView = null;
    this.layerState = new Map();
    this.selectedItems = new Set();
    this.draggedItem = null;
    this.renderAllDebounced = this.debounce(this.renderAll.bind(this), 50);
    this.excalidrawUnsubscribe = null;
    this.leafChangeObserver = null;
  }

  debounce(func, delay) {
    let timeout;
    return (...args) => {
      clearTimeout(timeout);
      timeout = setTimeout(() => func.apply(this, args), delay);
    };
  }

  onOpen() {
    this.isOpen = true;
    const initialView = this.app.workspace.getActiveViewOfType(ea.obsidian.ItemView);
    if (initialView && initialView.getViewType() === "excalidraw") {
        this.setExcalidrawView(initialView);
    } else {
        this.close();
        new Notice("Please open an Excalidraw drawing first.");
        return;
    }

    this.contentEl.empty();
    this.contentEl.createEl('h2', { text: 'Layer Manager' });

    const searchInput = this.contentEl.createEl('input', { type: 'text', placeholder: 'Search...', cls: 'search-input' });
    searchInput.oninput = () => this.renderAllDebounced();

    const listContainer = this.contentEl.createDiv({ cls: 'list-container' });
    listContainer.onclick = (e) => {
      if (e.target === listContainer) {
        this.selectedItems.clear();
        ea.selectElementsInView([]);
      }
    };

    const controls = this.contentEl.createDiv({ cls: 'controls' });
    const btnMoveUp = controls.createEl('button', { title: 'Move Up' });
    const btnMoveDown = controls.createEl('button', { title: 'Move Down' });
    const btnNewLayer = controls.createEl('button', { title: 'Create New Layer' });
    const btnAddToLayer = controls.createEl('button', { title: 'Add to Existing Layer' });

    ea.obsidian.setIcon(btnMoveUp, "arrow-up");
    ea.obsidian.setIcon(btnMoveDown, "arrow-down");
    ea.obsidian.setIcon(btnNewLayer, "plus-square");
    ea.obsidian.setIcon(btnAddToLayer, "layers");

    btnMoveUp.onclick = () => this.moveItems('up');
    btnMoveDown.onclick = () => this.moveItems('down');
    btnNewLayer.onclick = () => this.newLayer();
    btnAddToLayer.onclick = () => this.addToLayer();

    this.leafChangeObserver = this.app.workspace.on('active-leaf-change', this.handleLeafChange.bind(this));
    this.renderAll();
  }

  onClose() {
    this.isOpen = false;
    if (this.excalidrawUnsubscribe) this.excalidrawUnsubscribe();
    if (this.leafChangeObserver) this.app.workspace.offref(this.leafChangeObserver);
    const styleEl = document.getElementById('layer-manager-style');
    if (styleEl) styleEl.remove();
    window.layerManager = null;
  }

  setExcalidrawView(view) {
    if (this.currentView === view && ea.targetView === view) return;
    this.currentView = view;
    ea.setView(view);

    if (this.excalidrawUnsubscribe) this.excalidrawUnsubscribe();

    const api = ea.getExcalidrawAPI();
    this.excalidrawUnsubscribe = api.onChange((elements, appState) => {
        this.handleSelectionChange(appState);
        this.renderAllDebounced();
    });

    const viewContainer = ea.targetView.containerEl;
    viewContainer.removeEventListener('pointerup', this.handleCanvasClick);
    viewContainer.addEventListener('pointerup', this.handleCanvasClick.bind(this));

    //this.renderAll(); // DENIS
  }

  handleCanvasClick() {
      const currentSelection = new Set(Object.keys(ea.getExcalidrawAPI().getAppState().selectedElementIds));
      const areSetsEqual = (a, b) => a.size === b.size && [...a].every(value => b.has(value));
      if (!areSetsEqual(currentSelection, this.selectedItems)) {
        this.handleSelectionChange(ea.getExcalidrawAPI().getAppState());
      }
  }

  handleLeafChange(leaf) {
    if (!leaf || (leaf.view.getViewType() !== 'excalidraw' && this.isOpen)) {
      this.close();
    } else if (leaf.view.getViewType() === 'excalidraw') {
      this.setExcalidrawView(leaf.view);
    }
  }

  handleSelectionChange(appState) {
    const newSelectedIds = new Set(Object.keys(appState.selectedElementIds));
    const areSetsEqual = (a, b) => a.size === b.size && [...a].every(value => b.has(value));
    if (!areSetsEqual(newSelectedIds, this.selectedItems)) {
      this.selectedItems = newSelectedIds;
      this.renderAll();
    }
  }

  getElementsAndLayers() {
    if (!ea || !this.currentView) return { allItems: [], elementMap: new Map() };
    const elements = ea.getViewElements();
    const elementMap = new Map(elements.map(el => [el.id, el]));
    const groupIds = new Set();
    elements.forEach(el => el.groupIds.forEach(gid => groupIds.add(gid)));

    const layers = Array.from(groupIds).map(gid => {
      const elementsInGroup = elements.filter(el => el.groupIds.includes(gid));
      if (elementsInGroup.length === 0) return null;

      const directChildren = elementsInGroup.filter(el => el.groupIds[0] === gid);

      const zIndex = Math.max(...elementsInGroup.map(el => elements.indexOf(el)));
      const isVisible = elementsInGroup.some(el => el.opacity > 0);
      const isLocked = elementsInGroup.some(el => el.locked);

      let name = `Layer ${gid.substring(0, 4)}`;
      const namedElement = elementsInGroup.find(el => el.customData?.layerName && el.customData.layerId === gid);
      if(namedElement) name = namedElement.customData.layerName;

      if (!this.layerState.has(gid)) this.layerState.set(gid, { isCollapsed: false });

      return {
        id: gid, type: 'layer', name: name,
        children: directChildren.map(el => el.id),
        zIndex: zIndex, isVisible: isVisible, isLocked: isLocked,
        isCollapsed: this.layerState.get(gid).isCollapsed,
      };
    }).filter(Boolean);

    const topLevelElements = elements.filter(el => el.groupIds.length === 0);
    const allItems = [ ...topLevelElements.map(el => ({ ...el, zIndex: elements.indexOf(el) })), ...layers ];
    allItems.sort((a, b) => b.zIndex - a.zIndex);

    return { allItems, elementMap };
  }

  renderAll() {
    if (!this.contentEl || !this.isOpen) return;
    const { allItems, elementMap } = this.getElementsAndLayers();
    const searchInput = this.contentEl.querySelector('.search-input');
    const filter = searchInput ? searchInput.value.toLowerCase() : '';

    const listContainer = this.contentEl.querySelector('.list-container');
    if (!listContainer) return;
    listContainer.empty();

    const renderedItems = new Set();

    const renderItem = (item, container, depth) => {
      if (!item || renderedItems.has(item.id)) return;
      let itemName = item.customData?.name || (item.type === 'layer' ? item.name : `${item.type.charAt(0).toUpperCase() + item.type.slice(1)} ${item.id.substring(0,4)}`);
      if (filter && !itemName.toLowerCase().includes(filter) && !this.isParentOfFiltered(item, filter, allItems)) return;

      renderedItems.add(item.id);
      const itemEl = this.createItemElement(item, itemName, depth);
      container.appendChild(itemEl);

      if (item.type === 'layer' && !item.isCollapsed) {
        item.children.forEach(childId => {
          const childItem = allItems.find(i => i.id === childId);
          if(childItem) renderItem(childItem, container, depth + 1);
        });
      }
    };

    const topLevelItems = allItems.filter(item => {
        const el = elementMap.get(item.id);
        if(el) return el.groupIds.length === 0;
        const elementsInGroup = ea.getViewElements().filter(e => e.groupIds.includes(item.id));
        if (elementsInGroup.length === 0) return true;
        const parentGroup = elementsInGroup[0].groupIds[1];
        return !parentGroup;
    });

    topLevelItems.forEach(item => renderItem(item, listContainer, 0));
  }

  isParentOfFiltered(layer, filter, allItems) {
      if (layer.type !== 'layer') return false;
      for (const childId of layer.children) {
          const child = allItems.find(i => i.id === childId);
          if (child) {
              const childName = child.customData?.name || (child.type === 'layer' ? child.name : `${child.type.charAt(0).toUpperCase() + child.type.slice(1)} ${child.id.substring(0,4)}`);
              if (childName.toLowerCase().includes(filter)) return true;
              if (this.isParentOfFiltered(child, filter, allItems)) return true;
          }
      }
      return false;
  }

  createItemElement(item, itemName, depth) {
    const isLayer = item.type === 'layer';
    const itemEl = document.createElement('div');
    itemEl.className = 'list-item';
    itemEl.style.marginLeft = `${depth * 20}px`;
    itemEl.dataset.id = item.id;

    if (this.selectedItems.has(item.id) || (isLayer && item.children.some(childId => this.selectedItems.has(childId)))) {
        itemEl.classList.add('is-selected');
    }

    const dragHandle = itemEl.createDiv({ cls: 'drag-handle' });
    ea.obsidian.setIcon(dragHandle, "grip-vertical");
    dragHandle.draggable = true;
    dragHandle.ondragstart = (e) => { e.stopPropagation(); this.draggedItem = item; e.dataTransfer.setData('text/plain', item.id); };

    if (isLayer) {
      const collapseBtn = itemEl.createEl('button', { cls: 'collapse-btn', text: item.isCollapsed ? '▶' : '▼' });
      collapseBtn.onclick = (e) => { e.stopPropagation(); const state = this.layerState.get(item.id) || {}; state.isCollapsed = !state.isCollapsed; this.layerState.set(item.id, state); this.renderAll(); };
    }

    const nameEl = itemEl.createSpan({ text: itemName, cls: 'item-name' });
    nameEl.contentEditable = "true";
    nameEl.spellcheck = false;
    nameEl.onkeydown = (e) => { if (e.key === "Enter") { e.preventDefault(); e.target.blur(); } };
    nameEl.onblur = (e) => { const newName = e.target.innerText.trim(); if (newName && newName !== itemName) this.renameItem(item, newName); else e.target.innerText = itemName; };
    nameEl.onclick = (e) => { e.stopPropagation(); this.selectedItems.clear(); this.selectedItems.add(item.id); ea.selectElementsInView([item.id]); };

    const buttonsEl = itemEl.createDiv({ cls: 'item-buttons' });
    const visibilityBtn = buttonsEl.createEl('button', { cls: 'visibility-btn'});
    ea.obsidian.setIcon(visibilityBtn, (item.type === 'layer' ? item.isVisible : item.opacity > 0) ? 'eye' : 'eye-off');
    visibilityBtn.onclick = (e) => { e.stopPropagation(); this.toggleVisibility(item); };

    console.log(item.id, item.locked);
    const lockBtn = buttonsEl.createEl('button', { cls: 'lock-btn' });
    ea.obsidian.setIcon(lockBtn, item.locked ? 'lock' : 'unlock');
    lockBtn.onclick = (e) => { e.stopPropagation(); this.toggleLock(item); };

    itemEl.ondragover = (e) => { e.preventDefault(); e.stopPropagation(); itemEl.classList.add('drag-over'); };
    itemEl.ondragleave = () => itemEl.classList.remove('drag-over');
    itemEl.ondrop = (e) => { e.preventDefault(); e.stopPropagation(); itemEl.classList.remove('drag-over'); if (this.draggedItem) this.handleDrop(this.draggedItem, item); };

    itemEl.onclick = (e) => {
      if (e.target.closest('button, .drag-handle, .item-name')) return;
      if (this.selectedItems.has(item.id) && this.selectedItems.size === 1) this.selectedItems.clear();
      else if (this.selectedItems.has(item.id)) this.selectedItems.delete(item.id);
      else this.selectedItems.add(item.id);
      ea.selectElementsInView(Array.from(this.selectedItems));
    };

    itemEl.oncontextmenu = (e) => {
      e.preventDefault();
      const menu = new ea.obsidian.Menu();
      menu.addItem((item) => item.setTitle("Add to New Layer").onClick(() => this.newLayer()));
      menu.addItem((item) => item.setTitle("Add to Existing Layer").onClick(() => this.addToLayer()));
      menu.showAtMouseEvent(e);
    };

    return itemEl;
  }

  // --- ACTION IMPLEMENTATIONS ---
  renameItem(item, newName) { ea.copyViewElementsToEAforEditing(ea.getViewElements()); if (item.type === 'layer') { ea.getElements().filter(el => el.groupIds.includes(item.id)).forEach(el => { ea.addAppendUpdateCustomData(el.id, { layerName: newName, layerId: item.id }); }); } else { const el = ea.getElement(item.id); if(el) ea.addAppendUpdateCustomData(el.id, { name: newName }); } ea.addElementsToView().then(() => this.renderAllDebounced()); }
  toggleVisibility(item) { const elementsToToggle = this.getElementsOfItem(item, true); const isCurrentlyVisible = item.type === 'layer' ? item.isVisible : item.opacity > 0; ea.copyViewElementsToEAforEditing(elementsToToggle); elementsToToggle.forEach(el => { const eaEl = ea.getElement(el.id); if (eaEl) { if (isCurrentlyVisible) { ea.addAppendUpdateCustomData(eaEl.id, { originalOpacity: eaEl.opacity }); eaEl.opacity = 0; } else { const originalOpacity = eaEl.customData?.originalOpacity ?? 100; eaEl.opacity = originalOpacity; ea.addAppendUpdateCustomData(eaEl.id, { originalOpacity: undefined }); } } }); ea.addElementsToView().then(() => this.renderAllDebounced()); }
  toggleLock(item) { const elementsToToggle = this.getElementsOfItem(item, true); const shouldBeLocked = !elementsToToggle.every(el => el.locked); ea.copyViewElementsToEAforEditing(elementsToToggle); elementsToToggle.forEach(el => { const eaEl = ea.getElement(el.id); if (eaEl) eaEl.locked = shouldBeLocked; }); ea.addElementsToView().then(() => this.renderAllDebounced()); }
  getElementsOfItem(item, returnFullElement = false) { const allElements = ea.getViewElements(); if (item.type !== 'layer') { return returnFullElement ? allElements.filter(el => el.id === item.id) : [{id: item.id}]; } const elementsInLayer = allElements.filter(el => el.groupIds.includes(item.id)); return returnFullElement ? elementsInLayer : elementsInLayer.map(el => ({id: el.id})); }
  moveItems(direction) { if (this.selectedItems.size === 0) return; const api = ea.getExcalidrawAPI(); const elementsToMove = ea.getViewElements().filter(el => this.selectedItems.has(el.id) || el.groupIds.some(gid => this.selectedItems.has(gid))); if (direction === 'up') api.bringForward(elementsToMove); else api.sendBackward(elementsToMove); setTimeout(() => this.renderAllDebounced(), 200); }
  handleDrop(dragged, target) { if (target.type === 'layer' && dragged.id !== target.id) { if (this.selectedItems.size === 0) this.selectedItems.add(dragged.id); this.addToLayer(target.id); } else { const api = ea.getExcalidrawAPI(); const { allItems } = this.getElementsAndLayers(); const allSceneElements = ea.getViewElements(); const draggedZ = allItems.findIndex(i=>i.id === dragged.id); const targetZ = allItems.findIndex(i=>i.id === target.id); if (draggedZ === -1 || targetZ === -1) return; const diff = Math.abs(draggedZ - targetZ); const elementsToMove = ea.getViewElements().filter(el => this.selectedItems.has(el.id) || el.groupIds.some(gid => this.selectedItems.has(gid))); for(let i = 0; i < diff; i++) { if (draggedZ > targetZ) api.sendBackward(elementsToMove); else api.bringForward(elementsToMove); } setTimeout(() => this.renderAllDebounced(), 200); } }
  async newLayer() { const name = await utils.inputPrompt("New layer name:"); if(!name) return; if (this.selectedItems.size === 0) { const newLayerId = ea.addToGroup([]); const newLayer = {id: newLayerId, type: 'layer', name: name}; this.renameItem(newLayer, name); return; } const newLayerId = ea.addToGroup(Array.from(this.selectedItems)); ea.copyViewElementsToEAforEditing(ea.getViewElements()); ea.addElementsToView().then(() => { const { allItems } = this.getElementsAndLayers(); const newLayer = allItems.find(i => i.id === newLayerId); if(newLayer) this.renameItem(newLayer, name); }); }
  async addToLayer(targetLayerId) { if (!targetLayerId) { const { allItems } = this.getElementsAndLayers(); const layers = allItems.filter(i => i.type === 'layer'); if (layers.length === 0) { this.newLayer(); return; } targetLayerId = await utils.suggester(layers.map(l => l.name), layers.map(l => l.id), "Select a layer"); if (!targetLayerId) return; } if (this.selectedItems.size === 0) { new Notice("Select items to add."); return; } ea.copyViewElementsToEAforEditing(ea.getViewElements()); Array.from(this.selectedItems).forEach(id => { const el = ea.getElement(id); if (el && !el.groupIds.includes(targetLayerId)) { el.groupIds.unshift(targetLayerId); } }); await ea.addElementsToView(); this.renderAllDebounced(); }
}

// --- INITIALIZATION ---
const style = document.createElement('style');
style.id = 'layer-manager-style';
style.textContent = `
  .list-container { max-height: 400px; overflow-y: auto; cursor: default; }
  .list-item { display: flex; align-items: center; padding: 4px 8px; border-bottom: 1px solid var(--background-modifier-border); gap: 4px;}
  .list-item:hover { background-color: var(--background-modifier-hover); }
  .list-item.is-selected { background-color: var(--interactive-accent); color: var(--text-on-accent); }
  .list-item.is-selected:hover { background-color: var(--interactive-accent-hover); }
  .list-item.is-selected button, .list-item.is-selected .drag-handle, .list-item.is-selected .collapse-btn { color: var(--text-on-accent); }
  .list-item.drag-over { border-top: 2px solid var(--interactive-accent); }
  .drag-handle { cursor: grab; padding: 0 4px; color: var(--text-muted); display: flex; align-items: center; }
  .item-name { flex-grow: 1; outline: none; padding: 2px 4px; border-radius: 4px; cursor: text;}
  .item-name:focus { background-color: var(--background-primary); color: var(--text-normal); box-shadow: 0 0 0 1px var(--interactive-accent); }
  .item-buttons { display: flex; }
  .item-buttons button { background: none; border: none; cursor: pointer; padding: 2px; display: flex; align-items: center; justify-content: center; color: var(--text-muted); }
  .item-buttons button:hover { color: var(--text-normal); }
  .search-input { width: 100%; padding: 8px; margin-bottom: 10px; background-color: var(--background-secondary); border: 1px solid var(--background-modifier-border); color: var(--text-normal); border-radius: 4px; }
  .controls { margin-top: 10px; display: flex; gap: 5px; }
  .controls button { flex-grow: 1; display: flex; align-items: center; justify-content: center; }
  .collapse-btn { width: 20px; text-align: center; background: none; border: none; cursor: pointer; color: var(--text-muted); }
`;
document.head.appendChild(style);

window.layerManager = new LayerManagerModal(ea.plugin.app);
window.layerManager.open();
