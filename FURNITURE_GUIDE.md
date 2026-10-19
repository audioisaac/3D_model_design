# Furniture guide (three.js) — snappable furniture

How to build cartoon furniture and props that **snap into rooms** made with `ROOM_GUIDE.md`. Each piece is a self-contained `defineFurniture({...})` module: it builds its model, says how it mounts (floor, wall, corner, surface…), declares its footprint, exposes **child snaps** (a tabletop other items can sit on) and describes how the character uses it.

The visual language matches `Fox Room.html`: soft, chunky, hand-made, every edge rounded, colours from the Organic design tokens. The older Fox-Room-specific guide is kept as `FURNITURE_GUIDE v1 (Fox Room).md`.

---

## 1. The contract

```js
// furniture/nightstand.js
import { defineFurniture, rbox, lathe, ell, group, piece, R } from '../room-kit.js';

export const Nightstand = defineFurniture({
  id: 'nightstand',
  label: 'Nightstand',
  mount: ['wall-floor', 'floor'],     // snap types it accepts (ROOM_GUIDE §5.2)
  origin: 'back',                     // built back-centred (§2)
  footprint: { w: 0.42, d: 0.36 },    // meters, local frame: w along x, d along z
  height: 0.43,                       // top of the piece (used for window clearance)
  build(node) { /* add meshes to node — see §3 */ },
  snaps: [                            // child snaps, local frame
    { id: 'top', kind: 'area', mount: 'surface', origin: [0, 0.43, 0.18], length: 0.36, width: 0.3 },
  ],
  use: null,                          // or an interaction (§6)
});
```

`defineFurniture()` just validates the object and adds `create()`:

```js
export function defineFurniture(def) {
  for (const k of ['id', 'mount', 'footprint', 'height', 'build']) if (def[k] == null) throw new Error(`${def.id || '?'}: missing ${k}`);
  def.create = (opts = {}) => {
    const node = new THREE.Group(); node.name = def.id;
    const parts = def.build(node, opts) || {};          // build may return handles (doors, lights…) for update()
    return { node, parts, footprint: def.footprint, height: def.height, snaps: def.snaps || [], use: def.use, wallSpan: def.wall,
             update: def.update ? (dt, ctx) => def.update(parts, dt, ctx) : null };
  };
  return Object.freeze(def);
}
```

| Field | Required | Meaning |
|---|---|---|
| `id` | ✓ | Registry key, used in saved layouts |
| `label` | | Display name |
| `mount` | ✓ | Array of accepted snap mounts: `floor`, `wall-floor`, `corner`, `wall`, `window`, `surface`, `ceiling` |
| `footprint` | ✓ | `{ w, d }` floor (or surface) rectangle, local frame |
| `height` | ✓ | Overall height in meters |
| `origin` | | `'back'` if built back-centred (§2). Required for wall pieces that also accept `floor`/`surface` |
| `wall` | wall pieces | `{ w, h }` area taken on the wall |
| `flat` | | `true` for rugs and mats: no collision, no obstacle |
| `build(node, opts)` | ✓ | Adds meshes to `node`; may return handles |
| `snaps` | | Child snaps (§5) |
| `use` | | Interaction (§6) |
| `update(parts, dt, ctx)` | | Per-frame animation (§7) |
| `variants` | | Named option sets passed as `opts` (§8) |

---

## 2. Origin and orientation (critical for snapping)

Build every piece in its **local frame** with the **front facing +z**. Where the origin sits depends on how it mounts:

| Mount | Origin | Picture |
|---|---|---|
| `wall-floor` | **back-centre** of the footprint, on the floor (y = 0). The piece extends from z = 0 to z = +d | bed, desk, wardrobe, mirror, bookshelf |
| `wall` | **back-centre** of the object, at its vertical centre. The back face lies on z = 0 | board, picture frame, wall shelf, clock |
| `floor`, `corner`, `surface`, `window` | **footprint centre**, base at y = 0 | table, stool, rug, plant, lamp, vase |
| `ceiling` | top-centre, hanging down along −y | pendant lamp |

A piece that accepts both `wall-floor` and `floor` (a nightstand) should be built back-centred. The room offsets it by `d / 2` when it's placed on open floor (`place()` handles this when `def.origin === 'back'`).

```
        +z (front, toward the room)
         ▲
   ┌─────┼─────┐   wall-floor: origin ● at the back edge, centred
   │     │     │
   │     │     │   d
   └─────●─────┘
       wall (z = 0)        x →
```

Rules:
- Nothing may poke behind z = 0 for `wall-floor` and `wall` pieces, or it clips into the wall.
- Keep the model inside its declared `footprint` (handles, knobs and overhangs ≤ 0.03 m).
- `height` must be the true top, because it decides whether the piece fits under a window.

---

## 3. Modelling toolkit

These are shared helpers in `room-kit.js`, copied from `Fox Room.html`:

| Helper | Makes |
|---|---|
| `rbox(w, h, d, r = 0.02)` | Rounded bevelled box, centred on the origin |
| `lathe([[r, y], …], segs = 40)` | Turned profile around y, with smooth normals. List the points bottom → top and end at `[0, yTop]` |
| `ell(rx, ry, rz)` | Ellipsoid: cushions, pillows, leaves, knobs |
| `rrect(w, h, r)` / `archShape(w, h, rise)` / `archInto(path, …)` | 2D shapes for extrusion and holes |
| `extrude(shape, depth, bevel = 0.02)` | Bevelled extrusion along +z |
| `curtainGeo(w, h, folds, amp, thick)` | Wavy fabric panel |
| `group(parent, kind, pos, rot)` | Sub-group (pass `null` for `kind` inside furniture) |
| `piece(geo, mat, parent, pos, rot)` | Mesh with shadows on |

### Style rules

- **No raw boxes.** Use `rbox` for slabs, with bevels: 0.005 m for tiny parts, 0.012–0.02 m for panels, 0.03–0.06 m for big soft shapes.
- **Turned parts are lathes** with a small lip: legs, tops, lamp bases, pots, knobs, feet.
- **Outlines are extrusions**: headboards, frames, arches.
- **Soft goods are ellipsoids**: cushions, pillows, folded clothes.
- **Chunky proportions** for a 1 m character: seats 0.28–0.32 m, tabletops 0.56 m, panel thickness 0.03 m, leg radius 0.016–0.035 m.
- **Overlap parts by ~1 mm** rather than touching exactly, to avoid gaps and z-fighting.

### Recipes (local frame)

```js
// Turned leg, flaring under the top
piece(lathe([[0.03, 0], [0.024, 0.02], [0.02, 0.2], [0.026, 0.27], [0.035, 0.28]], 20), R.dark, node, [x, 0, z]);

// Round tabletop with a rounded lip at height T
piece(lathe([[0, T - 0.04], [0.4, T - 0.04], [0.418, T - 0.032], [0.425, T - 0.02], [0.418, T - 0.008], [0.4, T], [0, T]], 64), R.wood, node);

// Rectangular top (back-centred piece: centre it at z = d/2)
piece(rbox(w, 0.045, d, 0.02), R.wood, node, [0, T - 0.0225, d / 2]);

// Three splayed stool legs
for (let k = 0; k < 3; k++)
  piece(new THREE.CylinderGeometry(0.016, 0.02, 0.27, 12), R.dark, group(node, null, [0, 0, 0], [0, k * TAU / 3, 0]), [0.11, 0.135, 0], [0, 0, 0.12]);

// Drawer front + knob on the front face (z = d)
piece(rbox(0.36, 0.13, 0.02, 0.012), R.trim, node, [0, 0.31, d]);
piece(new THREE.SphereGeometry(0.016, 14, 10), R.clay, node, [0, 0.31, d + 0.016]);

// Hinged door: pivot group on the hinge line, panel offset inside it
const door = group(node, null, [-w / 2, 0, d]);
piece(rbox(w, h, 0.03, 0.012), R.trim, door, [w / 2, h / 2, -0.015]);
// open: door.rotation.y = -1.75 * t

// Frame with an arched hole
const outer = archInto(new THREE.Shape(), fw + 0.09, fh + 0.09, rise + 0.045, -0.045);
outer.holes.push(archInto(new THREE.Path(), fw, fh, rise, 0));
piece(extrude(outer, 0.022, 0.012), R.wood, node);

// Cushion
piece(ell(0.15, 0.03, 0.15), R.cushion, node, [0, 0.29, 0]);

// Fanned plant leaves
for (let k = 0; k < 7; k++) {
  const h = 0.2 + (k % 2) * 0.06;
  const tilt = group(group(node, null, [0, 0.3, 0], [0, k * 2.4, 0]), null, [0, 0, 0], [0, 0, -(0.3 + (k % 3) * 0.22)]);
  piece(ell(0.024, h, 0.085), [R.leaf1, R.leaf2, R.leaf3][k % 3], tilt, [0, h * 0.95, 0]);
}
```

Canvas-textured faces (cork, posters, book covers): draw on a `<canvas>`, wrap it in a `CanvasTexture` with `colorSpace = SRGBColorSpace`, and put it on a plane 1 mm in front of the frame.

---

## 4. Materials

Use the shared `R` set only. Every material is matte (`metalness: 0`) and coloured from Organic tokens via `tok()`:

| Key | Use | Token |
|---|---|---|
| `R.wood` | Tops, frames, carcasses | `--color-accent-700` |
| `R.dark` | Legs, rails, poles, bases | `--color-accent-800` |
| `R.trim` | Painted panels, doors, sheets | `--color-neutral-100` |
| `R.panel` | Recessed panels | `--color-neutral-200` |
| `R.inside` | Interiors, backs | `--color-neutral-700` |
| `R.clay` / `R.terra` | Knobs, pots, blankets | `--color-accent-600` / `-500` |
| `R.cushion` / `R.pad` | Upholstery | `--color-accent-2-500` / `-400` |
| `R.rugA` / `R.rugB` | Rugs, pillows | `--color-accent-300` / `--color-neutral-100` |
| `R.leaf1–3` | Foliage | `--color-accent-2-400…600` |
| `R.china`, `R.paper`, `R.curtain` | Crockery, paper, fabric | neutral / accent ramps |

For a new material, add it to `R` in `room-kit.js` with the same pattern (`mat(name, tok('--color-…'), roughness)`), never inside a furniture file. For glowing parts, use `emissive: WARM, emissiveIntensity: 0` and let the light helper (§7) drive it.

---

## 5. Child snaps

A piece can offer places for other pieces. Declare them in the local frame; the room converts them to world space and names them `<instanceId>.<snapId>`:

```js
snaps: [
  { id: 'top',    kind: 'area',  mount: 'surface', origin: [0, 0.56, 0.3], length: 0.9, width: 0.5 },   // desk top
  { id: 'shelf1', kind: 'area',  mount: 'surface', origin: [0, 0.4, 0.15], length: 0.72, width: 0.26 }, // bookshelf shelf
  { id: 'hook',   kind: 'point', mount: 'surface', origin: [0.1, 1.12, 0] },                            // hat stand peg
]
```

- `origin` is the centre of the supporting surface. Its `y` is the surface height.
- `length` runs along local x and `width` along local z.
- Items on a child snap move and get removed with their parent.
- `surface` items need `footprint.w ≤ length` and `footprint.d ≤ width`.

---

## 6. Interaction (`use`)

Describe how the character uses the piece, in the **local frame**:

```js
use: {
  label: 'Study at the desk',      // hover label / button text
  labelActive: 'Stop studying',    // optional, while in use
  at: [0, 0.95],                   // where the character stands: [x, z] local (in front of the piece)
  yaw: Math.PI,                    // facing, relative to the piece (π = facing the piece's front)
  action: 'sit',                   // 'sit' | 'lie' | 'open' | 'toggle' | 'look' | custom string
  seat: { pos: [0, 0.32, 0.55], clip: 'Study' },   // for sit / lie: where the root lands
  panel: 'todo',                   // for 'open': which close-up UI to show
}
```

- `at` must be on walkable floor: at least 0.22 m (agent radius) outside every footprint, including this one. A rule of thumb is `[0, d + 0.3]`.
- Clips available on the characters: `Sit`, `Study` and `Lie` for seats; one-shots `Wave`, `Hop`, `Nudge`, `Reach` and `Twirl` for actions.
- `seat.pos` is where the **character root** goes: on the seat surface for `Sit`/`Study`, on the mattress for `Lie`.
- The room converts all of this to world space (`ROOM_GUIDE §6.3`). Your controller does the walking and plays the clips.

---

## 7. Moving parts and lights

Return handles from `build()` and animate them in `update()`:

```js
build(node) {
  const doorL = group(node, null, [-0.45, 0, 0.5]), doorR = group(node, null, [0.45, 0, 0.5]);
  // … panels inside doorL / doorR …
  return { doorL, doorR, open: 0, target: 0 };
},
update(p, dt) {
  p.open += (p.target - p.open) * Math.min(1, dt * 8);
  p.doorL.rotation.y = -1.75 * p.open; p.doorR.rotation.y = 1.75 * p.open;
},
// app: inst.parts.target = 1  → doors swing open
```

**Lights**: return `{ light, glowMats, on: false, t: 0, intensity }` and fade in `update()`:

```js
update(p, dt) {
  p.t += ((p.on ? 1 : 0) - p.t) * Math.min(1, dt * 8);
  p.light.intensity = p.intensity * p.t;
  p.glowMats.forEach((m, i) => (m.emissiveIntensity = p.t * (i === p.glowMats.length - 1 ? 2.2 : 0.55)));
},
use: { label: 'Turn on the lamp', labelActive: 'Turn off the lamp', action: 'toggle', at: null },  // at: null → no walking
```

`ctx` in `update(parts, dt, ctx)` carries `{ time, weather, room }`, so pieces can react to the weather.

---

## 8. Variants

One definition can produce several looks:

```js
variants: {
  oak:   { top: 'wood', legs: 'dark' },
  sage:  { top: 'cushion', legs: 'wood' },
  round: { shape: 'round' },
},
build(node, opts = {}) {
  const top = R[opts.top || 'wood'];
  // …
}
// room.place(Table, 'floor', { t: [0, 0.8], variant: 'sage' })  → create(def.variants[variant])
```

Variants must not change `footprint` or `height` unless you also declare them per variant (`variants.round.footprint`).

---

## 9. Worked examples

### Bed (`wall-floor`, lie)

```js
export const Bed = defineFurniture({
  id: 'bed', label: 'Bed', mount: ['wall-floor'], origin: 'back',
  footprint: { w: 0.92, d: 1.42 }, height: 0.95,
  build(node) {
    const W = 0.86, L = 1.4;
    for (const [x, z] of [[-1, 0.06], [1, 0.06], [-1, L - 0.06], [1, L - 0.06]])
      piece(new THREE.CylinderGeometry(0.03, 0.024, 0.08, 14), R.dark, node, [x * (W / 2 - 0.02), 0.04, z]);
    piece(rbox(W + 0.06, 0.15, L, 0.04), R.wood, node, [0, 0.145, L / 2]);              // base
    piece(rbox(W, 0.15, L - 0.06, 0.06), R.trim, node, [0, 0.29, L / 2 + 0.01]);         // mattress
    piece(rbox(W + 0.05, 0.05, 0.86, 0.025), R.terra, node, [0, 0.36, L - 0.46]);         // blanket
    piece(ell(0.3, 0.065, 0.16, 40, 24), R.rugB, node, [0, 0.4, 0.26]);                  // pillow
    piece(extrude(archShape(W + 0.06, 0.95, 0.22), 0.04, 0.02), R.wood, node, [0, 0, 0.0]);   // headboard (back at z = 0)
    piece(rbox(W + 0.06, 0.46, 0.05, 0.025), R.wood, node, [0, 0.23, L - 0.025]);         // footboard
  },
  use: { label: 'Take a nap', labelActive: 'Wake up', at: [-0.78, 0.83], yaw: Math.PI / 2, action: 'lie', seat: { pos: [0, 0.39, 1.16], clip: 'Lie' } },
});
```

### Bedside lamp (`surface`, toggle)

```js
export const BedsideLamp = defineFurniture({
  id: 'bedside-lamp', label: 'Bedside lamp', mount: ['surface', 'floor'],
  footprint: { w: 0.26, d: 0.26 }, height: 0.32,
  build(node) {
    piece(lathe([[0, 0], [0.06, 0], [0.075, 0.05], [0.07, 0.11], [0.035, 0.16], [0.016, 0.19], [0, 0.19]]), R.cushion, node);
    const shade = mat('shade', R.trim.color, 0.95, { side: THREE.DoubleSide, emissive: WARM, emissiveIntensity: 0 });
    const bulb = mat('bulb', R.trim.color, 0.4, { emissive: WARM, emissiveIntensity: 0 });
    piece(lathe([[0.13, 0.15], [0.12, 0.2], [0.09, 0.32]], 40), shade, node);
    piece(new THREE.SphereGeometry(0.03, 16, 12), bulb, node, [0, 0.21, 0]).castShadow = false;
    const light = new THREE.PointLight(WARM, 0, 3, 1.3); light.position.set(0, 0.24, 0); node.add(light);
    return { light, glowMats: [shade, bulb], on: false, t: 0, intensity: 1.3 };
  },
  update(p, dt) { p.t += ((p.on ? 1 : 0) - p.t) * Math.min(1, dt * 8); p.light.intensity = p.intensity * p.t; p.glowMats.forEach((m, i) => (m.emissiveIntensity = p.t * (i ? 2.2 : 0.55))); },
  use: { label: 'Turn on the lamp', labelActive: 'Turn off the lamp', action: 'toggle', at: null },
});
```

### Wall shelf (`wall`, with child snaps)

```js
export const WallShelf = defineFurniture({
  id: 'wall-shelf', label: 'Wall shelf', mount: ['wall'], origin: 'back',
  footprint: { w: 0.7, d: 0.2 }, height: 0.2, wall: { w: 0.7, h: 0.2 },
  build(node) {
    piece(rbox(0.7, 0.03, 0.2, 0.012), R.wood, node, [0, -0.085, 0.1]);
    for (const s of [-1, 1]) piece(extrude(rrect(0.03, 0.16, 0.01), 0.12, 0.008), R.dark, node, [s * 0.28, -0.02, 0.01]);
  },
  snaps: [{ id: 'top', kind: 'area', mount: 'surface', origin: [0, -0.07, 0.1], length: 0.66, width: 0.18 }],
});
```

---

## 10. Exporting to `.glb` (React Three Fiber)

```js
const inst = Bed.create();
const buf = await new GLTFExporter().parseAsync(inst.node, { binary: true });
download(new Blob([buf], { type: 'model/gltf-binary' }), 'bed.glb');
```

Snap and interaction data can travel with the model as glTF extras:

```js
inst.node.userData.furniture = { id: Bed.id, mount: Bed.mount, footprint: Bed.footprint, height: Bed.height, snaps: Bed.snaps, use: Bed.use };
// GLTFExporter writes userData into "extras"; read it back in R3F with scene.userData.furniture
```

```jsx
const { scene } = useGLTF('/bed.glb');
const meta = scene.userData.furniture;   // footprint, snaps, use spot
return <primitive object={scene.clone()} position={[x, 0, z]} rotation={[0, yaw, 0]} />;
```

---

## 11. Checklist

- [ ] Front faces +z. Origin follows §2 for its mount (back-centre for wall pieces)
- [ ] Nothing behind z = 0 for `wall-floor` / `wall`; model stays inside `footprint`
- [ ] `height` is the true top
- [ ] Only `rbox` / `lathe` / `extrude` / `ell`, all with bevels
- [ ] Colours only from `R` (design tokens)
- [ ] Child snaps sit exactly on their surfaces
- [ ] `use.at` is ≥ 0.22 m clear of the footprint
- [ ] Moving parts and lights are driven from `update()`
- [ ] Tested by placing it on each mount type it declares, with `room.showSnaps(true)`

---

## 12. Prompt template

> Create a snappable furniture piece following `FURNITURE_GUIDE.md` (and the snap system in `ROOM_GUIDE.md`).
> - **Item:** _(e.g. armchair, bookshelf, piano, toy chest, floor lamp)_
> - **Mounts:** _(floor / wall-floor / corner / wall / surface / window / ceiling)_
> - **Size:** _(footprint w × d, height, or "pick sensible child-sized dimensions")_
> - **Look:** _(materials from `R`, details)_
> - **Child snaps:** _(e.g. the top surface, each shelf)_
> - **Interaction:** _(none / sit / lie / open / toggle / custom + label)_
>
> Output a single `defineFurniture({...})` module, then show a `room.place(...)` call that puts it in a sensible spot.
