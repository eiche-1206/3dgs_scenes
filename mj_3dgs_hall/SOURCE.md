# mj_3dgs_hall — XGrids LCC2 ConferenceHall，从零重做

扫描源 `/home/limx/data/3DGS/scenes/LCC_studio/ConferenceHall/meta.lcc2`
（XGrids，`version 0.0.3`，LOD0 6,174,150 个高斯，**SH degree 0**）。
工作树 `/home/limx/data/3DGS/scenes/CH/`，工具 `3dgs2sim` 仓库（镜像同名）。
构建于 2026-08-20/21。**尚未部署。**

> ⚠️ 这个 id 上曾有过另一个场景，走的是 LiDAR 网格 + 高斯填充、2 cm 体素的路线，
> 已废弃并移除。本包与它没有任何共用文件：碰撞是纯高斯空间雕刻、4 cm，
> 由 3dgs2sim 管线从零重做。

## 内容

| | |
|---|---|
| 碰撞 | 36 块 sdfvoxel 瓦片，4 cm，纯高斯空间雕刻 + 合成平板 + FLATTEN |
| 视觉 | **8 chunk / 729,721 面**（剥掉地板）+ **独立地板件 60,000 面**，两张 8192² 图集 |
| 相机层 | `CH_gs_sh3_zup_flat.ply`（去漂移，0.35 GB）。**SH degree 0**，要设 `sh_degree: 0` |
| 解析平面 | z = **−0.270** |
| assets | 643 MB |

## 场景形状（决定了一半的设计）

L 形整层，不是「一个会议厅」：走廊/工位带（X −2…8, Y −6…26，吊顶 **2.79 m**）
+ 大厅（X 10…33, Y 19…32，约 25×13 m，藻井 **5.24 m**，最高结构 5.99 m）。
足迹 46.6 × 42.0 m，室内 **727.9 m²**。

**两套层高**因此贯穿全程：碰撞切 2.793（最低密集层），视觉保持全高 6.74，
rig 的俯仰角必须加到 +70° 才够得到藻井。

## 实测常量（**不要搬到别的扫描**）

| 常量 | 值 | 取法 |
|---|---|---|
| `SCENE_MIN/MAX` xy | `[-11.905,-8.431] .. [34.595,35.169]` | 3dgs-probe 最大连通足迹 + 0.6 m |
| `SCENE_MIN` z | **−0.360** | **不是** `FLOOR_Z−0.05`；为体素栅格对齐特意选的，见下 |
| `SCENE_MAX` z | 2.793 | = `Z_MAX_COLLISION` |
| `FLOOR_Z` | −0.332 | 反漂移后室内逐柱最低点中位 |
| `FLOOR_SURFACE` | **−0.270** | `floor_layers.py` 的 `visual_median`（**独立地板件**口径）|
| `FLOOR_SOLID_TO` | −0.300 | → `kz=4`，板顶 −0.280 |
| `FLOOR_NOMINAL_Z` | −0.280 | 第一遍烘焙实测的板顶（= p5 = p25，**不能用中位数**）|
| `Z_MAX_VISUAL` | 6.74 | 反漂移后不透明高斯 z max |
| `YAW_DEG` | **0.0** | 墙面投影锐度峰 0° 收 49.1%，次优 −1.0° 42.4% |

### 那个必须算出来的常量：`SCENE_MIN` 的 z

4 cm 体素把 SDF 零穿越量化成 **40 mm 一档**。按惯例取 `FLOOR_Z−0.05 = −0.382` 时，
可达板顶是 −0.342/−0.302/−0.262，离目标（`FLOOR_SURFACE − 10 mm`）最近 **15 mm**，
**过不了 3dgs-floor 的 5 mm 线**。挪到 −0.360 之后可达 −0.280，误差 0。

**对齐只能靠挪包围盒，调 `FLOOR_SOLID_TO` 没用**——相邻取值落在同一档里。

## 验收

| 判据 | 结果 |
|---|---|
| `3dgs-mjcf` | error 0 warn 0 |
| `3dgs-floor` | **problems 空**：视觉−平面 **0.14 mm**、平整度 **0.11 mm**、SDF−平面 −10.0 mm |
| `3dgs-collision` | 60/60 落地 · 穿透 0 · 幽灵面 0 · t=0 接触 0 · 中位嵌入 −0.11 mm |
| `3dgs-sdf` | 残差中位 3.74 mm（线 8.0）· 梯度模长 1.089 |
| `3dgs-tile` | 边界带 0 / 59,832 重叠 |
| `3dgs-atlas` | 主件 0.65 cm / 填充 99.4%；地板件 0.37 cm / 98.8% |
| `3dgs-mesh` | **不合格**，见「已知问题」 |
| 帧时间 | 1280×800 **2.42 ms**；4000×2560 38.7 ms（填充受限）|

`3dgs-collision` 要用 `--clear 0.10` 跑：默认值下选点函数的半体素缺陷会让
60 个投放点只剩 4 个（见 `skills/3dgs-collision/SKILL.md`）。

## 已知问题

**两处视觉地板洞，共 5.75 m²（0.8%）**，在最西端 (−8.30, +24.12) 和 (−4.88, +18.94)。
格内 100% 的高斯（含低不透明度）与 100% 的网格顶点都在 2.60 m 以上——
**源扫描在那两块的吊顶以下没有数据**，大概率是没进去扫的小房间。跑再多遍也不会变。
碰撞地板在那里**是存在的**（合成平板铺满整个盒子），所以机器人能走进去、
脚下踩着一块看不见的地。

**spawn 还是占位值**，部署前必须用 freespace 实测。

## 复现

完整命令与每一步的实测数字在工作树的 `docs/scenes/ch.md`（3dgs2sim 仓库）。
**通用的教训不在那里**——已分别落到 `docs/lessons/*.md` 和 `skills/*/SKILL.md`。

关键的四步（其余照 `docs/guides/new-scene.md`）：

```bash
# 1. .lcc2 -> ply（镜像里 splat-transform 钉死 3.1.7）
splat-transform -L 0 meta.lcc2 -r 90,0,0 -N -w CH_lod0_zup.ply
splat-transform -L 0 meta.lcc2 -r 90,0,0 data/3dgs/env.sog -w CH_gs_sh3_zup.ply

# 2. 反漂移（本场景漂移 p5..p95 = 265 mm，比 15F 的 180 更重）
#    --ref 是最脆的地方：逐列最低值要先与 4 m 滚动中位比、偏离 >12 cm 的剔除，
#    否则桌面会被采成地板（基准面跨度 604 mm，被 floor_datum 的闸门拦下）
floor_harvest.py --depth pgsr --ref CH_floor_ref.obj
floor_datum.py --build --from-harvest ... --write-warp
dewarp.py --apply --smooth 4.0 --mesh <视觉件>
dewarp_ply.py --src CH_gs_sh3_zup.ply --out assets/CH_gs_sh3_zup_flat.ply

# 3. 独立地板件（默认常量差一个数量级，见 skills/3dgs-floor/SKILL.md）
floor_patch.py --full --strip <剥地板视觉件> --floor-surface -0.2593 \
    --harvest <重采样到补丁栅格的 harvest> \
    --trust-lo 20 --trust-hi 60 --hband 0.025 --resid-smooth 8
#    然后：抽面到 60k（UV 是 (x,y) 的精确线性函数，重算即可）
#          压成近平面（加 1 mm 峰峰正弦，否则 MuJoCo 惯量退化）
#          用 bake_texture_render 单独重烘它的贴图

# 4. 分块 + 合成；compose_scene 接不了第二张贴图，四个元素手工加进 XML
split_visual.py --k 3      # 出 8 块（L 形空掉一格），实测 11.9× 提速
```

## 与 PentHouse 的关系

本次是独立从零推导的，事后发现 **PentHouse 的 SOURCE.md 走过同一条路**
（`v6 — the floor is REPLACED, not patched` / `v7 — the floor's height comes from
the gaussians` / `v11 — the scan's drift is removed` / `v12 — the visual mesh stops
being one geom`）。**下一个场景应当先读那一份**，能省掉大量试错。

## 2026-09-14 —— 删掉门洞里那扇门扇

走廊尽头通宴会厅的门洞里立着一扇门扇：world 系 x 7.78~8.10 / y 24.44~25.52 /
z -0.22~2.54，厚 0.16 m、宽 1.1 m、通到吊顶，从 y≈25.6 的横墙往南伸进门洞，
把 2.15 m 的门洞切成 1.0 m 一条缝。e2e 夜测里宴会厅那侧的 6 个导航点因此全 FAIL。

扫描原样是对的（现场那扇门就半开着），但它让这个场景的一半跑不了导航，所以按
后处理删掉。改两个文件：

- `assets/sdf/tile_02_04.npz` —— 碰撞 SDF，占据体素 230467 → 223943
- `assets/vis_chunks/chunk_01_02.obj` —— 视觉网格，面 192420 → 191231

挖除盒 world x 7.74~8.16 / y 24.18~25.50 / z -0.28~2.52，上下界都避开了
y≥25.53 与 y≤23.45 的两道横墙、x≥8.22 的横墙、z≤-0.30 的地板板、z≥2.54 的走廊吊顶。

**用 CSG 差集 `max(sdf, -sdf_box)`，不是往盒内赋常数**：sdfvoxel 的碰撞检测是在距离场
上做梯度下降的，赋常数会在盒壁上留一圈断崖，梯度指向错误方向。

**没动的**：`assets/CH_gs_sh3_zup_flat.ply`。高斯点云是独立的一层，物理与网格里门没了，
3DGS 相机渲出来门还在。要一致得重新 carve 点云。

脚本（幂等，可 `--dry-run`）：工作区 `oneoff-tools/hall_remove_door.py`。
场景重新生成后需要再跑一次。
