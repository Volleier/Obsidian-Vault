**NLERP（Normalized Linear Interpolation，归一化线性插值）是对两个单位四元数先做普通的分量线性插值、再归一化回单位长度的旋转插值方法。它的路径和 SLERP 完全相同（都在单位超球面的同一段大圆弧上），只是沿弧的速度不均匀：两端慢、中间快。它计算便宜、满足交换律，适合多路动画混合，是游戏动画系统里实际用得最多的旋转混合方式。UE 里对应 `FQuat::FastLerp` 加 `Normalize()`，以及姿势混合时 `FTransform` 的"按最短路径累加旋转再归一化"。**

> 参考：Jonathan Blow, *Understanding Slerp, Then Not Using It*（Game Developer, 2004 年 4 月）；《Game Engine Architecture》第 3 版数学与动画两章；UE 部分对照 5.x `TQuat` API 文档。笔记标题"球面非线性插值"是沿用的叫法，NLERP 的 N 实际指 Normalized（归一化），它并不是在"球面上非线性地插值"。

## 公式

给定单位四元数 $\mathbf{q}_1$、$\mathbf{q}_2$ 和插值参数 $t\in[0,1]$：

$$
\text{nlerp}(\mathbf{q}_1, \mathbf{q}_2, t) = \frac{(1-t)\,\mathbf{q}_1 + t\,\mathbf{q}_2}{\left\lVert (1-t)\,\mathbf{q}_1 + t\,\mathbf{q}_2 \right\rVert}
$$

分子是 4D 空间里连接两点的弦上的一点，它不在单位超球面上（长度小于 1），除以长度就把它投影回球面。因为投影是从原点出发的，弦上每一点投影到的正是两点之间那段大圆弧，所以 NLERP 和 [[EAADK-球面线性插值]]（SLERP）走的是同一条路径。

## 最短路径

四元数是旋转的"双重覆盖"：$\mathbf{q}$ 和 $-\mathbf{q}$ 表示同一个旋转。在 4D 球面上从 $\mathbf{q}_1$ 到 $\mathbf{q}_2$ 有两条弧，一条去 $\mathbf{q}_2$，一条去 $-\mathbf{q}_2$，它们对应的三维旋转一个转了 $\theta$，另一个转了 $360^\circ - \theta$。

判断的依据是点积：$\mathbf{q}_1\cdot\mathbf{q}_2 = \cos\Omega$，$\Omega$ 是两者在 4D 空间中的夹角，而对应的三维旋转角差是 $2\Omega$。点积为负说明 $\Omega > 90^\circ$，也就是旋转差大于 $180^\circ$，此时应该把 $\mathbf{q}_2$ 取负，改走另一条更短的弧。原笔记里说点积为负意味着"两个四元数之间的夹角大于 180 度"，这里要分清：4D 夹角大于 90°，对应的三维旋转差才大于 180°。

$$
\text{if}\ \ \mathbf{q}_1\cdot\mathbf{q}_2 < 0:\quad \mathbf{q}_2 \leftarrow -\mathbf{q}_2
$$

不做这一步，角色在某些插值中会"绕远路"转一大圈，比如从朝向 $170^\circ$ 转到 $-170^\circ$ 时，本应转 $20^\circ$，却转了 $340^\circ$。

## 速度为什么不均匀

把问题放到两个四元数张成的平面上：令 $\mathbf{q}_1 = (1, 0)$、$\mathbf{q}_2 = (\cos\Omega, \sin\Omega)$。弦上的点是 $\big((1-t) + t\cos\Omega,\ t\sin\Omega\big)$，归一化后的角度为

$$
\varphi(t) = \operatorname{atan2}\!\big(t\sin\Omega,\ 1 - t + t\cos\Omega\big)
$$

求导可得 $\varphi'(t) = \sin\Omega / r(t)^2$，$r(t)$ 是弦上点到原点的距离。两端 $r=1$，速度是 $\sin\Omega$；中点 $r^2 = (1+\cos\Omega)/2$，速度是 $2\tan(\Omega/2)$。弦的中间离原点最近，投影后被"放大"得最多，所以中间快、两端慢。

差异随夹角增大：三维旋转 90°（$\Omega=45^\circ$）时，中点速度约为端点的 1.17 倍；三维旋转 180°（$\Omega=90^\circ$）时达到 2 倍。动画里相邻关键帧之间的旋转通常只有几度，这个差异完全看不出来；对于混合权重而言，速度的不均匀表现为权重和实际旋转比例之间的轻微非线性，一般也可以接受。

## 为什么游戏里更常用 NLERP

Blow 在那篇文章里把旋转插值的理想性质归结为三条：**交换律**（多个旋转混合时结果和顺序无关）、**恒定角速度**、**最小力矩**（走最短的大圆弧）。任何方法最多只能满足其中两条：

| 方法 | 交换律 | 恒定角速度 | 最小力矩 |
| --- | --- | --- | --- |
| SLERP | 否 | 是 | 是 |
| NLERP | 是 | 否 | 是 |
| 对数四元数线性插值 | 是 | 是 | 否 |

对动画混合来说，交换律比恒定角速度重要得多。动画混合树里经常要把三个、五个甚至更多姿势按权重混合（比如混合空间），NLERP 可以直接写成加权和再归一化：

$$
\mathbf{q} = \operatorname{normalize}\left(\sum_{k} w_k\,s_k\,\mathbf{q}_k\right),\qquad s_k = \operatorname{sign}(\mathbf{q}_{\text{ref}}\cdot\mathbf{q}_k)
$$

$s_k$ 是相对某个参考四元数的最短路径符号修正。SLERP 只定义了两个量之间的插值，多路混合只能两两嵌套，结果依赖嵌套顺序。再加上 NLERP 没有三角函数，也不存在 SLERP 在夹角很小时除以 $\sin\Omega$ 的数值问题，它就成了动画混合的默认选择。

## 在 UE 里

`FQuat::FastLerp(A, B, Alpha)` 就是不带归一化的 NLERP：API 文档写明"Fast Linear Quaternion Interpolation. Result is NOT normalized."，实现里已经按点积符号做了最短路径修正。需要单位四元数时要自己归一化：

```cpp
// NLERP：FastLerp 已处理最短路径，但结果需要手动归一化
FQuat NLerp(const FQuat& A, const FQuat& B, float Alpha)
{
	FQuat Result = FQuat::FastLerp(A, B, Alpha);
	Result.Normalize();
	return Result;
}
```

同族的还有 `FQuat::FastBilerp`（双线性，同样不归一化）。对照来看，`FQuat::Slerp` 会修正对齐并返回归一化结果，`Slerp_NotNormalized` 修正对齐但不归一化，`SlerpFullPath` 不做最短路径检查。

动画姿势的多路混合走的是同一个思路。据社区对源码的分析，`FAnimationRuntime` 混合多个姿势时对每个骨骼调用 `FTransform::AccumulateWithShortestRotation(Source, Weight)` 做"按最短路径符号修正后的加权累加"，全部累加完再调用 `NormalizeRotation()`，这正是上面加权和再归一化的公式；具体调用链以你的引擎源码为准。

## 容易踩的坑

**忘记归一化。** 用 `FastLerp` 的结果直接构造变换，非单位四元数会给旋转带上缩放。多次累加后误差还会积累。

**最短路径检查放错位置。** 多路混合时，所有四元数都要相对同一个参考（通常是第一个或权重最大的那个）判断符号，而不是两两比较。

**把 NLERP 用在大角度的匀速转动上。** 比如摄像机匀速转 180°，NLERP 中段速度会快一倍，这种场合用 SLERP。

## 相关

[[EAADK-球面线性插值]] [[EAADO-姿势的插值]] [[EBAC-线性插值]] [[EBCB-Cross Fades]] [[EBDC-叠加混合（Additive Blending）]]
