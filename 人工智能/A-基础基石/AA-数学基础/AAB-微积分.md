
微积分是研究变化与累积的数学分支，在人工智能中充当优化引擎——它通过导数和梯度告诉我们应如何调整模型参数来降低损失函数值。从线性回归的最小二乘目标到深度神经网络的反向传播，微积分贯穿了机器学习模型训练的每个环节。

一元微分基础

导数的直观意义是函数在某一点的瞬时变化率。对于函数 $f(x)$，在点 $x_0$ 处的导数定义为：

$$f'(x_0) = \lim_{h \to 0} \frac{f(x_0 + h) - f(x_0)}{h}$$

几何上，导数表示函数曲线在 $x_0$ 处的切线斜率。若 $f'(x_0) > 0$，函数在 $x_0$ 处递增；若 $f'(x_0) < 0$，函数递减；若 $f'(x_0) = 0$，该点可能是极值点（驻点）。

常用导数规则：

常数规则：$(c)' = 0$

幂函数规则：$(x^n)' = nx^{n-1}$

和差规则：$(f \pm g)' = f' \pm g'$

乘积规则：$(fg)' = f'g + fg'$

商规则：$\left(\frac{f}{g}\right)' = \frac{f'g - fg'}{g^2}$

链式法则：$(f(g(x)))' = f'(g(x)) \cdot g'(x)$

链式法则是反向传播算法的数学基础——它让我们可以处理嵌套函数（深层神经网络）的复合求导。

泰勒展开将光滑函数在一点附近近似为多项式之和：

$$f(x) \approx f(a) + f'(a)(x-a) + \frac{f''(a)}{2!}(x-a)^2 + \frac{f'''(a)}{3!}(x-a)^3 + \dots$$

一阶泰勒展开 $f(x) \approx f(a) + f'(a)(x-a)$ 就是线性近似，梯度下降每一步实质上是在当前点用一阶近似来指引下降方向。保留到二阶的展开则用于牛顿法等二阶优化方法。

多元微积分与梯度

当函数有多个输入变量时，每个变量单独求导得到偏导数。对于函数 $f(x_1, x_2, \dots, x_n)$，关于 $x_i$ 的偏导数记作：

$$\frac{\partial f}{\partial x_i}$$

梯度（Gradient）是所有偏导数组成的向量：

$$\nabla f = \left(\frac{\partial f}{\partial x_1}, \frac{\partial f}{\partial x_2}, \dots, \frac{\partial f}{\partial x_n}\right)$$

梯度的方向是函数增长最快的方向，梯度的反方向 $-\nabla f$ 则是函数下降最快的方向。这正是梯度下降算法的几何直觉。

梯度具有以下重要性质：

梯度在极值点处为零向量：$\nabla f(\mathbf{x}^*) = \mathbf{0}$

梯度与水平集垂直：在 $f$ 的等值面上，梯度方向沿法线方向

方向导数：沿单位向量 $\mathbf{u}$ 方向的变化率为 $\nabla f \cdot \mathbf{u}$

在机器学习中，损失函数 $J(\mathbf{w})$ 的梯度 $\nabla_{\mathbf{w}} J$ 指示了每个参数 $w_i$ 对损失值的贡献方向，参数更新规则为：

$$\mathbf{w} \leftarrow \mathbf{w} - \eta \nabla_{\mathbf{w}} J$$

其中 $\eta$ 为学习率。

雅可比矩阵与向量值函数求导

当函数从一个向量空间映射到另一个向量空间时，需要雅可比矩阵来描述导数。对于映射 $\mathbf{f}: \mathbb{R}^n \to \mathbb{R}^m$，雅可比矩阵 $\mathbf{J} \in \mathbb{R}^{m \times n}$ 的各元素为：

$$\mathbf{J}_{ij} = \frac{\partial f_i}{\partial x_j}$$

雅可比矩阵的每一行是输出分量 $f_i$ 的梯度，每一列表示所有输出对某个输入的敏感度。在神经网络中，每一层的前向和后向运算都涉及雅可比矩阵的计算。

海森矩阵与曲率

海森矩阵是二阶偏导数组成的方阵，对于函数 $f: \mathbb{R}^n \to \mathbb{R}$：

$$\mathbf{H}_{ij} = \frac{\partial^2 f}{\partial x_i \partial x_j}$$

海森矩阵描述函数在某点处的曲率信息：

海森矩阵的正定性：若 $\mathbf{H} \succ 0$（所有特征值 > 0），该点是局部极小值；若 $\mathbf{H} \prec 0$（所有特征值 < 0），该点是局部极大值；混合符号的特征值对应鞍点。

条件数（Condition Number）：$\kappa(\mathbf{H}) = \frac{\lambda_{\max}}{\lambda_{\min}}$ 越大，梯度下降越慢、路径越振荡——因为不同方向上的曲率差异悬殊。

牛顿法利用海森矩阵进行二阶优化：$\mathbf{w} \leftarrow \mathbf{w} - \mathbf{H}^{-1}\nabla f$，一步到位调整曲率影响，收敛更快但计算代价高。实际中常用拟牛顿法（如 L-BFGS）来近似海森矩阵的逆。

矩阵微积分

当函数以矩阵为输入或输出时，需要矩阵微积分来高效求导。常见约定分为分子布局（Numerator Layout）和分母布局（Denominator Layout）。关键结果包括：

标量对向量的梯度：$\frac{\partial}{\partial \mathbf{x}}(\mathbf{a}^\top\mathbf{x}) = \mathbf{a}$

标量对矩阵求梯度的二次型：$\frac{\partial}{\partial \mathbf{x}}(\mathbf{x}^\top\mathbf{A}\mathbf{x}) = (\mathbf{A} + \mathbf{A}^\top)\mathbf{x}$

标量对矩阵求梯度的线性型：$\frac{\partial}{\partial \mathbf{X}}(\mathbf{a}^\top\mathbf{X}\mathbf{b}) = \mathbf{a}\mathbf{b}^\top$

迹的矩阵求导：$\frac{\partial}{\partial \mathbf{X}}\text{tr}(\mathbf{X}\mathbf{A}) = \mathbf{A}^\top$

这些规则在手动推导机器学习目标函数的梯度时极为实用。例如推导线性回归闭式解时，将 $\|\mathbf{X}\mathbf{w} - \mathbf{y}\|^2$ 展开并对 $\mathbf{w}$ 求导，令梯度为零可得法方程。

反向传播的微积分基础

反向传播（Backpropagation）是计算图上的自动微分实现，其数学本质是链式法则的系统化应用。考虑一个简单的三层网络：

$$\mathbf{z}_1 = \mathbf{W}_1\mathbf{x}, \quad \mathbf{a}_1 = \sigma(\mathbf{z}_1)$$
$$\mathbf{z}_2 = \mathbf{W}_2\mathbf{a}_1, \quad \hat{\mathbf{y}} = \sigma(\mathbf{z}_2)$$
$$L = \|\hat{\mathbf{y}} - \mathbf{y}\|^2$$

要计算 $\frac{\partial L}{\partial \mathbf{W}_1}$，链式法则展开为：

$$\frac{\partial L}{\partial \mathbf{W}_1} = \frac{\partial L}{\partial \hat{\mathbf{y}}} \cdot \frac{\partial \hat{\mathbf{y}}}{\partial \mathbf{z}_2} \cdot \frac{\partial \mathbf{z}_2}{\partial \mathbf{a}_1} \cdot \frac{\partial \mathbf{a}_1}{\partial \mathbf{z}_1} \cdot \frac{\partial \mathbf{z}_1}{\partial \mathbf{W}_1}$$

反向传播从顶层损失出发，逐层将误差信号 $\delta$（局部梯度与上游梯度的乘积）向底层传播。这种从后往前递归应用链式法则的方式，使得计算所有参数的梯度只需一次前向传播加一次反向传播，复杂度与网络深度线性相关。

对于深度学习框架（PyTorch、TensorFlow）中的自动微分（Autograd），计算图上的每个节点不仅存储前向计算结果，还记录了局部的雅可比矩阵（或向量-雅可比积），反向遍历计算图时自动合成全局梯度。

积分与概率密度

积分在机器学习中的主要作用体现在连续概率模型中。概率密度函数 $p(x)$ 满足：
$$\int_{-\infty}^{\infty} p(x) \,dx = 1$$

连续随机变量 $X$ 的期望定义为：
$$\mathbb{E}[X] = \int x \, p(x) \,dx$$

在高斯分布中：
$$p(x) = \frac{1}{\sqrt{2\pi\sigma^2}} \exp\left(-\frac{(x-\mu)^2}{2\sigma^2}\right)$$

归一化常数 $\frac{1}{\sqrt{2\pi\sigma^2}}$ 由高斯积分 $\int_{-\infty}^{\infty} e^{-x^2}dx = \sqrt{\pi}$ 导出。在高维空间中，多元高斯分布的归一化涉及更复杂的矩阵积分。

在变分推断中，对后验分布 $p(\mathbf{z}|\mathbf{x})$ 的积分通常是不可解的（intractable），因此引出近似方法——用可解的分布 $q(\mathbf{z})$ 去逼近真实后验，优化证据下界（ELBO）：
$$\mathcal{L} = \mathbb{E}_{q}[\log p(\mathbf{x},\mathbf{z})] - \mathbb{E}_{q}[\log q(\mathbf{z})]$$

微积分在机器学习中的典型应用总结

梯度下降：利用一阶导数（梯度）信息迭代更新参数，使损失函数收敛到局部最小值

反向传播：链式法则的系统化应用，高效计算深层网络的参数梯度

损失函数设计：交叉熵损失、均方误差等目标函数的梯度需手动推导，均依赖微分

激活函数分析：Sigmoid 的导数 $\sigma'(x) = \sigma(x)(1 - \sigma(x))$，ReLU 的导数分段函数特性，影响梯度流动和网络训练

二阶优化方法：牛顿法、自然梯度下降等利用海森矩阵或 Fisher 信息矩阵的曲率信息

变分推断：通过泛函微分（变分法）推导最优近似分布的条件

连续强化学习：策略梯度定理 $\nabla_\theta J = \mathbb{E}[\nabla_\theta \log \pi_\theta(a|s) Q(s,a)]$ 涉及对策略参数的期望梯度
