# Room guide (three.js) — rooms with snapping points

How to generate a cosy, open-fronted cartoon room like the one in `Fox Room.html`, with a **snapping system** that furniture attaches to. Furniture built with `FURNITURE_GUIDE.md` declares how it mounts; the room declares where things can mount; one `place()` call puts a piece in, validates it, blocks walking, and wires up its interaction.

Hand this file and `FURNITURE_GUIDE.md` to an AI assistant (or follow them yourself) and ask, e.g., *"Build a 4 × 3 m bedroom following ROOM_GUIDE.md with a window on the back wall, then place a bed, desk and lamp."*

---

## 1. Overview

```
room-kit.js        helpers + materials shared by rooms and furniture (rbox, lathe, extrude, ell, group, piece, R, tok)
room.js            buildRoom(spec)  → Room { group, snaps, place(), remove(), layout(), walkable(), findPath(), showSnaps() }
furniture/*.js     defineFurniture({...}) per piece (see FURNITURE_GUIDE.md)
layout.json        optional saved arrangement: [{ item, snap, t, rotate }]
```

Flow:

1. `buildRoom(spec)` builds floor, walls, openings, lights, and **generates snap points**.
2. `room.place(item, snapId, opts)` positions a furniture instance on a snap, checks it fits, registers its footprint as an obstacle, marks space as occupied, and registers the furniture's own **child snaps** (e.g. a tabletop for a lamp).
3. Your app handles clicks: `room.pick(ray)` returns the furniture instance; its `use` spot tells the character where to stand.

---

## 2. Conventions (shared with furniture)

- **Meters, y-up**, floor top at **y = 0**.
- Room centred on the origin: x ∈ [−W/2, W/2], z ∈ [−D/2, D/2]. **+z is the open front** (toward the camera).
- **Yaw**: rotation about +y. `yaw = 0` means *the piece's front faces +z*. Back wall items face +z (yaw 0); left wall items face +x (yaw π/2); right wall items face −x (yaw −π/2).
- The character is ~1 m tall: seats 0.28–0.32 m, tabletops ~0.56 m, window sill ~0.85 m, wall height ~2.1 m.
- All placement yaws are **multiples of 90°** (keeps footprints axis-aligned and collision trivial).

---

## 3. Room spec

```js
const spec = {
  size: { w: 5.2, d: 3.6, h: 2.1 },          // interior
  wallThickness: 0.12,
  walls: {                                    // omit a side to leave it open
    back: { openings: [{ type: 'window', x: 0.05, y: 0.875, w: 1.1, h: 0.85 }] },
    left: { openings: [] },
    // right: { openings: [{ type: 'door', x: 0.4, w: 0.8, h: 1.6 }] },
  },
  floor: { style: 'planks', rows: 12 },       // 'planks' | 'tiles' | 'flat'
  skirting: true,                             // trim strip at the wall base
  snaps: { floorPitch: 0.1, wallHeights: [1.2, 1.45], ceiling: false },
  lighting: { sun: true, hemi: true, windowGlow: true },
  palette: 'organic',                         // R materials from design tokens
};
```

`x` for an opening is measured **along the wall from its centre**, in the wall's own left→right direction as seen from inside the room. `y` is the bottom of the opening.

---

## 4. Building the shell

Use the shared helpers (all exist in `Fox Room.html`; move them into `room-kit.js`):

```js
export function buildRoom(spec) {
  const { w, d, h } = spec.size, T = spec.wallThickness;
  const root = new THREE.Group(); root.name = 'Room';
  const room = { spec, group: root, snaps: new Map(), items: new Map(), obstacles: [], wallRects: [] };

  // Floor: one slab, top at y = 0, planks drawn on a canvas texture
  const floorMat = mat('floor', 0xffffff, 0.8, { map: floorTexture(spec.floor) });
  const floor = group(root, 'floor');
  piece(new THREE.BoxGeometry(w, 0.12, d), [R.edge, R.edge, floorMat, R.edge, R.edge, R.edge], floor, [0, -0.06, 0]);

  for (const side of Object.keys(spec.walls)) buildWall(room, side, spec.walls[side]);
  addLights(room);
  generateSnaps(room);
  return Object.assign(room, api(room));
}
```

### Walls with real openings

A wall is a set of boxes around its openings (no CSG needed):

```js
// Wall frame: origin at the wall's centre on the floor, local +x = "along", local +z = into the room
const WALLS = {
  back:  ({ w, d }) => ({ pos: [0, 0, -d / 2], yaw: 0,            len: w }),
  left:  ({ w, d }) => ({ pos: [-w / 2, 0, 0], yaw: Math.PI / 2,  len: d }),
  right: ({ w, d }) => ({ pos: [ w / 2, 0, 0], yaw: -Math.PI / 2, len: d }),
};
function buildWall(room, side, cfg) {
  const { len, pos, yaw } = WALLS[side](room.spec.size), H = room.spec.size.h, T = room.spec.wallThickness;
  const wall = group(room.group, 'wall', pos, [0, yaw, 0]); wall.userData.side = side;
  const box = (x0, x1, y0, y1) => x1 - x0 > 1e-3 && y1 - y0 > 1e-3 &&
    piece(new THREE.BoxGeometry(x1 - x0, y1 - y0, T), R.wall, wall, [(x0 + x1) / 2, (y0 + y1) / 2, -T / 2]);
  const ops = [...(cfg.openings || [])].sort((a, b) => a.x - b.x);
  let cursor = -len / 2;
  for (const o of ops) {
    const x0 = o.x - o.w / 2, x1 = o.x + o.w / 2, y0 = o.y ?? 0;
    box(cursor, x0, 0, H);                 // solid run before the opening
    box(x0, x1, 0, y0);                    // below (sill wall; 0 height for doors)
    box(x0, x1, y0 + o.h, H);              // lintel above
    buildOpening(wall, o);                 // frame, glass, sill, curtains / door leaf
    cursor = x1;
  }
  box(cursor, len / 2, 0, H);
  if (room.spec.skirting) piece(new THREE.BoxGeometry(len, 0.09, 0.025), R.trim, wall, [0, 0.045, 0.0125]);
  room.walls = { ...(room.walls || {}), [side]: { len, pos: V(...pos), yaw, openings: ops, node: wall } };
}
```

- **Window**: four `rbox` frame bars, a cross mullion, a transparent glass plane, a protruding sill (`rbox`), optional curtain rod + two `curtainGeo` panels. An outdoor sky plane sits ~0.4 m behind the wall (see `Fox Room.html` weather section).
- **Door**: frame bars + a leaf in a hinge group (see FURNITURE_GUIDE §5 *hinged door*).
- Corners: extend the side walls by `T` at the back so corners close.

---

## 5. Snap points

### 5.1 Schema

Every snap is one object in `room.snaps` (a `Map` by id):

```js
{
  id: 'back.floor',            // unique, human-readable
  kind: 'rail',                // 'point' | 'rail' | 'area'
  mount: 'wall-floor',         // what furniture mount types it accepts (see 5.2)
  // geometry (world space)
  origin: V(0, 0, -1.8),       // point: the spot; rail: centre; area: centre
  axis:   V(1, 0, 0),          // rail/area: the "along" direction
  length: 5.2,                 // rail/area: extent along axis
  width:  0,                   // area: extent across (perpendicular, in the floor plane)
  normal: V(0, 0, 1),          // out of the supporting surface (into the room for walls, up for floors)
  yaw: 0,                      // yaw a piece gets when snapped here (front faces away from the wall)
  y: 0,                        // height of the supporting surface (rails at wall heights, tabletop height…)
  tags: ['under-window'],      // free-form hints, e.g. 'corner', 'by-window', 'under-window'
  parent: null,                // furniture instance id for child snaps (tabletop, shelf)
  blocked: [],                 // [t0, t1] intervals along a rail taken by openings or other pieces
}
```

- **point** — one exact spot (a corner, the window sill centre, a ceiling hook).
- **rail** — a line; a piece slides along it with `t ∈ [-length/2, length/2]` (walls).
- **area** — a rectangle; a piece goes anywhere inside with `t = [along, across]` (open floor, a tabletop).

### 5.2 Mount types

| `mount` | Where | Piece origin (see FURNITURE_GUIDE) | Examples |
|---|---|---|---|
| `floor` | open floor (area) | footprint centre, base y = 0 | tea table, rug, armchair, floor lamp |
| `wall-floor` | along a wall, standing on the floor (rail) | **back-centre** of footprint, base y = 0 | bed, desk, wardrobe, bookshelf, mirror |
| `corner` | room corners (point) | footprint centre, base y = 0 (`place()` pushes it out of the corner by half its footprint) | plant, hat stand, corner shelf |
| `wall` | on a wall surface at a height (rail) | back-centre of the mounted object | to-do board, picture, wall shelf, clock |
| `window` | in a window opening (point) | centre of the sill | potted herb, cat bed, curtains |
| `surface` | on top of furniture (area, child snap) | footprint centre, base y = 0 | lamp, vase, books, cup |
| `ceiling` | ceiling (area) | top-centre | pendant light, mobile |

A furniture piece declares one or more acceptable mounts (e.g. a plant: `['floor', 'corner', 'surface', 'window']`).

### 5.3 Generating snaps

```js
function generateSnaps(room) {
  const { w, d, h } = room.spec.size, S = room.spec.snaps;
  // Open floor: one area covering the room (minus a margin)
  addSnap(room, { id: 'floor', kind: 'area', mount: 'floor', origin: V(0, 0, 0), axis: V(1, 0, 0), length: w - 0.2, width: d - 0.2, normal: V(0, 1, 0), yaw: 0, y: 0 });

  for (const [side, W] of Object.entries(room.walls)) {
    const yawQ = new THREE.Quaternion().setFromAxisAngle(V(0, 1, 0), W.yaw);
    const axis = V(1, 0, 0).applyQuaternion(yawQ), normal = V(0, 0, 1).applyQuaternion(yawQ);
    // Floor rail along the wall (inner face)
    const floorRail = addSnap(room, { id: `${side}.floor`, kind: 'rail', mount: 'wall-floor', origin: W.pos.clone(), axis, length: W.len, normal, yaw: W.yaw, y: 0 });
    // Wall-mount rails at each height
    for (const y of S.wallHeights) addSnap(room, { id: `${side}.wall@${y}`, kind: 'rail', mount: 'wall', origin: W.pos.clone().setY(y), axis, length: W.len, normal, yaw: W.yaw, y });
    // Openings block rails; windows add a point snap on their sill
    for (const o of W.openings) {
      const span = [o.x - o.w / 2 - 0.05, o.x + o.w / 2 + 0.05];
      if (o.type === 'door') floorRail.blocked.push({ span, maxH: Infinity });            // nothing in front of a door
      else floorRail.blocked.push({ span, maxH: o.y - 0.02 });                              // only low pieces under a window
      for (const y of S.wallHeights) if (y + 0.25 > o.y && y - 0.25 < o.y + o.h) room.snaps.get(`${side}.wall@${y}`).blocked.push({ span, maxH: Infinity });
      if (o.type === 'window') addSnap(room, { id: `${side}.window`, kind: 'point', mount: 'window', origin: W.pos.clone().addScaledVector(axis, o.x).addScaledVector(normal, 0.06).setY(o.y), normal: V(0, 1, 0), yaw: W.yaw, y: o.y, tags: ['by-window'] });
    }
  }
  // Corners where two walls meet (and front corners of each wall, for plants etc.)
  for (const [a, b, x, z, yaw] of [['back', 'left', -w / 2, -d / 2, Math.PI / 4], ['back', 'right', w / 2, -d / 2, -Math.PI / 4]])
    if (room.walls[a] && room.walls[b]) addSnap(room, { id: `${a}-${b}.corner`, kind: 'point', mount: 'corner', origin: V(x, 0, z), normal: V(0, 1, 0), yaw: 0, y: 0, tags: ['corner'], cornerDir: V(Math.sign(-x), 0, 1).normalize() });
  if (S.ceiling) addSnap(room, { id: 'ceiling', kind: 'area', mount: 'ceiling', origin: V(0, h, 0), axis: V(1, 0, 0), length: w - 0.4, width: d - 0.4, normal: V(0, -1, 0), yaw: 0, y: h });
}
function addSnap(room, s) { const snap = { tags: [], blocked: [], parent: null, ...s }; room.snaps.set(snap.id, snap); return snap; }
```

### 5.4 Resolving a placement

`resolve(snap, opts)` turns a snap + parameters into a world transform for the piece's origin:

```js
function resolve(snap, { t = 0, rotate = 0 } = {}) {
  const p = snap.origin.clone();
  if (snap.kind === 'rail') p.addScaledVector(snap.axis, t);
  if (snap.kind === 'area') {
    const across = V().crossVectors(snap.normal, snap.axis).normalize();          // in the surface plane
    p.addScaledVector(snap.axis, t[0]).addScaledVector(across, t[1]);
  }
  return { position: p, yaw: snap.yaw + rotate };
}
```

Snapping a cursor to the nearest valid `t`:

```js
function nearestT(snap, worldPoint, pitch = room.spec.snaps.floorPitch) {
  const rel = worldPoint.clone().sub(snap.origin), q = v => Math.round(v / pitch) * pitch;
  if (snap.kind === 'rail') return clamp(q(rel.dot(snap.axis)), -snap.length / 2, snap.length / 2);
  const across = V().crossVectors(snap.normal, snap.axis).normalize();
  return [clamp(q(rel.dot(snap.axis)), -snap.length / 2, snap.length / 2), clamp(q(rel.dot(across)), -snap.width / 2, snap.width / 2)];
}
```

---

## 6. Placing furniture

```js
const bed = room.place(Bed, 'back.floor', { t: 1.75 });
const nightstand = room.place(Nightstand, 'back.floor', { t: 0.95 });
room.place(BedsideLamp, `${nightstand.id}.top`);                    // child snap registered by the nightstand
room.place(Plant, 'back-right.corner');
room.place(TeaTable, 'floor', { t: [0.85, 1.25] });
room.place(TodoBoard, 'left.wall@1.45', { t: 0.75 });
```

### 6.1 `place()`

```js
function place(room, def, snapId, opts = {}) {
  const snap = room.snaps.get(snapId);
  if (!snap) throw new Error(`No snap "${snapId}"`);
  if (!def.mount.includes(snap.mount)) throw new Error(`${def.id} can't mount on ${snap.mount}`);
  const { position, yaw } = resolve(snap, opts);
  if (snap.mount === 'corner') {                      // push the footprint centre out of the corner
    const { w: fw, d: fd } = def.footprint;
    position.x += Math.sign(snap.cornerDir.x) * fw / 2; position.z += fd / 2;
  }
  if (def.origin === 'back' && !['wall-floor', 'wall'].includes(snap.mount)) {   // back-centred piece on open floor/surface:
    position.addScaledVector(V(Math.sin(yaw), 0, Math.cos(yaw)), -def.footprint.d / 2); // shift so the snap point is its footprint centre
  }
  const check = canPlace(room, def, snap, position, yaw, opts);
  if (!check.ok) throw new Error(`${def.id} doesn't fit at ${snapId}: ${check.reason}`);

  const inst = def.create();                          // { node, footprint, height, snaps, use, update, dispose }
  inst.id = opts.id || `${def.id}#${room.items.size + 1}`;
  inst.def = def; inst.snapId = snapId; inst.opts = opts;
  inst.node.position.copy(position);
  if (snap.mount === 'wall') inst.node.position.y = snap.y;   // wall pieces hang at the rail height
  if (snap.mount === 'surface') inst.node.position.y = snap.y;
  inst.node.rotation.y = yaw;
  inst.node.userData.kind = inst.id;                  // clicks resolve to this instance
  room.group.add(inst.node); inst.node.updateMatrixWorld(true);

  inst.rect = footprintRect(inst, snap);              // world AABB on the floor (or on the wall/surface plane)
  if (['floor', 'wall-floor', 'corner'].includes(snap.mount)) room.obstacles.push({ ...inst.rect, owner: inst.id });
  if (snap.mount === 'wall') room.wallRects.push({ side: snapId.split('.')[0], ...inst.wallSpan, owner: inst.id });
  for (const s of inst.snaps || []) addSnap(room, toWorldSnap(inst, s));   // child snaps: `${inst.id}.${s.id}`
  room.items.set(inst.id, inst);
  room.invalidateNav();
  return inst;
}
```

### 6.2 Footprints and validation

Furniture declares `footprint: { w, d }` in its local frame (w along local x, d along local z). Because yaws are multiples of 90°, the world rectangle is:

```js
function footprintRect(inst, snap) {
  const { w, d } = inst.footprint, yaw = inst.node.rotation.y, p = inst.node.position;
  const swap = Math.abs(Math.sin(yaw)) > 0.5, ww = swap ? d : w, dd = swap ? w : d;
  // origin is back-centre for wall-floor / wall, centre otherwise
  const back = snap.mount === 'wall-floor' || snap.mount === 'wall';
  const fwd = V(Math.sin(yaw), 0, Math.cos(yaw));            // the piece's front direction
  const c = back ? p.clone().addScaledVector(fwd, d / 2) : p.clone();   // footprint centre
  return { x0: c.x - ww / 2, x1: c.x + ww / 2, z0: c.z - dd / 2, z1: c.z + dd / 2, h: inst.height };
}

function canPlace(room, def, snap, position, yaw, opts) {
  const probe = { footprint: def.footprint, height: def.height, node: { position, rotation: { y: yaw } } };
  const r = footprintRect(probe, snap), { w, d } = room.spec.size, eps = 0.005;
  if (r.x0 < -w / 2 - eps || r.x1 > w / 2 + eps || r.z0 < -d / 2 - eps || r.z1 > d / 2 + eps) return { ok: false, reason: 'outside the room' };
  if (snap.kind === 'rail') {
    const half = (snap.mount === 'wall' ? def.wall?.w ?? def.footprint.w : def.footprint.w) / 2, t = opts.t || 0;
    if (Math.abs(t) + half > snap.length / 2 + eps) return { ok: false, reason: 'past the end of the wall' };
    for (const b of snap.blocked) if (t + half > b.span[0] && t - half < b.span[1] && def.height > b.maxH) return { ok: false, reason: 'blocks an opening' };
  }
  const overlaps = (a, b) => a.x0 < b.x1 - eps && a.x1 > b.x0 + eps && a.z0 < b.z1 - eps && a.z1 > b.z0 + eps;
  if (['floor', 'wall-floor', 'corner'].includes(snap.mount))
    for (const o of room.obstacles) if (overlaps(r, o) && !def.flat && !o.flat) return { ok: false, reason: `overlaps ${o.owner}` };
  if (snap.mount === 'surface') {
    const s = snap;                                           // must fit inside the parent's surface area
    if (def.footprint.w > s.length + eps || def.footprint.d > s.width + eps) return { ok: false, reason: 'too big for that surface' };
  }
  return { ok: true };
}
```

- `def.flat: true` (rugs, mats) skips overlap checks and doesn't become an obstacle.
- Wall items check their `wall: { w, h }` span against other wall items on the same side (same idea, 1D on the rail plus height).
- Child pieces are removed with their parent: `remove(id)` removes the node, its obstacle, its child snaps, and any items on those snaps.

### 6.3 Interaction spots

Furniture declares where the character stands to use it, **in its local frame**:

```js
use: { at: [0, 0.75], yaw: Math.PI, label: 'Take a nap', action: 'lie', seat: { pos: [0, 0.39, 0.6], clip: 'Lie' } }
```

`place()` converts it to world space:

```js
function useSpot(inst) {
  const u = inst.use; if (!u) return null;
  const m = inst.node.matrixWorld, at = V(u.at[0], 0, u.at[1]).applyMatrix4(m);
  return { x: at.x, z: at.z, yaw: inst.node.rotation.y + u.yaw, label: u.label, action: u.action,
           seat: u.seat && { pos: V(...u.seat.pos).applyMatrix4(m), clip: u.seat.clip } };
}
```

Your controller walks the character to `(x, z)`, turns it to `yaw`, then runs the action (sit, lie, open a panel, toggle a lamp…). `at` must land on walkable floor — `place()` warns if it doesn't.

---

## 7. Navigation

The walk grid is rebuilt from `room.obstacles` whenever furniture changes:

```js
const AGENT_R = 0.22, CELL = 0.08;
function buildNav(room) {
  const { w, d } = room.spec.size, NX = Math.ceil(w / CELL), NZ = Math.ceil(d / CELL);
  const free = (x, z) => {
    if (x < -w / 2 + AGENT_R || x > w / 2 - AGENT_R || z < -d / 2 + AGENT_R || z > d / 2 - AGENT_R) return false;
    for (const o of room.obstacles) { const dx = Math.max(o.x0 - x, 0, x - o.x1), dz = Math.max(o.z0 - z, 0, z - o.z1); if (dx * dx + dz * dz < AGENT_R ** 2) return false; }
    return true;
  };
  // A* over cells + string-pulling — copy findPath()/los()/nearestFree() from Fox Room.html unchanged
  return { free, NX, NZ, CELL };
}
```

Open walls (front/right) still clamp the agent to the room rectangle. Use `room.walkable(x, z)` to validate `use.at` spots and click targets.

---

## 8. Layouts (save / load)

```js
room.layout();   // → [{ item: 'bed', snap: 'back.floor', t: 1.75, rotate: 0, id: 'bed#1' }, { item: 'lamp', snap: 'nightstand#2.top', t: [0, 0] }, …]
room.load(layout, registry);   // registry: { bed: Bed, lamp: BedsideLamp, … } — parents are placed before children
```

Store layouts in `localStorage` or a JSON file; they survive model changes because they reference snaps by id, not coordinates.

---

## 9. Debug view

```js
room.showSnaps(true);   // draws every snap
```

- **area** → translucent rectangle (sage `--color-accent-2-300`)
- **rail** → line with ticks every `floorPitch`; blocked spans in terracotta `--color-accent-500`
- **point** → ring + arrow showing `yaw`
- child snaps in a lighter tint; hovering a snap shows its id

Use `canPlace()` live while dragging to tint the ghost preview sage (fits) or terracotta (doesn't), with `check.reason` in a tooltip.

---

## 10. Lighting, look and atmosphere

- `scene.background` and floor/wall/trim colours come from the Organic tokens through `tok()` (see `Fox Room.html`): walls `--color-accent-2-200`, trim `--color-neutral-100`, floor planks `--color-neutral-200/300`.
- Lights: a `HemisphereLight` (`--color-neutral-100` sky / `--color-neutral-400` ground, ~1.5) + a shadow-casting `DirectionalLight` from front-right-above (2.4, PCF, 2048 map, `normalBias 0.02`) + a soft `PointLight` just inside each window.
- Renderer: `NeutralToneMapping`, `SRGBColorSpace`, shadows on, `localClippingEnabled` if characters use clip-plane effects.
- Camera: perspective FOV 30, orbiting the room centre with azimuth 0.1–1.2 rad and elevation 0.32–1.05 rad; auto-fit so all room corners are on screen; zoom 0.5–1.15× toward the character. (Copy `fit()`/`place()` from `Fox Room.html`.)
- Weather behind windows: a canvas-painted sky plane plus rain/snow particles between the sky and the wall — copy the weather section of `Fox Room.html`.

---

## 11. Checklist

- [ ] Room centred on origin, floor at y = 0, open side at +z
- [ ] Walls built around openings; corners closed
- [ ] Snaps generated: `floor`, `<side>.floor`, `<side>.wall@h`, corners, `<side>.window`, optional `ceiling`
- [ ] Openings block rails (doors fully, windows above sill height)
- [ ] `place()` validates mount type, bounds, rail ends, openings, overlaps, surface size
- [ ] Obstacles and nav rebuilt after every place/remove
- [ ] Every interactive piece's `use.at` is walkable
- [ ] Layout round-trips through `layout()` / `load()`
- [ ] Colours only from design tokens

---

## 12. Prompt templates

**New room**

> Build a room following `ROOM_GUIDE.md`.
> - **Size:** _(e.g. 4.4 × 3.2 m, wall height 2.1 m)_
> - **Walls:** _(which sides exist; windows/doors with position and size)_
> - **Floor:** _(planks / tiles / flat)_
> - **Snaps:** _(wall heights, ceiling yes/no, floor pitch)_
>
> Generate the snap map and turn on `showSnaps()` so I can see it.

**Furnish it**

> Using `ROOM_GUIDE.md` and `FURNITURE_GUIDE.md`, place: _(list of pieces with the snap and position you want, or "find good spots")_. If something doesn't fit, report the `canPlace` reason and suggest the nearest valid spot.
