/*
 * Excalidraw Script: Shape Intersections
 * Select exactly 2 shapes, then run this script.
 * Draws a small red circle at each intersection point.
 *
 * Supported shapes: circle, ellipse, rectangle, diamond, line, arrow, freedraw
 * Method: convert every shape to polyline segments, then find segment intersections.
 * Circles/ellipses are sampled at high resolution for accuracy.
 */

const CIRCLE_SAMPLES = 360;   // points sampled around a circle/ellipse perimeter
const MERGE_DISTANCE = 5;     // px — deduplicate points closer than this
const MARKER_MIN     = 6;     // px minimum marker radius

// ── geometry helpers ─────────────────────────────────────────────────────────

function segmentIntersection(p1, p2, p3, p4) {
  const d1x = p2.x - p1.x, d1y = p2.y - p1.y;
  const d2x = p4.x - p3.x, d2y = p4.y - p3.y;
  const cross = d1x * d2y - d1y * d2x;
  if (Math.abs(cross) < 1e-9) return null; // parallel

  const dx = p3.x - p1.x, dy = p3.y - p1.y;
  const t = (dx * d2y - dy * d2x) / cross;
  const u = (dx * d1y - dy * d1x) / cross;

  if (t < 0 || t > 1 || u < 0 || u > 1) return null;
  return { x: p1.x + t * d1x, y: p1.y + t * d1y };
}

function dedup(points) {
  const out = [];
  for (const p of points) {
    const close = out.some(q =>
      Math.hypot(q.x - p.x, q.y - p.y) < MERGE_DISTANCE
    );
    if (!close) out.push(p);
  }
  return out;
}

// Rotate a point around an origin by `angle` radians
function rotate(px, py, ox, oy, angle) {
  const cos = Math.cos(angle), sin = Math.sin(angle);
  const dx = px - ox, dy = py - oy;
  return {
    x: ox + dx * cos - dy * sin,
    y: oy + dx * sin + dy * cos,
  };
}

// ── shape → polyline segments ─────────────────────────────────────────────────

function shapeToPoints(el) {
  const cx = el.x + el.width  / 2;
  const cy = el.y + el.height / 2;
  const angle = el.angle || 0;

  function rotPt(px, py) {
    return angle === 0 ? { x: px, y: py } : rotate(px, py, cx, cy, angle);
  }

  switch (el.type) {

    case "ellipse": {
      const rx = Math.abs(el.width)  / 2;
      const ry = Math.abs(el.height) / 2;
      const pts = [];
      for (let i = 0; i <= CIRCLE_SAMPLES; i++) {
        const a = (2 * Math.PI * i) / CIRCLE_SAMPLES;
        pts.push(rotPt(cx + rx * Math.cos(a), cy + ry * Math.sin(a)));
      }
      return pts;
    }

    case "rectangle":
    case "image":
    case "text": {
      const tl = rotPt(el.x,            el.y);
      const tr = rotPt(el.x + el.width, el.y);
      const br = rotPt(el.x + el.width, el.y + el.height);
      const bl = rotPt(el.x,            el.y + el.height);
      return [tl, tr, br, bl, tl]; // closed
    }

    case "diamond": {
      const top   = rotPt(cx,            el.y);
      const right = rotPt(el.x + el.width, cy);
      const bot   = rotPt(cx,            el.y + el.height);
      const left  = rotPt(el.x,          cy);
      return [top, right, bot, left, top]; // closed
    }

    case "line":
    case "arrow": {
      // el.points is an array of [dx, dy] offsets from el.x, el.y
      if (!el.points || el.points.length < 2) return [];
      return el.points.map(([dx, dy]) => rotPt(el.x + dx, el.y + dy));
    }

    case "freedraw": {
      if (!el.points || el.points.length < 2) return [];
      return el.points.map(([dx, dy]) => rotPt(el.x + dx, el.y + dy));
    }

    default:
      return [];
  }
}

function pointsToSegments(pts) {
  const segs = [];
  for (let i = 0; i < pts.length - 1; i++) {
    segs.push([pts[i], pts[i + 1]]);
  }
  return segs;
}

// ── main ──────────────────────────────────────────────────────────────────────

const selected = ea.getViewSelectedElements()
  .filter(el => el.type !== "selection");

if (selected.length !== 2) {
  new Notice("Select exactly 2 shapes. Found: " + selected.length);
  return;
}

const pts0 = shapeToPoints(selected[0]);
const pts1 = shapeToPoints(selected[1]);

if (pts0.length < 2) {
  new Notice("First shape type is not supported: " + selected[0].type);
  return;
}
if (pts1.length < 2) {
  new Notice("Second shape type is not supported: " + selected[1].type);
  return;
}

const segs0 = pointsToSegments(pts0);
const segs1 = pointsToSegments(pts1);

const rawPoints = [];
for (const s0 of segs0) {
  for (const s1 of segs1) {
    const pt = segmentIntersection(s0[0], s0[1], s1[0], s1[1]);
    if (pt) rawPoints.push(pt);
  }
}

const points = dedup(rawPoints);

if (points.length === 0) {
  new Notice("No intersections found between the two shapes.");
  return;
}

// Marker size relative to the bounding boxes of both shapes
const avgSize = (
  Math.max(Math.abs(selected[0].width), Math.abs(selected[0].height)) +
  Math.max(Math.abs(selected[1].width), Math.abs(selected[1].height))
) / 2;
const markerR = Math.max(MARKER_MIN, avgSize * 0.025);

// ── draw markers ──────────────────────────────────────────────────────────────

ea.style.strokeColor     = "#e03131";
ea.style.backgroundColor = "transparent";
ea.style.fillStyle       = "solid";
ea.style.strokeWidth     = 0.5;
ea.style.roughness       = 0;
ea.style.opacity         = 100;

for (const pt of points) {
  ea.addEllipse(
    pt.x - markerR,
    pt.y - markerR,
    markerR * 2,
    markerR * 2
  );
}

await ea.addElementsToView(false, false, false);
new Notice("Drew " + points.length + " intersection marker(s).");
