**蒙皮矩阵调色板（Skinning Matrix Palette）是一个角色所有关节的蒙皮矩阵 $\mathbf{K}_j$ 组成的数组。每帧在 CPU 上算好后整体交给 GPU，顶点只存几个指向这个数组的索引和对应权重，顶点着色器按索引"查表"取矩阵完成蒙皮。名字来自索引色图像的调色板：像素只存颜色编号，真正的颜色在调色板里。线性混合蒙皮也因此常被称为矩阵调色板蒙皮（matrix palette skinning）。UE 里调色板按渲染 Section 划分，每个 Section 通过 `BoneMap` 把局部索引映射到骨架骨骼。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（Skinning and Matrix Palette Generation）；UMD CMSC 425 蒙皮讲义；UE 部分对照 5.x API（`FSkelMeshRenderSection`、`GPUSkinVertexFactory.h`）及论坛讨论。本文采用行向量约定。

## 为什么是"调色板"

蒙皮有两种分工方式。一种是 CPU 蒙皮：CPU 每帧把所有顶点变换好，再把整个顶点缓冲上传给 GPU。一个 5 万顶点的角色，每帧光位置和法线就要传一两 MB，而且 CPU 要做几十万次矩阵—向量乘法。

另一种是 GPU 蒙皮：CPU 只算每个关节的蒙皮矩阵，一个 100 根骨骼的角色每帧只需上传 100 个矩阵（3×4 格式约 4.8 KB），顶点缓冲是静态的，从不更新。顶点着色器读取顶点的关节索引，从矩阵数组里取出对应矩阵，完成混合。

UMD 的讲义把这点说得很清楚：骨架可能只有几十个关节，模型却有成百上千个顶点；每个关节只需要一个矩阵，每个顶点只需要一小组关节索引和权重，而不需要为每个顶点关联完整的矩阵。原笔记说调色板"减少 CPU 到 GPU 的数据传输"，指的正是和 CPU 蒙皮相比，每帧上传的数据从"所有顶点"变成"所有关节矩阵"。

## 数据布局

每帧的调色板生成（见 [[EAADH-蒙皮矩阵]]）：

$$
\mathbf{K}_j = \mathbf{B}_{j\to M}^{-1}\,\mathbf{C}_{j\to M},\qquad j = 0,\dots,N-1
$$

顶点属性里存关节索引 $(j_1, j_2, j_3, j_4)$ 和权重 $(w_1, w_2, w_3, w_4)$，着色器里做

$$
\mathbf{v}' = \mathbf{v}\left(\sum_{i=1}^{4} w_i\,\mathbf{K}_{j_i}\right)
$$

几个常见的优化：

**用 3×4 矩阵。** 仿射变换的第 4 列（行向量约定下）恒为 $(0,0,0,1)^T$，只存 12 个浮点数就够了，比 4×4 省 25% 的带宽和寄存器。GEA 在讨论 SRT 格式时也提到 4×3 矩阵需要 12 个浮点数。

**用对偶四元数或 SRT 代替矩阵。** 对偶四元数只要 8 个浮点数，还能避免 LBS 的塌陷，但着色器里要多做转换。

**预乘模型到世界矩阵。** 把 $\mathbf{M}_{M\to W}$ 乘进每个 $\mathbf{K}_j$，着色器里省一次乘法；但动画实例化（多个角色共用一份调色板）时不能这样做。

下面是顶点着色器里的混合示意，用 HLSL 风格书写。它是说明原理的简化代码，**不是** UE 的 `GpuSkinVertexFactory.ush` 源码：

```hlsl
// 矩阵调色板蒙皮示意：BonePalette 每根骨骼 3 个 float4，存 3x4 蒙皮矩阵的转置
StructuredBuffer<float4> BonePalette;

float3x4 FetchBone(uint Index)
{
	return float3x4(BonePalette[Index * 3 + 0], BonePalette[Index * 3 + 1], BonePalette[Index * 3 + 2]);
}

float3 SkinPosition(float3 Pos, uint4 Indices, float4 Weights)
{
	float3x4 M = FetchBone(Indices.x) * Weights.x
	           + FetchBone(Indices.y) * Weights.y
	           + FetchBone(Indices.z) * Weights.z
	           + FetchBone(Indices.w) * Weights.w;
	return mul(M, float4(Pos, 1.0)); // 以转置形式存储，这里用列向量乘法
}
```

CPU 端存的是行向量约定的矩阵，上传时转置成"3 行 × 4 列"再按列向量乘，这样只需要 3 个 float4，这也是很多引擎的做法。写自己的蒙皮着色器时，存储布局和 `mul` 的参数顺序必须和 CPU 端对应，否则得到的是转置矩阵的结果，旋转方向会反。

## 调色板大小的限制与分段

早期 GPU 通过常量寄存器传调色板，寄存器数量有限（例如 Shader Model 2/3 时代顶点着色器常见的是 256 个 float4 常量），一个 3×4 矩阵占 3 个寄存器，再扣掉其他常量，一次绘制只能容纳几十根骨骼。骨骼更多的角色要把网格按骨骼分成若干段（section / chunk），每段只引用一部分骨骼，单独提交一次绘制，并为这一段建一个局部调色板。顶点里的骨骼索引是段内局部索引，每段有一张表把局部索引映射到骨架的全局骨骼索引。

现在调色板一般放在结构化缓冲或纹理缓冲里，容量不再是瓶颈，但"按段划分、段内局部索引"这套结构在很多引擎里保留了下来，因为它和材质分段天然对齐，也让 8 位索引够用。

## 在 UE 里

UE 的骨骼网格体渲染数据按 LOD 和 Section 组织。每个 `FSkelMeshRenderSection` 有一个 `BoneMap`：顶点蒙皮权重缓冲（`FSkinWeightVertexBuffer`）里存的骨骼索引是 Section 内的局部索引，通过 `BoneMap[LocalIndex]` 才能得到骨架的骨骼索引。论坛上常有人直接用权重缓冲里的索引去查骨骼名字，结果对不上，原因就在这里。

每帧的调色板来源是 ReferenceToLocal 矩阵数组（逆参考矩阵乘组件空间变换，见 [[EAADH-蒙皮矩阵]]）。渲染线程按每个 Section 的 `BoneMap` 从这个数组里取出所需的矩阵，以 3×4 的形式写进骨骼矩阵缓冲，供 GPU 蒙皮顶点工厂（`GPUSkinVertexFactory.h` 中声明，影响类型有 `DefaultBoneInfluence` 和 `UnlimitedBoneInfluence` 两种）读取。上传细节和缓冲格式随版本变化，以源码为准。

有两种蒙皮执行路径：默认在各渲染通道的顶点着色器里做蒙皮；开启 GPU Skin Cache 后，改为用计算着色器每帧蒙皮一次，把结果写进缓冲，深度、阴影、BasePass、光线追踪等通道直接复用。硬件光线追踪要求骨骼网格体开启 Skin Cache，无限骨骼影响模式也依赖它。

Leader Pose 让多个子网格体共用主网格体的组件空间骨骼变换，但每个子网格体仍然用自己的逆参考矩阵生成自己的调色板，因为它们的参考姿势可能不同。

## 容易踩的坑

**把局部索引当全局索引。** 顶点里的骨骼索引是段内（Section 内）索引，必须经过 BoneMap 映射。

**CPU 与着色器的矩阵布局不一致。** 行主序 / 列主序、行向量 / 列向量、是否转置，四个因素组合起来很容易错一个。最简单的检查是绑定姿势：所有调色板矩阵都应该是单位矩阵。

**调色板里夹带了不需要的骨骼。** 为每个 Section 上传整副骨架的矩阵是浪费，只上传 BoneMap 里用到的即可。UE 的 LOD 还会裁掉低 LOD 不需要的骨骼。

**用调色板做 CPU 端的精确命中检测。** 读回蒙皮结果或在 CPU 上重算都很贵，大多数命中检测应该用物理资产的简单碰撞体，而不是逐顶点。

## 相关

[[EAADH-蒙皮矩阵]] [[EAADD-多骨骼的蒙皮权重]] [[EAADL-权重蒙皮的混合]] [[EAADB-单关节蒙皮]] [[EAABH-3D蒙皮动画]] [[BDGB-渲染技术]]
