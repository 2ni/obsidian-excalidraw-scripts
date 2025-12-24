/*
 * Create a perpendicular line between a selected line and selected circles
 */
const selectedElements = ea.getViewSelectedElements();

let line = null;

// find line
for (const el of selectedElements) {
  if (el.type === "line" && el.points.length === 2) {
    line = el;
    break;
  }
}

if (!line) {
  return console.error("You must select at leeast one circle and one straight line");
}

// --- GET LINE POINTS IN ABSOLUTE COORDINATES ---
const [p0, p1] = line.points;
const x1 = line.x + p0[0];
const y1 = line.y + p0[1];
const x2 = line.x + p1[0];
const y2 = line.y + p1[1];

// --- VECTOR OF LINE ---
const vx = x2 - x1;
const vy = y2 - y1;

for (const circle of selectedElements) {
  if (circle.type !== "ellipse") continue;

  // --- GET CIRCLE CENTER ---
  const cx = circle.x + circle.width / 2;
  const cy = circle.y + circle.height / 2;

  // --- PROJECT CIRCLE CENTER ON LINE ---
  const t = ((cx - x1) * vx + (cy - y1) * vy) / (vx * vx + vy * vy);
  const ix = x1 + t * vx;
  const iy = y1 + t * vy;

  // --- DRAW THE NEW LINE ---
  ea.style.strokeWidth=line.strokeWidth;
  ea.style.roughness = line.roughness;
  ea.style.strokeColor =  line.strokeColor;
  ea.style.strokeStyle = line.strokeStyle;
  ea.addLine([[cx, cy], [ix, iy]]);
}

// newElementsOnTop = true (3rd param)
await ea.addElementsToView(false, false, true);
