**SLERP（Spherical Linear Interpolation，球面线性插值）是在单位四元数构成的 4D 超球面上，沿两点之间的大圆弧以恒定角速度插值的方法，由 Ken Shoemake 在 1985 年引入计算机动画。它给出两个旋转之间"最自然"的过渡：转轴固定、角速度恒定、路径最短。代价是需要三角函数，且只定义了两个旋转之间的插值。UE 里对应 `FQuat::Slerp` 及其变体 `Slerp_NotNormalized`、`SlerpFullPath`。**

> 参考：Ken Shoemake, *Animating Rotation with Quaternion Curves*（SIGGRAPH 1985）；Jonathan Blow, *Understanding Slerp, Then Not Using It*（2004）；《Game Engine Architecture》第 3 版数学一章（四元数插值）；UE 部分对照 5.x `TQuat` API 文档。

## 为什么不能直接插值旋转

平移可以直接线性插值，旋转不行。欧拉角逐分量插值会得到奇怪的弯曲路径，还受万向节锁影响；旋转矩阵逐元素插值，中间结果不再是正交矩阵，会带上缩放和剪切。四元数把旋转表示成 4D 单位球面上的点，问题就变成"在球面上从一点走到另一点"，自然的答案就是沿大圆弧匀速走，这就是 SLERP。

## 公式与推导

设 $\mathbf{q}_1$、$\mathbf{q}_2$ 为单位四元数，夹角 $\Omega$ 满足 $\cos\Omega = \mathbf{q}_1\cdot\mathbf{q}_2$：

$$
\text{slerp}(\mathbf{q}_1, \mathbf{q}_2, t) = \frac{\sin\big((1-t)\Omega\big)}{\sin\Omega}\,\mathbf{q}_1 + \frac{\sin(t\Omega)}{\sin\Omega}\,\mathbf{q}_2
$$

这个式子可以在两者张成的平面里推出来。要找的点 $\mathbf{q}(t)$ 在单位圆上，和 $\mathbf{q}_1$ 夹角 $t\Omega$、和 $\mathbf{q}_2$ 夹角 $(1-t)\Omega$。设 $\mathbf{q}(t) = a\,\mathbf{q}_1 + b\,\mathbf{q}_2$，分别和 $\mathbf{q}_1$、$\mathbf{q}_2$ 做点积：

$$
\cos(t\Omega) = a + b\cos\Omega,\qquad \cos\big((1-t)\Omega\big) = a\cos\Omega + b
$$

解这个二元一次方程组，用和差化积整理，就得到 $a = \sin((1-t)\Omega)/\sin\Omega$，$b = \sin(t\Omega)/\sin\Omega$。

等价的写法是用四元数的幂：

$$
\text{slerp}(\mathbf{q}_1, \mathbf{q}_2, t) = \mathbf{q}_1\left(\mathbf{q}_1^{-1}\mathbf{q}_2\right)^{t}
$$

$\mathbf{q}_1^{-1}\mathbf{q}_2$ 是从 $\mathbf{q}_1$ 到 $\mathbf{q}_2$ 的相对旋转，取 $t$ 次幂就是"绕同一根轴只转 $t$ 倍的角度"。这个形式说明了 SLERP 的几何意义：转轴固定，转角随 $t$ 线性增长，所以角速度恒定。注意三维空间里的旋转角是 $2\Omega$，因为四元数用的是半角。

## 实现细节

实际实现有两处必须处理：

**最短路径。** $\mathbf{q}$ 和 $-\mathbf{q}$ 是同一个旋转，点积为负时把 $\mathbf{q}_2$ 取负，保证走短弧（和 [[EAADJ-球面非线性插值]] 中的讨论相同）。

**小角度退化。** $\Omega \to 0$ 时 $\sin\Omega \to 0$，公式变成 $0/0$。夹角很小时弧和弦几乎重合，直接退化为线性插值再归一化即可。

```cpp
// SLERP 算法示意（UE 已有 FQuat::Slerp，此处仅说明实现要点）
FQuat SlerpSketch(const FQuat& Q1, FQuat Q2, float T)
{
	double CosOmega = Q1 | Q2;          // FQuat 的点积运算符
	if (CosOmega < 0.0)                  // 最短路径：翻转到同一半球
	{
		Q2 = -Q2;
		CosOmega = -CosOmega;
	}

	double S1, S2;
	if (CosOmega > 0.9999)               // 夹角极小，退化为线性插值
	{
		S1 = 1.0 - T;
		S2 = T;
	}
	else
	{
		const double Omega = FMath::Acos(CosOmega);
		const double InvSin = 1.0 / FMath::Sin(Omega);
		S1 = FMath::Sin((1.0 - T) * Omega) * InvSin;
		S2 = FMath::Sin(T * Omega) * InvSin;
	}

	FQuat Result = Q1 * S1 + Q2 * S2;   // 这里的 * 是标量缩放，不是四元数乘法
	Result.Normalize();                  // 退化分支必须归一化；正常分支用来消除浮点误差
	return Result;
}
```

最容易混淆的是最后那一行：`FQuat * double` 是逐分量缩放，`FQuat * FQuat` 才是旋转复合。阈值 0.9999 没有标准答案，太大时 `acos` 在 1 附近精度很差，太小时线性近似误差可见。

## SLERP 的局限

Blow 的文章指出，旋转插值的三条理想性质（交换律、恒定角速度、最小力矩）任何方法最多只能满足两条。SLERP 满足后两条，但不满足交换律：三个以上的旋转混合时，只能两两嵌套，比如先 slerp(A, B)，再和 C 做 slerp，结果依赖嵌套顺序和权重的分配方式。动画混合树恰恰经常需要多路混合，所以游戏的姿势混合更常用 NLERP；SLERP 留给真正需要匀速转动的场合，比如摄像机、炮塔、平滑转身。

性能上，SLERP 每次需要一次 `acos` 和若干次 `sin`，在需要对上百根骨骼每帧插值的动画系统里，这比 NLERP 的一次归一化贵不少。相邻关键帧之间的夹角通常很小，SLERP 和 NLERP 的差别肉眼不可见，这也是很多引擎在动画解压和采样时只用 NLERP 的原因。

## 多个关键帧：从 SLERP 到 SQUAD

对一串关键帧 $\mathbf{q}_0, \mathbf{q}_1, \dots$ 逐段做 SLERP，姿势是连续的，但每经过一个关键帧，转轴和角速度都会突变，只有 $C^0$ 连续。关键帧稀疏、每段旋转较大时，这种"折角"在摄像机运动里很明显。

对应平移里的三次样条，四元数有 SQUAD（球面四边形插值）：在每段两端各加一个控制四元数 $\mathbf{s}_i$，做一次"SLERP 套 SLERP"：

$$
\operatorname{squad}(\mathbf{q}_i, \mathbf{q}_{i+1}, \mathbf{s}_i, \mathbf{s}_{i+1}, t) = \operatorname{slerp}\big(\operatorname{slerp}(\mathbf{q}_i, \mathbf{q}_{i+1}, t),\ \operatorname{slerp}(\mathbf{s}_i, \mathbf{s}_{i+1}, t),\ 2t(1-t)\big)
$$

控制点常取

$$
\mathbf{s}_i = \mathbf{q}_i \exp\left(-\frac{\log(\mathbf{q}_i^{-1}\mathbf{q}_{i+1}) + \log(\mathbf{q}_i^{-1}\mathbf{q}_{i-1})}{4}\right)
$$

这样相邻两段在关键帧处的切向一致，角速度不再突变。骨骼动画的采样率通常够高，用不上 SQUAD；它主要用在稀疏关键帧的摄像机路径和过场动画里。UE 的 `FQuat` 提供了 `Squad`、`SquadFullPath` 和计算控制点的 `CalcTangents`。

## 在 UE 里

| 函数 | 最短路径 | 结果归一化 | 说明 |
| --- | --- | --- | --- |
| `FQuat::Slerp(Quat1, Quat2, Slerp)` | 是 | 是 | 最常用 |
| `FQuat::Slerp_NotNormalized` | 是 | 否 | 调用方自己归一化 |
| `FQuat::SlerpFullPath` / `SlerpFullPath_NotNormalized` | 否 | 是 / 否 | API 文档描述为"不做最短距离检查的简化版 Slerp"，需要绕远路时用 |
| `FQuat::FastLerp` | 是 | 否 | 即 NLERP 的分子，见 [[EAADJ-球面非线性插值]] |

游戏逻辑里想让一个朝向每帧平滑地追向目标，用 `FMath::QInterpTo(Current, Target, DeltaTime, InterpSpeed)`，它内部基于 Slerp 按时间逐步逼近，写在 Tick 里就行：

```cpp
void AMyTurret::Tick(float DeltaSeconds)
{
	Super::Tick(DeltaSeconds);

	const FQuat Current = GetActorQuat();
	const FQuat Target = (AimPoint - GetActorLocation()).ToOrientationQuat();
	SetActorRotation(FMath::QInterpTo(Current, Target, DeltaSeconds, 5.0f));
}
```

`InterpSpeed` 越大追得越快，这是一种"每帧按比例逼近"的缓动，不是固定时长的插值；需要固定时长时，自己维护 $t$ 并调用 `FQuat::Slerp`。蓝图里对应的是 RInterp To，以及带 Shortest Path 选项的 Lerp (Rotator)。

## 容易踩的坑

**不处理最短路径。** 自己写 SLERP 时漏掉点积符号检查，某些朝向之间会绕远路转一大圈。UE 的 `Slerp` 已经处理，`SlerpFullPath` 故意不处理。

**不处理小角度。** 两个几乎相同的四元数做 SLERP，除以接近 0 的 $\sin\Omega$ 会得到 NaN 或巨大误差。

**把 SLERP 当作多路混合工具。** 嵌套 SLERP 的结果依赖顺序，混合空间这类多输入混合应该用加权 NLERP。

**用 FRotator 做插值。** 欧拉角逐分量插值在跨越 ±180° 时会绕远路，大角度时路径也不是最短的。需要平滑旋转时先转成 `FQuat`。

## 相关

[[EAADJ-球面非线性插值]] [[EAADO-姿势的插值]] [[EBAC-线性插值]] [[EBCB-Cross Fades]]
