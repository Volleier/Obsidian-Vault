
概率论为人工智能提供了处理不确定性的数学语言。在机器学习中，数据本身就包含噪声和随机性，模型需要量化预测的置信度，而贝叶斯框架则将先验知识与观测数据结合起来进行推理。从朴素贝叶斯分类器到深度生成模型，概率论是贯穿 AI 各领域的核心工具。

概率空间与公理

概率论建立在三个基本要素之上：样本空间 $\Omega$（所有可能结果的集合）、事件集合 $\mathcal{F}$、以及概率测度 $P$。柯尔莫哥洛夫公理定义了概率的基本性质：对于任意事件 $A \subseteq \Omega$，有 $0 \leq P(A) \leq 1$；$P(\Omega) = 1$；对于互不相容的事件 $A_1, A_2, \dots$，有 $P(\bigcup_i A_i) = \sum_i P(A_i)$。

条件的概率是机器学习中最常用的操作之一。事件 $B$ 发生的前提下，事件 $A$ 发生的条件概率为：

$$P(A|B) = \frac{P(A \cap B)}{P(B)}, \quad P(B) > 0$$

全概率公式：若 $\{B_i\}$ 构成样本空间的一个划分，则：
$$P(A) = \sum_i P(A|B_i) P(B_i)$$

贝叶斯定理

贝叶斯定理是概率视角下机器学习的基石公式。它将条件概率 $P(A|B)$ 反转过来：

$$P(A|B) = \frac{P(B|A) P(A)}{P(B)}$$

在机器学习语境下，这个公式被重新诠释为后验概率的更新规则：
$$P(\theta|\mathcal{D}) = \frac{P(\mathcal{D}|\theta) P(\theta)}{P(\mathcal{D})}$$

其中 $\theta$ 为模型参数，$\mathcal{D}$ 为观测数据：

$P(\theta)$ 是先验概率，代表在观测数据前对参数的已有信念

$P(\mathcal{D}|\theta)$ 是似然函数，量化在给定参数下观测到当前数据的可能性

$P(\theta|\mathcal{D})$ 是后验概率，综合了先验信念和数据的证据后对参数的更新信念

$P(\mathcal{D}) = \int P(\mathcal{D}|\theta)P(\theta) d\theta$ 是边际似然，充当归一化常数

贝叶斯定理体现了"从数据中学习"的核心逻辑：先验被数据更新为后验，后验又成为新一轮学习的先验，如此迭代逼近真实分布。

随机变量与分布

随机变量是从样本空间到实数的映射。离散随机变量取有限或可数个值，其概率质量函数（PMF）满足 $P(X = x_i) = p_i$，$\sum_i p_i = 1$；连续随机变量的概率密度函数（PDF）$p(x)$ 满足 $\int p(x)dx = 1$，且 $P(a \leq X \leq b) = \int_a^b p(x)dx$。累积分布函数 $F(x) = P(X \leq x)$ 对两类变量均适用。

常见离散分布：

伯努利分布 Bernoulli$(p)$：一次二元试验，$P(X=1)=p, P(X=0)=1-p$，用于二分类输出的建模

二项分布 Binomial$(n,p)$：$n$ 次独立伯努利试验中成功次数的分布，$P(X=k) = \binom{n}{k} p^k (1-p)^{n-k}$

多项分布 Multinomial$(n,\mathbf{p})$：每次试验有多个类别，描述各类别出现频次的联合分布

分类分布 Categorical$(\mathbf{p})$：一次多类别试验的结果分布，是多项分布在 $n=1$ 时的特例，广泛用于多分类的输出层

泊松分布 Poisson$(\lambda)$：$P(X=k) = \frac{\lambda^k e^{-\lambda}}{k!}$，描述单位时间内随机事件发生的次数

常见连续分布：

均匀分布 Uniform$(a,b)$：在区间 $[a,b]$ 上等概率取值，$p(x) = \frac{1}{b-a}$，常用作无信息先验

高斯（正态）分布 $\mathcal{N}(\mu, \sigma^2)$：$p(x) = \frac{1}{\sqrt{2\pi\sigma^2}} \exp\left(-\frac{(x-\mu)^2}{2\sigma^2}\right)$，由中心极限定理保证其在自然界的广泛出现，是整个机器学习的核心分布

多元高斯分布 $\mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$：$p(\mathbf{x}) = \frac{1}{(2\pi)^{d/2}|\boldsymbol{\Sigma}|^{1/2}} \exp\left(-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^\top\boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})\right)$，协方差矩阵 $\boldsymbol{\Sigma}$ 刻画了各维度之间的关联

指数分布 Exponential$(\lambda)$：$p(x) = \lambda e^{-\lambda x}, x \geq 0$，用于 Poisson 过程中事件间隔的建模

Beta 分布 Beta$(\alpha,\beta)$：定义在 $[0,1]$ 区间上，形式灵活，是二项分布的共轭先验，在 Beta-Binomial 模型中用于参数估计

Dirichlet 分布：Beta 分布的多元推广，是多项分布的共轭先验，常用于主题模型（LDA）中的主题-词分布建模

指数族（Exponential Family）统一了大多数常见分布的形式：
$$p(x|\boldsymbol{\eta}) = h(x) \exp\{\boldsymbol{\eta}^\top\mathbf{T}(x) - A(\boldsymbol{\eta})\}$$

其中 $\boldsymbol{\eta}$ 为自然参数，$\mathbf{T}(x)$ 为充分统计量，$A(\boldsymbol{\eta})$ 为对数配分函数。指数族具有优良的性质——凸的对数似然保证了优化问题的全局最优性，也为广义线性模型的推导提供了统一框架。

共轭先验是指后验分布与先验分布属于同一分布族的情况：Beta-Binomial、Dirichlet-Multinomial、Gaussian-Gaussian（均值未知）等均为典型例子。共轭性使贝叶斯更新拥有闭式解，极大简化了推论。

期望、方差与矩

期望衡量随机变量的"平均"位置，方差衡量其"散布"程度。对于连续随机变量：
$$\mathbb{E}[X] = \int x p(x) dx, \quad \text{Var}(X) = \mathbb{E}[(X - \mathbb{E}[X])^2] = \mathbb{E}[X^2] - (\mathbb{E}[X])^2$$

协方差量化两个随机变量的线性关联：
$$\text{Cov}(X,Y) = \mathbb{E}[(X - \mathbb{E}[X])(Y - \mathbb{E}[Y])]$$

相关系数将其归一化到 $[-1, 1]$：
$$\rho_{XY} = \frac{\text{Cov}(X,Y)}{\sqrt{\text{Var}(X)\text{Var}(Y)}}$$

注意相关系数只捕获线性关系——$X$ 和 $X^2$ 的相关系数可能为零，尽管它们有确定的函数关系。

Law of the Unconscious Statistician：对于函数变换 $Y = g(X)$，其期望无需先求 $Y$ 的分布，直接使用：
$$\mathbb{E}[g(X)] = \int g(x) p(x) dx$$

条件期望 $\mathbb{E}[Y|X=x]$ 给出在已知 $X$ 的情况下 $Y$ 的最佳预测（在均方误差意义下）。迭代期望定律（Adam 定律）$\mathbb{E}[Y] = \mathbb{E}[\mathbb{E}[Y|X]]$ 将嵌套期望展开，用于 EM 算法和强化学习中的期望分解。

随机变量的高阶矩——偏度（Skewness）衡量分布的不对称性，峰度（Kurtosis）衡量分布的尾部厚度——在分析金融数据和非高斯噪声时有一定应用。

随机变量的变换规则给出了从 $p_X(x)$ 推导 $p_Y(y)$ 的方法。若 $Y=g(X)$ 且 $g$ 单调可逆，则：
$$p_Y(y) = p_X(g^{-1}(y)) \left| \frac{d}{dy}g^{-1}(y) \right|$$

对于多元变换，雅可比行列式取代了导数的绝对值。

熵与信息量

熵（Entropy）度量随机变量的不确定性或"平均信息量"。离散随机变量 $X$ 的香农熵为：
$$H(X) = -\sum_x p(x) \log p(x)$$

熵值越高，不确定性越大。极端情况：若某个 $x$ 的 $p(x)=1$（完全确定），$H(X)=0$；若 $n$ 个取值等概率，$H(X)=\log n$（最大熵原理）。

联合熵 $H(X,Y) = -\sum_{x,y} p(x,y) \log p(x,y)$ 量化两个变量的联合不确定性。

条件熵 $H(Y|X) = -\sum_{x,y} p(x,y) \log p(y|x)$ 量化已知 $X$ 后 $Y$ 的剩余不确定性。

KL 散度（相对熵）衡量两个概率分布之间的差异：
$$D_{KL}(P \| Q) = \sum_x P(x) \log \frac{P(x)}{Q(x)}$$

KL 散度是非负的（吉布斯不等式）且不对称：$D_{KL}(P \| Q) \neq D_{KL}(Q \| P)$。在机器学习中，KL 散度被用于变分推断（优化近似后验）、策略梯度方法（限制策略更新幅度）等。

交叉熵（Cross-Entropy）与 KL 散度关系紧密：
$$H(P, Q) = H(P) + D_{KL}(P \| Q) = -\sum_x P(x) \log Q(x)$$

当 $P$ 为真实分布、$Q$ 为预测分布时，最小化交叉熵等价于最小化 KL 散度（因为 $H(P)$ 是常数）。这正是分类任务中交叉熵损失函数的理论基础。

互信息（Mutual Information）量化了两个变量共享的信息量：
$$I(X; Y) = D_{KL}(P(X,Y) \| P(X)P(Y)) = H(X) - H(X|Y)$$

互信息对称且非负，在特征选择中——互信息高的特征与目标变量的关联紧密，值得纳入模型。

概率论在机器学习中的典型应用

朴素贝叶斯分类器：基于条件独立性假设 $P(\mathbf{x}|y) = \prod_i P(x_i|y)$，用贝叶斯定理计算后验类概率，虽假设粗糙但实践效果可靠

逻辑回归：将线性组合 $z = \mathbf{w}^\top\mathbf{x}$ 通过 Sigmoid 映射到 $[0,1]$，输出 $P(y=1|\mathbf{x}) = \frac{1}{1+e^{-z}}$，用最大似然估计训练

贝叶斯神经网络：不是学习确定的权重值，而是在权重上放置概率分布，进行不确定性量化和模型平均

变分自编码器（VAE）：用变分推断学习潜在变量模型，优化 ELBO = 重构损失 + 对后验的正则化（KL 散度）

生成对抗网络（GAN）：生成器和判别器的对抗博弈本质上是极小化 $P_{\text{data}}$ 和 $P_G$ 之间的某种散度（原始 GAN 对应 JS 散度，WGAN 对应 Wasserstein 距离）

高斯过程：将函数视为无限维多元高斯的采样，提供带有置信区间的非参数函数逼近，用于贝叶斯优化中的代理模型
