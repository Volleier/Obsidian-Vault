**线性插值（Linear Interpolation，LERP）按一个参数 $t$ 在两个值之间取直线上的点：$t=0$ 得到起点，$t=1$ 得到终点，中间匀速过渡。它是动画混合最底层的运算：两段动画按权重混合时，每根骨骼的平移和缩放就是做 Lerp，旋转则用 Lerp 的四元数变体（NLerp）或 Slerp。UE 里对应 `FMath::Lerp`、蓝图的 Lerp 节点、材质的 `LinearInterpolate` 节点。**

## 公式和两种写法

$$
\operatorname{lerp}(a, b, t) = (1 - t)\,a + t\,b = a + t\,(b - a)
$$

$a$、$b$ 可以是标量、向量、颜色，只要它们所在的空间支持加法和数乘。结果对 $t$ 的导数恒为 $b - a$，所以 $t$ 匀速变化时，结果也匀速移动。这正是“线性”的含义，也是它听起来生硬的原因：起步和停下都没有加减速。

两种写法数学上等价，浮点下不等价：

| 写法 | 优点 | 缺点 |
| --- | --- | --- |
| $a + t(b - a)$ | 一次乘法，$a = b$ 时结果严格等于 $a$ | $t = 1$ 时由于舍入，结果可能不严格等于 $b$ |
| $(1-t)a + tb$ | $t = 0$ 和 $t = 1$ 时都严格返回端点 | 两次乘法；$a = b$ 时结果可能有微小偏差 |

UE 的 `FMath::Lerp` 用的是第一种（`A + Alpha * (B - A)`）。大多数情况下无所谓，但如果依赖“插值到 1 时恰好等于目标”来判断结束，最好直接在 $t \ge 1$ 时赋值为目标，而不是比较浮点相等。

$t$ 不必限制在 $[0, 1]$：$t > 1$ 或 $t < 0$ 就是沿直线外推。`FMath::Lerp` 不会替你截断，需要的话自己 `FMath::Clamp`。

## 在动画混合里

两个姿势按权重 $\alpha$ 混合，本质就是对每根骨骼的局部变换分量分别插值：

$$
\mathbf{T} = \operatorname{lerp}(\mathbf{T}_A, \mathbf{T}_B, \alpha),\quad
\mathbf{S} = \operatorname{lerp}(\mathbf{S}_A, \mathbf{S}_B, \alpha),\quad
\mathbf{q} = \frac{(1-\alpha)\,\mathbf{q}_A + \alpha\,\mathbf{q}'_B}{\lVert (1-\alpha)\,\mathbf{q}_A + \alpha\,\mathbf{q}'_B \rVert}
$$

平移、缩放直接 Lerp 没有问题。旋转不能直接 Lerp 欧拉角（会走奇怪的路径、遇到 ±180° 跳变），也不能 Lerp 旋转矩阵（结果不再正交）。四元数分量做 Lerp 后再归一化，叫 NLerp；其中 $\mathbf{q}'_B$ 是和 $\mathbf{q}_A$ 点积为正的那个（$\mathbf{q}$ 和 $-\mathbf{q}$ 表示同一旋转，取同号才走短弧）。NLerp 的路径和 Slerp 一样在大圆弧上，只是角速度不均匀，两端慢、中间快，偏差在夹角很大时才明显。多个姿势按权重混合时，NLerp 可以直接推广成“加权求和再归一化”，Slerp 做不到这一点，这也是引擎姿势混合普遍选 NLerp 的原因。两者的推导和比较见 [[EAADK-球面线性插值]]、[[EAADJ-球面非线性插值]]。

UE 的姿势混合就是这样做的：混合节点（`Blend`、`Blend Poses by bool` 等）把各输入姿势按权重逐骨骼累加，旋转用 `FTransform::AccumulateWithShortestRotation` 保证同号，最后统一归一化；单个变换之间的插值有 `FTransform::Blend`。混合空间（Blend Space）也是在样本之间算权重再做同样的加权混合，参见 [[EAADO-姿势的插值]]。

## 从两个值到多个值

Lerp 可以嵌套。在矩形网格的一个格子里，先沿 $x$ 方向对上下两条边各做一次 Lerp，再沿 $y$ 方向对两个结果做一次，就是双线性插值：

$$
f(u, v) = \operatorname{lerp}\big(\operatorname{lerp}(f_{00}, f_{10}, u),\ \operatorname{lerp}(f_{01}, f_{11}, u),\ v\big)
$$

展开后四个角的权重分别是 $(1-u)(1-v)$、$u(1-v)$、$(1-u)v$、$uv$，加起来恒为 1。在三角形里对应的是重心坐标，三个顶点的权重同样和为 1。动画里的混合空间就是这个思路：1D 混合空间在相邻两个样本之间做一次 Lerp，2D 混合空间在包含当前参数点的格子或三角形里算出几个样本的权重，再把这些样本的姿势按权重加权混合。权重和为 1 是关键，它保证混合结果不会整体偏移或缩放；叠加动画（Additive）之所以要单独处理，就是因为它的权重不满足这个约束，见 [[EBDC-叠加混合（Additive Blending）]]。

纹理的双线性过滤、GPU 光栅化时顶点属性在三角形内的插值（加上透视校正），用的也是同一套数学。

## 在 UE 里的常用接口

| 接口 | 作用 | 备注 |
| --- | --- | --- |
| `FMath::Lerp(A, B, Alpha)` | 模板，标量、`FVector`、`FLinearColor` 等都能用 | 不截断 Alpha |
| 蓝图 `Lerp`、`Lerp (Vector)`、`Lerp (Rotator)` | 同上 | `Lerp (Rotator)` 有 `Shortest Path` 选项 |
| `FQuat::Slerp` / `FQuat::FastLerp` | 四元数球面插值 / 不归一化的快速 Lerp | `FastLerp` 结果需要自己 `Normalize` |
| `FMath::InterpEaseInOut(A, B, Alpha, Exp)` | 对 Alpha 先做缓入缓出再 Lerp | 另有 `InterpEaseIn`、`InterpSinInOut` 等 |
| `FMath::FInterpTo` / `VInterpTo` / `RInterpTo` / `QInterpTo` | 每帧向目标靠近一段，距离越近靠得越慢 | 需要传 `DeltaTime` 和速度 |
| `FMath::FInterpConstantTo` 等 | 每帧以固定速度靠近目标 | 真正的匀速 |
| `FLinearColor::LerpUsingHSV` | 在 HSV 空间插值颜色 | 避免 RGB 插值中间发灰 |
| 材质 `LinearInterpolate`（Lerp） | 着色器里的 Lerp | 对应 HLSL 的 `lerp` |

Lerp 本身只负责“给定 $t$ 求值”，曲线的形状取决于 $t$ 怎么随时间变化。状态机过渡的 Blend Settings 里的 `Mode`（Linear、Cubic、Sinusoidal 等）、Timeline 里的曲线、`InterpEaseInOut` 的指数，都是在 Lerp 外面套一层 $t \mapsto f(t)$ 的缓动函数，Lerp 的公式本身不变。

## 每帧 Lerp 向目标

游戏代码里常见这样的写法：

```cpp
// 每帧执行：朝目标走剩余距离的一个固定比例
Current = FMath::Lerp(Current, Target, 0.1f);
```

它已经不是线性插值了，而是指数衰减：每帧剩下 90% 的距离，永远无限接近而不到达。而且它和帧率绑定，60 fps 和 30 fps 下同样一秒走过的比例不同。`FMath::FInterpTo` 的实现是 `Current + Dist * Clamp(DeltaTime * InterpSpeed, 0, 1)`，引入了 `DeltaTime`，但 `DeltaTime * InterpSpeed` 本身只是指数函数的一阶近似，帧率差距大时仍有偏差。要严格与帧率无关，用精确的指数形式：

```cpp
// 与帧率无关的平滑跟随：Lambda 越大越快，单位 1/秒
FVector SmoothFollow(const FVector& Current, const FVector& Target, float Lambda, float DeltaTime)
{
	const float Alpha = 1.f - FMath::Exp(-Lambda * DeltaTime);
	return FMath::Lerp(Current, Target, Alpha);
}
```

推导很简单：连续形式下剩余距离满足 $\frac{d}{dt}e = -\lambda e$，一帧后剩下 $e^{-\lambda \Delta t}$，所以这一帧应该走 $1 - e^{-\lambda \Delta t}$ 的比例。无论每帧多长，同样经过一秒剩下的都是 $e^{-\lambda}$。

## 容易踩的坑

**直接 Lerp 欧拉角。** 从 170° 插值到 -170°，Lerp 会绕远路转 340°。用 `Lerp (Rotator)` 勾上 `Shortest Path`，或者转成四元数插值。

**Lerp 四元数后不归一化。** `FQuat::FastLerp` 的结果模长小于 1，直接用来旋转会带缩放效果；多次累积后误差更大。

**在 sRGB 空间 Lerp 颜色。** 在 gamma 编码的值上插值，中间色会偏暗。先转成线性空间（`FLinearColor`）再插值，或者改用 HSV。

**把 `Lerp(Current, Target, 常数)` 当成匀速运动。** 那是帧率相关的指数逼近，见上一节。

## 相关

[[EAADK-球面线性插值]] [[EAADJ-球面非线性插值]] [[EAADO-姿势的插值]] [[EBC-动画状态机]] [[EBCB-Cross Fades]] [[EBDC-叠加混合（Additive Blending）]]
