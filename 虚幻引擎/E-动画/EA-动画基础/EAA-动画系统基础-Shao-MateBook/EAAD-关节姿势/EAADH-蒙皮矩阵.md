**蒙皮矩阵（Skinning Matrix）是把一个顶点从"绑定姿势下的模型空间位置"变换到"当前姿势下的模型空间位置"的矩阵，每个关节一个。用《Game Engine Architecture》的行向量记法，关节 $j$ 的蒙皮矩阵是 $\mathbf{K}_j = \mathbf{B}_{j\to M}^{-1}\,\mathbf{C}_{j\to M}$：先用逆绑定矩阵把顶点变到关节空间，再用关节当前的全局矩阵变回模型空间。多关节蒙皮（线性混合蒙皮，LBS）就是把各关节蒙皮矩阵变换的结果按权重加权。UE 里这个矩阵叫 ReferenceToLocal，由 `USkeletalMesh::GetRefBasesInvMatrix()` 和组件空间骨骼变换相乘得到。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（The Mathematics of Skinning）；MIT 6.837（2012 秋）蒙皮讲义；UE 部分参考社区源码分析（Zhirui Li《Unreal 骨骼动画源码剖析》、Rodolphe Vaillant 的 Skeletal Mesh 笔记）和官方论坛中引用的源码片段，函数细节以你的引擎源码为准。本文统一采用行向量约定：$\mathbf{v}' = \mathbf{v}\mathbf{M}$。

## 从局部姿势到全局姿势

动画片段存的是每个关节相对父关节的局部姿势 $\mathbf{P}_{j\to p(j)}$（通常是 SRT：缩放、四元数、平移），而蒙皮需要的是关节在模型空间中的全局姿势。先把局部姿势沿层级逐级相乘：

$$
\mathbf{C}_{j\to M} = \mathbf{P}_{j\to p(j)}\,\mathbf{C}_{p(j)\to M}
$$

根关节的全局姿势就是它的局部姿势。关节数组按"父先于子"排序，一次顺序遍历就能全部算完（见 [[EAABC-刚性层级动画]]）。同样的计算作用在绑定姿势上，得到绑定姿势下的全局矩阵 $\mathbf{B}_{j\to M}$，它只需要算一次。

## 推导蒙皮矩阵

关键观察来自单关节的情况（[[EAADB-单关节蒙皮]]）：顶点在所绑定关节的局部空间里的坐标，不随关节运动而改变。所以对绑定姿势下的模型空间顶点 $\mathbf{v}_M^{B}$：

$$
\underbrace{\mathbf{v}_M^{B}\,\mathbf{B}_{j\to M}^{-1}}_{\text{关节 } j \text{ 空间中的坐标，恒定}}\;\mathbf{C}_{j\to M} = \mathbf{v}_M^{C}
\quad\Longrightarrow\quad
\mathbf{K}_j = \mathbf{B}_{j\to M}^{-1}\,\mathbf{C}_{j\to M}
$$

可以把 $\mathbf{K}_j$ 理解为"关节 $j$ 从绑定姿势到当前姿势的相对变化"。绑定姿势下 $\mathbf{C}=\mathbf{B}$，$\mathbf{K}_j = \mathbf{I}$，网格不动，这是检验实现是否正确的第一个测试。

**一个具体的例子。** 设关节 $j$ 在绑定姿势下位于 $(0, 0, 10)$，没有旋转，即 $\mathbf{B}_{j\to M}$ 是平移 $(0,0,10)$。一个顶点在模型空间的 $(5, 0, 10)$。

- 乘 $\mathbf{B}^{-1}$（平移 $(0,0,-10)$）得到关节空间坐标 $(5, 0, 0)$。
- 当前姿势下关节绕 $Z$ 轴转了 90°，位置不变。先旋转：$(5,0,0)$ 变成 $(0,5,0)$；再平移 $(0,0,10)$，得到 $(0, 5, 10)$。

顶点绕着关节转了 90°，和直觉一致。如果漏掉 $\mathbf{B}^{-1}$，顶点会先绕模型原点旋转再被抬高 10，落在 $(0,5,20)$，这就是"漏掉逆绑定矩阵"的典型症状。

## 多关节：线性混合蒙皮

顶点绑定到 $n$ 个关节，权重 $w_i$ 之和为 1（见 [[EAADD-多骨骼的蒙皮权重]]）：

$$
\mathbf{v}_M^{C} = \sum_{i=1}^{n} w_i\,\mathbf{v}_M^{B}\,\mathbf{K}_{j_i}
$$

由于矩阵乘法对加法满足分配律，也可以先把矩阵加权再乘一次：

$$
\mathbf{v}_M^{C} = \mathbf{v}_M^{B}\left(\sum_{i=1}^{n} w_i\,\mathbf{K}_{j_i}\right)
$$

两种写法结果相同。GPU 上通常用后一种：先混合出一个矩阵，再对位置、法线、切线各乘一次，比分别变换再混合省。原笔记里把 LBS 描述为"变换矩阵的加权平均"，指的就是这个形式；但要记住，加权平均后的矩阵一般不再是刚体变换，这就是 LBS 体积塌陷的来源（见 [[EAABH-3D蒙皮动画]]）。

## 加上模型到世界变换

渲染最终需要世界空间（或裁剪空间）坐标。可以把模型到世界的矩阵预先乘进每个蒙皮矩阵：

$$
\mathbf{K}_j^{W} = \mathbf{B}_{j\to M}^{-1}\,\mathbf{C}_{j\to M}\,\mathbf{M}_{M\to W}
$$

这样每个顶点少做一次矩阵乘法。GEA 提到一个例外：用动画实例化渲染大群角色时，多个角色共用一份矩阵调色板，模型到世界的矩阵就必须单独保留，在着色器里再乘。

## 各矩阵的更新频率

| 矩阵 | 何时计算 | 说明 |
| --- | --- | --- |
| $\mathbf{P}_{j\to p(j)}$ 局部姿势 | 每帧 | 动画采样、混合的输出 |
| $\mathbf{C}_{j\to M}$ 当前全局姿势 | 每帧 | 局部姿势逐级相乘 |
| $\mathbf{B}_{j\to M}^{-1}$ 逆绑定矩阵 | 导入时一次 | 和骨架 / 网格一起存储 |
| $\mathbf{K}_j$ 蒙皮矩阵 | 每帧 | 每关节一次矩阵乘法 |
| 顶点蒙皮 | 每帧每顶点 | 通常在 GPU 上完成 |

所有 $\mathbf{K}_j$ 组成的数组就是蒙皮矩阵调色板，整体上传给 GPU，见 [[EAADI-蒙皮矩阵调色板]]。

## 算法示意

下面用 UE 的数学类型写出每帧生成蒙皮矩阵的过程。UE 的 `FTransform` 和 `FMatrix` 都是"`A * B` 先 A 后 B"的约定，和本文公式顺序一致。这是示意代码，不是引擎源码：

```cpp
// 由局部姿势生成蒙皮矩阵（算法示意）
// ParentIndices[i] < i；InvBindPose 在导入时预计算
void BuildSkinningMatrices(const TArray<FTransform>& LocalPose, const TArray<int32>& ParentIndices,
                           const TArray<FMatrix44f>& InvBindPose, TArray<FMatrix44f>& OutSkinning)
{
	const int32 Num = LocalPose.Num();
	TArray<FTransform> Global;
	Global.SetNum(Num);
	OutSkinning.SetNum(Num);

	for (int32 i = 0; i < Num; ++i)
	{
		const int32 Parent = ParentIndices[i];
		Global[i] = (Parent == INDEX_NONE) ? LocalPose[i] : LocalPose[i] * Global[Parent]; // C_j
		OutSkinning[i] = InvBindPose[i] * FMatrix44f(Global[i].ToMatrixWithScale());       // K_j = B^-1 · C
	}
}
```

两处乘法顺序都不能反。`ToMatrixWithScale` 而不是 `ToMatrixNoScale`，否则骨骼缩放动画会丢失。

## 在 UE 里

UE 的实现和上面的推导一一对应，只是换了名字：

| 本文记号 | UE 中的对应物 |
| --- | --- |
| $\mathbf{B}_{j\to M}^{-1}$ | `USkeletalMesh::GetRefBasesInvMatrix()`，参考姿势（绑定姿势）下组件空间矩阵的逆，按骨骼索引排列 |
| $\mathbf{C}_{j\to M}$ | `USkinnedMeshComponent::GetComponentSpaceTransforms()`，动画求值后每根骨骼的组件空间变换 |
| $\mathbf{K}_j$ | ReferenceToLocal 矩阵（`FMatrix44f`） |

根据社区对源码的分析，渲染模块在 `UpdateRefToLocalMatrices` 里为当前 LOD 需要的骨骼计算 `ReferenceToLocal[i] = RefBasesInvMatrix[i] * ComponentSpaceTransforms[i].ToMatrixWithScale()`，结果随动态数据送到渲染线程，最终作为骨骼矩阵缓冲交给 GPU 蒙皮着色器；论坛里引用的 `USkinnedMeshComponent::GetTypedSkinnedVertexPosition`（CPU 端读取蒙皮后顶点位置）里也是同样的写法。这里 RefBasesInvMatrix 在左、组件空间矩阵在右，正是行向量约定下的 $\mathbf{B}^{-1}\mathbf{C}$。组件空间到世界空间的变换不在这个矩阵里，由图元的本地到世界矩阵在着色器中单独处理。

被隐藏的骨骼（`HideBoneByName`）不会简单地跳过：官方论坛里的一次讨论解释过，隐藏骨骼会取父骨骼的矩阵并把缩放设为 0，否则从第一根隐藏骨骼开始蒙皮就会出错。此外，使用 Leader Pose 的子网格体要用主组件的组件空间变换，但逆参考矩阵仍然是子网格体自己的。

需要在 CPU 上拿到当前的蒙皮矩阵时（比如做顶点级的命中检测），`USkinnedMeshComponent` 提供了 `GetCurrentRefToLocalMatrices`，但它只更新当前 LOD 活动骨骼的条目，其他条目可能是旧值。

## 容易踩的坑

**绑定姿势下网格就变形。** 首先检查逆绑定矩阵和乘法顺序，$\mathbf{K}=\mathbf{I}$ 是最简单的单元测试。

**列向量公式照搬到 UE。** 教材里常见的 $\mathbf{C}\,\mathbf{B}^{-1}$ 是列向量约定，到 UE 里要写成 `InvBind * Current`。

**丢掉缩放。** 用 `ToMatrixNoScale` 或只用四元数和平移构造矩阵，骨骼缩放动画会失效，而且会和导入时的参考姿势不一致。

**法线用了带非均匀缩放的矩阵。** 蒙皮矩阵带非均匀缩放时，法线严格来说要用逆转置变换；自己写蒙皮着色器时直接用蒙皮矩阵变换法线，在非均匀缩放下光照会出错。

## 相关

[[EAADB-单关节蒙皮]] [[EAADD-多骨骼的蒙皮权重]] [[EAADI-蒙皮矩阵调色板]] [[EAADL-权重蒙皮的混合]] [[EAABH-3D蒙皮动画]] [[EAABC-刚性层级动画]]
