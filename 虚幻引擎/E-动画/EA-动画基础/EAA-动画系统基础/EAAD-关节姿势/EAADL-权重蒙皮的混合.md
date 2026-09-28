**权重蒙皮的混合，指的是蒙皮计算中"把一个顶点所受的几个关节变换按权重合成一个变换"这一步。最常见的做法是线性混合蒙皮（LBS）：直接对蒙皮矩阵做加权和，再用这个矩阵变换位置、法线和切线。它简单、快、完全适合 GPU，但线性混合刚体变换会导致关节扭转和大角度弯曲处体积塌陷；对偶四元数蒙皮（DQS）等方法就是为了解决这个问题。UE 默认的 GPU 蒙皮就是 LBS。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章；Kavan et al., *Geometric Skinning with Approximate Dual Quaternion Blending*（TOG 2008）；Le & Hodgins, *Real-time Skeletal Skinning with Optimized Centers of Rotation*（SIGGRAPH 2016）。本文采用行向量约定。

## 混合矩阵还是混合结果

设顶点 $\mathbf{v}$ 受关节 $j_1,\dots,j_n$ 影响，权重 $w_i$，蒙皮矩阵 $\mathbf{K}_{j_i}$（见 [[EAADH-蒙皮矩阵]]）。有两种等价写法：

$$
\mathbf{v}' = \sum_{i} w_i\,(\mathbf{v}\,\mathbf{K}_{j_i}) \qquad\text{或}\qquad \mathbf{v}' = \mathbf{v}\,\Big(\underbrace{\textstyle\sum_{i} w_i\,\mathbf{K}_{j_i}}_{\mathbf{M}}\Big)
$$

前者是"分别变换再加权平均"，后者是"先混合出一个矩阵再变换一次"。矩阵乘法的线性保证两者结果完全相同。GPU 实现几乎都用后者：位置、法线、切线都要变换，先混合出 $\mathbf{M}$ 再各乘一次，比每个关节都变换三个向量再混合省得多。

法线和切线只受 $\mathbf{M}$ 的线性部分（左上 3×3，记为 $\mathbf{M}_{3\times3}$）影响，不受平移影响：

$$
\mathbf{n}' = \operatorname{normalize}\left(\mathbf{n}\,\mathbf{M}_{3\times3}\right)
$$

严格来说法线应该用 $\mathbf{M}_{3\times3}$ 的逆转置变换，只有在 $\mathbf{M}_{3\times3}$ 是旋转乘均匀缩放时两者才一致。但 $\mathbf{M}$ 是几个矩阵的加权和，本身就不是纯旋转，多数实时实现仍然直接用 $\mathbf{M}_{3\times3}$ 变换再归一化，误差在大部分情况下看不出来。

## 流程

一个顶点的蒙皮在 GPU 上是这样完成的：

1. 从顶点属性读出关节索引和权重（见 [[EAADD-多骨骼的蒙皮权重]]）。
2. 用索引从矩阵调色板里取出对应的蒙皮矩阵（见 [[EAADI-蒙皮矩阵调色板]]）。
3. 按权重把矩阵加起来，得到 $\mathbf{M}$。
4. 用 $\mathbf{M}$ 变换位置，用 $\mathbf{M}_{3\times3}$ 变换法线和切线并归一化。
5. 位置再乘视图投影矩阵，输出到光栅化。

```hlsl
// LBS 顶点着色器示意（HLSL 风格，非 UE 源码）；FetchBone 返回转置存储的 3x4 蒙皮矩阵
void SkinVertex(float3 Pos, float3 Normal, float3 Tangent, uint4 Idx, float4 W,
                out float3 OutPos, out float3 OutNormal, out float3 OutTangent)
{
	float3x4 M = FetchBone(Idx.x) * W.x + FetchBone(Idx.y) * W.y
	           + FetchBone(Idx.z) * W.z + FetchBone(Idx.w) * W.w;

	OutPos     = mul(M, float4(Pos, 1.0));                        // 位置带平移
	OutNormal  = normalize(mul((float3x3)M, Normal));             // 方向只用 3x3 部分
	OutTangent = normalize(mul((float3x3)M, Tangent));
}
```

常见的写错之处是法线也用 `float4(Normal, 1.0)` 去乘，把平移带进了方向向量；应该用 `w = 0` 或者只取 3×3 部分。切线的副切线符号（handedness）要原样保留，不参与混合。

## 线性混合为什么会塌陷

LBS 的根本问题是：刚体变换的加权和一般不是刚体变换。

看最简单的情况：一个顶点被两个关节各以 0.5 的权重影响，一个关节不动（单位矩阵），另一个绕顶点所在的轴转了 $\theta$。只看垂直于转轴的平面，两个旋转矩阵的平均是

$$
\tfrac12\big(\mathbf{I} + \mathbf{R}(\theta)\big) = \cos\tfrac{\theta}{2}\;\mathbf{R}\!\left(\tfrac{\theta}{2}\right)
$$

结果确实转了一半角度，这是我们想要的；但它还多了一个 $\cos(\theta/2)$ 的缩放。顶点到转轴的距离被乘上了 $\cos(\theta/2)$：$\theta = 90^\circ$ 时缩到约 0.71 倍，$\theta = 180^\circ$ 时缩为 0。前臂绕自身轴扭转接近 180° 时，手腕附近的一圈顶点被压成一点，像被拧紧的糖纸两端，这就是"糖纸"瑕疵（candy-wrapper artifact）。肘、膝大角度弯曲时，内侧的塌陷也是同一个原因。

## 对偶四元数蒙皮

DQS 不混合矩阵，而是把每个蒙皮变换表示成单位对偶四元数 $\hat{\mathbf{q}}_i$（8 个数，同时编码旋转和平移），加权相加再归一化：

$$
\hat{\mathbf{q}} = \frac{\sum_i w_i\,s_i\,\hat{\mathbf{q}}_i}{\left\lVert \sum_i w_i\,s_i\,\hat{\mathbf{q}}_i \right\rVert}
$$

$s_i = \pm 1$ 是和四元数插值一样的"同半球"符号修正。归一化后的单位对偶四元数一定表示刚体变换，所以不会出现缩放，糖纸问题消失。在上面两关节各 0.5 的例子里，DQS 得到的就是一个纯粹的 $\theta/2$ 旋转。

DQS 并不是 LBS 的无代价替代。它在关节弯曲处外侧会产生鼓包；对偶四元数不能表示缩放，骨骼缩放需要单独处理；着色器也更复杂。Le 和 Hodgins 在 2016 年提出的"优化旋转中心"蒙皮（CoR）是一种折中：为每个顶点预计算一个旋转中心，用四元数混合旋转、用线性混合处理平移，兼顾体积保持和不鼓包。

| 方法 | 混合的量 | 结果是否刚体 | 典型问题 |
| --- | --- | --- | --- |
| LBS | 矩阵 | 否 | 糖纸扭曲、弯曲内侧塌陷 |
| DQS | 单位对偶四元数 | 是 | 弯曲外侧鼓包，缩放需另行处理 |
| CoR | 旋转用四元数，平移按预计算旋转中心 | 旋转部分是 | 需要预处理每顶点的旋转中心 |

## 在 UE 里

UE 骨骼网格体的标准 GPU 蒙皮是 LBS，混合在 GPU 蒙皮顶点工厂的着色器里完成；开启 GPU Skin Cache 时改为在计算着色器里完成，结果写进缓冲供后续通道复用。我没有在官方文档里查到内置的 DQS 选项（未核实）。

UE 里弥补 LBS 瑕疵的常规手段不在混合公式上，而是在数据上：

- 在前臂、上臂、大腿上加扭转辅助骨（twist bone），把一个大扭转分摊给几根骨骼，每根只转一小部分，$\cos(\theta/2)$ 就接近 1。UE 的默认人形骨架上就有 `lowerarm_twist_01_l` 这类骨骼。
- 用修正变形目标（corrective morph target）在特定关节角度下补回体积，通常由 Pose Driver 节点根据骨骼朝向驱动（见 [[EAABI-变形目标动画]]）。
- ML Deformer：在 DCC 里做高质量的肌肉 / 布料模拟，训练一个模型学习"LBS 结果与模拟结果的差"，运行时在 LBS 之上叠加这个修正。

需要完全自定义蒙皮算法时，5.x 的 Deformer Graph（基于计算着色器的可编程网格变形框架）可以替换默认的蒙皮步骤，功能状态以版本为准。

## 容易踩的坑

**法线带上了平移。** 用齐次坐标 $w=1$ 变换法线，光照会随角色位置变化。方向向量只用 3×3 部分。

**混合后忘了归一化法线。** $\mathbf{M}_{3\times3}$ 不是正交矩阵，变换后的法线长度不是 1，光照会偏暗或偏亮。

**以为多加权重就能解决塌陷。** 塌陷来自线性混合本身，增加影响骨骼数不会让它消失，反而可能让过渡带更宽、塌陷范围更大。要用辅助骨、修正形状或换混合方法。

**DQS 下使用骨骼缩放。** 对偶四元数不包含缩放，直接套用会丢失缩放动画，需要单独混合缩放分量。

## 相关

[[EAADH-蒙皮矩阵]] [[EAADI-蒙皮矩阵调色板]] [[EAADD-多骨骼的蒙皮权重]] [[EAADB-单关节蒙皮]] [[EAABH-3D蒙皮动画]] [[EAABI-变形目标动画]]
