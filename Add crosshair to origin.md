/*
* @name Add Origin Crosshair
* @description Adds a locked red crosshair at 0,0 to mark the origin.
*/

const elements = ea.getViewElements();

// Check if a crosshair already exists to avoid duplicates
const existing = elements.find(el => el.customData?.type === "origin-crosshair");

if (!existing) {
  ea.style.strokeColor = "#e03131";
  ea.style.backgroundColor = "#ffc9c9";
  ea.style.fillStyle = "solid";
  ea.style.strokeWidth = 1;
  ea.style.roughness = 0; // Professional, clean lines

  // 1. Create the Circle (10px diameter, centered at 0,0)
  // In Excalidraw, x/y is the top-left, so we offset by radius
  const circleId = ea.addEllipse(-5, -5, 10, 10);

  // 2. Create Horizontal Line (20px, centered at 0,0)
  const hLineId = ea.addLine([[-10, 0], [10, 0]]);

  // 3. Create Vertical Line (20px, centered at 0,0)
  const vLineId = ea.addLine([[0, -10], [0, 10]]);

  // Group them, add metadata, and lock
  const groupElements = [circleId, hLineId, vLineId];
  const groupId = ea.addToGroup(groupElements);

  // Tag the elements so we can find them later
  groupElements.forEach(id => {
    const el = ea.getElement(id);
    el.customData = { type: "origin-crosshair" };
    el.locked = true;
  });

  ea.addElementsToView();

  // Optional: Center view on the new crosshair
  // ea.centerLookAtPoint([0, 0]);
} else {
  new Notice("Origin crosshair already exists.");
}
