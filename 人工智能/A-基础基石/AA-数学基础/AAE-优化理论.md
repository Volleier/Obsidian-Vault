
优化理论是机器学习的引擎——它将"从数据中学习"这一抽象目标转化为具体的数学问题：寻找使损失函数最小化（或目标函数最大化）的模型参数。从线性回归的简单凸优化到深度神经网络的高维非凸优化，优化算法的选择直接影响训练效率、收敛质量和最终模型性能。

优化问题的基本形式

无约束优化问题的一般形式为：
$$\min_{\mathbf{w} \in \mathbb{R}^d} f(\mathbf{w})$$

其中 $f: \mathbb{R}^d \to \mathbb{R}$ 为目标函数（在机器学习中通常为损失函数或负对数似然），$\mathbf{w}$ 为待优化参数。优化的目标是找到 $\mathbf{w}^*$ 使得 $f(\mathbf{w}^*)$ 达到全局最小值。

在机器学习中，$f(\mathbf{w})$ 通常写为训练集上各样本损失的平均值：
$$f(\mathbf{w}) = \frac{1}{n} \sum_{i=1}^n \ell(\mathbf{w}; \mathbf{x}_i, y_i) + \lambda R(\mathbf{w})$$

其中 $\ell$ 为每个样本的损失，$R(\mathbf{w})$ 为正则化项。这种"经验风险 + 正则化"的结构驱动了绝大多数监督学习算法。

梯度下降法

梯度下降是最基础的迭代优化算法。每一步沿负梯度方向更新参数：
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \eta \nabla f(\mathbf{w}_t)$$

其中 $\eta > 0$ 为学习率（步长）。当 $f$ 为凸函数且 $\nabla f$ Lipschitz 连续时，取 $\eta = 1/L$（$L$ 为 Lipschitz 常数）可保证收敛。

学习率的选择至关重要——过大会导致振荡甚至发散，过小则收敛缓慢。实践中常用学习率衰减策略：

阶梯衰减：每经过固定轮次将学习率乘以衰减因子

指数衰减：$\eta_t = \eta_0 \cdot \gamma^t$

余弦退火：$\eta_t = \eta_{\min} + \frac{1}{2}(\eta_{\max} - \eta_{\min})(1 + \cos(\frac{t\pi}{T}))$，让学习率按照余弦曲线平滑下降

预热（Warmup）：训练初期从极小学习率线性增加至目标值，避免初期不稳定

梯度下降的三种主要变体根据每次更新使用的数据量区分：

批量梯度下降（BGD）：每步扫描全部 $n$ 个样本计算精确梯度 $\nabla f = \frac{1}{n}\sum_{i=1}^n \nabla \ell_i$。每步代价为 $O(nd)$，大数据集上不可行，但能保证稳定下降

随机梯度下降（SGD）：每步随机抽取一个样本 $i$，用 $\nabla \ell_i$ 作为梯度的无偏估计。更新极快但噪声大、收敛轨迹剧烈震荡

小批量梯度下降（Mini-batch SGD）：每步使用 $m$ 个样本（如 $m=32$ 或 $m=256$），在梯度精度和计算效率之间取得平衡，是深度学习中实际使用的标准方式

动量法

动量法引入"惯性"来加速收敛并抑制振荡：
$$\mathbf{v}_{t+1} = \beta \mathbf{v}_t + \eta \nabla f(\mathbf{w}_t)$$
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \mathbf{v}_{t+1}$$

动量系数 $\beta \in [0, 1)$（常用 $0.9$）控制历史梯度方向对当前更新的持续影响。在梯度方向一致时，动量累积加速了前进；在梯度方向反复反转时，动量互相抵消，减轻了锯齿状震荡。物理类比：球沿山谷滚落——平行的走向使球加速，垂直上下的震荡被惯性平滑。

Nesterov 加速梯度（NAG）是动量法的改进，先沿动量方向位移一步，在该前瞻位置评估梯度——"先看一眼再决定怎么走"：
$$\mathbf{v}_{t+1} = \beta \mathbf{v}_t + \eta \nabla f(\mathbf{w}_t - \beta \mathbf{v}_t)$$
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \mathbf{v}_{t+1}$$

在凸优化中，NAG 的理论收敛率为 $O(1/t^2)$，优于标准动量的 $O(1/t)$。

自适应学习率方法

自适应方法为每个参数独立调整学习率，特别适合参数尺度差异大的问题（如嵌入层与全连接层的学习率需求不同）。

AdaGrad 累积历史梯度的平方和，用于缩放当前学习率：
$$\mathbf{g}_t = \nabla f(\mathbf{w}_t), \quad \mathbf{G}_t = \sum_{\tau=1}^t \mathbf{g}_\tau \odot \mathbf{g}_\tau$$
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \frac{\eta}{\sqrt{\mathbf{G}_t + \epsilon}} \odot \mathbf{g}_t$$

对于频繁更新的参数，累积量 $\mathbf{G}_t$ 大，学习率被压得很小；对于稀疏更新的参数，学习率保持较大。但 AdaGrad 的累积量会单向增长，导致训练后期学习率趋向零。

RMSProp 修正了 AdaGrad 的单调衰减缺陷，改用指数移动平均：
$$\mathbf{G}_t = \beta \mathbf{G}_{t-1} + (1 - \beta)(\mathbf{g}_t \odot \mathbf{g}_t)$$
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \frac{\eta}{\sqrt{\mathbf{G}_t + \epsilon}} \odot \mathbf{g}_t$$

指数衰减使较旧的梯度信息逐渐"遗忘"，保持了学习率的适应性而不走向零。

Adam（Adaptive Moment Estimation）是动量法和 RMSProp 的结合，已成为深度学习中最常用的优化器。Adam 同时维护梯度的一阶矩（均值）和二阶矩（未中心化的方差）的指数移动平均：
$$\mathbf{m}_t = \beta_1 \mathbf{m}_{t-1} + (1 - \beta_1)\mathbf{g}_t$$
$$\mathbf{v}_t = \beta_2 \mathbf{v}_{t-1} + (1 - \beta_2)(\mathbf{g}_t \odot \mathbf{g}_t)$$

初始化 $\mathbf{m}_0 = \mathbf{0}, \mathbf{v}_0 = \mathbf{0}$ 会导致早期步骤中的估计向零偏置（因为 $\beta_1, \beta_2$ 接近 1）。偏差校正修正了这个问题：
$$\hat{\mathbf{m}}_t = \frac{\mathbf{m}_t}{1 - \beta_1^t}, \quad \hat{\mathbf{v}}_t = \frac{\mathbf{v}_t}{1 - \beta_2^t}$$
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \frac{\eta}{\sqrt{\hat{\mathbf{v}}_t} + \epsilon} \hat{\mathbf{m}}_t$$

默认超参数：$\eta = 0.001, \beta_1 = 0.9, \beta_2 = 0.999, \epsilon = 10^{-8}$。

AdamW 将权重衰减从 Adam 的梯度步骤中分离出来单独施加，解决了 Adam 中 L2 正则化等同于自适应学习率缩放下的权重衰减这一微妙问题。这是当前大规模训练的推荐优化器。

凸优化

凸集：集合 $C$ 中任意两点连线上的所有点仍在 $C$ 内。即对任意 $\mathbf{x}, \mathbf{y} \in C$ 及 $\lambda \in [0,1]$，有 $\lambda\mathbf{x} + (1-\lambda)\mathbf{y} \in C$。

凸函数满足：$f(\lambda\mathbf{x} + (1-\lambda)\mathbf{y}) \leq \lambda f(\mathbf{x}) + (1-\lambda)f(\mathbf{y})$——函数图像始终在弦之下。凸函数的局部最小值即为全局最小值，这使凸优化问题有良好的理论性质。

凸函数的判别条件：

一阶条件：$f(\mathbf{y}) \geq f(\mathbf{x}) + \nabla f(\mathbf{x})^\top(\mathbf{y} - \mathbf{x})$

二阶条件：海森矩阵 $\nabla^2 f(\mathbf{x}) \succeq \mathbf{0}$（半正定），对所有 $\mathbf{x}$ 成立

常见的机器学习凸损失包括：逻辑回归的对数损失（二次可微且强凸）、线性 SVM 的 Hinge 损失（凸但非光滑）、Lasso（L1 正则化）的绝对值惩罚（凸但非光滑）。

拉格朗日乘子法将约束优化转化为无约束问题。对于等式约束 $\min f(\mathbf{x})$ s.t. $h_i(\mathbf{x}) = 0$，构造拉格朗日函数：
$$\mathcal{L}(\mathbf{x}, \boldsymbol{\lambda}) = f(\mathbf{x}) + \sum_i \lambda_i h_i(\mathbf{x})$$

对 $\mathbf{x}$ 和 $\boldsymbol{\lambda}$ 分别求导并置零，解出候选极值点。

KKT 条件（Karush-Kuhn-Tucker）推广到不等式约束 $g_j(\mathbf{x}) \leq 0$。对于 $\min f(\mathbf{x})$ s.t. $g_j(\mathbf{x}) \leq 0, h_i(\mathbf{x}) = 0$，在最优解 $\mathbf{x}^*$ 处存在乘子 $\mu_j^* \geq 0$ 和 $\lambda_i^*$ 满足：

平稳性：$\nabla f(\mathbf{x}^*) + \sum_j \mu_j^* \nabla g_j(\mathbf{x}^*) + \sum_i \lambda_i^* \nabla h_i(\mathbf{x}^*) = \mathbf{0}$

可行性：$g_j(\mathbf{x}^*) \leq 0, h_i(\mathbf{x}^*) = 0$

互补松弛性：$\mu_j^* g_j(\mathbf{x}^*) = 0$——若约束是非紧的（$g_j < 0$），则其乘子必为零；若乘子非零，其对应的约束必为零

拉格朗日对偶性给出了原问题与对偶问题的关系。原问题 $\inf_{\mathbf{x}} \sup_{\boldsymbol{\lambda} \geq \mathbf{0}} \mathcal{L}$ 的对偶问题为 $\sup_{\boldsymbol{\lambda} \geq \mathbf{0}} \inf_{\mathbf{x}} \mathcal{L}$。弱对偶性保证对偶最优值不超过原最优值；强对偶性（在 Slater 条件下成立）使得在满足约束规约时两者相等。对偶间隙为零是 SVM 的对偶推导能成立的理论保证。

二阶优化方法

牛顿法利用二阶泰勒展开逼近目标函数：
$$f(\mathbf{w} + \Delta\mathbf{w}) \approx f(\mathbf{w}) + \nabla f^\top \Delta\mathbf{w} + \frac{1}{2}\Delta\mathbf{w}^\top \mathbf{H} \Delta\mathbf{w}$$

对 $\Delta\mathbf{w}$ 求导并置零得到更新：
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \mathbf{H}_t^{-1} \nabla f(\mathbf{w}_t)$$

在凸函数上，牛顿法达到二次收敛率——每步有效数字翻倍。但在深度学习中，参数数量 $d$ 往往达数百万甚至数亿级别，存储和求逆 $d \times d$ 的海森矩阵在实际中完全不可行——即便存储就需要 $\frac{d(d+1)}{2}$ 个浮点数。

拟牛顿法（Quasi-Newton）通过梯度变化来近似海森矩阵的逆，只存储和更新一个近似逆矩阵。

BFGS（Broyden-Fletcher-Goldfarb-Shanno）是最著名的方法，更新公式为：
$$\mathbf{H}_{t+1}^{-1} = \left(\mathbf{I} - \frac{\mathbf{s}_t\mathbf{y}_t^\top}{\mathbf{y}_t^\top\mathbf{s}_t}\right)\mathbf{H}_t^{-1}\left(\mathbf{I} - \frac{\mathbf{y}_t\mathbf{s}_t^\top}{\mathbf{y}_t^\top\mathbf{s}_t}\right) + \frac{\mathbf{s}_t\mathbf{s}_t^\top}{\mathbf{y}_t^\top\mathbf{s}_t}$$

其中 $\mathbf{s}_t = \mathbf{w}_{t+1} - \mathbf{w}_t$，$\mathbf{y}_t = \nabla f_{t+1} - \nabla f_t$。BFGS 保证正定性并达到超线性收敛率。

L-BFGS（Limited-memory BFGS）仅存储最近 $m$ 步的 $\mathbf{s}_t, \mathbf{y}_t$（通常 $m=10\sim20$），通过两层递归隐式计算矩阵-向量乘积，将内存复杂度从 $O(d^2)$ 降至 $O(md)$。L-BFGS 广泛用于大规模机器学习中的凸问题，如逻辑回归和条件随机场的训练。

EM 算法

EM 算法（Expectation-Maximization）适用于含隐变量模型的极大似然估计。许多概率模型中，直接优化对数似然 $\log p(\mathbf{X}|\theta)$ 很困难，但若隐变量 $\mathbf{Z}$ 被观测到，则 $\log p(\mathbf{X},\mathbf{Z}|\theta)$ 更为简单。

E 步：基于当前参数 $\theta^{(t)}$ 计算隐变量的后验分布 $p(\mathbf{Z}|\mathbf{X}, \theta^{(t)})$，进而构造 Q 函数——完整数据对数似然的期望：
$$Q(\theta|\theta^{(t)}) = \mathbb{E}_{\mathbf{Z}|\mathbf{X},\theta^{(t)}}[\log p(\mathbf{X},\mathbf{Z}|\theta)]$$

M 步：最大化 Q 函数得到新参数：
$$\theta^{(t+1)} = \arg\max_\theta Q(\theta|\theta^{(t)})$$

EM 算法的每次迭代单调提升观测数据的对数似然（或至少不降低），但并不保证收敛到全局最优——对于非凸问题（如高斯混合模型），初始化会影响最终解的质量。

典型应用包括：高斯混合模型（GMM）的参数估计（E 步计算每个样本属于各成分的责任概率，M 步更新均值、方差和混合系数）、隐马尔可夫模型（HMM）的 Baum-Welch 训练、含缺失数据的统计分析等。

推导 EM 算法的另一种视角是通过 Jensen 不等式构造对数似然的下界，交替优化该下界（变分下界 / ELBO）。这一观点将 EM 与变分推断统一到了一个框架中。

约束优化与正则化

正则化通过在损失函数中增加对参数大小的惩罚，抑制过拟合。从优化视角，均值损失和正则化的最小化等价于在参数空间中约束参数来解决损失函数的最小化。

L1 正则化（Lasso）：$R(\mathbf{w}) = \|\mathbf{w}\|_1 = \sum_i |w_i|$。几何上，L1 约束区域（菱形）的顶点恰好落在坐标轴上，导致解中许多参数恰好为零，实现天然的特征选择。

L2 正则化（Ridge）：$R(\mathbf{w}) = \|\mathbf{w}\|_2^2 = \sum_i w_i^2$。L2 约束区域为圆形，最优解将每个参数均匀收缩但不为零。L2 使线性回归中 $(\mathbf{X}^\top\mathbf{X} + \lambda\mathbf{I})$ 总是可逆，解决了共线性的数值不稳定问题。

弹性网络（Elastic Net）：$R(\mathbf{w}) = \alpha\|\mathbf{w}\|_1 + (1-\alpha)\|\mathbf{w}\|_2^2$，结合了 L1 的稀疏性和 L2 的稳定性，在高度相关的特征组上优于单独的 Lasso。

非凸优化与深度学习中的挑战

深度神经网络的损失函数是非凸的——海森矩阵的特征值有正有负，存在大量鞍点。在极高维空间中，鞍点远比局部极小点常见（海森矩阵的 $2^d$ 种特征值符号组合中，只有全正对应极小点）。

鞍点处的梯度为零，但并非最优解——标准梯度下降可能被鞍点"困住"。幸运的是，SGD 中的随机梯度噪声有去鞍效应：由于前后梯度方向不完全一致，噪声带来的扰动有助于从鞍点逃逸。动量法也有类似效果。

高维非凸损失曲面上，局部极小点的质量通常不差。有理论表明：在过参数化的神经网络中，绝大多数局部极小值与全局最小值在函数值上非常接近——问题不在于陷入"坏"的极小值，而在于找到"好"的极小值。

其他重要优化方法

共轭梯度法：在二次问题上能在 $d$ 步内精确收敛，每步计算仅为梯度求值和向量内积。共轭方向确保沿新方向搜索不会破坏之前已最小化的方向。

坐标下降法：每次只优化一个坐标（一个参数）而固定其余，轮替遍历。在 Lasso（问题可分解为对每个系数的单变量软阈值处理）和矩阵分解等问题中极其高效。

近端梯度法（Proximal Gradient Method）：对于形式为 $\min f(\mathbf{w}) + g(\mathbf{w})$ 的问题（$f$ 可微，$g$ 仅需近端算子存在），每步迭代为：
$$\mathbf{w}_{t+1} = \text{prox}_{\eta g}(\mathbf{w}_t - \eta \nabla f(\mathbf{w}_t))$$

近端算子 $\text{prox}_g(\mathbf{v}) = \arg\min_{\mathbf{x}} \{g(\mathbf{x}) + \frac{1}{2}\|\mathbf{x} - \mathbf{v}\|^2\}$ 对于许多正则化项（如 L1、核范数）有闭式解。近端梯度法将梯度步骤和正则化步骤交替执行，是处理不可微正则化的通用框架。

贝叶斯优化：当目标函数的评估代价昂贵时（如超参数搜索中的模型重训），使用高斯过程作为目标函数的代理模型，采集函数（期望改进 EI、置信上界 UCB 等）在探索未知区域和利用已知好点之间权衡，指导下一步在何处采样。
