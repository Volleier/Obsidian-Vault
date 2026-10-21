**多骨骼蒙皮权重（Skin Weights for Multiple Bones）是让一个顶点同时绑定到若干个关节、每个关节配一个非负权重且权重之和为 1 的做法。它决定了线性混合蒙皮里"每个关节对这个顶点贡献多少"，是蒙皮网格数据里除了位置、法线、UV 之外最重要的顶点属性。引擎通常限制每个顶点的影响数（传统上是 4，UE 默认可以更多），并把关节索引和权重量化后打包进顶点缓冲。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（Per-Vertex Skinning Information）；Baran & Popović, *Automatic Rigging and Animation of 3D Characters*（SIGGRAPH 2007，Blender 自动权重的"骨骼热量"方法出处）；UE 部分对照 5.x。

## 为什么一个顶点要受多个关节影响

只绑一个关节时（[[EAADB-单关节蒙皮]]），关节两侧的顶点分别刚性地跟着两根骨骼走，交界处的三角形被硬拉开。让交界附近的顶点同时受两根骨骼影响，比如手肘处的顶点 50% 跟上臂、50% 跟前臂，弯曲时它就落在两者之间，皮肤被平滑地拉开。越靠近上臂的顶点上臂权重越大，越靠近前臂的越小，形成一个过渡带。

蒙皮后的位置就是各关节蒙皮结果的加权平均（行向量约定，$\mathbf{K}_j$ 为关节 $j$ 的蒙皮矩阵，见 [[EAADH-蒙皮矩阵]]）：

$$
\mathbf{v}' = \sum_{i=1}^{n} w_i\,\mathbf{v}\,\mathbf{K}_{j_i},\qquad w_i \ge 0,\quad \sum_{i=1}^{n} w_i = 1
$$

权重之和必须为 1。直观地说，如果所有关节都做同一个刚体变换，顶点应该做同样的变换；只有权重和为 1 时加权平均才等于这个变换本身。和大于 1 时顶点会被向外推，小于 1 时会被拉向模型原点。

## 每顶点影响数的上限

GEA 说明了为什么游戏引擎通常限制每个顶点最多绑定 4 个关节：一是 4 个 8 位关节索引正好打包进一个 32 位字，很方便；二是 2 个、3 个、4 个影响之间的画质差异很容易看出来，但超过 4 个以后，大多数人看不出区别。书中的顶点结构只存 3 个权重，第 4 个用 1 减去前 3 个得到，省一个字段：

$$
w_4 = 1 - (w_1 + w_2 + w_3)
$$

现在的硬件对顶点带宽没那么敏感，面部和高精度角色经常需要 8 个甚至更多影响，UE 就允许超过 4 个（见下文）。但上限依然存在，DCC 里刷出的权重超过上限时，引擎导入会保留最大的几个并重新归一化，形变会和 DCC 里有差异。

## 权重的来源和处理

**自动生成。** DCC 绑定时先自动算一版初始权重。最简单的是按顶点到骨骼的距离衰减；Blender 的"自动权重"用的是 Baran 和 Popović 提出的骨骼热量（bone heat）方法：把骨骼当热源，在网格表面求解热扩散，平衡后的温度就是权重，这样权重会沿着表面扩散，而不会穿过身体跑到另一条腿上。

**手工修正。** 自动结果几乎总要修：腋下、胯部、肩部这些多根骨骼交汇的地方最难。美术用权重笔刷逐区域调整，并反复把角色摆到极端姿势检查。

**归一化和剔除。** 刷完的权重要做三件事：剔除很小的权重（比如小于 0.01），它们对形变几乎没有贡献，却占用一个影响名额；截断到引擎的影响数上限，保留最大的几个；最后重新归一化，让剩下的权重和为 1。

**量化。** 顶点缓冲里权重通常用 8 位或 16 位无符号整数存储。量化时一个容易忽略的问题是：四个权重各自四舍五入后，和不一定正好等于 255。差值要补到最大的那个权重上，否则所有顶点都会有一点点缩放。下面是这个处理过程的示意代码（算法示意，不是引擎源码）：

```cpp
// 把任意数量的影响截断到 MaxInfluences 个并量化为 8 位，保证量化后之和恰为 255
struct FInfluence { int32 BoneIndex; float Weight; };

void QuantizeWeights(TArray<FInfluence> Influences, int32 MaxInfluences, uint8 OutBones[], uint8 OutWeights[])
{
	Influences.Sort([](const FInfluence& A, const FInfluence& B) { return A.Weight > B.Weight; });
	Influences.SetNum(FMath::Min(Influences.Num(), MaxInfluences));

	float Sum = 0.f;
	for (const FInfluence& Inf : Influences) { Sum += Inf.Weight; }

	int32 Total = 0;
	for (int32 i = 0; i < MaxInfluences; ++i)
	{
		const bool bValid = i < Influences.Num() && Sum > 0.f;
		OutBones[i]   = bValid ? (uint8)Influences[i].BoneIndex : 0;
		OutWeights[i] = bValid ? (uint8)FMath::RoundToInt(Influences[i].Weight / Sum * 255.f) : 0;
		Total += OutWeights[i];
	}
	OutWeights[0] = (uint8)FMath::Clamp(OutWeights[0] + (255 - Total), 0, 255); // 误差补给最大的权重
}
```

先排序再截断，保证丢掉的是最小的影响；归一化放在截断之后，否则截断后的和小于 1。这里把骨骼索引也压成 8 位只是为了示意，骨骼多于 256 根时要用 16 位索引，或者像 UE 那样按 Section 重映射。

## 在 UE 里

UE 的蒙皮权重存在 `FSkinWeightVertexBuffer` 里，每个顶点的影响数上限由项目设置里的默认骨骼影响上限（Default Bone Influence Limit）和每个网格 LOD 的设置决定，Interchange 导入管线也有对应的 `SetCustomBoneInfluenceLimit`，按官方 API 说明，设置得高于项目上限时无效，设为 0 时取项目设置。社区资料显示 UE5 源码中每顶点影响数的硬上限 `MAX_TOTAL_INFLUENCES` 为 12（UE4 时代常见的说法是 8），这个数字我没有在官方文档中核实到。此外可以通过 `r.GPUSkin.UnlimitedBoneInfluences` 开启无限骨骼影响，这种模式要求开启 GPU Skin Cache。项目设置里还有支持 16 位骨骼索引的选项，用于骨骼数超过 256 的网格。

顶点里存的骨骼索引**不是**骨架的全局骨骼索引，而是所在渲染 Section 的局部索引，要通过 `FSkelMeshRenderSection::BoneMap` 映射回骨架索引。直接拿权重缓冲里的索引去查骨骼是论坛上的常见问题（相关的调色板机制见 [[EAADI-蒙皮矩阵调色板]]）。

UE 还支持 Skin Weight Profiles：为同一个骨骼网格体准备多套替代权重，按平台或 LOD 切换，比如在低端平台上用影响数更少的权重。5.x 起骨骼网格体编辑器里也加入了直接编辑蒙皮权重的工具，状态和功能以手上的版本为准。

## 容易踩的坑

**权重和不为 1。** 最常见的症状是网格在绑定姿势下就轻微膨胀或收缩。程序化生成或合并网格后，一定要重新归一化。

**截断前没有排序。** 直接取前 4 个而不是最大的 4 个，会丢掉关键影响，形变出现尖刺。

**量化误差累积。** 各自四舍五入后和不等于 255（或 65535），整张网格会有微小的缩放。把误差补到最大权重上。

**权重跨越身体。** 大腿内侧的顶点被另一条腿的骨骼影响，抬腿时皮肤被拉向另一侧。用基于表面距离（测地线、热扩散）而不是欧氏距离的初始权重可以避免。

## 相关

[[EAADB-单关节蒙皮]] [[EAADH-蒙皮矩阵]] [[EAADI-蒙皮矩阵调色板]] [[EAADL-权重蒙皮的混合]] [[EAABH-3D蒙皮动画]]
