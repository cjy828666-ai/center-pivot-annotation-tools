# 中心支轴灌溉系统人工目视解译软件 v2.0

本项目是一个面向中心支轴灌溉农田解译任务的 PyQt6 桌面软件。软件定位是“人工目视解译辅助工具”，不是全自动识别系统。

当前主流程为：

```text
影像加载 -> 图层管理 -> 三点定圆标注 -> 边界辅助拟合 -> 属性表联动 -> 项目保存恢复 -> 结果导出 -> 数据集导出
```

## 软件定位

软件主要解决遥感影像中中心支轴灌溉农田人工判读过程中的几个实际问题：

- 多景 TIFF/GeoTIFF 影像加载后需要统一显示、浏览和管理。
- 人工标注圆形或近圆形农田时，需要比普通绘图工具更稳定的几何约束。
- 标注结果需要自动计算中心坐标、半径和面积，并与属性表同步。
- 标注过程需要可保存、可恢复、可导出，便于后续项目复查、成果统计和深度学习数据集制作。
- 对边界较模糊、内部纹理干扰明显的圆形农田，需要在三点定圆基础上提供辅助边界拟合，但不能替代人工判断。

## 当前核心功能

### 1. 项目管理

- 支持新建项目、打开项目、保存项目。
- 项目目录包含：
  - `imagery/`：影像数据。
  - `labels/`：标注数据。
  - `exports/`：导出结果。
  - `cache/`：缓存文件。
  - `project.json`：项目配置。
- 标注数据可写入 `labels/project.gpkg`，也会生成兼容用的 JSON 快照。

### 2. 影像加载与地图浏览

- 支持单景 TIFF/GeoTIFF 加载。
- 支持一次选择多景影像并加载到同一地图场景。
- 支持按目录导入 TIFF/TIF。
- 支持同坐标系多景影像按 GeoTransform 对齐显示。
- 支持图层显隐、顺序管理和图幅网格辅助检查。
- 大图支持异步瓦片渲染，缩放和平移时减少界面卡顿。

### 3. 图层树管理

- 左侧图层树区分影像图层、标注图层和参考数据。
- 每个标注对象在创建时会记录所属影像图层：
  - `source_raster`
  - `source_layer_id`
- 多影像场景下，后续微调和导出会根据标注绑定的影像图层处理，而不是默认使用最后加载的影像。

### 4. 三点定圆人工标注

- 左键依次点击 3 个边界点生成圆形农田标注。
- 两点后显示辅助预览圆、中心点和引导线。
- `Backspace` 撤销最后一个锚点。
- `Esc` 清空当前锚点。
- `Delete` 删除选中标注。
- `Ctrl+Z` / `Ctrl+Shift+Z` 支持撤销和重做。

### 5. 微调模式：径向主动轮廓辅助拟合

当前微调模式默认使用“基于三点定圆几何先验的径向主动轮廓边界辅助拟合”，相关代码位于：

- `utils/boundary_refinement.py`
- `ui/main_window.py`

算法特点：

- 不做全图自动检测。
- 不接入 YOLO。
- 不替代标准三点定圆结果。
- 在三点定圆的圆心和半径基础上，只允许边界在小范围内径向调整。
- 使用极坐标表示边界：

```text
P(theta_i) = C + r_i * [cos(theta_i), sin(theta_i)]
```

能量项包括：

- 影像边缘能量。
- 半径先验约束。
- 一阶平滑约束。
- 二阶曲率约束。
- 三个人工点击点的弱锚点约束。

显示逻辑：

- 绿色实线/半透明填充仍然表示标准三点定圆主标注。
- 青色虚线表示辅助拟合轮廓。
- 拟合失败时保留标准圆，不删除标注，不清空场景。

旧的 Canny/OpenCV 微调代码没有删除，但已经降级为 legacy/debug 对照逻辑；默认情况下不会自动执行 Canny 回退。

### 6. 坐标转换与 ROI 裁剪

边界拟合不再使用 `get_basemap_gray_array()` 作为输入，因为该函数返回的是缩略预览图数组，不能和原始 TIFF 像素坐标直接混用。

当前边界拟合数据流为：

```text
QGraphicsScene 坐标
-> 命中当前标注所属 raster layer
-> image item / root item 的 mapFromScene
-> 图层局部坐标
-> 原始 TIFF pixel 坐标
-> 裁剪 TIFF ROI
-> ROI 局部坐标
-> refine_radial_boundary()
-> polygon ROI 坐标
-> polygon TIFF pixel 坐标
-> polygon scene 坐标
-> QGraphicsScene 显示
```

调试日志会输出：

- `source_raster`
- `layer_id`
- `scene_center`
- `local_point_after_mapFromScene`
- `pixel_center`
- `radius_scene`
- `radius_pixel`
- `tiff_size`
- `image_array.shape`
- `roi_bbox`
- `roi_shape`
- `center_roi`
- `failure_reason`

日志位置：

- `logs/circular_farm_interp.log`
- 终端中也会打印 `RADIAL_REFINEMENT_DEBUG ...`

### 7. 属性表与统计

- 下方属性表展示编号、类别、中心坐标、半径、面积和辅助统计。
- 场景选择与表格选择双向联动。
- 有地理参考时显示米和公顷。
- 无地理参考时回退为像素单位。
- 右侧解释概览显示标注数量、总面积、平均统计和类别分布。

### 8. 导出能力

支持导出：

- 当前视图图片。
- JSON 标注结果。
- GeoPackage 标注结果。
- GeoJSON 标注结果。
- Shapefile 标注结果。
- CSV 统计报告。
- 深度学习数据集：
  - Patch JSON。
  - YOLO 检测框。
  - YOLO 分割。
  - COCO instances。
  - 可选二值圆形掩膜。

如果标注存在辅助拟合边界：

- JSON 中保存 `refined_boundary` 和 `refinement_metrics`。
- GPKG 中保存标准圆图层，并额外写入 `refined_boundaries` 图层。
- GeoJSON 中通过 `geom_role` 区分 `standard_circle` 和 `refined_boundary`。

## 运行环境

建议环境：

- Windows 11
- Python 3.10
- PyQt6
- NumPy
- SciPy
- GDAL/OGR（推荐，用于 GeoTIFF、GPKG/SHP、真实 TIFF ROI 读取）

安装依赖：

```bash
pip install -r requirements.txt
```

当前虚拟环境运行：

```bash
.\Scripts\python.exe main.py
```

## 关键依赖说明

### SciPy

用于径向主动轮廓优化：

- `scipy.optimize.minimize`
- `scipy.ndimage.gaussian_filter`

如果缺少 SciPy，软件不会崩溃，但微调模式会提示边界辅助拟合失败并保留原始三点定圆结果。

### GDAL/OGR

用于：

- 读取 GeoTIFF 原始影像窗口。
- 读取 GeoTransform 和 CRS。
- 多景影像按地理坐标对齐。
- 导出 GPKG/SHP。

### OpenCV

仅保留为 legacy/debug 对照算法，不作为默认微调路径。

## 当前验证状态

已通过：

```bash
.\Scripts\python.exe -m py_compile utils\boundary_refinement.py utils\annotation_utils.py core\project_annotation_store.py ui\main_window.py main.py
```

径向主动轮廓调试测试：

- 标准圆 + 内部纹理：成功。
- 轻微不规则圆：成功。
- 无边界梯度图像：失败并回退。

仍建议人工复测：

- 单景影像中的微调定圆。
- 多景影像中不同图层上的微调定圆。
- 保存项目后重新打开，检查辅助拟合轮廓是否恢复。
- 导出 GPKG/GeoJSON 后检查 `refined_boundary` 是否存在。
