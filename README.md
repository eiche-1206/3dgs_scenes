# 3dgs_scenes

MuJoCo 3DGS 扫描场景包，4 个。

| 场景 | 说明 | 资产 | 点云 ply |
|---|---|---|---|
| `mj_3dgs_5f` | 办公楼 5F，约 76×42 m，60 块 sdfvoxel 碰撞瓦片 | 333 MB | 1783 MB |
| `mj_3dgs_15f` | 办公楼 15F，约 40×74 m，60 块 sdfvoxel 碰撞瓦片 | 330 MB | 1258 MB |
| `mj_3dgs_hall` | XGrids LCC2 会议厅，L 形地面 727.9 m²，走廊层高 2.79 m / 宴会厅 5.24 m | 313 MB | 330 MB |
| `mj_3dgs_penthouse` | 顶层公寓扫描，室内约 24×33 m，8 块视觉网格 + 15 块 sdfvoxel 碰撞瓦片 | 402 MB | 1336 MB |

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

## 高斯点云 `.ply`

各场景的 `assets/*_gs_sh3_zup*.ply`（合计 4.7 GB）走 **Git LFS** 存放，普通 `git clone` 只会拿到 132 字节的指针文件。

取回实际点云：

```bash
git lfs install
git clone https://github.com/eiche-1206/3dgs_scenes.git
# 已经 clone 过的：
git lfs pull
```

只要某个场景的：

```bash
git lfs pull --include='mj_3dgs_hall/**'
```

缺 ply 时场景不会报错，`meta.yaml` 的 `gs:` 段声明了降级行为——整场自动转为 `mujoco_only`，碰撞层与视觉网格照常工作，只是没有高斯相机渲染。

## 变体场景

`_crowd` / `_obstacle` 等变体、以及各场景的地图包（`*_maps`）均不在本仓库。
