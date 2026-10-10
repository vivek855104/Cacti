# Azure Bay Resort — 3D viewer. Project handoff

Written for a fresh Claude Code session in VS Code. Everything below is current as of
10 Oct 2026. Read this first, then open `azure-bay-local.html`.

---

## 1. What this is

A 3D resort viewer for hotel/resort websites. A prospective guest sees a drone view of
the whole property, taps a building, gets a room list, taps a room, and walks into a
photoreal-ish 3D interior with a hotel-room info panel beside it (photos, price,
amenities, reviews, booking CTA).

Built as a demo/prototype for a product that would be sold to real resort clients, with
their actual property replicated. Not yet client work.

### Navigation model — a three-state machine

```
resort  ──tap building──▶  building  ──tap room──▶  room
   ▲                          │                      │
   └──────── Back ────────────┴──── Back ────────────┘
```

- **resort** — orbiting drone camera, auto-rotates until touched, resumes after 7s.
  HTML labels float over each building. Raycast click also works on the geometry.
- **building** — camera flies to a 3/4 view; side panel lists that building's rooms.
- **room** — first-person look-around (camera locked in place, orbit radius 0.012),
  six viewpoint chips (Entry / Bed / Lounge / Window / Balcony / Bath) that tween the
  camera. Side panel shows the room detail page.

---

## 2. Hard constraints (these drove most decisions)

1. **It must be hostable and light.** This is why there are no downloaded textures and
   no authored models in the core build. Every material is a procedural canvas texture
   generated at load; every object is code-built geometry. Core file ≈ 72 KB of HTML
   plus three.js from CDN. **Do not casually add asset downloads.**
2. **Blender is not in the pipeline.** We started down a Blender route (there's a
   leftover `resort_room_deluxe.py`), but the Blender MCP bridge never worked reliably
   on the user's machine (Blender 5.2.2 LTS, add-on "MCP for Blender"; connection
   accepted one request then dropped, repeated 4-minute timeouts). We pivoted to doing
   everything in three.js. Don't reintroduce a Blender dependency without asking.
3. **Published-artifact CSP.** The claude.ai-hosted copy can only load scripts from a
   few CDNs — no local models, no remote images. That's why there are two builds
   (see §3).

---

## 3. Files

| File | What it is |
|---|---|
| `azure-bay-local.html` | **The main build.** Full resort + rooms, entirely procedural. Rename to `index.html`. |
| `room-deluxe.html` | Earlier standalone room-only scene. Superseded; keep for reference. |
| `resort3d.html` | The very first rough prototype (three r128, flat colours). Historical. |
| `resort_room_deluxe.py` | Blender build script for the deluxe room. Unused after the pivot. |

**Keep `azure-bay-local.html` and `azure-bay.html` in sync.** Changes so far have been
applied to both via scripted string replacement. If that gets annoying, collapse to one
file and delete the hosted copy.

### Local setup

```
Resort/
  index.html          (azure-bay-local.html renamed)
```

```powershell
cd "C:\Users\vivek\Downloads\Resort"
npx serve .
```

Must be served over HTTP. `file://` blocks ES modules and model loading.

---

## 4. Architecture of the main file

Single HTML file, one `<script type="module">`, importmap pointing at
`three@0.170.0` on jsDelivr. Addons used: OrbitControls, RoundedBoxGeometry,
RoomEnvironment, RectAreaLightUniformsLib, BufferGeometryUtils, EffectComposer,
RenderPass, UnrealBloomPass, ShaderPass, OutputPass.

Sections, in source order (line numbers approximate):

| Lines | Section |
|---|---|
| 141 | `data` — `TYPES` (deluxe/suite/villa) and `BUILDINGS` arrays |
| 192 | `progress` — loading-bar `step()` helper |
| 197 | `texture lab` — noise + all procedural texture generators |
| 365 | `renderer` — WebGLRenderer, `LOW` tier detection, camera, controls |
| 382 | `materials` — `buildMaterials()`, async, fills the `M` object |
| 500 | `build helpers` — `bx` `cy` `sph` `plane` `instance` `crumple` |
| 527 | `scene: resort` — geometry builders + `buildResort()` |
| 884 | `scene: room` — `buildRoom(type)`, built lazily on first entry |
| 1080 | `post` — composer and the film/vignette/saturation shader |
| 1105 | `state machine` — `applyLight`, camera flights, mode switching |
| 1225 | `panel` — HTML side-panel rendering |
| ~1300 | resize, label projection, render loop, `boot()` |

### Key conventions

- **Coordinates:** X across, Z depth, Y up. Room interior is
  `x ∈ [-2.3, 2.3]`, `z ∈ [-3.5, 3.5]`, `y ∈ [0, 2.9]` — 4.6 × 7.0 × 2.9 m = 32 m².
  Window/balcony at `+z`, entry door at `-z`.
- **Two scenes, one camera.** `resort` and `room` are separate `THREE.Scene`s.
  Switching sets `renderPass.scene` and the `active` variable. The room scene is only
  constructed on first entry and cached in `room`/`roomType`.
- **`LOW` tier** (`max-width:860px` or ≤4 cores): no post-processing, 512px textures,
  1024 shadow maps, glass falls back to cheap transparency instead of transmission.
- **Instancing is the perf strategy.** Palms, trees, windows, balustrades, loungers,
  parasols, hedges, lamps all go through `instance()`. Keep it that way.

### The texture lab

All tileable, all generated on `<canvas>` at load, all with matching normal maps
derived by Sobel from a height canvas (`normalFrom`).

`wood` (planks + grain), `plaster`, `marble` (domain-warped veins), `weave` (fabric),
`tiles`, `roofTiles` (barrel clay), `speckle` (lawn/foliage), `caustics`, `ripple`,
`artwork`, `hillBand`.

Sizes: 1024 on desktop, 512 on `LOW`. Generation takes ~1–2 s, reported via the
loading bar.

---

## 5. What's in the scene

**Resort:** infinity pool (transparent water, animated caustics on the floor, entry
steps, foam lines), teak deck, loungers + parasols, three building types, hedged paths,
garden lamps, instanced palms (33) and Quaternius trees, beach, sea, distant hills.

**Buildings** (`BUILDINGS` array, each with `id`, `pos`, `foot`, `floors`, `bays`,
`rooms`):
- `A` Lakeview Wing — 3 floors, 6 rooms
- `B` Garden Suites — 2 floors, 4 rooms
- `R` Reception & Spa — `lobby:true`, no rooms, glass entrance + canopy
- `BE` The Beach — `beach:true`, no rooms, shack + loungers + catamaran
- `C` Palm Villas — `villas:true`, 3 villas with walled plunge pools

**Room interior:** oak floor with clearcoat, plaster walls, skirting + picture rail,
cove lighting, full-height glazing with folded sheer curtains, balcony with furniture,
bathroom pod (marble vanity, mirror, rain shower, WC, towels, amenity bottles), bed with
upholstered channel-tufted headboard, noise-crumpled duvet/pillows/throw, nightstands
with lamps, TV console with tray and cups, desk with laptop, lounge chair, luggage
bench, planter, artworks.

**Room type variation is currently shallow:** all three types share one layout; only the
accent fabric colour, rug tint and (for villa) an extra plunge pool outside differ.
**This is the most obvious next improvement** — the suite should get a real lounge area
and the villa a different footprint.

**Lighting:** a time-of-day slider (top bar) runs Day → Evening → Night. `applyLight(k)` blends three keyframes (`KF`) for sky, fog, sun, ambient, lamps, windows, pool, stars and exposure. Starts on Evening (k=.5).

---

## 6. Landscaping (procedural, no downloaded assets)

The Quaternius nature kit and `nature-kit.js` were removed (10 Oct 2026). Vegetation is now
composed in `plantGardens()`:

- Shapes (all merged once, instanced everywhere): shrub, blossom dots, ornamental-grass clump,
  tropical-leaf clump, round / flat-topped / columnar trees, palms, rocks.
- Composition: `rectBed()` (limestone-edged borders), `island()` (round planting islands with a
  palm / tree / tropical centrepiece), a palm allee beside the pool terrace, and a far tree belt of
  loose clusters outside the grounds. Tall planting is kept away from building facades.
- Colour comes from per-instance colours on shared foliage materials (`M.shrub`, `M.tropLeaf`,
  `M.crownRound`…), which darken toward the base via `onBeforeCompile`. Palette: emerald, lime,
  white, coral, lavender, pink.
- `LOW` tier thins beds and the tree belt (`D` factor, fewer clusters).

## 7. Bugs already found and fixed (don't reintroduce)

1. **Pool was invisible.** Water plane sat at `y=0`, terrace paving at `y=0.03` — paving
   drew over it. Water now at `y=0.09`, floor at `-0.42`, basin deeper.
2. **Palms looked like agaves.** They were 8 flat crossed `PlaneGeometry` blades. Now a
   ring-by-ring tapered curved trunk + 13 drooping V-section fronds (3 short upright
   young ones) + crown shaft + coconuts, all merged into 2 instanced meshes.
3. **`stars` identifier collision.** The night-sky variable clashed with the
   star-rating helper `const stars = r => '★'...`. Module failed to parse; nothing
   rendered at all. Renamed to `starField` / `moonMesh`.
   **Lesson: syntax-check before handing over.**
   ```bash
   # extract the module and check it
   node --check extracted.mjs
   ```
4. **Grey distance read as pollution.** Fog was `0xbcd6e4` at 120–460. Now `0xe2eff2`
   at 210–680, and the hill billboard went pale atmospheric blue.

---

## 7b. Art-direction pass (10 Oct 2026)

- Pool is now a real hole: `slab()` cuts it out of the lawn, paving and teak deck; basin walls,
  floor and water sit inside it. (Before, the lawn plane covered the water.)
- Late-afternoon light: sun direction `SUN_DIR[0]`, ACES tone mapping in the resort, AgX kept for
  rooms (`applyTone()`), sky-and-ground environment map (`skyEnvScene()`), cyan sky, far-only haze.
- Day/night is a slider, `applyLight(k)`, k in 0..1 (day, evening, night keyframes).
- Architecture: limestone base, shutters, pergolas, sand/terracotta roofs, per-building palette.
- Amenities: fountain at the path crossing, dining deck (east) and pergola lounge (west) by
  reception, two garden corners, cream parasols around the pool, low bollards.
- UI: ivory/gold labels, time-of-day slider. `?dev` in the URL exposes `window.__dbg` for testing.

## 7c. Room pass (10 Oct 2026)

- `buildRoom()` was rewritten in place (same footprint, furniture positions and camera views). New helpers:
  `pillowGeo()` (inflated cushions), `drapeGeo()` (duvet / sheet / runner draped over the mattress),
  `contactMat()` (soft contact shadows), `poolHole()` and `roomExterior()`.
- The window view is the resort's own pool terrace, palms, gardens and sea, built with `gardenKit(S,Y,seed)`
  (the same kit `plantGardens()` uses). Hills billboard and fog give atmospheric depth.
- Lighting: sun 3.0, RectAreaLight sky fill 1.8, ceiling lamps ~1.1, AgX exposure 1.0 (was 1.42, which washed
  everything out). The room follows the time-of-day slider through the `r*` keys in `KF`
  (`S.userData.apply`); lamp shades glow at evening and night.
- Bathroom mirror is a `Reflector` (skipped on the `LOW` tier, where a warm gradient mirror is used).
- `disposeRoom()` frees geometries, owned materials/textures and the reflector target when the room type changes.
- Colour tone per room type (`tone` in `buildRoom`) drives headboard, runner, cushions and upholstery.

## 7d. Beach (10 Oct 2026)

- Before, the 900x900 lawn plane covered the sand and sea, so there was no visible beach. The lawn now ends at the
  dunes (z=84); `buildShore()` adds sand, a damp band, vertex-coloured shallows that deepen to blue (waterline z=118),
  three animated surf foam lines (`animateShore`), a teak pier at x=26, dune planting and a distant headland.
- `RESORT_CAM` now looks south-east from the north-west so the beach is in the default frame.

## 7e. UI redesign (10 Oct 2026)

- Frosted-glass interface over a full-screen canvas: `.glass` tokens in `:root`, with `body.night` (set by `applyLight`
  when the slider passes .65) switching the glass to a smoky warm style. Works without `backdrop-filter` (opaque fallback).
- The info panel floats at the right (`#panel`, collapsible via `#pclose` / `#popen`; bottom sheet under 860px via `#grab`).
  The camera uses `setViewOffset` (`shift` / `shiftTo`) so the 3D view stays centred in the clear area when the panel is open.
- Panel content is built by `resortPanel` / `buildingPanel` / `roomPanel` (items are `.item[data-b|data-room]`;
  amenity icons come from `IC` / `AMI`). Hover or focus on a list item, a label or the building itself lights all three
  (`setHover`, gold ground outline from `ringMesh`).
- Time-of-day control shows its mode (Daylight / Golden hour / Night). Viewpoint toolbar is a bottom-centre glass bar.
- `TYPES[*].sleeps` now holds the capacity text that used to be hardcoded in the panel.

## 7f. Performance and camera pass (10 Oct 2026)

- Shadows are static (`renderer.shadowMap.autoUpdate=false`); `renderer.shadowMap.needsUpdate=true` after anything that moves the sun or swaps the scene.
- Resort night lights cut from 21 to 8 (4 on `LOW`); lamp heads/bollards still glow. Room rig is lighter on phones, mirror only redraws when the view or light changes.
- Shader warm-up at load (`compileScene`, lamps off/on, stars, hover outlines, first room), so dragging the slider or entering a room does not stall.
- Adaptive quality (`Q`, `qualityTick`): lowers pixel ratio first, then drops glass blur (`body.lite`), then bloom; raises back cautiously. Bloom runs at half resolution and only when something glows.
- Camera: `fly()` orbits in spherical coordinates (curved path, log-zoom, shortest turn, duration from distance); room viewpoints use a quaternion slerp (`kind:'look'`); both use real elapsed time and a quintic ease. Entering/leaving a room is a quick fade (`fadeSwap`, ~0.2 s) with the build and shader compile hidden inside it.
- Labels move with the CSS `translate` property and only when the camera changed (`placeLabels`).
- `?perf` shows an fps / pixel-ratio / draw-call overlay; `?nointro` skips the opening swoop.

## 8. Open work, roughly in priority order

1. **Distinct layouts per room type.** Suite needs a lounge; villa needs its own
   footprint. Currently colour-swap only.
2. **Real photos in the room panel.** The panel has no gallery yet — the earlier
   `resort3d.html` had placeholder tiles; the premium build dropped them.
3. **Deploy pipeline.** Vercel. No heavy assets now, so nothing to optimize first.
4. **Data layer.** `TYPES` and `BUILDINGS` are hardcoded consts. Move to Supabase or a
   CMS. Booking via Razorpay was discussed, not built.
5. **Lightmap baking** for real GI. Options: `three-gpu-pathtracer` in a one-off browser
   page, or back to Blender. Would close the remaining gap to archviz quality.
6. **Mobile pass.** `LOW` tier exists but has never been tested on a real phone.
7. **Loading UX.** Currently a bar with cute status lines; fine, but first paint is
   gated on ~1–2 s of texture generation that could be deferred.

---

## 9. Client-replication playbook (discussed, not built)

For doing a real resort, we settled on three tiers:

1. **Stylized** — from drawings and photos only, no site visit. Fast, approximate.
2. **Accurate** — drone photogrammetry (200–500 overlapping photos → RealityCapture /
   Metashape / Polycam) + a Matterport or LiDAR scan per room type, modelled over the
   scans.
3. **Digital twin** — the above plus baked Cycles lighting, 360° renders per viewpoint,
   day/night variants.

**Collect from the client:** site plan and master layout (CAD/PDF), floor plans and
elevations, room-type schedule, interior drawings, material/finish list, furniture
vendors, brand assets, existing photos/drone footage, landscape plan, and the room data
for the panel (prices, amenities, reviews, booking system access).

**Accuracy method:** import CAD into the modelling tool and build on top of it; overlay
the model on the scan mesh and check deviation; compare renders against photos from the
same camera angle; client sign-off at massing → materials → final. Quote a tolerance up
front (±5 cm interiors, ±30 cm exteriors).

**India-specific:** DGCA/Digital Sky rules and no-fly zones for drone capture, written
client permission, DPDP-compliant handling if guest reviews or personal data appear.

**Contract points:** scope, revision count, asset ownership, publishing permission,
maintenance plan (renovations outdate the model).

---

## 10. Working notes for the next session

- The user is on Windows, Blender 5.2.2 LTS installed, Node available via `npx`.
- Folder names with brackets break PowerShell `-Path`; use `-LiteralPath`.
- When something doesn't render, ask for **F12 → Console, first red line**. A failure
  before the loading screen clears is a parse error; after, it's a runtime fault in a
  specific builder.
- The user wants premium/polished output and gives direct visual feedback
  ("looks dull", "palms look bad"). Act on it concretely rather than explaining.
- Preference: concise responses, minimal padding.