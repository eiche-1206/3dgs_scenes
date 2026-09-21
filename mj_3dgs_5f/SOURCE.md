# mj_3dgs_5f — source & regeneration

MuJoCo environment for the 3DGS-derived 5F SDF scene.

## Source of truth
- Scene assets live beside this wrapper in `assets/`:
  - `5F_3dgs_visual.obj`
  - `5F_3dgs_atlas.png`
  - `sdf/tile_XX_YY.npz` (60 tiles total)
  - `5F_gs_sh3_zup.ply` (~1.9 GB, gaussian splats for the gs camera layer;
    generated locally for now, to be committed in a follow-up)

## 3DGS camera layer (`meta.yaml` `gs:` block)
- The ply is exported from the same NuRec scan the mesh/SDF were baked from,
  in the original Z-up frame — identity-aligned with this scene's world:

## Composition conventions
- `scene.xml` contains only the shared map, SDF collision, textures, lights and world bodies.
- Robot include, mesh directory and spawn are selected from `meta.yaml`.
- The scene materializer combines the selected robot with `scene.xml` in its runtime cache.
- Environment mesh files and `sdfvoxel.grid` NPZ paths are rewritten to absolute paths before
  the environment is copied to the cache.
- The Tron2 entry reuses the `mj_msp_5f` world frame and verified spawn
  `(16.975, -14.115, 0.95, yaw=0.0)`.
- Reuses the existing `mj_msp_5f` occupancy map as `../../maps/mj_3dgs_5f.{pgm,yaml}`.
- The visual OBJ is declared as `assets/5F_3dgs_visual.obj` in the source scene and made
  absolute by the materializer.
- Texture resolves through `texturedir="assets"`, so the atlas is referenced as `5F_3dgs_atlas.png`.
- The visible/primary collision floor is the same `floor5f_mat` plane at `z=0` used by
  `mj_msp_5f`. The source SDF slab remains around `z=-0.15` as an unreachable backing surface.
- Every SDF plugin config path remains relative in the source
  (`assets/sdf/tile_XX_YY.npz`) and is made absolute in the materialized environment.

## Runtime requirements
- Requires MuJoCo 3.2.3 with `libsdfvoxel.so` loaded before `MjModel.from_xml_*`.
- The shared environment preserves the source scene's 60 tiled `geom type="sdf"` collision bodies and plugin
  bindings on both the mesh assets and the geoms.

## Local asset edits (diverged from the 3dgs2sdf source)

Hand-applied to the assets in this directory. A re-bake from the source pipeline drops them
unless they are reapplied.

### Corridor floor flattened
- 4 cm voxelisation leaves 1–3 extra voxel layers on the scanned floor, so the collision
  surface stops at `-0.13 / -0.09 / -0.05` instead of the nominal `-0.17`. With the fallback
  plane at `z=-0.14` those ridges poke through and catch the feet — that is why the plane had
  been raised to `-0.05`, at the cost of 12 cm of standing height against the visual floor.
- Every *floor-only* column whose surface sat in `(-0.17, -0.05]` was pushed back to `-0.17`
  over the corridor `x∈[-9.0, 11.3] y∈[-12.5, -8.3]`: 6 tiles (`04_01 04_02 05_01 05_02
  06_01 06_02`), 837 columns (1.34 m²), 884 voxels flipped inside→outside. Walls and
  furniture keep every voxel — their columns run far above `-0.05`.
- 881 of those 884 voxels were already buried under the `-0.05` plane, so the edit changes
  nothing the robot could previously touch. The plane is back at `z=-0.14`.
- Not applied scene-wide: the same ridges cover ~21 m² elsewhere, mostly `x < -29`. Flatten
  those before lowering the plane any further.

### Floating visual fragments removed
- `extract_scene_geometry()` feeds *every* geom to the taichi LiDAR regardless of `contype`,
  and it skips geom type 8 (`mjGEOM_SDF`) entirely — so in this scene the visual mesh is the
  only obstacle source navigation ever sees, and detached scan debris in it becomes a phantom
  obstacle in the costmap.
- Six components welded-disconnected from the main mesh, all floating clear of the floor
  (`z_min` from 0.02 to 1.97 m), were dropped from the left-centre area around
  `x∈[-33.8, -28.8] y∈[-12.2, -9.0]`: 531 of 300000 faces.
- Only the `f` lines were removed; `v/vt/vn/usemtl` are byte-identical, so vertex indices and
  the `5F_3dgs_atlas.png` UV binding are unchanged. Some vertices are now unreferenced, which
  is harmless for a `contype=0` visual mesh.
- Not swept scene-wide: 270 non-main components (20819 faces) remain, 55 of them in the
  0.6–1.4 m band. "Floating" does not imply "debris" — scanned desks and cabinets often lost
  their legs, so deleting those would let the planner route straight through them.

## Known SDF limitations (from the source README)
- 4 cm voxels limit geometric detail; thin features can blob together.
- The source SDF still contains its synthetic floor slab below the wrapper's `z=0` floor plane.
- High-speed small objects can tunnel through the SDF collision at the 2 ms timestep.
- The compiled model is heavy: ~635 MB of SDF grids resident and ~3.1 GB RSS in the source build.
- Keep `sdf_initpoints` conservative; the source notes values above 40 can trigger
  `mjc_SDF: too many contact points`.
