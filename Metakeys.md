const view = app.workspace.getActiveViewOfType(customElements.get("excalidraw-view")?.constructor || Object);
if (!view) return;

const existingHud = view.contentEl.querySelector(".excalidraw-modifier-hud");
if (existingHud) existingHud.dispatchEvent(new CustomEvent("destroy"));

let state = { Shift: false, Control: false, Option: false, Command: false };
const canvas = view.contentEl.querySelector(".excalidraw__canvas");

const fireGlobalKey = (key, type) => {
    const keyMap = { Shift: "Shift", Control: "Control", Option: "Alt", Command: "Meta" };
    const event = new KeyboardEvent(type, {
        key: keyMap[key],
        code: keyMap[key],
        shiftKey: state.Shift,
        ctrlKey: state.Control,
        altKey: state.Option,
        metaKey: state.Command,
        bubbles: true,
        composed: true
    });
    window.dispatchEvent(event);

    // If we just deactivated Control, run the Purge Sequence
    if (key === "Control" && type === "keyup") {
        setTimeout(() => {
            // 1. Send Escape key
            window.dispatchEvent(new KeyboardEvent("keydown", { key: "Escape", bubbles: true }));

            // 2. Dispatch a 'pointerdown' on the canvas background
            // This is the most likely to trigger Excalidraw's internal "ClickAway"
            const rect = canvas.getBoundingClientRect();
            const clickEvent = new PointerEvent("pointerdown", {
                bubbles: true,
                cancelable: true,
                clientX: rect.left + 1,
                clientY: rect.top + 1,
                pointerType: "mouse"
            });
            canvas.dispatchEvent(clickEvent);

            // 3. Force focus back to the workspace
            view.contentEl.focus();
        }, 10);
    }
};

const handleEvents = (e) => {
    if (e.target.closest(".excalidraw-modifier-hud")) return;

    if (state.Shift) Object.defineProperty(e, 'shiftKey', { get: () => true });
    if (state.Control) Object.defineProperty(e, 'ctrlKey', { get: () => true });
    if (state.Option) Object.defineProperty(e, 'altKey', { get: () => true });
    if (state.Command) Object.defineProperty(e, 'metaKey', { get: () => true });

    if (state.Control && (e.type === "pointerdown" || e.type === "mousedown")) {
        e.stopPropagation();
        const contextEvent = new MouseEvent("contextmenu", {
            bubbles: true,
            cancelable: true,
            view: window,
            button: 2,
            buttons: 2,
            clientX: e.clientX,
            clientY: e.clientY,
            ctrlKey: true
        });
        e.target.dispatchEvent(contextEvent);
    }
};

const hud = document.createElement("div");
hud.className = "excalidraw-modifier-hud";
hud.style.cssText = "position:absolute; top:40px; left:20px; display:flex; gap:8px; z-index:10000; pointer-events:none;";

const cleanup = () => {
    ["Shift", "Control", "Option", "Command"].forEach(k => { if(state[k]) fireGlobalKey(k, "keyup") });
    window.removeEventListener("pointerdown", handleEvents, true);
    window.removeEventListener("pointermove", handleEvents, true);
    window.removeEventListener("pointerup", handleEvents, true);
    hud.remove();
};

hud.addEventListener("destroy", cleanup);

const createBtn = (key) => {
    const btn = document.createElement("div");
    const labels = { Shift: "⇧", Control: "⌃", Option: "⌥", Command: "⌘", Exit: "✕" };
    btn.innerHTML = `<div style="font-size:9px; opacity:0.8; line-height:1;">${key}</div><div style="font-size:16px;">${labels[key]}</div>`;

    btn.style.cssText = `width:40px; height:40px; background:var(--background-secondary-alt); border:1px solid var(--divider-color); border-radius:8px; display:flex; flex-direction:column; align-items:center; justify-content:center; font-weight:bold; pointer-events:auto; touch-action:none; color:var(--text-normal); cursor:pointer; box-shadow:0 2px 8px rgba(0,0,0,0.15);`;

    if (key === "Exit") {
        btn.onclick = cleanup;
        return btn;
    }

    let startTime;
    btn.onpointerdown = (e) => {
        e.stopPropagation();
        e.preventDefault();
        startTime = Date.now();
        state[key] = !state[key];

        if (state[key]) fireGlobalKey(key, "keydown");
        else fireGlobalKey(key, "keyup");

        btn.style.background = state[key] ? "var(--interactive-accent)" : "var(--background-secondary-alt)";
        btn.style.color = state[key] ? "white" : "var(--text-normal)";
    };

    btn.onpointerup = (e) => {
        e.stopPropagation();
        if (Date.now() - startTime > 450) {
            state[key] = false;
            fireGlobalKey(key, "keyup");
            btn.style.background = "var(--background-secondary-alt)";
            btn.style.color = "var(--text-normal)";
        }
    };

    return btn;
};

["Shift", "Control", "Option", "Command", "Exit"].forEach(k => hud.appendChild(createBtn(k)));
view.contentEl.appendChild(hud);

window.addEventListener("pointerdown", handleEvents, true);
window.addEventListener("pointermove", handleEvents, true);
window.addEventListener("pointerup", handleEvents, true);

view.register(cleanup);
