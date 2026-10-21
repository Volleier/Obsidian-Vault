**姿势的插值（Pose Interpolation）是在两个骨架姿势之间逐关节计算中间姿势的过程：平移和缩放做线性插值，旋转做四元数的 NLERP 或 SLERP。它有两个用途：一是在动画片段的相邻采样帧之间插值，让任意时刻都能取到姿势；二是在两个不同动画的姿势之间按权重混合（GEA 称为 LERP 混合），这是交叉淡化、混合空间等一切动画混合的基础。插值总是在关节的局部空间、以 SRT 形式进行，而不是插值矩阵或全局姿势。UE 里对应动画序列的采样和 `FAnimationRuntime`、`FTransform` 提供的混合函数。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（Poses、Clips、Frames/Samples and Looping、LERP Blending 各节）；UE 部分对照 5.x API 文档与社区源码分析。

## 姿势是什么

GEA 的定义：关节的姿势是它相对某个参考系的位置、朝向和缩放；骨架的姿势就是所有关节姿势的集合，通常表示为一个 SRT 数组。每个关节的局部姿势几乎总是相对父关节给出，并以 SRT（缩放、四元数旋转、平移）存储：

$$
\mathbf{P}^{\text{skel}} = \{\mathbf{P}_j\}_{j=0}^{N-1},\qquad \mathbf{P}_j = (\mathbf{S}_j,\ \mathbf{q}_j,\ \mathbf{T}_j)
$$

GEA 指出了用 SRT 而不是 4×3 矩阵存储的两个理由：体积更小（均匀缩放时 8 个浮点数，非均匀缩放时 10 个，矩阵要 12 个），以及更容易插值。直接插值矩阵不可行，两个旋转矩阵逐元素平均的结果不再是旋转矩阵，这正是局部姿势通常用 SRT 表示的原因之一。

## 逐关节的插值公式

给定两个姿势 $\mathbf{P}^A$、$\mathbf{P}^B$ 和混合系数 $\beta\in[0,1]$，对每个关节 $j$：

$$
\mathbf{T}_j = (1-\beta)\,\mathbf{T}_j^A + \beta\,\mathbf{T}_j^B
$$

$$
\mathbf{q}_j = \operatorname{nlerp}(\mathbf{q}_j^A, \mathbf{q}_j^B, \beta) \quad\text{或}\quad \operatorname{slerp}(\mathbf{q}_j^A, \mathbf{q}_j^B, \beta)
$$

$$
\mathbf{S}_j = (1-\beta)\,\mathbf{S}_j^A + \beta\,\mathbf{S}_j^B
$$

旋转的两种选择见 [[EAADJ-球面非线性插值]] 和 [[EAADK-球面线性插值]]：NLERP 便宜且满足交换律，适合混合；SLERP 角速度恒定，但只能两两插值。相邻采样帧之间夹角很小，两者差别看不出来，所以动画采样多用 NLERP。

线性插值本身见 [[EBAC-线性插值]]。

## 为什么在局部空间插值

插值必须在局部空间做，而不是先算出全局姿势再插值。设想一条手臂在姿势 A 里向前伸、在姿势 B 里向上举：局部空间里，肩关节的旋转从"向前"插值到"向上"，前臂和手的局部变换不变，整条手臂沿圆弧扫过去，骨骼长度始终保持。如果对手的全局位置做线性插值，手会沿着"前方点到上方点"的直线走，途中离肩膀的距离变短，骨骼看上去被压缩了。

这也说明了姿势插值的另一个性质：插值的结果还要经过"局部 → 全局"的逐级相乘（见 [[EAABC-刚性层级动画]]），才能生成蒙皮矩阵。插值和层级传播的顺序是固定的：先插值局部姿势，再算全局姿势。

## 在片段内采样

动画片段是按固定间隔采样的姿势序列。GEA 提醒了一个容易出错的计数问题：非循环片段有 $N$ 帧时有 $N+1$ 个不同的采样点（首尾都要）；循环片段的最后一个采样和第一个相同，是冗余的，所以 $N$ 帧只有 $N$ 个不同采样。

取任意时刻 $t$ 的姿势时，先换算成帧坐标 $f = t\cdot\text{fps}$，取前后两个采样 $\lfloor f\rfloor$ 和 $\lfloor f\rfloor+1$，以 $\beta = f - \lfloor f\rfloor$ 做上面的逐关节插值。循环片段的下一帧要回绕到第 0 帧。下面是这个过程的示意代码，用了 UE 的数学类型，但它是算法示意，不是 UE 的动画采样实现（UE 的动画数据是压缩过的，由编解码器解压采样）：

```cpp
// 在两个相邻采样姿势之间逐关节插值（算法示意）
void InterpolatePose(const TArray<FTransform>& PoseA, const TArray<FTransform>& PoseB, float Beta,
                     TArray<FTransform>& OutPose)
{
	OutPose.SetNum(PoseA.Num());
	for (int32 j = 0; j < PoseA.Num(); ++j)
	{
		FQuat Rot = FQuat::FastLerp(PoseA[j].GetRotation(), PoseB[j].GetRotation(), Beta); // 已做最短路径修正
		Rot.Normalize();                                                                   // FastLerp 不归一化

		OutPose[j].SetRotation(Rot);
		OutPose[j].SetTranslation(FMath::Lerp(PoseA[j].GetTranslation(), PoseB[j].GetTranslation(), Beta));
		OutPose[j].SetScale3D(FMath::Lerp(PoseA[j].GetScale3D(), PoseB[j].GetScale3D(), Beta));
	}
}
```

旋转那两行是最容易出错的：不做最短路径修正会绕远路，不归一化会给旋转带上缩放。`FQuat::FastLerp` 已经处理了前者，后者要自己做。

线性插值只保证姿势连续，不保证速度连续：每经过一个采样点，速度会突变一次。采样率足够高（30 Hz 以上）时看不出来；如果片段被压缩成稀疏关键帧，就需要更高阶的曲线。GEA 提到 RAD Game Tools 的 Granny 把每个关节的 S、Q、T 通道存成 n 阶非均匀 B 样条，就是用曲线同时解决压缩和平滑的问题。

## 两个不同动画之间的混合

同样的逐关节公式用在两个不同动画的姿势上，就是 LERP 混合。$\beta$ 随时间从 0 变到 1 就是交叉淡化（见 [[EBCB-Cross Fades]]）；$\beta$ 由游戏参数（速度、方向）决定就是混合空间。两个姿势必须属于同一副骨架，逐关节一一对应。

混合多于两个姿势时，平移和缩放直接按权重加权，旋转用加权 NLERP（加权和再归一化），权重之和为 1。只对部分骨骼混合是分部混合（见 [[EBDD-分部混合（Skeleton Masked Blending）]]），把"差异姿势"叠加上去则是叠加混合（见 [[EBDC-叠加混合（Additive Blending）]]），后者不是插值，而是姿势的复合。

## 在 UE 里

动画序列（`UAnimSequence`）有一个 Interpolation 属性，可选 Linear（相邻帧之间线性插值，默认）或 Step（不插值，直接取前一帧，适合故意做出卡帧感的风格化动画）。采样出的姿势是每根骨骼一个局部空间 `FTransform`，由 `FTransform` 统一持有平移、旋转四元数和三维缩放。

两个姿势之间的混合有现成的接口：`FTransform::Blend(Atom1, Atom2, Alpha)` 和 `BlendWith(OtherAtom, Alpha)` 混合单个变换；`FAnimationRuntime::BlendTwoPosesTogether` 混合两个完整姿势，多个姿势则用 `BlendPosesTogether`。据社区对源码的分析，多姿势混合时对每根骨骼用 `FTransform::AccumulateWithShortestRotation` 按权重、按最短路径符号修正累加旋转，最后 `NormalizeRotation()`，也就是加权 NLERP，具体实现以你的引擎源码为准。动画蓝图里的 Blend 节点、Blend Poses by Bool / by Int、混合空间和状态机过渡，底层都是这套逐骨骼的混合。

动画数据在资产里是压缩存储的，采样时由压缩编解码器解码再插值。UE 5.x 的引擎插件里带有 ACL（Animation Compression Library）编解码器，它从哪个版本开始随引擎发布、是否为默认编解码器，我没有核实，以你的版本为准。

## 容易踩的坑

**插值全局姿势。** 骨骼长度在插值过程中会改变，肢体看上去被压缩或拉长。始终插值局部姿势。

**循环片段首尾处理错误。** 循环片段的最后一个采样和第一个相同，如果在播放时多算一帧，循环点会停顿一帧；如果少了回绕，最后一段会从末帧跳回首帧。

**不同骨架的姿势直接混合。** 骨骼数量或顺序不一致时逐关节对应就错了。UE 通过同一个 `USkeleton` 和重定向机制保证对应关系。

**缩放用线性插值时出现负值或零。** 从 1 插值到 -1（镜像）时中间会经过 0，网格瞬间消失。需要镜像的动画不要靠负缩放实现。

## 相关

[[EAADJ-球面非线性插值]] [[EAADK-球面线性插值]] [[EBAC-线性插值]] [[EBCB-Cross Fades]] [[EBDC-叠加混合（Additive Blending）]] [[EBDD-分部混合（Skeleton Masked Blending）]] [[EBC-动画状态机]] [[EAABC-刚性层级动画]]
