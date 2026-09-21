# 3dgs_scenes

MuJoCo 3DGS 扫描场景包，4 个。

| 场景 | 说明 | 入库大小 |
|---|---|---|
| `mj_3dgs_5f` | 办公楼 5F，约 76×42 m，60 块 sdfvoxel 碰撞瓦片 | 333 MB |
| `mj_3dgs_15f` | 办公楼 15F，约 40×74 m，60 块 sdfvoxel 碰撞瓦片 | 330 MB |
| `mj_3dgs_hall` | XGrids LCC2 会议厅，L 形地面 727.9 m²，走廊层高 2.79 m / 宴会厅 5.24 m | 313 MB |
| `mj_3dgs_penthouse` | 顶层公寓扫描，室内约 24×33 m，8 块视觉网格 + 15 块 sdfvoxel 碰撞瓦片 | 402 MB |

每个包的结构：

```
<scene>/
  scene.xml        MJCF 入口
  meta.yaml        场景元数据（尺寸、建图参数、gs 相机层声明）
  SOURCE.md        扫描来源
  assets/
    *.obj *.mtl *.png   视觉网格与贴图
    sdf/*.npz           sdfvoxel 碰撞场
```

## 未包含：高斯点云 `.ply`

各场景的 `assets/*_gs_sh3_zup*.ply`（330 MB ~ 1.8 GB，合计 4.7 GB）**不在本仓库**，超出 GitHub 单文件 100 MB 上限。

缺 ply 不影响加载——`meta.yaml` 的 `gs:` 段声明了这一行为，整场自动降级为 `mujoco_only`，碰撞层与视觉网格照常工作，只是没有高斯相机渲染。需要相机层时从内网资产服务取对应 ply 放回 `assets/` 即可。

## 变体场景

`_crowd` / `_obstacle` 等变体、以及各场景的地图包（`*_maps`）均不在本仓库。
