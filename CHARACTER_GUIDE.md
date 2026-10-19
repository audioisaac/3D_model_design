# Character model guide (three.js)

How to add new cartoon characters to `fox-model.js` — the procedural system behind the fox, bunny, bear, cat and chibi human — and export them as rigged, animated `.glb` files for React Three Fiber.

You can hand this whole file to an AI assistant (or follow it yourself) together with `fox-model.js`, and ask for a new character, e.g. *"Add a penguin following CHARACTER_GUIDE.md"*.

---

## 1. How the system works

Characters are **sculpted, not assembled**. There are no stacked spheres and cones. Each character body is:

1. **A signed-distance sculpt.** A function `sculpt(x, y, z)` returns the distance from a point to the character's surface (negative inside). Body parts are ellipsoids and tapered capsules, blended with smooth-union (`smin`) so they flow into each other like clay.
2. **Polygonised** into one continuous, watertight mesh by `surfaceNet()` (surface nets plus a Newton projection onto the exact surface, so the result is smooth).
3. **Painted** with per-vertex colours. Markings such as a cream belly, dark paws and an ink nose are soft-edged masks, not separate meshes.
4. **Skinned** to a shared 10-bone rig with automatic weights, so it bends at the joints.
5. **Decorated** with rigid face parts (eyes, mouth, whiskers, brows) parented to the `Head` bone.

Because **every character shares the same rig**, all animation clips, outfits, headwear and the hat work on every character automatically.

```
fox-model.js
├─ math: clamp, sstep, smin, smax, sdEll, sdCone
├─ surfaceNet(f, min, max, h)           → { position, normal, index, count }
├─ SPECIES   (UI list: key, label, swatches)
├─ HUMAN     (human customisation options)
├─ DEFAULT_LOOK / normalizeLook()
└─ buildCharacter(opts)
   ├─ Rig (bones)
   ├─ base()   – torso, arms, legs     (shared)
   ├─ DEFS     – one entry per species (head, ears, tail, paint, face, hat)
   ├─ sculpt() – combines base + species parts
   ├─ skinning, outfits, headwear, face, hair
   ├─ applyLook() / prepare() / needsBuild()
   └─ animation clips
```

---

## 2. Coordinate conventions

- Units are **meters**, **y-up**, and the character **faces +z**.
- The feet stand at **y = 0**, and the character is about **1 m tall** including the ears.
- `_L` means the **character's left (+x)**; `_R` is −x.

### The rig (shared)

| Bone | Parent | Rest position (local) | Notes |
|---|---|---|---|
| `Rig` | – | `[0, 0, 0]` | Root. Move it for hops and sitting |
| `Body` | Rig | `[0, 0.16, 0]` | Torso pivot (hips) |
| `Head` | Body | `[0, 0.37, 0]` | Neck pivot |
| `Ear_L` / `Ear_R` | Head | `[±0.15, 0.315, -0.02]`, rot `[-0.12, 0, ∓0.38]` | Animal ears |
| `Arm_L` / `Arm_R` | Body | `[±0.15, 0.28, 0.01]`, rot `[-0.2, 0, ±0.75]` | Shoulder pivot. The arm hangs along local −y |
| `Leg_L` / `Leg_R` | Rig | `[±0.085, 0.16, 0]` | Hip pivot. The leg hangs along local −y |
| `Tail` | Body | `[0, 0.07, -0.13]` | Tail root |

Also on `Head`: `Eye_L`, `Eye_R` (rigid groups, scaled by blink tracks), `FaceFeatures`, `HatSocket`, and `Hair` (human only).

**Do not rename or re-parent bones.** Clips, outfits and the room page address them by name.

---

## 3. The sculpt toolkit

All of these are already defined at the top of `fox-model.js`.

| Function | Meaning |
|---|---|
| `sdEll(x, y, z, rx, ry, rz)` | Distance to an ellipsoid centred at the origin. Offset the point to move it: `sdEll(x - cx, y - cy, z - cz, …)` |
| `sdCone(px,py,pz, ax,ay,az, bx,by,bz, r1, r2)` | Tapered capsule from A (radius r1) to B (radius r2). Sets the global `coneH` (0–1 along the segment) |
| `smin(a, b, k)` | Smooth union. `k` is the blend radius in meters (0.01 is crisp, 0.08 very soft) |
| `smax(a, b, k)` | Smooth intersection. `smax(a, -b, k)` carves `b` out of `a` |
| `sstep(e0, e1, x)` | Smoothstep, used for soft paint masks |
| `faceZ(fn, x, y)` | Ray-marches from the front (+z) to find where the surface `fn` sits at (x, y). Useful for placing pads and noses on a face |

### The G and L arrays

`sculpt()` fills two scratch arrays per sample point. Your species code **reads** `L` and **writes** `G`.

**`L`** holds the sample's position in each bone's local space (3 floats per bone):

| Bone | Index | x, y, z |
|---|---|---|
| Body | 1 | `L[3]`, `L[4]`, `L[5]` |
| Head | 2 | `L[6]`, `L[7]`, `L[8]` |
| Ear_L / Ear_R | 3 / 4 | `L[9..11]`, `L[12..14]` |
| Arm_L / Arm_R | 5 / 6 | `L[15..17]`, `L[18..20]` |
| Leg_L / Leg_R | 7 / 8 | `L[21..23]`, `L[24..26]` |
| Tail | 9 | `L[27]`, `L[28]`, `L[29]` |

**`G`** holds per-part distances. Every slot you are responsible for **must be written** on every call; use `INF` (Infinity) when the part doesn't exist.

| Slot | Part | Written by |
|---|---|---|
| `G[0]` | torso | `base()` (shared) |
| `G[1]` | scarf | `sculpt()` when `scarf: true` |
| `G[2]` | **head** | species `head()` |
| `G[3]` | **head extra** (cheek fluff, muzzle, whisker pads) | species `head()` (`INF` if unused) |
| `G[4]` | **nose** | species `head()` |
| `G[5]`, `G[6]` | **ears L / R** | species `ears()` |
| `G[7]`, `G[8]` | arms | `base()` |
| `G[9]`, `G[10]` | legs | `base()` |
| `G[11]` | **tail** | species `tail()` (`INF` if none) |
| `G[12]` | **tail parameter u** (0 at root, 1 at tip), used for painting | species `tail()` |
| `G[13]`, `G[14]` | arm cones (for sleeves) | `base()` |
| `G[15]`, `G[16]` | **ear cups** (inner-ear hollows) | species `ears()` (`INF` if unused) |
| `G[17]` | body without scarf | `sculpt()` |

Skin weights are derived from these slots automatically: the head slots go to `Head`, `G[5]`/`G[6]` to the ear bones, and `G[11]` to `Tail`.

---

## 4. Anatomy of a species definition

Every species is one entry in the `DEFS` object inside `buildCharacter()`:

```js
penguin: {
  label: 'Penguin',
  bounds: [[-0.38, -0.03, -0.32], [0.38, 1.0, 0.4]], // polygonise box (meters). Must contain every part with ~2 cm spare
  kN: 0.1,   // neck blend: head ↔ torso
  k3: 0.03,  // blend: head ↔ head-extra (G[3])
  k4: 0.012, // blend: head ↔ nose (G[4])
  kE: 0.03,  // blend: head ↔ ears
  kT: 0.03,  // blend: body ↔ tail
  // scarf: true,      // optional: sculpt the fox's knitted scarf into the body
  // earsOnHead: true, // optional: skin the ears to Head instead of the ear bones (human-style ears)
  // human: true,      // optional: use the skin material and enable hair

  head(hx, hy, hz) { /* write G[2], G[3], G[4] in HEAD-local coords */ },
  ears()           { /* write G[5], G[6], G[15], G[16] */ },
  tail(lx, ly, lz) { /* write G[11], G[12] in TAIL-local coords */ },
  paint(c, x, y, z){ /* set THREE.Color c for this vertex */ },
  face: () => ({ /* eye, mouth and extras description */ }),
  hat: { p: [0, 0.375, 0.03], r: [0.1, 0, 0.06], s: 1 }, // HatSocket pose on Head
},
```

Then add the species to the exported `SPECIES` list so the mirror UI shows it:

```js
{ key: 'penguin', label: 'Penguin', swatch: '#2e2b25', swatch2: '#f9f4ed' },
```

The `key` must match the `DEFS` key.

### 4.1 `head(hx, hy, hz)`, the head sculpt

Coordinates are **Head-bone local**: the neck is at the origin, the face points to +z, and the crown is around y ≈ 0.38–0.42.

Reference proportions (the existing species):

```js
// Main skull: wide, slightly squashed sphere
const skull = sdEll(hx, hy - 0.18, hz, 0.24, 0.2, 0.2);
// Jaw/cheek mass, lower and forward: blends the face into a soft "bean"
const jaw   = sdEll(hx, hy - 0.10, hz - 0.04, 0.2, 0.1, 0.16);
G[2] = smin(skull, jaw, 0.06);
```

Patterns used:

- **Snout / muzzle** (fox, bear): an extra ellipsoid pushed forward in z (the fox tilts it with a rotation; see `MC`/`MS`).
- **Whisker pads** (cat, bunny): two small ellipsoids at `x = ±0.03`, placed on the face with `faceZ()`:
  ```js
  const zc = faceZ(baseHead, 0.032, 0.103);  // surface z at that point
  G[3] = sdEll(Math.abs(hx) - 0.032, hy - 0.103, hz - (zc - 0.012), 0.042, 0.034, 0.03);
  ```
  Use `Math.abs(hx)` to mirror any part across both sides.
- **Cheek fluff** (fox): `sdCone` pointing sideways and outward from the cheek.
- **Nose** (`G[4]`): a small ellipsoid on the front surface, about 0.02–0.035 wide.

Expensive constants such as `faceZ` results should be computed once with the `memo()` helper (see `catZ`, `bunZ`, `bearZ`).

### 4.2 `ears()`, ears and inner-ear cups

The `boneEars(fn)` helper loops over both ears and gives you **ear-bone local** coordinates (the ear root is at the origin, and the ear grows along +y):

```js
ears: boneEars((s, lx, ly, lz) => {      // s = 0 for left, 1 for right
  const E   = sdEll(lx, ly - 0.035, lz, 0.066, 0.06, 0.033);         // the ear
  const cup = sdEll(lx, ly - 0.04, lz - 0.026, 0.04, 0.035, 0.014);  // hollow on the front
  G[5 + s]  = smax(E, -cup, 0.008);  // carve the hollow
  G[15 + s] = cup;                   // keep the cup so paint can tint it
}),
```

Shapes used so far:

- **Pointed** (fox, cat): `sdCone(… 0,-0.02,0 → 0,0.21,0, r 0.085 → 0.01)` flattened with `smax(…, Math.abs(lz) - 0.03, 0.03)`.
- **Long** (bunny): a tall flat ellipsoid, `0.056 × 0.19 × 0.03`.
- **Round** (bear): a small flat ellipsoid, `0.066 × 0.06 × 0.033`.
- **No ears**: set `G[5] = G[6] = G[15] = G[16] = INF`.

For ears that should not wiggle (head-attached, like the human's), write a custom `ears()` that reads `L[6..8]` (head-local), and set `earsOnHead: true`.

### 4.3 `tail(lx, ly, lz)`, the tail

Coordinates are **Tail-bone local**: the root is at the origin, and −z is backwards.

- **Swept tail** (fox, cat): use `pathTail(points, radiusFn)`, a curve through control points with a radius profile over `u ∈ [0, 1]`. It writes `G[11]` and `G[12]` for you.
  ```js
  tail: pathTail(
    [[0,0,0], [0.03,-0.04,-0.1], [0.08,0,-0.2], [0.11,0.12,-0.26], [0.1,0.27,-0.27], [0.05,0.38,-0.22]],
    u => 0.034 - 0.01 * u + 0.005 * Math.sin(Math.PI * u)      // slim cat tail
  ),
  ```
  Keep the tail **clear of the head** (stay behind z ≈ −0.2 above y ≈ 0.3), otherwise it fuses into the skull.
- **Pom / stub** (bunny, bear): a single sphere or ellipsoid, with `G[12] = 0` or `1`.
- **No tail**: `G[11] = INF; G[12] = 0;`.

### 4.4 `paint(c, x, y, z)`, colours and markings

This is called once per vertex **after** `sculpt()` has run on that vertex, so all `G` and `L` values are valid. Start from the base colour, then `c.lerp(colour, mask)` for each marking. Masks go from 0 to 1 and should be soft-edged.

Helpers available inside `buildCharacter`:

| Helper | Mask |
|---|---|
| `own(i)` | This vertex belongs to part `G[i]` (it's the closest part) |
| `bellyMask(rx, ry)` | Oval patch on the front of the torso |
| `pawMask(s)` / `footMask(s)` | The tips of arm / leg `s` |
| `innerEar(s)` | Inside the ear cup |
| `noseMask()` | The nose |
| `blushPaint(c, x, y, z, amount)` | Soft blush circles at the `face().blush` positions |
| `band(y, a, b)` | 1 between `a` and `b` (stripes) |

Example (penguin):

```js
paint(c, x, y, z) {
  const hx = L[6], hy = L[7], hz = L[8];
  c.copy(PEN.black);
  c.lerp(PEN.white, bellyMask(0.14, 0.17));                         // white belly
  c.lerp(PEN.white, own(2) * sstep(0.012, -0.012,                   // white face patch
         sdEll(hx, hy - 0.15, hz - 0.12, 0.15, 0.11, 0.12)));
  for (let s = 0; s < 2; s++) c.lerp(PEN.orange, footMask(s));      // orange feet
  c.lerp(PEN.orange, noseMask());                                   // beak
  blushPaint(c, x, y, z, 0.5);
},
```

Define colour constants next to the others (`const PEN = { black: C(0x2e2b25), white: C(0xf9f4ed), orange: C(0xe8913a) };`).

Tabby stripes: `sstep(0.55, 0.8, Math.sin(L[4] * 48 + 1))` on the torso, and `Math.sin(G[12] * 22)` along the tail.

### 4.5 `face()`, eyes, mouth and extras

The face is built from rigid meshes snapped onto the head surface. The function returns:

```js
face: () => ({
  eyes:  { x: 0.09, y: 0.2, style: 'round', k: 1, iris: 0x6b4a2e },
  mouth: { style: 'cat', y: 0.09, w: 0.03, philtrum: [0.117, 0.09] },
  blush: [0.14, 0.12],  // [|x|, y] on the head, mirrored
  whiskers: true,       // optional
  teeth: true,          // optional: bunny buck teeth
  brows: true,          // optional: coloured from the hair colour
}),
```

- **Eye `style`**: `'round'` (glossy ink), `'gem'` (coloured iris with a slit pupil; needs `iris`), `'sparkle'` (white, iris and lid; needs `iris`), `'happy'` (closed arcs), or `'lashes'`.
- **`eyes.x` / `eyes.y`**: head-local position of the left eye (it's mirrored). `k` scales the eye.
- **Mouth `style`**: `'cat'` (ω shape), `'smile'`, `'smirk'`, or `'open'`. `philtrum` draws the line from the nose down to the mouth: `[yTop, yBottom]`.

New eye or mouth styles go in `buildEye()` / `buildFace()`.

### 4.6 `hat`, the headwear seat

`{ p: [x, y, z], r: [rx, ry, rz], s: scale }` places `HatSocket` on the head so that the hat, chef toque, police cap and other headwear sit right. Put `p.y` at the top of the skull (around 0.37–0.47) and tilt it slightly back with `r[0]` ≈ 0.1.

---

## 5. Step-by-step: adding a character

1. **Add the swatch entry** to `SPECIES`.
2. **Add colour constants** next to `FOX`, `CAT`, `BUN`, `BEAR`.
3. **Add a `DEFS` entry** with `label`, `bounds`, the blends, `head`, `ears`, `tail`, `paint`, `face` and `hat`.
4. **Check `bounds`.** If parts are clipped flat, enlarge the box. If generation is slow, shrink it.
5. **Preview** in `Fox.html`: the character switcher is built from `SPECIES`. Orbit around it and check:
   - there are no holes or fused parts (tail into head, ears into each other),
   - the paint edges are soft and on the right parts,
   - the eyes sit on the surface (not sunk in or floating),
   - the clips Idle, Walk, Wave and Hop look right.
6. **Check it in the room**: open the mirror, pick the new character, and try the wardrobe outfits and headwear.

### Debugging tips

- **A part is missing or flat**: it's outside `bounds`, or you forgot to write its `G` slot.
- **Spiky artefacts or a torn surface**: a distance function is badly scaled. Avoid scaling ellipsoids to radii under 0.006 m, and keep features larger than about 2 × the grid step `H` (0.0075 m).
- **Parts fuse unexpectedly**: lower the relevant blend (`kN`, `k3`, `kE`, `kT`) or move the parts apart.
- **Paint bleeds onto a neighbouring part**: multiply the mask by `own(i)`.
- **Eyes float or sink**: tweak `face().eyes.y`. Eyes are projected onto the surface and inset by 0.013 m.
- **Generation time**: about 0.3–0.6 s per character. `tail()` with `pathTail` is the most expensive part, because it has a bounding-sphere early-out but still samples 28 segments near the tail.

---

## 6. Using the API

```js
import { buildCharacter, SPECIES, HUMAN } from './fox-model.js';

const ch = buildCharacter({ look: { species: 'cat' } });
scene.add(ch.fox);                        // the character root (named after the species)

ch.applyLook({ ...ch.look(), species: 'bear' });   // switch character (cached after the first build)
ch.prepare({ species: 'bunny' });                  // pre-build without switching
ch.needsBuild({ species: 'bunny' });               // → true if the first build is still pending

// Human customisation
ch.applyLook({ species: 'human', hair: 'buns', hairColor: '#e8a0b4',
               eyes: 'happy', nose: 'button', mouth: 'open', ears: 'elf', skin: '#9f6a45' });

// Outfits & headwear
ch.outfits;   // { sweater, police, chef, fire, … } → { parts: [meshes] }. Toggle .visible
ch.headwear;  // { … } → groups on HatSocket. Show one at a time
ch.hat;       // the felt hat (attach it to ch.hatSocket to wear it)

// Animation
const mixer  = new THREE.AnimationMixer(ch.fox);
const action = mixer.clipAction(THREE.AnimationClip.findByName(ch.fox.animations, 'Walk'));
action.play();
ch.resetPose();                           // back to the rest pose (use before exporting)
```

**Clips** (all characters): `Idle`, `Walk`, `Wave`, `Hop`, `Sit`, `Nudge`, `Reach`, `Twirl`, `Study`, `Lie`.
`Sit` and `Study` expect the root on the seat surface. `Lie` expects the root on the mattress.

### Adding a new animation clip

Clips are authored as functions of normalised time `t ∈ [0, 1]` that return **offsets from the rest pose**:

```js
clip('Dance', 1.2, {
  'Rig.pos':   t => [0, 0.04 * Math.abs(sin(t, 2)), 0],          // position offset (m)
  'Body.rot':  t => [0, 0.3 * sin(t), 0.1 * sin(t, 2)],           // Euler offset (rad)
  'Arm_L.rot': t => [0, 0, 1.2 + 0.4 * sin(t, 2)],
  'Arm_R.rot': t => [0, 0, -1.2 - 0.4 * sin(t, 2, 0.5)],
  'Eye_L.scale': t => [1, 1 - 0.9 * pulse(t, 0.5, 0.02), 1],      // blink
}),
```

Helpers: `sin(t, k, phase)`, `cos(…)`, `pulse(t, centre, width)`, `win(t, a, b)` (a smooth bump between a and b), and `hold(t, a, b, w)`. Make looping clips start and end on the same pose (use whole-number frequencies `k`).

---

## 7. Exporting a `.glb` for React Three Fiber

From any page with the character loaded:

```js
import { GLTFExporter } from 'three/addons/exporters/GLTFExporter.js';

ch.resetPose();
ch.fox.position.set(0, 0, 0); ch.fox.rotation.set(0, 0, 0);
ch.fox.updateMatrixWorld(true);
const buf = await new GLTFExporter().parseAsync(ch.fox, { binary: true, animations: ch.fox.animations });
const url = URL.createObjectURL(new Blob([buf], { type: 'model/gltf-binary' }));
Object.assign(document.createElement('a'), { href: url, download: 'penguin.glb' }).click();
```

(The **Download GLB** button in `Fox Room.html` does exactly this.)

### Using it in React Three Fiber

```jsx
import { useRef, useEffect } from 'react';
import { useGLTF, useAnimations } from '@react-three/drei';

export function Character({ url = '/penguin.glb', clip = 'Idle', outfit, ...props }) {
  const ref = useRef();
  const { scene, animations } = useGLTF(url);
  const { actions } = useAnimations(animations, ref);

  useEffect(() => {
    actions[clip]?.reset().fadeIn(0.3).play();
    return () => actions[clip]?.fadeOut(0.3);
  }, [clip, actions]);

  // Optional: show/hide named parts, e.g. scene.getObjectByName('Sweater').visible = false
  return <group ref={ref} {...props}><primitive object={scene} /></group>;
}
```

The GLB contains:
- one skinned body mesh with vertex colours (`CharacterBody`),
- the outfit meshes that were visible at export, all skinned to the same skeleton,
- the rigid eyes, mouth and headwear under `Head`,
- the 10-bone skeleton with the names in section 2,
- all 10 animation clips.

Vertex colours are used instead of textures, so make sure your material keeps `vertexColors: true`. `useGLTF` handles this automatically.

---

## 8. Prompt template

Paste this to an AI assistant along with `fox-model.js` and this guide:

> Add a new character to `fox-model.js` following `CHARACTER_GUIDE.md`.
> - **Species:** _(e.g. penguin)_
> - **Silhouette:** _(e.g. round egg body, no visible neck, tiny flipper arms)_
> - **Ears:** _(none / pointed / round / long / custom)_
> - **Tail:** _(none / stub / swept / pom)_
> - **Colours & markings:** _(e.g. black body, white belly and face patch, orange beak and feet)_
> - **Face:** _(eye style, mouth style, extras like whiskers or teeth)_
>
> Keep the shared rig and every `G` slot contract. Add the `SPECIES` entry, a `DEFS` entry and colour constants. Don't change other characters. Then check it in `Fox.html`.
