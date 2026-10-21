**循环坐标下降（Cyclic Coordinate Descent，CCD）是一种迭代 IK 算法：从末端往根逐个处理关节，每次只转动一个关节，让“关节→末端”的方向对准“关节→目标”的方向，一轮处理完再从头开始，直到末端足够接近目标。它是“坐标下降”这一通用优化思想在 IK 上的应用，每次只沿一个关节的自由度求局部最优。UE 里对应 AnimGraph 的 CCDIK 节点（`FAnimNode_CCDIK`，官方标为 Experimental）。**

> 名称说明：本笔记文件名里的“循环星标下降”是误译，正确译名是“循环坐标下降”，coordinate 是“坐标”。文件名保持不变，正文一律用 CCD / 循环坐标下降。
>
> 出处：L.-C. T. Wang, C. C. Chen, *A Combined Optimization Method for Solving the Inverse Kinematics Problems of Mechanical Manipulators*, IEEE Transactions on Robotics and Automation, 1991；Chris Welman 1993 年在西蒙弗雷泽大学的硕士论文 *Inverse Kinematics and Geometric Constraints for Articulated Figure Manipulation* 把它引入角色动画，并加入了关节约束。

## 坐标下降：通用优化里的 CCD

原笔记描述的其实是通用的坐标下降法，这部分本身是对的：要最小化多元函数 $F(x_1, \dots, x_n)$，每次固定其他变量，只沿一个坐标方向优化，

$$
x_i^{(k+1)} = \arg\min_{x_i} F\big(x_1^{(k+1)}, \dots, x_{i-1}^{(k+1)}, x_i, x_{i+1}^{(k)}, \dots, x_n^{(k)}\big)
$$

按 $i = 1, \dots, n$ 循环。每一步是一维问题，常常有闭式解；它不需要梯度，也不需要求解线性方程组。代价是变量之间耦合很强时收敛慢，沿“斜着”的山谷只能走锯齿路线。坐标下降在机器学习里很常见（例如 Lasso 回归的求解），但和角色动画的关系只在于思路相同。

## 用在 IK 上

IK 的目标函数是末端到目标的距离 $F(\boldsymbol\theta) = \lVert \mathbf{e}(\boldsymbol\theta) - \mathbf{t} \rVert^2$，变量是各关节的旋转。固定其他关节、只转关节 $j$ 时，末端绕关节位置 $\mathbf{p}_j$ 做刚体旋转，一维最优解有几何闭式解：把向量

$$
\mathbf{u} = \frac{\mathbf{e} - \mathbf{p}_j}{\lVert \mathbf{e} - \mathbf{p}_j \rVert},\qquad
\mathbf{v} = \frac{\mathbf{t} - \mathbf{p}_j}{\lVert \mathbf{t} - \mathbf{p}_j \rVert}
$$

对齐即可，旋转轴 $\mathbf{a} = \mathbf{u} \times \mathbf{v}$（归一化），角度 $\phi = \arccos(\mathbf{u} \cdot \mathbf{v})$。对于球关节，这就是把 $\mathbf{u}$ 转到 $\mathbf{v}$ 的最短旋转；对于只有一个转轴 $\mathbf{h}$ 的铰链关节（肘、膝），先把 $\mathbf{u}$、$\mathbf{v}$ 投影到垂直于 $\mathbf{h}$ 的平面上，再求平面内的夹角。

一轮迭代从离末端最近的关节开始，依次往根处理（也可以从根开始，UE 的 CCDIK 节点有 `Start from Tail` 选项）。每转动一个关节，它下面的所有关节点都要跟着更新位置，然后处理下一个关节。

```cpp
// CCD 单链求解（球关节，无约束），演示用
// Joints[0] 为根，Joints.Num()-1 为末端
void SolveCCD(TArray<FVector>& Joints, const FVector& Target, float Tolerance, int32 MaxIterations)
{
	const int32 N = Joints.Num() - 1;
	for (int32 Iter = 0; Iter < MaxIterations; ++Iter)
	{
		for (int32 j = N - 1; j >= 0; --j)
		{
			const FVector ToEnd = (Joints[N] - Joints[j]).GetSafeNormal();
			const FVector ToTarget = (Target - Joints[j]).GetSafeNormal();
			// 把“关节→末端”转到“关节→目标”的最短旋转
			const FQuat Delta = FQuat::FindBetweenNormals(ToEnd, ToTarget);

			// 关节 j 以下的所有点绕 Joints[j] 旋转
			for (int32 k = j + 1; k <= N; ++k)
			{
				Joints[k] = Joints[j] + Delta.RotateVector(Joints[k] - Joints[j]);
			}
		}
		if (FVector::Dist(Joints[N], Target) <= Tolerance) { break; }
	}
}
```

实际实现里还要把 `Delta` 累乘到对应骨骼的旋转上，并在每步后施加关节限制。这段代码最容易写错的地方是忘了更新子关节的位置，导致后面的关节用的是旧的末端位置。

## 特点

**每一步极便宜。** 一次叉积、一次点积、一次旋转，和雅可比法的矩阵运算相比开销很小。

**约束好加。** 每次只转一个关节，转完立刻把它夹到允许的角度范围里即可。这是 CCD 相对 FABRIK 的主要优势；UE 里也正好是 CCDIK 节点提供每关节旋转限制，而 FABRIK 节点没有。

**弯曲集中在链尾。** 离末端最近的关节先转，常常一步就把末端带到目标附近，后面靠根的关节几乎不用动。结果是手腕、指尖扭得厉害，肩膀不动，链容易卷成钩状或螺旋状，看起来不自然。

**可能振荡、收敛慢。** 目标在链“背后”时可能要很多轮；接近伸直时每一步的改进很小。

常见的改良：

| 手段 | 作用 |
| --- | --- |
| 每步限制最大转角（阻尼） | 避免链尾一步转过头，让调整分摊到更多关节 |
| 按关节加权 | 让靠根的关节多承担一些 |
| 从根开始处理（或交替方向） | 改变弯曲的分布 |
| 从上一帧的解开始迭代 | 目标连续移动时几轮就能跟上 |

## 和其他方法比较

| | CCD | FABRIK | 雅可比（DLS） |
| --- | --- | --- | --- |
| 迭代变量 | 关节角 | 关节位置 | 关节角 |
| 单次迭代开销 | $O(n)$，含三角运算 | $O(n)$，只有归一化 | 至少一次 $m\times m$ 或 $n\times n$ 线性方程求解 |
| 姿势自然度 | 偏向链尾弯曲 | 较自然 | 取决于参数 |
| 关节约束 | 容易 | 需要投影，稍复杂 | 可纳入，但实现复杂 |
| 多末端 | 不直接支持 | 支持（按分叉点合并） | 自然支持 |

## 在 UE 里

CCDIK 节点在 AnimGraph 右键添加，工作在组件空间，属性有：

| 属性 | 作用 |
| --- | --- |
| `Effector Location` / `Effector Location Space` / `Effector Target` | 目标位置、所在空间，空间为 Bone 时参照的骨骼 |
| `Tip Bone` / `Root Bone` | 链的末端与起点 |
| `Precision` | 末端与目标的容差，默认 1 |
| `Max Iterations` | 最大迭代次数，默认 10 |
| `Start from Tail` | 文档描述为调试绘制从 Tip 开始画到 Root |
| `Enable Rotation Limit` | 启用每关节旋转限制 |
| `Rotation Limit Per Joints` | 每个关节一项，索引 0 对应紧挨 Tip 的关节，依次往 Root 方向 |

Epic 文档里的示例是让角色食指（`index_03_l`）去按按钮，Root 选左锁骨（`clavicle_l`），用旋转限制防止手指和手臂过度弯折。节点的底层求解函数是 AnimationCore 模块里的 `AnimationCore::SolveCCDIK`，Control Rig 也有对应的 CCDIK 节点（`RigUnit_CCDIK`）。节点是 Experimental 状态，发行项目中使用要谨慎。

## 容易踩的坑

**以为 `Start from Tail` 会改变求解方向。** 按 Epic 文档的描述，这个选项影响的是调试绘制的方向；实际的求解顺序以源码为准，不要依赖文档之外的推测。

**旋转限制的索引顺序弄反。** 索引 0 是靠近 Tip 的关节，不是靠近 Root 的。

**没有限制时链卷成一团。** CCD 的无约束解经常很难看，至少给末端几个关节加角度上限。

**用 CCD 解两骨骼四肢。** 三关节链有解析解（Two Bone IK），CCD 在这里既慢又不稳定。

## 相关

[[ECAB-IK和FK]] [[ECAA-FABRIK]] [[ECAI-雅可比矩阵法]] [[EAABAA-IK]]
