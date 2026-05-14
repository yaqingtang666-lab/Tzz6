
# 基于可微渲染的 3D 重建实验：从球体到奶牛

## 1. 项目简介

本实验利用 **PyTorch3D** 的可微渲染技术，通过 2D 多视角剪影（Silhouette）和 RGB 图像作为监督信号，将一个初始的 **各向同性球体（IcoSphere）** 变形并渲染优化，最终重建出目标对象（奶牛）的 3D 几何形状与表面纹理。

## 2. 核心实验：初始球体细分数（level）的影响

在实验过程中，我通过调整初始球体的细分数（Level）发现它是决定重建精细度和模型拓扑结构的关键参数。通过修改 `ico_sphere(level, device)` 中的 `level` 参数，观察到了明显的重建差异：

| 细分数 (Level) | 实验现象描述 | 对应截图 |
| --- | --- | --- |
| **level=3** | **拓扑不足**：球体面数较少，无法提供足够的自由度来捕捉奶牛的腿部和角部细节，模型整体呈现块状，过渡生硬。 | ![Uploading 小牛3.png…]()
 |
| **level=4** | **均衡状态**：面数适中，能够较好地拟合奶牛的主体轮廓（如耳朵、身体），但在极细微的转折处仍存在轻微的锯齿感。 | <img width="1802" height="1070" alt="小牛4" src="https://github.com/user-attachments/assets/cff10362-afa1-48ab-bd94-f881e4ec5669" />
 |
| **level=6** | **高精拟合**：提供极高的顶点密度，重建结果非常圆润，能捕捉到复杂的边缘起伏。但过高的细分数显著增加了显存负担和计算耗时。 | <img width="1658" height="1032" alt="小牛6" src="https://github.com/user-attachments/assets/c745e47a-82c7-449d-b17c-7e95579b9ed5" />
|

> **结论**：较高的细分数能显著提升几何精度，但若没有配合足够强的拉普拉斯平滑（Laplacian Smoothing），过密的顶点容易在优化中产生局部破碎或“刺猬状”突起。

## 3. 技术管线

1. **初始形状**：生成 `ico_sphere` 作为 Source Mesh。
2. **形变机制**：通过 `offset_verts` 为每个顶点学习一个位移向量。
3. **纹理映射**：利用 `TexturesVertex` 为顶点分配 RGB 颜色。
4. **可微渲染器**：
* `MeshRasterizer`：进行光栅化。
* `SoftSilhouetteShader`：渲染剪影，计算几何形状损失。
* `SoftPhongShader`：渲染带光照的色彩，计算纹理损失。
5. **损失函数**：
* **Image Loss**：$L_{rgb}$ (MSE) + $L_{silhouette}$。
* **Mesh Regularization**：拉普拉斯平滑、边缘长度约束、法线一致性约束。

## 4.核心技术架构

### 1. 几何表示层 (Geometry Representation)

* **初始拓扑 (Initial Topology)**：采用对称的二十面体球体（IcoSphere）作为基础网格。通过调整细分数（Level 3/5/6）来控制顶点密度，从而决定重建精度的上限。
* **形变模型 (Deformation Model)**：学习一个顶点偏移张量 $\Delta V$，通过 `offset_verts` 作用于原始顶点，在保持拓扑连接（Faces）不变的情况下改变几何形状。

### 2. 纹理与材质层 (Texture & Appearance)

* **顶点着色 (Vertex Coloring)**：利用 `TexturesVertex` 将 RGB 颜色信息直接绑定在顶点上。
* **色彩空间映射**：为了保证导出的颜色符合物理规范，对学习到的颜色参数进行 **Sigmoid** 激活处理，将其映射至 $[0, 1]$ 区间。

### 3. 可微渲染引擎 (Differentiable Rendering Engine)

* **光栅化配置 (Rasterization)**：使用 `MeshRasterizer` 将 3D 网格投影至 2D 像素平面。通过设置 `blur_radius` 使得光栅化过程可微，允许梯度回传至几何顶点。
* **多任务着色器 (Shaders)**：
* **SoftSilhouetteShader**：负责提取 2D 剪影。它不考虑光照，仅通过透明度通道（Alpha Channel）约束几何轮廓。
* **SoftPhongShader**：集成 `PointLights`（点光源）模型。通过计算法线与光源的夹角生成带有光影效果的 RGB 图像，用于监督表面纹理的生成。



### 4. 损失函数与正则化 (Losses & Regularization)

为了引导模型在拟合图像的同时保持平滑的几何特征，架构采用了联合损失函数：

* **图像对齐损失**：包含剪影误差（Silhouette Loss）和色彩均方误差（RGB MSE Loss）。
* **几何正则化项 (Constraint)**：
* **拉普拉斯平滑 (Laplacian Smoothing)**：约束相邻顶点的位移量，防止表面出现尖锐毛刺或“刺猬状”破碎。
* **边长偏移 (Edge Alignment)**：抑制三角形边长的过度拉伸。
* **法线一致性 (Normal Consistency)**：确保相邻面片的法向量方向平滑过渡。



