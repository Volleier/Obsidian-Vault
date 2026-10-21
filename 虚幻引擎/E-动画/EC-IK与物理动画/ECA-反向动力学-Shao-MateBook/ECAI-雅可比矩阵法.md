**雅可比矩阵法是一类数值 IK 方法：在当前姿势处把“关节角 → 末端位置”的非线性映射线性化为雅可比矩阵 $J$，用它的转置、伪逆或阻尼最小二乘逆求出一小步关节角增量，更新后重新线性化，反复迭代逼近目标。它通用性最强，能同时处理多个末端、位置和朝向目标，以及次要目标，代价是每步都要做矩阵运算，并且要处理奇异姿势。**

> 参考：Samuel R. Buss, *Introduction to Inverse Kinematics with Jacobian Transpose, Pseudoinverse and Damped Least Squares methods*（2004 年讲义，常被引用的入门综述）；阻尼最小二乘法最早见于 Wampler（1986）、Nakamura 与 Hanafusa（1986）。

## 雅可比矩阵

设链有 $n$ 个关节角 $\boldsymbol\theta = (\theta_1, \dots, \theta_n)^T$，末端（可以有多个）的位置拼成 $m$ 维向量 $\mathbf{e}(\boldsymbol\theta)$，一个末端只管位置时 $m = 3$，再加朝向是 $m = 6$。雅可比矩阵是 $m \times n$ 的偏导数矩阵：

$$
J(\boldsymbol\theta) = \frac{\partial \mathbf{e}}{\partial \boldsymbol\theta},\qquad J_{ij} = \frac{\partial e_i}{\partial \theta_j}
$$

它描述的是“每个关节转一点点，末端怎么动”：$\Delta\mathbf{e} \approx J\,\Delta\boldsymbol\theta$。

对转动关节，不需要真的求导。关节 $j$ 位于 $\mathbf{p}_j$，转轴（世界空间单位向量）为 $\mathbf{a}_j$，绕它转动时末端的线速度就是

$$
\frac{\partial \mathbf{e}}{\partial \theta_j} = \mathbf{a}_j \times (\mathbf{e} - \mathbf{p}_j)
$$

这就是第 $j$ 列。若同时约束末端朝向，对应的角速度部分就是 $\mathbf{a}_j$ 本身。球关节可以拆成三个互相垂直轴的转动关节。这样整张 $J$ 用一次 FK 就能算出来，开销是 $O(mn)$。

## 求增量的几种方法

误差 $\mathbf{r} = \mathbf{t} - \mathbf{e}$，要找 $\Delta\boldsymbol\theta$ 使 $J\,\Delta\boldsymbol\theta \approx \mathbf{r}$。$J$ 一般不是方阵（人体链通常 $n > m$，冗余），也可能奇异，不能直接求逆。

| 方法 | 公式 | 特点 |
| --- | --- | --- |
| 雅可比转置 | $\Delta\boldsymbol\theta = \alpha\,J^T\mathbf{r}$ | 最便宜，不求逆；本质是梯度下降，收敛慢，步长 $\alpha$ 要调 |
| 伪逆 | $\Delta\boldsymbol\theta = J^{+}\mathbf{r}$ | 最小范数的最小二乘解；接近奇异时增量爆炸 |
| 阻尼最小二乘（DLS） | $\Delta\boldsymbol\theta = J^T(JJ^T + \lambda^2 I)^{-1}\mathbf{r}$ | 最常用；$\lambda$ 越大越稳、越慢 |

伪逆的形式要按 $J$ 的形状选：冗余链（$n > m$，$J$ 行满秩）用右伪逆 $J^{+} = J^T(JJ^T)^{-1}$；原笔记写的 $(J^TJ)^{-1}J^T$ 是左伪逆，只在 $n \le m$ 且列满秩时成立，对冗余链 $J^TJ$ 必然奇异。一般情况下用 SVD 求 $J^{+}$ 最稳妥：$J = U\Sigma V^T$，$J^{+} = V\Sigma^{+}U^T$，其中 $\Sigma^{+}$ 把非零奇异值取倒数。

DLS 可以看成最小化 $\lVert J\Delta\boldsymbol\theta - \mathbf{r} \rVert^2 + \lambda^2 \lVert \Delta\boldsymbol\theta \rVert^2$，用 SVD 写成

$$
\Delta\boldsymbol\theta = \sum_i \frac{\sigma_i}{\sigma_i^2 + \lambda^2}\,\mathbf{v}_i\,\mathbf{u}_i^T\mathbf{r}
$$

奇异值 $\sigma_i$ 远大于 $\lambda$ 时，系数约为 $1/\sigma_i$，和伪逆一样；$\sigma_i \to 0$ 时系数趋于 0，而不是像伪逆那样趋于无穷。这就是它在奇异附近稳定的原因。

## 冗余与零空间

冗余链在到达目标后还有剩余自由度（比如肘部可以绕肩腕连线转）。伪逆解之外，可以加上 $J$ 零空间里的任意分量而不影响末端：

$$
\Delta\boldsymbol\theta = J^{+}\mathbf{r} + (I - J^{+}J)\,\mathbf{z}
$$

$\mathbf{z}$ 可以取某个次要目标的梯度，例如“关节角靠近舒适姿势”“远离关节极限”。这是雅可比法相对 CCD、FABRIK 最有特色的能力：主要目标和次要目标在数学上分得很清楚。多个末端时，把它们的误差向量和雅可比矩阵按行堆叠即可，每个末端还可以加权。

## 迭代与实现

```python
import numpy as np

def solve_ik_dls(theta, fk, jacobian, target, lam=0.1, max_step=0.2,
                 tol=1e-3, max_iter=50):
    """阻尼最小二乘 IK。
    fk(theta) -> 末端位置 (m,)；jacobian(theta) -> (m, n)；target -> (m,)
    """
    for _ in range(max_iter):
        e = fk(theta)
        r = target - e
        if np.linalg.norm(r) < tol:
            break
        # 误差太大时截断，避免一步迈出线性化有效范围
        n = np.linalg.norm(r)
        if n > max_step:
            r = r * (max_step / n)
        J = jacobian(theta)
        m = J.shape[0]
        # 解 (J J^T + λ²I) y = r，再 Δθ = J^T y；不显式求逆
        y = np.linalg.solve(J @ J.T + (lam ** 2) * np.eye(m), r)
        theta = theta + J.T @ y
    return theta
```

原笔记的示例在更新关节角后才更新末端位置，却用更新前的误差判断收敛，这里改成每轮开始时先做 FK 再判断。要点有两个：误差向量要截断，因为线性化只在小范围内成立，目标很远时一步迈过去会跑飞；解线性方程而不是显式求逆，数值更稳、更快。$\lambda$ 常见的做法是固定一个经验值，或者根据最小奇异值、离目标的距离自适应调整（Buss 讲义里还介绍了按奇异值分别阻尼的 Selectively Damped Least Squares）。

## 代价与局限

每步至少要解一个 $m \times m$ 线性方程组（$m$ 是约束维数），或做一次 SVD；关节越多、末端越多越贵。它是局部线性化方法，只会走向附近的局部解，目标在链“背后”时可能卡住。关节角限制不能直接写进上面的公式，常见做法是更新后截断，或者把超限关节从 $J$ 中临时去掉，或者改用带约束的二次规划。

| | 雅可比（DLS） | CCD | FABRIK |
| --- | --- | --- | --- |
| 单步开销 | 矩阵运算，较高 | 低 | 低 |
| 多末端、朝向目标 | 自然支持 | 不直接支持 | 支持多末端，朝向较麻烦 |
| 次要目标（零空间） | 支持 | 不支持 | 不支持 |
| 奇异处理 | 靠阻尼 | 无此问题 | 无此问题 |

## 在 UE 里

UE 的 AnimGraph IK 节点里没有雅可比求解器：Two Bone IK 是解析解，FABRIK、CCDIK 各用自己的迭代法。多末端全身 IK 由 Control Rig 和 IK Rig 的 Full Body IK 提供，Epic 文档说明它建立在“基于位置的 IK”框架上，而不是雅可比法。社区资料提到 4.26 时代实验性的 FullBodyIK 插件曾采用雅可比伪逆 / 阻尼最小二乘，后来被基于位置的求解器取代，这一点未在官方文档中核实。

在 UE 里自己实现雅可比 IK 时，可以用 `FTransform` 链做 FK 求出各关节位置和轴，雅可比每一列就是 `FVector::CrossProduct(Axis, EffectorPos - JointPos)`；线性方程组规模小（位置约束 $m = 3$），手写 3×3 求解即可，放在 Control Rig 的自定义节点或 `FAnimNode_SkeletalControlBase` 子类里运行。

## 容易踩的坑

**冗余链用了左伪逆。** $n > m$ 时 $J^TJ$ 奇异，公式直接失效。用右伪逆、SVD 或 DLS。

**不截断误差。** 目标很远时一步增量巨大，姿势乱飞甚至发散。

**阻尼太小或太大。** 太小，接近伸直时关节抖动；太大，收敛慢、够不到目标。

**雅可比的轴用了局部空间。** 列公式里的 $\mathbf{a}_j$、$\mathbf{p}_j$、$\mathbf{e}$ 必须在同一空间（通常是组件空间），并且每次迭代后要用新姿势重算。

## 相关

[[ECAB-IK和FK]] [[ECAA-FABRIK]] [[ECAH-循环星标下降算法]] [[EAABAA-IK]]
