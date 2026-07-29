# Godot 3D Space Game — Build Plan

**Working title:** TBD
**Engine:** Godot 4.6, custom build (`precision=double`) + Voxel Tools 1.6 module
**Target:** Solo/small-team vertical slice — 4 planets, seamless space-to-surface, voxel terrain you can dig, modular ships, resource-driven crafting.

> This document is the plan only. No code has been written yet. The Godot project will live in a **new repository** (decision D3 below) — this file stays here as the design record.

---

## 0. Decisions already locked

| # | Decision | Choice | Consequence |
|---|---|---|---|
| D1 | Terrain look | **Hybrid** — fine voxels (0.25 m target), Transvoxel smooth mesh, low-poly faceted shading | Reads as "stylized voxel" *and* as Valheim ground. Costs ~8× the voxel data of 0.5 m. Fallback to 0.5 m is pre-authorised (see §7 R2). |
| D2 | Space travel | **Fully seamless** (No Man's Sky style) | Forces double-precision engine build + custom export templates + a representation ladder. This is the single largest cost driver in the plan. |
| D3 | Project home | **New repository** | Godot 3D shares nothing with this repo's 2D canvas game. New repo gets Git LFS, Godot `.gitignore`, and a CI engine-build pipeline. |

### What D2 actually costs, stated plainly

Choosing seamless roughly triples the engineering risk versus a cinematic transition. Concretely it buys these three obligations:

1. **We compile the engine.** Voxel Tools' `double` variant is a module build, and upstream does not publish export templates for it. Every platform we ship on needs a Godot+module compile we own. That is M0, and it never fully goes away — every engine or module bump re-runs it.
2. **LOD must stitch at every scale**, not just one. There is no scripted burn to hide a pop.
3. **Precision bugs are a permanent tax.** They show up as jitter, z-fighting, and physics tunnelling at distance, and they are diagnosed far from where they're caused.

The plan below front-loads all three into M0–M1 and M3, with explicit pass/fail gates, so we learn whether seamless is affordable in **week 3**, not month six. If gate G3 fails, §7 R1 has a fallback that preserves most of the feel without the engine build.

---

## 1. Architecture at a glance

```
SolarSystem (Node3D, double-precision world)
├── Star (DirectionalLight3D re-aimed per frame + billboard)
├── PlanetBody × 4          ← data-driven from PlanetProfile.tres
│   ├── FarSphere           ← MeshInstance3D, procedural shader, always on
│   ├── Atmosphere          ← inverted sphere, scattering shader
│   └── VoxelLodTerrain     ← INSTANTIATED ON APPROACH ONLY (one at a time)
│       ├── VoxelGeneratorGraph   (the density field)
│       ├── VoxelStreamSQLite     (player edits, per planet)
│       └── VoxelInstancer        (trees / rocks / crystals)
├── ShipRoot (RigidBody3D)  ← assembled from ShipGrid at runtime
└── Player (CharacterBody3D)← spherical gravity, re-oriented per frame
```

Four systems, in dependency order. Everything else hangs off these.

1. **Voxel planet** — terrain generation, meshing, digging, persistence.
2. **Scale ladder** — how a planet is represented from 500 km out to standing on it.
3. **Ships** — modular grid assembly + 6DOF flight + the gravity-well handoff.
4. **Crafting** — items, inventory, recipes, and the per-planet resource tables that make travel worth doing.

---

## 2. The voxel planet (the core; everything else is downstream)

### 2.1 Why Voxel Tools and not a custom engine

Zylann's `godot_voxel` is the only mature option, and it already ships the exact feature set: Transvoxel SDF meshing, an octree LOD terrain, threaded generation/meshing, `VoxelTool` edit API, SQLite persistence, and a foliage instancer with LOD. Writing Transvoxel + octree LOD + threaded streaming from scratch is a multi-year project on its own, and GDScript cannot carry the hot path at all — it would mean a Rust/C++ GDExtension, i.e. rebuilding this library worse.

**We use the module build, not the GDExtension build**, because D2 requires the `double` variant and that variant is module-only.

### 2.2 Terrain shape — "Valheim-like" decomposed

Valheim's ground is *not* voxel — it's a biome-driven heightmap with a modifier list, which is why it has no caves or overhangs. We want its **feel** on voxel data, which strictly dominates:

- **Rolling, readable hills** — low-frequency FBM sets the base, so silhouettes are legible from distance.
- **Hard biome borders** — biomes are a separate low-frequency mask, not a smooth gradient. Valheim's crisp Meadows→Black Forest line is a deliberate look and it's what makes a planet feel like a place.
- **Ridged fractal mountains** — negated ridged noise for eroded cliff faces.
- **Caves and overhangs** — 3D worley/FBM subtracted from the SDF. This is the part heightmaps can't do and voxels get free.
- **Deformable ground** — pickaxe/shovel, radius brush, stamina-gated.

Composed as an SDF, evaluated in `VoxelGeneratorGraph`:

```
sdf = SdfSphere(radius)                    # the planet
    + fbm3d(p_normalized) * relief_amp     # continents & hills
    + ridged(p) * mountain_mask            # eroded peaks
    - cave_worley(p) * cave_mask           # caves, overhangs
```

Applying noise as *elevation projected onto the sphere normal* rather than as a raw SDF offset — otherwise high amplitude produces floating chunks (a documented failure mode of the naive approach).

**One rule that must not be broken:** the density field is authored **once**. Both the CPU generator and the far-field GPU shader (§3) evaluate the same function. A unit test samples both at N random points and asserts agreement within tolerance. If these two drift, the seamless crossfade tears, and it will tear *at range*, where it's hardest to debug.

### 2.3 Digging

`VoxelTool.do_sphere()` with a negative SDF value carves; positive fills. Three tools:

| Tool | Effect | Notes |
|---|---|---|
| Pickaxe | Carve sphere, r ≈ 0.75 m | Returns material at the dug voxels → inventory |
| Shovel | Carve/deposit soil, r ≈ 1.5 m | Deposit consumes carried soil (conservation matters for base-building) |
| Hoe | Flatten toward a plane | Valheim's levelling tool; snaps to a horizontal-relative-to-local-up plane |

Edits go to a `VoxelStreamSQLite` per planet, which stores **only modified blocks** — an untouched planet costs zero bytes. A per-planet modified-block budget with a soft warning keeps a player from carving a 3 km trench and destroying their own framerate.

**Material read-before-write**: `VoxelTool` reads the INDICES/WEIGHTS channels at the dig site *before* removing, to determine yield. That's the hook into crafting (§5.3).

### 2.4 Texturing

The SDF channel carries shape; INDICES/WEIGHTS carry material (4 blended textures per voxel out of 16). Biome material is written by the generator, so a mined cliff face exposes rock strata rather than grass. Surface shader is triplanar (no UVs on marching-cubes output) with a faceted-normal pass for the D1 look.

### 2.5 Foliage

`VoxelInstancer` scatters trees/rocks/crystals on voxel surfaces with its own LOD, and — critically — updates when terrain is edited, so digging out from under a tree removes it. Per-biome instance sets come from `PlanetProfile`.

---

## 3. The seamless scale ladder (the risky part)

### 3.1 The insight that makes this tractable

The naive fear is that seamless requires crossfading voxel terrain against something else *at ground level*, where any mismatch is glaring. It doesn't. Run the numbers:

- `VoxelLodTerrain` LOD level *L* covers `16 × 2^L` voxels. At 0.25 m/voxel that's `4 × 2^L` metres.
- A 3 km-radius planet is 6 km across. `4 × 2^L ≥ 6000` → **L = 11**, i.e. ~12 LOD levels covers the entire planet in one octree.

So the voxel terrain itself already spans from a 0.25 m pebble to the whole planet. It doesn't need help until we're well off the surface. That moves the crossfade up to ~20 km altitude, where max terrain relief (~1 km) subtends a tiny fraction of the planet's visual size and the silhouette is a smooth sphere anyway. **A mismatch there is subpixel.**

> Verify `lod_count`'s practical ceiling in the M0 spike. The math needs 12; the property allows far more, but memory and octree update cost at high LOD counts are measured, not assumed.

### 3.2 The ladder

| Band | Altitude | Representation | Editable |
|---|---|---|---|
| **L0 Surface** | 0 – 20 km | `VoxelLodTerrain`, full octree, instancer foliage, physics | Yes |
| **L1 Crossfade** | 15 – 25 km | Both, alpha-blended on altitude | Yes |
| **L2 Orbital** | 20 km – 1000 km | Quad-sphere mesh, same density field on GPU, atmosphere shader | No |
| **L3 Distant** | > 1000 km | Sphere impostor, no displacement, lit dot | No |

Rules that make it seamless rather than merely fast:

- **`VoxelLodTerrain` is instantiated at 50 km on approach** — 30 km *outside* the crossfade band. Streaming and meshing warm up entirely off-screen. By the time the player can resolve terrain, LOD0 is resident.
- **Exactly one planet has a `VoxelLodTerrain` at a time.** Others are L2/L3. Approaching a second planet before leaving the first is the edge case; handle it by allowing two and evicting on distance hysteresis.
- **No scene loads, ever.** Nodes are added/removed from a persistent `SolarSystem`. There is no `change_scene`, so there is nothing to hide.
- Deleting a terrain flushes its SQLite stream first. Losing a player's excavation on planet exit would be a brutal bug to ship.

### 3.3 Precision

The double build makes positions safe to ~9 quadrillion units; rendering uses Godot's camera-relative emulated-double path (shaders stay single-precision — that's fine, they only ever see camera-relative coordinates).

The structural detail that matters: **voxel coordinates are integers local to the terrain node**, which sits at the planet's centre. Voxel data therefore never sees large world coordinates — only the node's transform is huge. That's exactly the right shape for a double build, and it's why floating origin is *not* needed here. Dropping floating origin is the one real simplification D2 buys us; it removes an entire class of "which frame is this vector in" bugs.

---

## 4. Ships

### 4.1 Data model

- `ShipPart` (Resource): id, mesh, collision shape, mass, connector mask, category, power draw/gen, thrust N + direction, fuel rate, cargo volume.
- `ShipGrid` (Resource): `Dictionary[Vector3i → {part_id, rotation}]`, 1 m cells. This *is* the save format for blueprints.

### 4.2 Assembly

On any grid change, rebuild:
- **Visuals** — one `MultiMeshInstance3D` per part type. A 400-part ship is then a handful of draw calls, not 400.
- **Collision** — merged compound shape, boxes per contiguous run rather than per part.
- **Physics** — mass = Σmᵢ; centre of mass = Σ(mᵢpᵢ)/Σmᵢ set via `CUSTOM` mode; inertia tensor approximated from point masses and set directly.
- **Validation** — flood fill from the cockpit. Disconnected parts are rejected at build time, not silently ignored at flight time.

### 4.3 Flight

Sum thrusters into available force/torque per axis, then a control allocator maps 6DOF stick input → per-thruster throttle (greedy, weighted by alignment; least-squares if greedy proves inadequate). Flight assist damps residual velocity and is toggleable.

Because thrust is derived from actual placed thrusters, **a badly built ship flies badly** — under-thrust, off-centre torque, tumbling. That's the design intent, so the build UI must surface thrust-per-axis and CoM offset as live readouts while building. Without that feedback it reads as a bug rather than a consequence.

### 4.4 Gravity-well handoff

Ship physics switches between two integrators on a hysteresis band:
- **In-well** — planet gravity, atmospheric drag, aerodynamic-ish handling.
- **Out-of-well** — pure Newtonian, no drag.

Landing gear contact + near-zero velocity → "landed" state, which anchors the ship to the planet's frame and enables disembarking.

### 4.5 Explicit v1 scope cut

**Ships are piloted from a seat; you cannot walk around inside a moving ship.** Walking inside a moving reference frame is a genuinely hard, separate problem (interior as a local-space subscene with its own physics world, plus a transition at the airlock). It is not in this plan. Building it in later is possible — the ship interior would become its own zone — but it is a project-sized feature, not a milestone. Ship building happens landed, in a hangar/build mode.

---

## 5. Crafting & resources

### 5.1 Data model

- `ItemDef` (Resource): id, name, icon, stack size, **mass**, tags.
- `Recipe` (Resource): inputs[], outputs[], required station tag, craft time, unlock condition.
- `Inventory`: slot array, **mass-limited** — this is what makes cargo capacity a real ship design constraint rather than a number on a screen.
- Stations: Fabricator, Smelter, Ship Assembler. Each accepts recipes by tag.

### 5.2 Where materials come from

Three sources, deliberately different in feel:

1. **Surface props** (`VoxelInstancer`) — trees, boulders, crystal spires. Walk-up harvest. Early game, no digging required.
2. **Volumetric ore veins** — a deterministic 3D field per planet (thresholded worley), evaluated at the dig position. **Not stored** — it's a pure function of position and seed, so veins cost zero bytes and are identical for every player on that seed. Digging into one yields ore.
3. **Voxel material channel** — the INDICES/WEIGHTS read from §2.3 gives base materials (soil, stone, ice, obsidian).

### 5.3 The loop that makes travel matter

Each `PlanetProfile` carries a resource table, and **the tables are deliberately incomplete**. This is the actual game design load-bearing decision:

| Planet | Gravity | Hazard | Exclusive resource | Gates |
|---|---|---|---|---|
| **Terran** | 9.8 | None | Organics, common metals | Starting tech |
| **Moon** | 1.6 | Vacuum (O₂) | Rare metals, He-3 | Better reactors, suit O₂ tier 1 |
| **Ice** | 6.0 | Cold | Volatiles, water ice | **Fuel** — required for interplanetary range |
| **Fire** | 12.0 | Heat + ash | Obsidian, sulfur, high-temp alloys | Heat shielding, best hull parts |

The dependency chain is the progression: *Terran gets you to the Moon → Moon reactors get you to Ice → Ice fuel gets you to Fire → Fire alloys get you everything else.* Hazards drive suit upgrades, suit upgrades need off-world materials, and that's the loop. Without this table the four planets are reskins.

---

## 6. Milestones

Ordered to fail fast. The riskiest unknowns are M0/M1/M3, and all three land inside the first ~10 weeks.

| # | Milestone | Est. | Definition of done |
|---|---|---|---|
| **M0** | **Toolchain spike** | 1–2 wk | Custom Godot 4.6 `precision=double` + Voxel Tools 1.6 module compiles. CI produces editor + export templates for Linux & Windows, cached. Voxel demo scene runs in the double build. |
| **M1** | **Voxel planet vertical slice** | 2–3 wk | One 3 km planet. Spherical gravity character. Dig a cave with a pickaxe, quit, reload, cave is still there. 60 fps. **Gate G1.** |
| **M2** | **Look dev — Valheim pass** | 2 wk | Triplanar splat + faceted shading, biome materials, instancer foliage, sky/fog/lighting. Screenshot reads as stylized-Valheim. |
| **M3** | **Seamless ladder** | 3–4 wk | Fly from 500 km out to standing on the ground. No cut, no pop, no jitter. **Gate G3 — the make-or-break.** |
| **M4** | **Ships v1** | 3 wk | Build a ship in the hangar, take off, reach orbit, land elsewhere on the same planet. |
| **M5** | **Solar system** | 2 wk | 4 planets from `PlanetProfile`. Star map. Terran surface → Moon surface, seamless, both directions. |
| **M6** | **Resources & crafting** | 3 wk | Mine on the Moon, craft a part unobtainable on Terran, install it on the ship. |
| **M7** | **Survival loop** | 2 wk | Fire planet kills you without the suit tier that requires Ice volatiles. Loop closed. |
| **M8** | **Polish** | ongoing | Audio, UI, save/load, settings, perf pass, first playable build. |

**~5–6 months** of focused work to a rough but genuinely playable vertical slice. That estimate assumes M3 passes on the first architecture; if it doesn't, add 3–4 weeks for the R1 fallback.

### Gates

- **G1 (end of M1)** — dig + persist + 60 fps at the chosen voxel size. If this fails at 0.25 m, drop to 0.5 m (R2) and re-measure before proceeding. Do not start M2 look-dev until the voxel size is settled; the art pipeline depends on it.
- **G3 (end of M3)** — seamless descent with no visible seam. If this fails, invoke R1. **Decide within one week of failure — do not iterate on it indefinitely.**

---

## 7. Risks

| | Risk | Likelihood | Mitigation / fallback |
|---|---|---|---|
| **R1** | **Seamless descent can't be made invisible or stable** | Medium | Fall back to cinematic transition: keep the whole scale ladder, but mask the L1 crossfade behind a 3–5 s atmospheric-entry effect (heat glow, cloud layer, camera shake). Retains ~80% of the feel, drops the double build, and lets us ship stock export templates. Costs ~1 week to retrofit *if* we've kept the ladder modular — so keep the crossfade behind a single interface from day one. |
| **R2** | **0.25 m voxels blow the frame/memory budget** | **High** | 0.25 m is 8× the data of 0.5 m. Pre-authorised fallback: 0.5 m voxels, with 0.25 m retained only inside a small LOD0 radius if the octree allows. Measured at G1, before art depends on it. |
| **R3** | **Custom engine build rots** | Medium | CI builds the engine on every module/engine bump, not on demand. Pin exact Godot + Voxel Tools versions in a manifest. Budget a day per upgrade and don't upgrade mid-milestone. |
| **R4** | **CPU and GPU density fields drift apart** | Medium | Automated agreement test (§2.2) in CI. Non-negotiable — this failure is invisible until it tears at range. |
| **R5** | **Voxel Tools GDExtension/module API churn** | Low-Med | Wrap all `VoxelTool`/generator calls behind our own thin facade so an upstream rename is one file, not a hundred call sites. |
| **R6** | **Scope** | **High** | This is a large project. §8 lists what gets cut first, decided now rather than under pressure. |

---

## 8. Cut list — decided in advance

In priority order, first to go:

1. Multiplayer (not in this plan at all; architecture shouldn't *preclude* it, but nothing is built for it)
2. Walking inside moving ships (already cut, §4.5)
3. Base building beyond a landing-pad anchor
4. Planets 3 and 4 — ship with Terran + Moon if needed; the ladder and crafting loop are what's being proven
5. Fluid simulation — lava and water are SDF regions with a shader and a damage volume, not simulated
6. Orbital mechanics — planets on fixed circular rails, not n-body

Everything above the line in §6 M1–M3 is **not** cuttable. That's the game.

---

## 9. Repo & tooling (new repository)

- **Godot 4.6**, custom double build. Version manifest pins engine + module commit.
- **Git LFS** for `.png`, `.glb`, `.ogg`, `.wav` from the first commit — retrofitting LFS onto existing history is miserable.
- **CI (GitHub Actions):** build engine + module → cache artifact; build export templates once per engine version; run GDScript tests on every push.
- **Tests (GUT)** — pure logic only, which is where the real bugs live: recipe resolution, inventory mass limits, ship stat aggregation, control allocation, and the §2.2 generator determinism/agreement test.
- **Structure:** `addons/`, `scenes/`, `scripts/`, `resources/planets/`, `resources/items/`, `resources/recipes/`, `resources/ship_parts/`, `shaders/`, `tools/build/`.

---

## 10. Week 1 — concrete spike checklist

Nothing else starts until these are answered with measurements, not opinions.

- [ ] Compile Godot 4.6 + Voxel Tools 1.6 **module, double variant**, on the dev machine. Record wall-clock build time.
- [ ] Get the same build running in GitHub Actions with ccache. Record cold and warm times.
- [ ] Build export templates for Linux + Windows from the double build. **Confirm an exported binary actually launches** — this is the step most likely to surprise us.
- [ ] Open the upstream planet demo in the double build. Confirm it renders and edits.
- [ ] Measure `lod_count` ceiling and the memory cost at 12 LODs.
- [ ] Spawn a 3 km sphere at **0.25 m** voxels. Record: fps, RAM, meshing thread time, time-to-stream on teleport.
- [ ] Repeat at **0.5 m**. This is the R2 decision data.
- [ ] Place a camera at 1×10⁹ units from origin and check for rendering jitter and physics stability.

**Pass bar for week 1:** a double-precision exported binary that renders an editable 3 km voxel sphere at ≥60 fps at *some* voxel size, with CI reproducing the engine build. If any of that is unreachable, R1 gets invoked before M1 starts, not after M3.

---

## 11. Open questions

Not blocking — these can be answered during M1–M2 without changing the architecture:

1. **Art direction** — hand-painted stylised (Valheim) or flat-colour low-poly? Affects M2's shader work, not its structure.
2. **Planet radius** — 3 km is the working assumption. Larger is more impressive and linearly more expensive; it also pushes the LOD count. Settle at G1 with real numbers.
3. **Ship grid size** — 1 m cells assumed. 0.5 m gives finer builds and quadratically more parts to manage.
4. **First-person or third-person?** Affects character controller and the pole-handling in the camera rig. Third-person makes spherical-gravity camera work noticeably harder.
5. **Water on Terran** — a shader sphere with a buoyancy volume, or genuinely absent? Cheapest is a sphere at sea level with a damage/swim volume.

---

## Sources

- [Zylann/godot_voxel — releases](https://github.com/Zylann/godot_voxel/releases) — build variants, `double` availability, export-template caveat
- [Voxel Tools — getting the module](https://voxel-tools.readthedocs.io/en/latest/getting_the_module/) — module vs GDExtension
- [Voxel Tools — generators](https://github.com/Zylann/godot_voxel/blob/master/doc/source/generators.md) — spherical planets, `SdfSphere`, noise choice, floating-chunk failure mode
- [Godot — Large world coordinates](https://docs.godotengine.org/en/stable/tutorials/physics/large_world_coordinates.html) — double precision, performance caveats
- [Godot — Emulating double precision on the GPU](https://godotengine.org/article/emulating-double-precision-gpu-render-large-worlds/) — camera-relative rendering path
