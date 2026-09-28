**IK（Inverse Kinematics，逆向运动学）是给定关节链末端（末端执行器，end effector）想要到达的位置或朝向，反过来求链上各关节局部旋转的问题。它和 FK（正向运动学：由各关节局部变换逐级相乘得到末端位置）方向相反，在游戏里主要作为动画后处理，用来修正脚贴地、手抓物体、头看向目标这类"动画数据事先不知道"的情况。UE 里对应动画蓝图的 Two Bone IK、FABRIK、CCDIK、Leg IK 节点，Control Rig 的 Full Body IK，以及用于重定向的 IK Rig。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（动画后处理、IK 一节）；FABRIK 见 Aristidou & Lasenby 2011；UE 部分对照 5.x 官方节点文档。

## 为什么需要 IK

GEA 对 IK 的定义很直接：普通的动画片段属于 FK，输入是一组局部关节姿势，输出是全局姿势和蒙皮矩阵；IK 反过来，输入是某一个关节（末端执行器）期望的全局姿势，求其他关节的局部姿势，使末端到达那里。

动画是在平地上、对着固定位置做的，游戏里的地形和目标却是随时变化的。角色站在台阶上，一只脚踩空；伸手去按一个高度不固定的按钮；握枪的左手随换弹动画偏离护木。这些偏差通常只有几厘米到十几厘米，重做动画不现实，最自然的做法是在动画求值之后、蒙皮之前，用 IK 把末端拉到正确位置，只改动链上的几根骨骼。

IK 这个问题本身往往**不适定**：一条三段以上的链，到达同一个点的姿势有无穷多种（手掌不动时肘部可以绕肩—腕连线转一圈）；目标超出链长时又一个解都没有。所以所有 IK 求解器都要回答两个问题：解不唯一时选哪个（靠极向量 pole vector、关节限制、偏好姿势），无解时怎么办（尽量靠近、或允许拉伸）。

## 解析解：两骨骼 IK

人的手臂和腿都可以看成"两段骨骼 + 三个关节"的链：根（肩 / 髋）、中间关节（肘 / 膝）、末端（腕 / 踝）。这种情况有闭式解，是游戏里用得最多的 IK。

设上段长 $a$，下段长 $b$，根到目标的距离 $d$（先夹到 $[\,|a-b|,\ a+b\,]$ 区间内，避免无解）。由余弦定理，根关节处"根—目标连线"和上段骨骼之间的夹角 $\alpha$ 满足

$$
\cos\alpha = \frac{a^2 + d^2 - b^2}{2ad}
$$

中间关节的内角 $\beta$ 满足

$$
\cos\beta = \frac{a^2 + b^2 - d^2}{2ab}
$$

只有这两个角还不够，因为中间关节可以绕"根—目标"轴任意旋转。极向量（UE 里叫 Joint Target）用来确定弯曲平面：膝盖朝前、手肘朝后外侧。下面是一段示意实现，用了 UE 的数学类型，但它**不是引擎源码**，只用来说明几何关系：

```cpp
// 两骨骼 IK 的几何示意：求中间关节的新位置（算法示意，非 UE 源码）
FVector SolveTwoBoneJoint(const FVector& Root, const FVector& Target, const FVector& PoleTarget,
                          double UpperLen, double LowerLen)
{
	const FVector ToTarget = Target - Root;
	const double Dist = FMath::Clamp(ToTarget.Size(), FMath::Abs(UpperLen - LowerLen) + KINDA_SMALL_NUMBER,
	                                 UpperLen + LowerLen - KINDA_SMALL_NUMBER);
	const FVector U = ToTarget.GetSafeNormal();

	// 极向量去掉沿 U 的分量，得到弯曲平面内垂直于 U 的方向
	const FVector V = (PoleTarget - Root - U * FVector::DotProduct(PoleTarget - Root, U)).GetSafeNormal();

	// 余弦定理求根关节处的夹角
	const double CosAlpha = (UpperLen * UpperLen + Dist * Dist - LowerLen * LowerLen) / (2.0 * UpperLen * Dist);
	const double SinAlpha = FMath::Sqrt(FMath::Max(0.0, 1.0 - CosAlpha * CosAlpha));

	return Root + UpperLen * (CosAlpha * U + SinAlpha * V);
}
```

得到中间关节位置后，上段骨骼的旋转就是"从原方向转到 Root→Joint 方向"的最短旋转，下段同理（`FQuat::FindBetweenNormals` 可以做这件事）。容易写错的地方有两个：极向量和 U 几乎共线时 V 会退化成零向量，需要回退到一个默认方向；距离刚好等于 $a+b$ 时手臂完全伸直，稍有噪声就会在伸直和微弯之间抖动，所以上面的夹取故意留了一点余量。UE 里真正的实现在 AnimationCore 模块的 `AnimationCore::SolveTwoBoneIK`（`TwoBoneIK.h`），还支持拉伸。

## 数值解：多骨骼链

链更长时（脊柱、尾巴、触手）没有闭式解，要迭代：

| 方法 | 做法 | 特点 | 对应笔记 |
| --- | --- | --- | --- |
| CCD（循环坐标下降） | 从末端往根，逐个关节旋转，使"关节→末端"对准"关节→目标" | 实现最简单，单次迭代便宜；靠近末端的关节转得多，容易卷曲 | [[ECAH-循环星标下降算法]] |
| FABRIK | 在位置空间里前后两遍拉链：先从末端拖到目标，再把根拉回原位，重复 | 收敛快，姿势自然，不直接涉及角度；关节限制要额外处理 | [[ECAA-FABRIK]] |
| 雅可比方法 | 线性化末端位置对关节角的导数，解 $\Delta\boldsymbol\theta$ | 数学上最通用，能加权、加多个目标；每步要解线性方程 | [[ECAI-雅可比矩阵法]] |

雅可比方法的核心是 $\Delta\mathbf{e} \approx J\,\Delta\boldsymbol{\theta}$，其中 $J_{ij} = \partial e_i / \partial \theta_j$。由于 $J$ 通常不是方阵，常用阻尼最小二乘（DLS）求解：

$$
\Delta\boldsymbol{\theta} = J^{T}\left(JJ^{T} + \lambda^{2} I\right)^{-1}\Delta\mathbf{e}
$$

$\lambda$ 越大越稳定，但收敛越慢；它主要用来防止链接近奇异姿势（完全伸直）时角度暴增。

## 在 UE 里

| 节点 / 系统 | 运行时类型 | 适用场景 |
| --- | --- | --- |
| Two Bone IK | `FAnimNode_TwoBoneIK` | 手臂、腿；Effector 位置 + Joint Target（极向量），可选拉伸 |
| FABRIK | `FAnimNode_Fabrik` | 任意长度的链，指定 Root 和 Tip 骨骼 |
| CCDIK | `FAnimNode_CCDIK` | 任意长度的链，可以给每个关节设角度限制 |
| Leg IK | `FAnimNode_LegIK` | 多段腿（如动物的反关节腿） |
| Control Rig 的 Full Body IK | Control Rig 里的 FBIK 节点 | 全身联动：拉手时躯干跟着倾斜 |
| IK Rig / IK Retargeter | `UIKRigDefinition` 等资产 | 主要用于动画重定向，也能在动画蓝图里用 IK Rig 节点跑求解器 |

这些节点都是 Skeletal Control，放在动画蓝图 AnimGraph 里，输入输出是组件空间姿势，所以前面要接 Local to Component 转换节点（编辑器通常会自动插入）。蓝图里还有一个不依赖动画图的 `Two Bone IK Function`，可以在普通逻辑里直接算三点的 IK 结果。

一个典型的脚部 IK 流程是：在角色蓝图或动画蓝图的线程安全更新里，从每只脚的位置向下做射线检测，得到地面高度和法线；算出每只脚需要的偏移和旋转，同时把骨盆下压较高那只脚的偏移量（否则较低的脚够不着地面）；最后在 AnimGraph 里用两个 Two Bone IK 节点把脚踝拉到修正后的位置，Joint Target 放在膝盖前方。

## 容易踩的坑

**目标超出链长。** 不做夹取时解析解里会出现 `acos` 参数大于 1 得到 NaN，骨骼瞬间消失或乱飞。要么夹取距离，要么明确允许拉伸。

**极向量设错。** Joint Target 在错误的一侧，膝盖会反弯。通常把它放在参考姿势下膝盖（或肘部）前方一段距离，并跟随角色旋转。

**IK 结果逐帧跳变。** 射线检测结果不连续（台阶边缘）时，IK 目标会跳。对目标位置做插值平滑，或者对 IK 的 Alpha 做淡入淡出。

**空间搞混。** Effector 位置可以是世界、组件、骨骼等不同空间，节点上有对应的 Location Space 选项，传世界坐标却选了组件空间是最常见的错误。

## 相关

[[ECAB-IK和FK]] [[ECAA-FABRIK]] [[ECAH-循环星标下降算法]] [[ECAI-雅可比矩阵法]] [[EAABA-基于物理的动画]] [[EAABAC-布娃娃系统]]
