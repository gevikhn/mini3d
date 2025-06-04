# 数学与渲染原理简介

本项目 `mini3d.c` 通过不到一千行的代码实现了一个固定管线的软件渲染器。本文件从数学角度对项目中涉及的向量、矩阵、坐标系等知识做一个简要说明，并在后半部分介绍该渲染器的基本原理。

## 向量运算

代码中使用 `vector_t` 结构体表示四维向量：

```c
typedef struct { float x, y, z, w; } vector_t;
```

常用的向量运算包括加减、点乘、叉乘以及归一化等，例如 `vector_add`、`vector_dotproduct` 和 `vector_crossproduct` 等函数【F:mini3d.c†L45-L76】。
这些运算在模型的几何计算和矩阵变换中频繁出现，特别是叉乘和点乘被用于计算法线、摄像机方向等。

## 矩阵与变换

项目采用 4×4 的 `matrix_t` 来描述各种变换矩阵：

```c
typedef struct { float m[4][4]; } matrix_t;
```

矩阵运算函数包括矩阵相乘 `matrix_mul`、缩放 `matrix_set_scale`、平移 `matrix_set_translate` 以及旋转 `matrix_set_rotate` 等【F:mini3d.c†L160-L200】。此外还有 `matrix_set_lookat` 构造视图矩阵，`matrix_set_perspective` 构造投影矩阵等【F:mini3d.c†L203-L241】。

## 坐标系与齐次坐标

渲染流程中使用 `transform_t` 保存世界、视图和投影矩阵，并在 `transform_update` 中组合为整体变换矩阵【F:mini3d.c†L248-L261】。顶点经过 `transform_apply` 得到裁剪空间坐标，再通过 `transform_homogenize` 归一化到屏幕空间【F:mini3d.c†L274-L299】。

这里的 `w` 分量用于齐次坐标运算，它既能描述透视关系，也用于视锥裁剪的判断 `transform_check_cvv`【F:mini3d.c†L279-L289】。

## 软件光栅化渲染原理

### 顶点处理

1. **模型变换**：顶点先后乘以世界、视图和投影矩阵，得到裁剪空间坐标。
2. **齐次除法**：通过 `transform_homogenize` 将裁剪空间坐标转换到屏幕坐标并计算 `rhw`（`1/w`），为后续透视校正插值做准备。

### 组装与裁剪

三角形顶点经过裁剪测试后会被拆分为 1 或 2 个梯形（trapezoid）【F:mini3d.c†L360-L424】。每个梯形包含两条边的信息，用于后续扫描线填充。

### 光栅化

`device_render_trap` 会根据梯形生成扫描线，调用 `device_draw_scanline` 完成实际填充。该过程会依据渲染状态决定是否进行线框绘制、纹理贴图或颜色填充【F:mini3d.c†L652-L690】。

在扫描线内部，会对颜色、纹理坐标等属性进行透视校正插值，并与深度缓存比较后写入帧缓冲【F:mini3d.c†L626-L645】。

### 深度缓存

`device->zbuffer` 存储每个像素的 `rhw` 值，绘制像素时只有更大的 `rhw`（即更靠近相机）才会更新缓冲区【F:mini3d.c†L623-L645】。这样即可实现基本的遮挡消除。

### 渲染循环

在 Win32 环境下，`screen_init` 初始化窗口和帧缓冲，随后在主循环中不断清除屏幕、绘制三角形、更新窗口，从而实现实时渲染【F:mini3d.c†L533-L618】【F:mini3d.c†L793-L949】。

## 总结

本文件从数学基础到渲染流程简要介绍了项目中使用的核心概念。结合代码阅读，可更好地理解软件光栅化渲染器的实现细节。 
