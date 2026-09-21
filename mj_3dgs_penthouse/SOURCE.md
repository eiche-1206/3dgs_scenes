# mj_3dgs_penthouse — XGrids LCC2 → MuJoCo, source & regeneration

Source: `/home/limx/data/3DGS/scenes/LCC_studio/PentHouse` (`meta.lcc2`, quality tier,
10,675,771 gaussians / 181 MB, 23 `.sog` chunks + `env.sog`).

Layering: **3DGS renders the robot cameras; a mesh/SDF pair does collision and the
viewer.** Deployed at
`navi_ws/guide_ws/src/simulation/worlds/mjcf/scenes/mj_3dgs_penthouse/`.

---

## Pipeline decisions (v2 — locked)

The brief changed after v1: the pipeline must be **reusable on other scans and easy to
reproduce**, with visual and collision quality as the top priority. Everything below is
a decision, not an option. Prefer one measured number per scene over per-scene logic.

| decision | why |
|---|---|
| **Collision is SDF only.** No convex decomposition. | Measured here: 64 CoACD hulls filled **3,644 m³** against **1,318 m³** of real solid (2.8×); all 60 balls dropped into furniture voids spawned INSIDE a hull (`ncon=60` at t=0); the camera sits inside a featureless slab. A living room punishes hulls *harder* than an office floor — the furniture is lower and denser. |
| **Indoor only.** The 348 m² roof terrace is out. | The robot navigates indoors. Cuts the grid from 295 M to 100 M voxels @ 4 cm. |
| **One z cut for the roof** (`CEILING_Z = 2.02`), not per-column removal. | The ceiling UNDERSIDE p25 over 18,835 indoor columns. Same shape as 15F (2.45 under a 2.50 ceiling). Costs the upper half of the double-height space; buys one number per scene instead of an algorithm. **2.20 was wrong and shipped for one build**: it is the highest structure per column, i.e. the roof deck *above* the suspended ceiling, and it left a lid on 51.3% of columns / 90.5% of the double-height room. |
| **VOX 0.04 → 0.02.** | The indoor crop + z cut leave a 23.8×32.8×3.40 m box = 331 M voxels / 1.33 GB. Spend the headroom on the thing both P0s depend on. |
| **FLATTEN = True.** | 15F's own note: 4 cm voxelisation leaves 1–3 layers of floor noise, 9.7% of columns ride 4–27 cm high and poke through the analytic plane to catch the feet. Directly the "accurate floor" requirement. |
| **Visual mesh from pipeline A**, not from the SDF. | 2 cm TSDF vs 4 cm; RaDe-GS *median* depth vs alpha-weighted *expected* depth; xatlas charts vs triplanar; render-based texel bake vs nearest-gaussian KD. 15F measured planar sharpness 5.58 → 7.14 from the bake change alone. |
| **Keep `facades.close_envelope`.** | 3DGS never reconstructs glass — it puts gaussians on whatever is *beyond* the pane. Any glazed scan needs the boundary BUILT or the robot walks out of the building. General, not bespoke. |

### Deleted as measured-no-payoff — do not resurrect

| step | measurement that killed it |
|---|---|
| `facades.close_facades` (per-plane morphological closing) | 54% → 67% envelope coverage, while `close_envelope` alone reaches 82%. Pure redundancy, four extra knobs. |
| "glass rescue" (union high-opacity gaussian centres) | +0.4% voxels. The hypothesis was that glass gaussians exist to be recovered; they do not. |
| multi-level nadir sweeps in `gs_carve` | Only needed for the 8.2 m box with an open terrace. At 3.40 m one level suffices. |
| CoACD variant | See the table above. |
| splat-transform voxel-mesh path (`mesh_from_lcc.py`) | Superseded by pipeline B (p50 17.9 cm vs 6.0 cm to the source gaussians) and it carries a silent Rz180 frame bug. |
| the extra `<light directional diffuse 0.3>` | Six stations, with and without: sharpness within 1% and brightness within 2%. It is `castshadow="false"` and the albedo already carries the scan's lighting, so it does nothing. `--light` keeps it for A/B. |
| `env_ring.py` — the scanned surroundings on a 60 m ring | Built and rendered. Only 37% of gaussians are environment and they cluster in a few azimuths, so most of the ring is the pale fill, which is BRIGHTER than the gradient sky and blocks it: windows went from soft daylight to a blown-out white slab. Code kept (`--env-ring`, default off); the gradient skybox wins. |
| `--tex 16384` atlas | Died in PIL's decompression-bomb guard, and the premise was already measured away: the median gaussian is 2.6 cm and a texel is 0.44 cm, so the atlas was never the limit. The colour fix is the render bake below, not more texels. |
| `glass_edges.py` on the 1.36M mesh | Straightened 22 panes / 4.4 m² and left the ragged-edge fraction at 58.1% → 58.1%. The dense mesh fragments each opening into components too small to read as rectangles. Superseded by `mesh_prune.py`, which cuts on scan support instead of shape. |
| `mesh_prune --tol-edge` (tighter cut near open boundaries) | The idea was that a shard hanging off a mullion survives `--tol` only because the mullion's own gaussians are within 25 cm of it, and what really marks it is the open edge. But **481,602 of 1,360,097 faces (35%) already touch an open boundary**, and 3 rings out that is 79% of the mesh — the rule stops discriminating and just becomes a global 12 cm cut (9.46% of area). A 2 cm TSDF of a furnished interior is riddled with small holes; "near a boundary" is not a rare property here. Flag kept, default off. |


---

## v5 — the visual pass (2026-08-17)

Brief: *"地面绝对不允许有破洞"*, *"玻璃再处理一下"*, *"渲染效果能否再提升（参照 mj_3dgs_15f）"*.
Five changes, each with the measurement that justified it. Collision is untouched.

| # | change | measured |
|---|---|---|
| 1 | **face count 336k → 1.34M** (`--target_faces 1500000 --grid 3`) | The source TSDF mesh has 4.38 M faces / 894 m²; 350k was inherited from a build that covered a quarter of the plan and was never tuned. 15F's own sweep: 349k → 1.95M raised sharpness 29.0 → 37.0 monotonically at 13 stations. Here: +9…14% at six stations, and table legs / cushion seams / chair frames go from mush to readable. |
| 2 | **atlas colour: KD-tree → multi-view render bake** (`b_penthouse/src/bake_texture_render.py`) | The KD lookup uses 3 of 48 SH coefficients, never composites, and blurs at the gaussian scale (2.6 cm) into 0.44 cm texels. Re-projecting real renders instead: 68.5% of 43.6 M texels re-baked (block 8 m, 900 cams); the rest keep KD colour. +7…12% sharpness on top of #1, and rug weave / wood grain appear. |
| 3 | **floor holes → zero** (`src/floor_patch.py`) | **40.8 m² of 220.1 walkable (18.5%) → 0.000 m² (0.000%)**, verified by resampling both meshes. 63% of the holes were a band within 35 cm of a wall or furniture — where a tripod camera cannot see the floor — and 14.9 m² was genuinely unscanned interior. |
| 4 | **prune mesh the scan does not support** (`src/mesh_prune.py --tol 0.25`) | Per-face distance to the nearest opaque gaussian: p50 4.9, p90 13.2, p99 26.9 cm. Cutting at 25 cm deletes 17,752 faces / 10.18 m² / **1.16% of area** — the torn shards hanging off the mullions. The criterion IS the user's constraint: a face with no opaque gaussian within 25 cm is not something the LiDAR would return either. |
| 5 | **grey backing plane → gradient skybox** | The plane only ever existed to stop floor holes reading black; #3 removes the need, and the plane was visible through every window as a grey slab. `rgb1` (top) is near-black on purpose — the roof is cut, so a bright zenith makes every room a roofless courtyard. |

Reproduce:

```bash
D=/home/limx/data/3DGS/pipelines/D_lcc2sim
# 1  geometry + KD colour  (docker gggs:ext, ~12 min)
docker run --rm --gpus all -v /home/limx/data/3DGS:/work -w /work/GGGS gggs:ext \
  python3 tools/bake_texture_tiled.py -i <PC>/gggs/penthouse/recon_post.ply \
  -o <PC>/gggs/penthouse/pA_1p5M.obj --tex 8192 --target_faces 1500000 --grid 3 \
  --weld 0.02 --cache <PC>/gggs/penthouse/gauss_nurec.npz
# 2  render bake the atlas  (.venv_gs, 154 s)
cd $D/b_penthouse && .venv_gs/bin/python -s src/bake_texture_render.py \
  --obj $D/penthouse/assets/penthouse_A15_visual.obj \
  --atlas $D/penthouse/assets/penthouse_A15_atlas.png \
  --rig $D/penthouse/gggs/penthouse/rig.json --tex 8192 --block_m 8 --max_cams 900 \
  --out $D/penthouse/assets/penthouse_A15R2_atlas.png
# 3  prune, then patch the floor, then render-bake the patch too
cd $D/penthouse
<discoverse> -s src/mesh_prune.py --tol 0.25 --out assets/penthouse_A15P_visual.obj
<discoverse> -s src/floor_patch.py --mesh assets/penthouse_A15P_visual.obj \
                                   --out  assets/penthouse_A15P_floorpatch
cd $D/b_penthouse && .venv_gs/bin/python -s src/bake_texture_render.py \
  --obj <..>/penthouse_A15P_floorpatch.obj --atlas <..>/penthouse_A15P_floorpatch.png \
  --rig <..>/rig.json --tex 8192 --block_m 8 --max_cams 900 --out <..>/patchP_R.png
# 4  compose
<discoverse> -s src/compose_scene.py --tiles ../b_penthouse/assets/sdf \
  --visual assets/penthouse_A15P_visual.obj --atlas assets/penthouse_A15R2_atlas.png \
  --floor-patch assets/penthouse_A15P_floorpatch --sky --out scene.xml
```

### v6 — the floor is REPLACED, not patched

Feedback on v5: *"地面破洞修复了，但是看起来坑坑洼洼的"*. Measured on a 5 cm grid, residual
of the floor height field to its own 25 cm low pass: the TSDF mesh gave 31.4 mm sd /
75.0 mm p95 against 4.7 / 9.4 for the synthesised patch, so v6 replaced the floor.

> **THE REASON GIVEN HERE WAS WRONG, and the correction matters more than the
> conclusion.** v6 originally said "the pits are the scan's floor". Challenged on it
> (*"你确定是扫出来的地板本身有问题吗"*), four follow-up measurements say otherwise —
> see **Where the floor bumpiness actually comes from** below. The floor replacement is
> still the right call, but because the mesh *fabricates* a surface where the scan never
> saw one, not because the scan's floor is rough.

v6 replaces the floor: `floor_patch.py --full --strip` covers the whole indoor plan and deletes what it
replaces —

* **195.4 m² / 22.5% of the mesh**: near-horizontal faces (`|n_z| > 0.80`) within 12 cm
  of the new floor. The normal test is what keeps sofa sides and table legs.
* **70.6 m² more**: faces in a connected component under 500 cm² lying within 25 cm of
  the new floor. Once the bumpy floor is gone every torn shard that used to be buried in
  it sits in plain view on a clean surface, and a normal test cannot catch a tilted
  shard — but a size test can, because a real object at floor height is one big
  component and there are 58,909 components in this mesh.

Result: **residual sd 31.4 → 3.4 mm, p95 75.0 → 6.9 mm** (9×), holes still 0.000 m².
The colour comes from a second render bake at 61.7% coverage.

The rig for both bakes is now `rig_dense.json` — the same 89 stations, but 3 heights ×
16 yaws × 5 pitches = **21,360 cameras** against the original 2,136. Atlas coverage
68.5% → **78.5%**; it costs 869 s instead of 154 s and nothing else.

What this trades away: the floor no longer carries real geometric relief, so a rug reads
as a printed rug and a 2 cm threshold is smoothed. Everything above 12 cm off the floor
is untouched. `--full`/`--strip` are flags; without them v5's behaviour (fill holes only)
is still there.

Verified after: floor holes **0.000 m²**; robot 20,000 steps at two spawns, `base_z`
finite, MuJoCo warning counters clean; compile 7.8 s, `ngeom=17`, 2.54 M mesh faces.

### Where the floor bumpiness actually comes from

Four experiments, each killing one explanation. Scripts in the session scratchpad
(`floor_rough.py`, `floor_corr.py`, `nadir_depth.py`).

1. **Not the estimator, not decimation.** The 31.4 mm was a mean over a ±30 cm band over
   all faces, so it mixed the floor with rug edges, sofa undersides and the 70.6 m² of
   shards v6 deletes. Redone as the median z of `|n_z| > 0.8` samples per cell: 26.7 mm.
   On the 4.38 M-face mesh *before* decimation: 26.8 mm. Decimation contributes nothing.
2. **Not "the same bumps as the scan".** Mesh height field vs opaque-gaussian-centre
   height field, both detrended: **r = +0.04, R² 0.2%**, against a control of **+0.665**
   for the mesh against an independent resample of itself. (Weak evidence on its own —
   gaussian centres are a poor proxy for where a render puts the surface.)
3. **Not grazing geometry.** The TSDF rig is 2,136 cameras at one height (0.925 m) with
   pitches ≥ −30°, so the floor is only ever seen at ~10°, where 1 cm of depth error
   becomes 5 cm of z. Rebuilt the TSDF from 8,544 cameras at 3 heights with pitches to
   −70°: **27.1 → 29.8 mm**. Slightly worse. Hypothesis dead.
4. **It is the unobserved floor.** Rendering the gaussians from 54 nadir cameras at
   z = 1.70 with the carve's own alpha and depth-variance gates gives a floor at
   **11.1 mm sd / 22.0 mm p95** — 2.5× smoother than the mesh. Splitting the mesh's own
   cells by how well that render could see them:

   | nadir returns in the cell | share | mesh floor sd | p95 |
   |---|---|---|---|
   | 0 — never seen | 39.3% | **39.0 mm** | 86.1 mm |
   | 1–19 | 3.4% | 31.7 mm | 58.6 mm |
   | 20–99 | 42.6% | **13.7 mm** | 19.8 mm |
   | ≥ 100 | 14.7% | **11.4 mm** | 17.3 mm |

   Where the scan sees the floor the mesh is 11–14 mm, i.e. as good as the source. The
   global 27 mm is the unseen 39% dragging the average. **Same root cause as the 18.5%
   of holes: missing observations.** The mesh does not report a rough floor; it invents
   a floor where it has no data, and that invention is what reads as 坑坑洼洼.

   Side finding, unexplained: on well-seen cells the mesh floor sits a median **+31 mm
   ABOVE** where the rendered depth puts it (p25 +20, p75 +44). Worth chasing before
   trusting the mesh floor's absolute height for anything.

### v7 — the floor's height comes from the gaussians, not from the mesh

`src/floor_harvest.py` renders depth from **5,241 cameras placed to see the floor**
(1 m nadir grid + oblique fans at 0.35/0.75/1.30 m, pitches −60/−35/−18), gates each ray
with the carve's own alpha and depth-variance tests, weights each sample by cos²(incidence)
— a ray hitting at angle t turns its depth error into z error by 1/cos t — and runs a
second pass that rejects anything more than 3 cm from the first pass. Output: a per-cell
height **and an honest count `n` of the rays behind it**.

`floor_patch.py --harvest` then shrinks toward a smooth base by `n`: λ=1 above 200 rays,
0 below 20. The thresholds are not taste — they come from two measurements:

| rays per 5 cm cell | cells | relief sd | split-half r | full-set reliability |
|---|---|---|---|---|
| 20–99 | 5,344 | 26.8 mm | +0.788 | 0.881 |
| 100–199 | 4,158 | 21.7 mm | +0.867 | 0.929 |
| 200–399 | 6,127 | 17.5 mm | +0.883 | 0.938 |
| 400–999 | 14,891 | 11.9 mm | +0.887 | 0.940 |
| ≥1000 | 18,946 | 9.6 mm | +0.853 | 0.921 |

Two **disjoint** halves of the camera set (2,621 / 2,620) agree at r ≈ 0.85–0.89, so
94–97% of the measured relief amplitude is reproducible signal, not render noise. (What
that does *not* prove: both halves see the same gaussian field, so a systematic artefact
of the representation reproduces perfectly. Reproducible ≠ physically real.)

**The number that actually justifies v7 is alignment, not smoothness.** Against the
gaussian-rendered floor — i.e. against what the 3DGS camera layer will draw — measured on
the 47k cells with ≥200 rays:

| floor | median offset | p25 … p75 |
|---|---|---|
| TSDF mesh | +11.3 mm | +1.5 … +23.2 |
| v6 patch | +14.3 mm | +2.2 … +30.5 |
| **v7 patch** | **−0.0 mm** | **−3.4 … +3.4** |

Interquartile spread 28 mm → **6.8 mm**. In `hybrid` mode the mesh floor and the gaussian
floor now sit at the same height instead of 1–3 cm apart.

> **The alignment target is estimator-dependent, and the two candidates disagree by
> 2.5 cm.** `floor_harvest` renders alpha-normalised *expected* depth. PGSR (IEEE TVCG
> 2024, arXiv 2406.06521) argues that is biased toward the camera and renders the plane
> distance and normal instead, dividing them (eq 2–4); implemented here as
> `--depth pgsr`, it puts the floor a median **24.5 mm further** (sd 16.9, p5 −57.8,
> p95 −4.8) — the direction the paper predicts. Which one is the surface cannot be
> settled from this asset: the LCC2 is `quality` tier, `data/` holds only `3dgs`, and
> `meta.lcc2` contains no mesh/bvh/poses/lidar keys, so there is no independent geometry
> on disk. A `portable`/L2Pro export of the same scan would decide it in one measurement.
> For *visual* hybrid consistency the expected-depth target is arguably right anyway —
> what a camera shows is where the alpha mass is — but for metric geometry it is open.

Roughness of the shipped surface: TSDF 31.2 → v6 3.4 → v7 4.3 mm sd. v7 is not much
rougher than v6 because the corner-averaging that makes the patch mesh continuous halves
the relief (height field 9.3 mm → mesh 4.3 mm); that attenuation is accepted rather than
compensated, because "reproducible" is not the same as "the floor is really wavy".

Verified: holes **0.000 m²**; robot 20,000 steps at two spawns, `base_z` finite, warning
counters clean; compile 8.4 s, 2.56 M mesh faces. Shipped as `scene.xml`.

**Open, pre-existing, NOT from this pass:** spawning the robot at (4.67, −1.33) aborts
with `mjc_MeshSDF: too many contact points`. Reproduced on `scene_v3.xml`, i.e. it is a
property of the SDF tiles, not of the new visual mesh.

---

### v8 — the three floors become one floor (collision joins the alignment)

v7 aligned VISUAL to GAUSSIAN and stopped; the collision floor stayed the FLAT
scalar (`FLOOR_DATUM = False`), 76 mm above the shipped visual floor at the
median on the open plan (p95 +141). v8 closes that: collision now equals the
visual floor by construction, and every constant in the chain was re-measured.

**1. The estimator question was adjudicated on ConferenceHall first.** v7 left
open whether expected depth or PGSR unbiased depth is "the surface" (they
disagree by 24.5 mm here). CH is portable tier — its LiDAR mesh is independent
ground truth. `conferencehall/src/floor_adjudicate.py`, 371 nadir cameras, both
estimators, same gates, ±25 cm window: **expected −77.9 mm vs PGSR −81.4 mm
against the LiDAR floor — a 3.4 mm difference.** The 24.5 mm PentHouse
disagreement does not reproduce; PGSR provides no basis for shifting anything.
The basis stays expected-depth (what the camera layer draws). Verdict figure:
`conferencehall/outputs/M_adjudicate.png`.

  The adjudication's real finding is the offset itself: BOTH estimators put the
  gaussian floor ~8 cm below the LiDAR floor, and it is not a registration
  error — on CH the gaussian-derived surfaces sit outward of the LiDAR on every
  side (floor −32 mm, ceiling +30 mm, walls ±20 mm outward, TSDF convention).
  The gaussian representation inflates the room by 2–3 cm per face.
  **Consequence for PentHouse (no LiDAR): the absolute height of the whole
  aligned stack may be a few cm below physical reality, unmeasurable from this
  asset. Everything inside the sim is self-consistent; only the absolute datum
  is uncertain.** A portable/L2Pro export of this scan would measure it.

**2. The baked datum was rebuilt — the M18 surface was the wrong basis.**
`assets/sdf_datum/` (M18) was built from a gaussian CENTRE-histogram estimator
that predates floor_harvest; measured against the v7 floor it sits **+26.8 mm
high (IQR 33.5)** — a different estimator of the same floor. `floor_datum.py
--from-mesh` now builds the datum by sampling the SHIPPED A15G floorpatch
(10 cm cells, minimal 0.05 m smoothing): datum − visual floor = **+0.2 mm,
IQR 6.2**. Old surface kept at `cache/floor_datum_m18.npz`.

**3. bake_sdf grew step 5c (trusted open-floor flatten).** After the first v8
bake, 7.3 % of the open floor kept carve ridges up to +17 cm that the flatten
missed: the clearance-band test calls a column "not floor-only" whenever
anything (a curtain hem, a glass edge) drops one voxel into the band, and then
touches nothing in it. The gaussians say those ridges are air (median 0 opaque
gaussians in the 3–25 cm above the floor; control cells identical). Step 5c
force-flattens (datum, datum+0.20] where the harvest saw the floor ≥200 times
AND the gaussians hold no low mass (`cache/flatten_mask_v8.npz`, 121 m²):
348,839 voxels. Open-floor p95 went +96 → +15.6 mm.

**Acceptance (all seven gates, measured on the shipped scene.xml):**

| # | gate | target | v8 measured |
|---|---|---|---|
| 1 | visual floor holes on walkable set | 0.000 m² | **0.000 m²** (216.8 m² walkable) |
| 2 | visual floor flatness (residual to own 25 cm low pass) | ≤ ~5 mm sd | **4.3 mm sd**, p95 8.8 |
| 3 | visual vs gaussian floor | median \|Δ\| ≤ 5 mm | **−0.0 mm**, IQR 7.2 |
| 4 | collision vs visual floor (open plan) | median \|Δ\| ≤ 10 mm | **+0.8 mm**, p95 +15.6 (FLAT was +76.2 / p95 +141) |
| 5 | robot float | ≤ +10 mm | same measure as #4: **+0.8 mm** |
| 6 | drop 60×20k (no backstop plane) | 0 through, 0 ghost | **60/60 floor, 0/0**, penetration **−0.06 mm** |
| 7 | robot 20k steps × 2 spawns | finite, 0 warnings | **both finite, warnings none** (ncon max 54/52) |

Map: `outputs/M19_col_vis_map.png` (FLAT vs v8, same mask, same colour scale).

Shipped: `scene.xml` = v7 assets (A15G visual + floorpatch + A15R3 atlas,
byte-identical) + `../b_penthouse/assets/sdf_datum_v8/` (15 tiles, 256 MB, cut
from `cache/sdf_datum_v8.npy`). v7 collision kept at `scene_v7.xml` +
`assets/sdf/`. Constants re-measured on the v8 field: FLOOR_NOMINAL_Z −0.777
(p5 floor-only), COL_PLANE_Z −0.690 (test scenes only; the deployed scene has
no backstop plane).

Known, pre-existing, retest not attempted: the (4.67, −1.33) spawn abort
(`mjc_MeshSDF: too many contact points`) was a property of the FLAT tiles; not
yet re-tried on the datum tiles.

### v9/v10 — the floor becomes ONE surface, carried by a height field

**The complaint that started it:** the v8 floor still felt bumpy. Measured as the
thing a foot actually feels — the collision surface's residual to its own 0.55 m
low pass, i.e. the height change within one stride — the v8 SDF floor wanders by
**17.4 mm sd**. That is not the building; it is the 2 cm voxel grid representing a
sloped floor as a staircase.

**Why not the 5f/15f fix.** Both ship an analytic `<geom type="plane">` at
z = −0.140, one centimetre above their SDF floor, so the foot contacts the plane
and never reaches the SDF zero surface. Read from their shipped tiles, their
collision floors are dead flat: p5..p95 = **0.0 mm**, every column at −0.150 —
they were flattened to a constant during the bake, which is what makes a plane
correct there. PentHouse's floor is not flat: **140 mm p5..p95**, 4.4 mm/m
gradient, 37 mm residual to the best-fit plane. A single plane would be more than
30 mm wrong over **42%** of the floor and more than 50 mm over 22% — the v7
"robot floats" defect, restored. (The warp is not a reconstruction artefact
either: ConferenceHall's LiDAR ground truth has 184 mm of its own, gradient
4.7 mm/m, with the gaussians adding only 53 mm on top.)

**So: keep their trick, change the primitive.** A height field can be one
centimetre above the SDF floor *and* follow the drift.

**v10 goes further and makes the visual and collision floors the same surface.**
Define F = the datum low-passed at 0.20 m, ramped back onto the raw datum over
0.50 m at the patch boundary (without that ramp the low pass lifts the patch edge
by up to 74 mm and it pokes through the surrounding mesh; the ramp blends against
each vertex's OWN height, so the seam stays exactly where floor_patch put it —
boundary p95 +3.7 mm). Then:

| layer | v10 |
|---|---|
| collision | `<geom type="hfield">` = F exactly (`assets/floor.hfield`, float32, no quantisation) |
| visual | `penthouse_A15H_floorpatch.obj` — A15G with every vertex z set to F sampled **bilinearly**, i.e. the same function MuJoCo's hfield collider evaluates |
| SDF floor | re-baked at F − 10 mm, backstop only; nothing reaches it |

The visual mesh changes in nothing but vertex height. Its UV is a planar xy
projection (measured corr(u,x) = corr(v,y) = 1.000) so the 8K atlas stays valid
and no texture re-bake was needed.

**Measured (`src/floor_align.py`, `b_penthouse/src/floor_hfield.py`):**

| gate | v8 | **v10** |
|---|---|---|
| visual floor holes on the walkable set | 0.000 m² | **0.000 m²** (215.0 m²) |
| visual floor flatness, sd | 4.3 mm | **1.0 mm** (p95 1.9) |
| **collision − visual** | +0.98 mm, IQR 13.1, p95 32.9 | **−0.00 mm, IQR 0.01, p95 0.07** |
| **foot feel** (residual within 0.55 m) | 17.4 mm sd | **3.0 mm sd** (p95 6.1) |
| visual − gaussian floor | −0.04 mm, IQR 7.3 | +1.15 mm, IQR 17.1 |
| drop 60 × 20k, no backstop plane | 60/60, 0/0 | **60/60, 0 through, 0 ghost** |
| drop penetration vs the surface actually touched | −0.06 mm | **−0.85 mm** (p10/p90 −3.7/+2.7) |
| robot 20k × 2 spawns | finite, 0 warnings | **finite, 0 warnings** (ncon max 44 / 35) |
| step time with HU_D04_01 | 5.14 ms, **aborts** | 3.57 ms, no abort |

**The abort.** `mjc_MeshSDF: too many contact points` — the failure first seen at
the (4.67, −1.33) spawn — reproduces on the v8 field within 2,000 steps whenever
the robot flops onto the floor with many geoms at once. With the hfield taking
the foot contact it does not occur.

**What got worse, stated plainly.** The visual floor is now a smoothed surface, so
against the gaussian floor its IQR grows 7.3 → 17.1 mm (the median is unchanged at
+1.15 mm, so nothing shifts globally; the relief simply no longer matches point
for point). In `hybrid` mode the 3DGS camera still draws the gaussians' own
relief. `--smooth` is the knob: 0.05 → foot 7.0 mm / IQR 11.4, 0.20 (shipped) →
3.5 / 17.1, 0.50 → 1.5 / 24.1.

**Performance, in context.** The SDF narrowphase charges per candidate PAIR, not
per contact: 29 robot geoms × the tiles their AABBs touch, each running
sdf_initpoints × sdf_iterations gradient steps whether or not anything is close.
Measured with HU_D04_01: PentHouse pure SDF 5.14 ms/step, **the repo's own
mj_3dgs_5f 4.24 ms/step (0.47× realtime)**, ConferenceHall 5.64. So the slowness
is the approach, not this scene, and the hfield already beats the reference.
Further, measured but NOT shipped because it needs its own drop/robot validation:
hfield + `sdf_iterations` 5/`initpoints` 10 → 1.24 ms (1.61× realtime), 3/6 →
0.58 ms (3.47×).

Shipped as `scene_v10.xml`; v8 kept at `scene.xml`/`scene_v7.xml`, tiles at
`../b_penthouse/assets/sdf_v10` (15, 256 MB).

### v11 — the scan's drift is removed, and the floor becomes what 5f/15f's is

Three complaints drove this: the floor still looked uneven, the sim ran at a
third of realtime, and **the robot would not stand still** — none of which happen
on mj_3dgs_5f / mj_3dgs_15f.

**What the reference scenes actually do, measured on their shipped assets.** Not
what the code comments claim — what the files contain:

| | visual floor relief p5..p95 | collision plane − visual floor |
|---|---|---|
| mj_3dgs_15f | 197.1 mm | **+48.5 mm** median (−44 … +153) |
| mj_3dgs_5f | 211.0 mm | **+103.5 mm** median (+25 … +236) |

So they never flattened their VISUAL mesh, and their robot stands five to ten
centimetres above the floor it is drawn on. What they did do is flatten the
COLLISION floor to a constant during the bake (their tiles: p5..p95 = 0.0 mm,
every column at −0.150) and put an analytic `<geom type="plane">` one centimetre
above it, at −0.140.

**That plane is the whole answer to "站不稳".** A foot-sized box, dropped and left
to rest, reports:

| surface | resting-height sd | contacts | normal |
|---|---|---|---|
| **analytic plane / box top face** | **0.000 mm** | **4.0 ± 0.00** | 0.00° |
| height field (v10) | 0.087 mm | 0.8 ± 1.65 | 0.74° |
| raw SDF (v8 and every version before) | 0.433 mm | 0.5 ± 1.66 | 0.20° |

Both of our surfaces hand the controller a contact set that appears and vanishes
— mean below one contact, sd above 1.4 — while a plane holds four, forever.
Neither a 2 mm margin (0.3 ± 0.75), nor spherical feet (height sd 10.4 mm — it
bounces), nor a finer field fixes it; the hfield additionally aborts MuJoCo with
`mju_makeFrame: xaxis of contact frame undefined` at one of the two spawns.

**A plane is only correct on a flat floor, so v11 flattens the scene, not the
floor.** The 140 mm of relief is scan drift — a finished floor does not fall
14 cm across 20 m — so `src/dewarp.py` subtracts W(x,y) = F(x,y) − Z₀ from
**everything**: the visual mesh, the floor patch and all 5.5 M gaussians
(`cache/gauss_warped.npz` keeps the originals). Applying it to the whole scene is
what keeps furniture standing ON the floor and wall bases meeting it; the
deformation is a 0.7 % shear, far below what reads as a lean. Gaussian rotations
and scales are untouched — at that shear the shape change is under a voxel.

The collision then follows the reference recipe exactly: re-carve from the
de-warped gaussians, bake with the scalar `FLATTEN` (`FLOOR_DATUM = False`
again), measure, plane one centimetre above.

**Measured:**

| | v8 | v10 (hfield) | **v11** | reference 5f/15f |
|---|---|---|---|---|
| visual floor relief p5..p95 | 252 mm | 252 mm | **2.7 mm** | 197 / 211 mm |
| visual floor, foot roughness | 4.3 mm sd | 1.0 | **0.95 mm sd** | 16.4 / 18.9 mm |
| collision floor p5..p95 | 140 mm | 140 | **2.4 mm** | 0.0 mm |
| collision − visual | +0.98 mm | −0.00 | **−0.26 mm**, IQR 0.00 | +48.5 / +103.5 mm |
| foot: resting sd / contacts | 0.433 mm / 0.5±1.66 | 0.087 / 0.8±1.65 | **0.000 mm / 4.0±0.00** | 0.000 / 4.0±0.00 |
| drop 60 × 20k | 60/60, −0.06 mm | 60/60, −0.85 | **60/60, −0.0 mm**, 0 through | — |
| robot 20k × 2 spawns | ok, ncon max 54/35 | ok | **ok, ncon max 23/26**, 0 warnings | — |
| step time with the robot | 5.14 ms (0.39×) | 3.57 (0.56×) | **1.71 ms (1.17× realtime)** | 4.24 ms (0.47×) |

The speed came free with the plane: it takes the foot contacts, so far fewer
robot geoms are pressed into the SDF and the narrowphase — which charges per
candidate PAIR, not per contact — runs on far fewer of them. `sdf_iterations`
/`sdf_initpoints` stay at the reference's 10/20; lowering them is available and
measured (6/12 → 2.29×, 4/8 → 3.93×, 3/6 → 5.26×, 2/4 → 6.86× realtime) but is
not needed and would need its own contact-accuracy validation.

**The floor texture was re-baked** (`src/floor_recolour.py` → `bake_texture_render`):
the shipped atlas had been baked against the pre-v10 heights, so the shading of
the old relief was painted onto the new flat surface and read as bumps that were
not there. Re-sampled at the new heights (52.2 % of texels had a gaussian within
30 cm) and re-projected from 900 cameras per 8 m block (59.8 % of texels).

Shipped as `scene_v11.xml` and deployed. Assets: `penthouse_A15G_visual_flat.obj`,
`penthouse_A15H_floorpatch_flat.obj/.png`, `penthouse_A15R3_atlas.png`,
`../b_penthouse/assets/sdf_v11` (15 tiles, 253 MB).

**Open:** the gaussian PLY for the 3DGS camera layer (`penthouse_gs_sh3_zup.ply`)
has NOT been de-warped — only `cache/gauss.npz` has. The `gs:` block is disabled
in the deployed meta.yaml, so nothing reads it yet, but it must get the same
z' = z − W(x,y) before the camera layer is switched on or the rendered floor will
sit up to 14 cm off the mesh floor.

### v12 — the visual mesh stops being one geom, and the LiDAR stops gating startup

v11 was correct and unshippable: `start_sim.sh` tore the stack down before the
robot ever stood up. Two separate costs, found by measurement, fixed separately.

**Rendering.** MuJoCo frustum-culls per GEOM, so one geom is a scene with culling
switched off. Same mesh, same atlas, same faces, four stations:

| geoms | faces | frame | FPS |
|---|---|---|---|
| 1 | 996,982 | 67.3 ms | 14.9 |
| 4 | 996,982 | 33.4 ms | 29.9 |
| **8** | **996,982** | **1.3 ms** | **757** |
| 13 | 996,973 | 2.0 ms | 512 |

50× for free. Decimation was tried instead and **rejected three times over**:
uniformly to 347k (v13 — `start_sim` passes, quality does not), by tier to 600k
(A23 — "破洞太多"), and on the shipped mesh at 350k (it is 27.4% boundary edges,
so quadric collapse eats the connective triangles and the walls perforate,
`outputs/M30_dec350_grey.png`). The reference scenes ship at 93–131 faces/m²
because their source was closed; ours is not.

**Startup.** `mujoco_runner` advertises `/mujoco/respawn` only after the taichi
LiDAR backend has built its BVH, and `robot_control_sequence.py` waited 90 s for
that service. Build time is linear in the static triangle count — 2 + 0.097×(千面)
seconds across four points (299k→31 s, 347k→33 s, 600k→62 s, 1.04M→103/111 s) —
so v12 lands at ~111 s and missed the gate by ~21 s.

**Face count does not cost the LiDAR anything at run time** (`$SCR/lidar_bench.py`):

| scene | static tris | BVH build | per-frame trace p50 | at 10 Hz |
|---|---|---|---|---|
| v13 | 300,000 | 30.8 s | 1.80 ms | 1.8% |
| A23 | 599,997 | 61.7 s | 2.16 ms | 2.2% |
| **v12** | **996,982** | **103.3 s** | **1.76 ms** | **1.8%** |

A BVH is O(log n) to traverse; the 3.3× mesh traces no slower. More faces is if
anything better for the sensor — mullions and chair legs are the first things
decimation drops, and their returns go with them. So the fix belongs at startup,
and the user chose the smaller of the two: `service_wait_timeout` 90 → 180 s in
`sim_enhancement/scripts/robot_control_sequence.py`. (The alternative, caching the
BVH, was measured and works — 106.9 s build vs 0.0 s `np.load` of a 51 MB npz,
round-trip byte-identical — and is keyed on the static-triangle hash so the cache
file is machine-independent and could ship with the scene. It was not taken
because it needs a pre-warm step to help the very first launch.)

**Measured on the deployed scene**, `start_sim.sh start --scene mj_3dgs_penthouse`:

```
13:12:24.9  loading MJCF
13:14:15.8  BVH done -> /mujoco/respawn advertised   110.9 s, 1,043,664 tris / 367,485 nodes
13:14:17    Phase 1 ready (sensors + bridge online)
13:14:24    Standup complete -- robot in walk_mode
13:14:52    Self-nav simulation stack READY
```

physics **499.5–500.5 Hz** against a 500 Hz target (full realtime), viewer 83–140
fps, LiDAR 98–100 scans per 10 s, threads 7/7.

**Acceptance, run against the deployed `scene.xml` (not the work tree):**

| check | result | tool |
|---|---|---|
| deployed render vs work-tree v12 | mean \|Δ\| **0.62**/255, 0.04% of px >8 | `src/deploy_views.py` |
| foot probe, sites auto-picked from the shipped tiles | **8/8 touch only `floor_backstop`**, −0.7100, jitter **0.000 mm**, contacts **4.00±0.00** | `src/foot_probe.py` |
| 60 balls into voids under furniture | **60/60** to the real floor, 0 ghost, 0 through, penetration **−0.0 mm** | `src/drop_test.py --tiles` |
| live robot, 30 s, no teleop | base z p-p **0.04 mm**, tilt ≤0.19°, drift 0.2 mm | `$SCR/stand_live.py` |
| three floors | plane −0.7100, SDF crossing −0.7200 (IQR 0.0 mm), visual −0.7097; **plane − visual −0.26 mm** | `src/floor_layers.py` |
| robot 20k steps × 2 spawns (catalog + living room) | **qpos finite throughout, 0 MuJoCo warnings**, ncon max 23 / 24 (spawn 2) | `src/robot_soak.py` |
| chunk split conserves the mesh | **996,982 → 996,982 faces**, area +0.1 ppm | trimesh, vs `penthouse_A15G_visual_flat.obj` |

**Two things the acceptance tools had to be corrected for**, both of which had
been quietly returning nonsense:

* `find_voids` reads `b_penthouse/cache/sdf_global.npy`, which is the *pre-de-warp*
  field — v11 flattened the scene and re-baked, so the cache and the shipped tiles
  no longer describe the same floor. It picked **0** sites. `--tiles` reads the
  tiles MuJoCo actually loads.
* Hand-picked probe coordinates are how you drop a box inside a cabinet. One of
  four hand-picked "open floor" points at (0, −6) is solid from −0.89 to +0.61 —
  75 voxel layers — and returned 29 contacts and 67 mm of jitter. Sites are now
  picked from the field.

**The collision floor is one voxel higher under furniture** — crossing −0.7005 vs
−0.7200 in the open (51,008 open columns vs 52,290 covered ones), i.e. 10 mm above
the plane. That is `bake_sdf.py`'s step 5c doing what it was told: the flatten only
applies inside the trusted open-floor mask, and the scan never saw under the sofas.
It is why the dropped balls rest on the field rather than on the plane, and it is
harmless — the robot cannot go there. Do not "fix" it by flattening blind.

Shipped as `scene_v12.xml` → deployed. Assets: `assets/vis_chunks6/` (8 chunks,
996,982 faces), `penthouse_A15R3_atlas.png`, `penthouse_A16_floorpatch.obj/.png`
(46,680 faces), `../b_penthouse/assets/sdf_v12` (15 tiles). Figures:
`outputs/M41_deployed_v12.png`, `outputs/M42_v12_acceptance.png`.

## What runs where

| stage | interpreter |
|---|---|
| `.lcc2` → PLY | `node /home/limx/SuperSplat/node_modules/@playcanvas/splat-transform/bin/cli.mjs` (v3.1.7) |
| everything numeric | `/home/limx/data/3DGS/.venv_gs/bin/python -s` (gsplat 1.5.3 prebuilt, scipy, skimage, trimesh, fast_simplification, coacd) |
| mujoco + gsplat together | `/home/limx/miniconda3/envs/discoverse/bin/python -s` |
| MuJoCo 3.2.3 + `libsdfvoxel.so` | docker `guide-runtime:1.6.0.20260805004358` |

`-s` is mandatory: `~/.local/lib/python3.10/site-packages` leaks into the conda envs
and breaks Pillow with `module 'PIL.Image' has no attribute 'register_open'`.

The existing `libsdfvoxel.so` in `sim_enhancement_mujoco/prebuild/mujoco_plugins/` is
already built against 3.2.3 — **no plugin rebuild needed** (unlike the 5F chapter).

---

## Commands

```bash
CLI="node /home/limx/SuperSplat/node_modules/@playcanvas/splat-transform/bin/cli.mjs"
V=/home/limx/data/3DGS/.venv_gs/bin/python
cd /home/limx/data/3DGS/scenes/LCC_studio/PentHouse

# 0. lcc2 -> PLY. -r 90,0,0 undoes the reader's LCC2_TRANSFORM; env.sog joins as a
#    second input because it is already native Z-up (see the Rz180 trap below).
$CLI -L 0 meta.lcc2 -r 90,0,0 data/3dgs/env.sog -w …/assets/penthouse_gs_sh3_zup.ply
$CLI -L 0 meta.lcc2 -r 90,0,0 -N -w …/cache/penthouse_lod0_zup.ply

cd …/D_lcc2sim/penthouse
$V -s src/lcc_io.py --footprint      #  4 s  PLY -> cache/gauss.npz + measured constants
$V -s src/measure.py                 # 16 s  -> outputs/M2_scene_constants.png
$V -s src/freespace.py --n 10        #       -> cache/freespace.json (stations, spawn)

cd ../b_penthouse                    # pipeline B clone, config.py re-measured
$V -s src/gs_carve.py                # 22 s  [GPU] -> cache/occupancy.npz
$V -s src/bake_sdf.py                # 122 s -> cache/sdf_global.npy + surface_dense.ply
$V -s src/tile_sdf.py                # 43 s  -> assets/sdf/tile_*.npz  (42 tiles, 471 MB)

cd ../penthouse
$V -s src/mesh_clean_fast.py ../b_penthouse/cache/surface_dense.ply \
     --out cache/pB_clean.ply --visual cache/pB_visual.ply     # 21 s
$V -s src/decompose_coacd.py         # 22 s  -> assets/coacd/part_*.obj (A/B only)
python3 -s src/compose_scene.py --mode sdf --out scene.xml

# validation
docker run --rm --gpus all -v /home/limx:/home/limx -w $PWD \
  guide-runtime:1.6.0.20260805004358 bash -lc 'python3 src/drop_test.py --xml scene.xml --tag SDF'
MUJOCO_GL=egl /home/limx/miniconda3/envs/discoverse/bin/python -s src/gs_check.py
```

Whole chain ≈ **5 minutes** of compute, excluding the one-off 19-minute dead end.

---

## PentHouse constants (measured — do not carry these to another scan)

| | value | how |
|---|---|---|
| footprint | **41.6 × 54.0 m** | largest connected component of occupied XY columns, 98.8% of them |
| occupied area | **612 m²** = 265 indoor + **348 open roof terrace** | structure above floor+2.6 m |
| `FLOOR_Z` | **−0.896** | median of 12,840 column minima |
| `FLOOR_SURFACE` | **−0.724** | median per-column top of the floor band |
| structure top | **6.85 m** | top of the contiguous dense band; 543 gaussians (0.048%) sit past a 2 m gap |
| grid @ 4 cm | 1044×1354×209 = **295 M** voxels, 1.18 GB | |
| tiles | 6×7 = **42** | 8 m |
| cameras kept | **3,373 / 4,725** | 0.40 m < clearance < 6.0 m |
| σ-gate survival | **35%** of alpha-passing px | 15F 27%, 5F 49% |
| `COL_PLANE_Z` | **−0.750** | ★ measured zero crossing −0.760 + 1 cm |
| `YAW_DEG` | measured −4.00, **applied 0** | only +14.3% over 0° (15F's +13.5° was +160%) |

**More than half the walkable area is an open roof terrace.** Two stages had to change
for it: `gs_carve` orbit heights became floor-relative and its nadir sweep runs at
three heights (2.10 / 4.20 / 6.90) instead of one, because a single sweep plane cannot
see under a 2.9 m ceiling, under a 6 m double-height ceiling, and over open sky.

### The constant that must be measured

`COL_PLANE_Z`. Deriving it as `FLOOR_SURFACE + 0.01` = −0.714 puts the plane **36 mm**
above the field's actual zero crossing (−0.760, p5..p50 all −0.760, 80.3% within ±5 cm)
and every dropped object rests in mid-air. 15F recorded the same failure at 3 cm.

---

## Results

### SDF field
| check | PentHouse | 15F |
|---|---|---|
| \|SDF\| on the surface (trilinear) | **0.73 mm**, p95 3.16 mm | 0.72 mm, p95 3.34 |
| sign at camera centres | **100% +**, min +0.162 | 100% +, min +0.045 |
| sign inside the floor slab | **100% −**, max −0.064 | 100% −, max −0.077 |
| flood-fill air components / seeded | **225 / 1** | 66 / 1 |
| solid fraction of the box | **7.0%** | 10.4% |

### Physics (MuJoCo 3.2.3, 60 balls r=60 mm dropped into voids under furniture)
| | SDF tiles **+ backstop plane** | **SDF tiles ONLY** | CoACD hulls |
|---|---|---|---|
| collision geoms | 42 | 42 | 64 |
| solid volume | 1,318 m³ | 1,318 m³ | 3,644 m³ (2.8×) |
| ncon at spawn | 0 | 0 | **60** — every ball starts inside a hull |
| reached the real floor | 60 / 60 | **60 / 60** | 7 / 60 (after being ejected) |
| fell through | 0 | **0** | 0 |
| SDF at resting centre (r = 60) | 70.7 mm | **60.2 mm** | 245.6 mm |
| i.e. offset from contact | +10.7 mm | **+0.2 mm** | +185.6 mm |

★ **Read the middle column, not the left one.** With the analytic backstop plane in the
scene the balls come to rest ON THE PLANE, 10.7 mm before they ever touch the SDF — so
that column measures the helper plane, not the field. Only the plane-less run
(`scene_noplane.xml`) validates SDF contact accuracy, and it lands at +0.2 mm.
15F reports the same pair: 60.1 mm / −0.1 mm SDF-only vs 70.6 mm / −10.6 mm with plane.

15F's with-plane column is 70.6 mm. A living room is **not** more forgiving than an
office floor — the furniture is lower and denser, so hulls swallow the voids under
sofas, planters and coffee tables completely.

Robot (HU_D04_01) at the catalog spawn: compile 12.9 s, ngeom 128, **ncon = 2** at
spawn (15F: 213), base_z finite at step 500. Limp without a controller, as expected.

### Visual mesh
Textured via `B_3dgs2sdf/3dgs2sdf_15f/src/mesh_from_sdf.py`, no code changes, run against
`cache/pB_clean.ply` + `b_penthouse/cache/gauss.npz`. 8192² atlas, 34.3% filled after
4 rounds of seam dilation, 41.5% orphan texels (the synthetic envelope and the
out-of-building floor have no gaussians by construction). MuJoCo ignores the OBJ's .mtl —
the atlas is wired as an explicit `<texture>` + `<material>` on the visual geom.

### 3DGS camera layer
`gs_check.py` drives guide_ws's shipped `factory` / `worker_manager` / `strategy` /
`compositor` unchanged. Worker ready in 21 s; `3dgs_only` **8.29 ms/frame**,
`hybrid` **21.55 ms/frame** at 640×480, **0 fallback frames**.

`mj_3dgs_penthouse` is the **first scene in this repo to declare a `gs:` block** — the
machinery was fully written and tested but never switched on.

⚠️ **Scope of that measurement.** `gs_check.py` synthesizes its own minimal MJCF
(robot + backstop plane + visual mesh) — it deliberately omits the 42 SDF tiles, because
the plugin is 3.2.3-only and the working gsplat lives next to MuJoCo 3.10. So the
8.29 / 21.55 ms and "0 fallback frames" are true of the gs worker and of the camera
math, but were NOT measured on the deployed `scene.xml`, and the run predates the
texture bake (its `mujoco_only` panel is the untextured grey fallback). An end-to-end
number on the real scene needs the runner path (`gs_compare_render.py`) inside a
container that has both a 3.2.3-compatible plugin and a working gsplat.

#### The ply was in the pre-de-warp frame; the de-warp is now baked into it

v11 de-warped the mesh and `cache/gauss.npz` by z' = z − W(x,y). The shipped
`penthouse_gs_sh3_zup.ply` was never touched, so the camera layer and the collision
layer sat one drift field apart.

**What the difference actually was.** Matching the ply against the pre-de-warp backup
`cache/gauss_warped.npz`: nearest-neighbour **p50 0.00 mm**, Δz p50 +0.00 mm. Against
the de-warped `gauss.npz`: p50 21.6 mm. No rotation, no translation, no unit error —
W was the whole of it. (The ply carries 5,937,354 gaussians to the npz's 5,528,259;
the npz is the cropped subset, not a different export.)

**Where the correction goes — the part that is easy to get wrong.**
`sensors/gs/strategy.py:_anchor_t_world_scan` builds T_world←scan from the anchor
body's `body_pos`/`body_quat` and nothing else, so a *rigid* correction is free: a
plane through W (over the indoor mask, at the 4 m smoothing `dewarp.py` applies) is
78.4 mm down plus 0.284° of tilt and takes W's RMS from 68.9 mm to 15.9 mm. That was
built, measured, and **backed out**, because of what the reference scenes do:

| | anchor body | SDF tile bodies | floor plane |
|---|---|---|---|
| 5f | `body_3dgs_visual`, identity | identity | bare geom in worldbody, z = −0.14 |
| 15F | `body_3dgs_visual`, **posed** | **same pose on every tile** | bare geom, z = −0.140, untransformed |

15F's transform is `pos="0.624 0.825 0.000" quat="0.99307 0 0 0.11754"` — yaw 13.5°
with **z = 0 and a z-only quaternion**. It is horizontal by construction, which is
exactly why its floor plane, a bare geom outside every body, needs no transform.
Ours is the opposite: vertical, 78 mm plus a tilt. Carried on the bodies the 15F way
it would move the 15 SDF tiles off the analytic plane and undo v11's three-layer
alignment; carried on `body_3dgs_visual` alone it would drag the visual mesh off the
collision floor. And `body_3dgs_visual` is not a free choice of name — it is
`sensors/gs/config.py:DEFAULT_ANCHOR_BODY`, asserted in `test_gs_schema.py`.

So the correction goes in the data (`src/dewarp_ply.py`) and the scene keeps 5f's
shape: `body_3dgs_visual` at identity, no `anchor_body` override needed.

**Measured on the deployed mesh's up-facing near-horizontal faces** (113,918 of them;
a vertical error is invisible on a vertical wall, so splitting by orientation is what
makes this measurable at all — the unsplit number moves only 47.1 → 44.7 mm and looks
like nothing is happening):

| | \|Δz\| p50 | p5..p95 | RMS |
|---|---|---|---|
| original ply (what v12 shipped) | 25.2 mm | −49..+83 mm | 43.0 mm |
| rigid anchor (built, then backed out) | 16.1 mm | −68..+40 mm | 34.7 mm |
| **de-warped ply** | **13.2 mm** | −61..+39 mm | 32.0 mm |

Baking it in is not merely convention-compliant, it is better: the rigid plane
over/under-corrects by up to 40 mm where W departs from it, and on the render overlay
it made two of six stations *worse*. The de-warped ply improves five of six and
worsens none:

| station | original ply | de-warped ply |
|---|---|---|
| fireplace tier 1 | 24 mm | **0 mm** |
| sofa tier 1 | 58 mm | **0 mm** |
| curtain wall | 37 mm | **16 mm** |
| bedroom tier 2 | 17 mm | **6 mm** |
| floor, green box | 29 mm | **6 mm** |
| kitchen bar | 112 mm | 119 mm — alpha 0.24, view is mostly empty, unreliable |

The residual floor is the geometry's, not the alignment's: on vertical faces, where no
vertical correction can do anything, every variant reads −18 to −21 mm, because the
gaussians sit *inside* the mesh — the isosurface inflates outward, as 15F also saw.

**Verification of the rewrite.** 58 of the ply's 59 properties are byte-identical to
the source; only z moved (p5..p95 −112.9..+43.3 mm, every gaussian non-zero). Rotations,
scales and all 45 SH rest coefficients are untouched — a pure translation leaves the
view-dependent term alone, which is a second reason to prefer it to a rotation
(`strategy.py` transforms the camera into scan frame rather than the gaussians, so it
never rotates SH either). End to end: the de-warped ply matches `cache/gauss.npz` at
nearest-neighbour **p50 0.00 mm**, the same signature the original had against
`gauss_warped.npz`.

`dewarp_ply.py` clamps to the grid edge outside the drift field, byte-for-byte the
same as `dewarp.py`. Zeroing there instead is tempting — there is no measurement out
on the terrace — but it puts a step at the grid edge and lands the ply in a different
frame from `gauss.npz`, which is the thing being fixed. 63.0% of the gaussians fall
inside the grid; the rest are extrapolated.

Shipped as `assets/penthouse_gs_sh3_zup_flat.ply` (1.40 GB). Figures:
`outputs/M45_gs_align.png` (the rigid attempt), `outputs/M46_gs_ply_align.png` (shipped).

### Glass openings — straightened (optional post-process)

3DGS leaves a glazed opening as a hole with a 5-10 cm fringe of torn triangles:
measured, 33.8% of the mesh's edges are boundary edges and 61.7% of those run
neither vertical nor horizontal. `src/glass_edges.py` + `src/clip2d.py` square the
panes off.

Two attempts, the first wrong and worth recording:
* **Deleting faces inside the pane's bounding rectangle does nothing.** It only moves
  the same ragged boundary outwards -- 2.45% of faces removed, facade ragged fraction
  64.4% -> 64.8%. A straight edge has to be *created*.
* **True clipping works.** Triangles straddling the rectangle are cut on its lines and
  new vertices emitted, with position and UV barycentrically interpolated (exact --
  both are linear over a triangle). The rectangle's exterior is not convex, so it is
  decomposed into four disjoint convex regions (a pinwheel) and Sutherland-Hodgman is
  run on each.

The other thing the first attempt got wrong: a glazed facade is **not a wall with
holes punched in it**. Coverage on the facade planes is only 5-9% -- what survives is
a set of vertical mullion stripes with empty panes between them, so
`binary_fill_holes` finds nothing. The panes are recovered by closing the mullions
into a silhouette and subtracting them back out.

**The guard has to be structural, because the LiDAR eats this mesh.** The taichi
backend skips `type="sdf"` geoms entirely and traces the visual mesh, so every face
deleted here is a missing return on a real object. A guard that capped the *fraction*
of a pane bbox already covered by mesh could not tell a mullion from a sliver: at the
only setting where it straightened anything it also ate a transom (5,414 faces), and
at a safe setting it did nothing (85 faces).

Classify instead. **Real structure = whatever survives a morphological opening at the
mullion scale** (6 cm); the fringe does not. A pane whose bbox touches real structure
is skipped. Measured on the rebuilt mesh:

| | |
|---|---|
| facade cells classified real structure | 4,002 |
| real-structure cells deleted | **0 (0.000%)** |
| fringe cells deleted | 968 |
| surface area lost | 4.79 m² (0.57%) |

Settled: `--rect-thresh 0.35 --struct 0.06` -> 30 panes, 7.4 m², 454 faces removed.
Evidence: outputs/M22_glass_safe.png (and M21_glass_v3.png for the guard sweep that
was rejected).

Output: `assets/penthouse_A_visual_glass.obj` (kept separate; the untreated
`penthouse_A_visual.obj` remains the default).

---

## Traps hit here (all cost real time)

### 1. splat-transform v3.1.7 rotates its filters and voxel output by 180°

`--filter-box`, `--seed-pos`, `.voxel.json` and `--collision-mesh` all evaluate in a
frame with **x and y negated** relative to the PLY read/write frame. PLY→PLY passthrough
is exact, so nothing warns you. Measured: `-B 5,…` (meaning x ≥ 5) returned
x ∈ [−35.86, −5.00]; the collision mesh sat p50 **187.9 cm** from the source gaussians,
**18.35 cm** after negating x,y about the origin.

Workaround: pass the box as `-B -xmax,-ymax,zmin,-xmin,-ymin,zmax`, negate `--seed-pos`,
and transform any voxel/GLB output by `(x,y,z)→(−x,−y,z)` on read.

### 2. `-L -1` silently returns LOD 4

`-L 0,-1` errors out honestly, but `-L -1` quietly hands back 307,650 gaussians (the
coarsest LOD) instead of the 157,250-splat environment. `env.sog` must be passed as a
separate input — and read directly it is **already native Z-up**, so it must NOT get
the `-r 90,0,0` that the `meta.lcc2` path needs.

### 3. 3DGS cannot reconstruct glass, and no threshold recovers it

Looking through a window, 3DGS places its gaussians on whatever is **beyond** the pane,
at that thing's true position. The glazed band's opacity histogram is indistinguishable
from a solid wall's (33.2% vs 35.5% above 0.3), and unioning high-opacity gaussian
centres into the solid added 10k voxels — nothing. The depth-variance gate then removes
what little is left, because a ray through glass is genuinely bimodal.

Result: the envelope was **54% covered at torso height with gaps up to 1.9 m** — wide
open for a planner to route the robot out of a 20-storey window.

Fix (`src/facades.py::close_envelope`): stop trying to recover it and **build** it. A
column is indoors iff it has structure overhead; the line separating roofed columns from
open ones is the envelope, closed by construction. Mark it solid floor→2.6 m, keep the
widest run open as a 1.6 m door so the terrace stays reachable. Torso coverage
**54% → 82%**, head **81% → 93%**.

Two sub-traps: the raw roofed mask is speckled (the scan misses ceiling patches), so it
must be closed + hole-filled + reduced to its largest component first, or you build
walls through the middle of the living room; and per-plane morphological closing only
reaches 67% because the envelope steps between x=10.4 and x=12.0 and turns corners.

### 4. `trimesh.split()` on a 7.28 M-face mesh

Ran 19 minutes and died. `scipy.sparse.csgraph.connected_components` over
`mesh.face_adjacency` does the same labelling in **4 seconds** (`mesh_clean_fast.py`).

### 5. `map:` is a catalog-schema requirement, not a navigation deliverable

**No occupancy map is shipped, by design — navigation is out of scope for this phase.**
`meta.yaml` still carries a `map:` block because the schema requires one: without it
the scene is dropped from the catalog entirely (`skipping invalid scene`), and catalog
membership is what lets the runner discover the `gs:` block. `mj_3dgs_15f` is in exactly
the same state — a `map:` declaration and no map files.

### 6. `lidar_horizontal` is defined by the robot

`HU_D04_01.xml` already carries the site; a scene that also emits one fails to compile
with `repeated name 'lidar_horizontal' in site`. `mj_3dgs_15f/scene.xml` has none
either — the authoring guide's "every scene must contain one" is for hand-built scenes
with no robot of their own.

### 7. `<compiler meshdir>` must come AFTER the robot `<include>`

Otherwise `Error opening file '.../left_ankle_pitch_link.STL'` (lessons.md L7).

---

## Known limitations

1. **Texture resolution is capped by GEOMETRY, not by the atlas.** The visual mesh now
   ships textured (`penthouse_visual_tex.obj` + `penthouse_atlas.png`, 8192², 400k faces,
   TPM 60 = 1.67 cm/texel, 41.5% orphan texels filled neutral grey). Re-baking at TPM 80
   (1.25 cm/texel, 21.3 M distinct texels) is **visually indistinguishable** — see
   `outputs/M4_tpm_ab.png` — and TPM 100 trips the `atlas overflow at chart` guard.
   So 15F's "≤0.8 cm/texel" bar does not bind here: the limit is the 4 cm-derived
   geometry and the gaussian colour sampling. Do not spend effort raising TPM.
   Headlight must stay ambient 1.0 / diffuse 0 (the baked albedo already carries the
   scan's lighting).
2. **The glazing is modelled, not measured.** It is an opaque barrier at the roofed/open
   interface, ±1 voxel of where the real pane is, with one arbitrary 1.6 m door. If the
   real door position matters, punch it explicitly.
3. **Envelope still has 3 gaps > 0.6 m at torso height** (max 1.76 m), one of which is
   the intended door.
4. **Detail ceiling.** 4 cm voxels; chair legs and railings blur into blobs. More faces
   will not help — the information is not in the gaussians either.
5. **`YAW_DEG` measured but not applied.** −4.00° was worth only +14.3% of wall-projection
   sharpness (15F's +13.5° was worth +160%). If a later stage wants the scene squared to
   the axes, `compose_scene.py` applies it as a body quat and never bakes it — but then
   everything reasoning in world coordinates has to apply it too.
6. CoACD variant (`scene_coacd.xml`, `assets/coacd/`) exists only for the A/B and should
   not be deployed.
7. **No navigation map — out of scope for this phase.** `meta.yaml` carries a `map:`
   declaration only because the catalog schema demands one (see trap 5); no `.pgm`/`.yaml`
   is generated and none should be assumed.
