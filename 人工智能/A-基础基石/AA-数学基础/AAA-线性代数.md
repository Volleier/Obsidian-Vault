
线性代数是人工智能和机器学习的计算骨架，为数据表示、特征变换和模型计算提供了统一的数学语言。在机器学习中，一张数据表就是矩阵——每行是一个样本，每列是一个特征；模型的权重被组织为矩阵，神经网络中每一层的前向计算本质上是矩阵乘法后接非线性激活。理解线性代数意味着理解"数据在高维空间中如何流动、如何被变换"。

# 向量与向量空间

在人工智能中，向量是最基本的"数据容器"。一个用户的画像、一张图片的像素、一段文本的词嵌入、一个音频信号的频谱——这些都可以表示为向量。高维向量空间为这些数据提供了统一的几何语言：相似的样本在空间中距离近，不相似的样本距离远。

向量的基本概念：$n$ 维实向量 $\mathbf{v} \in \mathbb{R}^n$ 是一组 $n$ 个有序数值，通常写为列向量 $\mathbf{v} = (v_1, v_2, \dots, v_n)^\top$。向量可以相加（$\mathbf{u} + \mathbf{v}$）和乘以标量（$c\mathbf{v}$），这些操作满足结合律、交换律和分配律。

线性无关是理解"数据冗余"的关键概念：若一组向量 $\{\mathbf{v}_1, \dots, \mathbf{v}_k\}$ 中没有任何向量能被其余向量线性组合表示——即方程 $a_1\mathbf{v}_1 + \dots + a_k\mathbf{v}_k = \mathbf{0}$ 只有零解——则它们是线性无关的。线性相关意味着信息冗余——某些特征完全可以由其他特征线性表示，这在高维数据分析中非常普遍。

基与维数：一组线性无关且张成整个空间的向量构成该空间的基。维数等于基中向量的个数，代表空间的"自由度"。例如 $\mathbb{R}^3$ 的标准基为 $\{(1,0,0)^\top, (0,1,0)^\top, (0,0,1)^\top\}$。在机器学习中，PCA 降维的核心思想就是找到数据方差最大的 $k$ 个方向作为新的基，将数据从 $d$ 维投影到 $k$ 维子空间。

# 矩阵及其基本运算

矩阵将向量空间中的线性关系组织为紧凑的表格形式。在机器学习中，设计矩阵 $\mathbf{X} \in \mathbb{R}^{n \times d}$ 的每一行是一个训练样本，每一列是一个特征；权重矩阵 $\mathbf{W}$ 则编码了从输入到输出的线性映射。

矩阵的基本运算可类比于数的运算，但有重要区别。矩阵加法是同型矩阵的逐元素相加：$(\mathbf{A} + \mathbf{B})_{ij} = a_{ij} + b_{ij}$。标量乘法则将每个元素同时缩放：$(c\mathbf{A})_{ij} = ca_{ij}$。

矩阵乘法是最关键的操作。$\mathbf{C} = \mathbf{A}\mathbf{B}$（$\mathbf{A} \in \mathbb{R}^{m \times n}$，$\mathbf{B} \in \mathbb{R}^{n \times p}$）得到的 $\mathbf{C} \in \mathbb{R}^{m \times p}$ 中，每个元素由内积给出：
$$c_{ij} = \sum_{k=1}^{n} a_{ik} b_{kj}$$

矩阵乘法满足结合律和分配律，但通常不满足交换律$\mathbf{A}\mathbf{B} \neq \mathbf{B}\mathbf{A}$——在神经网络中这意味着"先做线性变换 A 再做变换 B"与"先做 B 再做 A"效果不同。

转置将行和列互换：$(\mathbf{A}^\top)_{ij} = a_{ji}$，可用于将行向量转为列向量或反向。重要性质：$(\mathbf{A}\mathbf{B})^\top = \mathbf{B}^\top\mathbf{A}^\top$。

逆矩阵 $\mathbf{A}^{-1}$ 满足 $\mathbf{A}\mathbf{A}^{-1} = \mathbf{A}^{-1}\mathbf{A} = \mathbf{I}$，仅满秩方阵可逆。在求解线性回归的闭式解 $\mathbf{w} = (\mathbf{X}^\top\mathbf{X})^{-1}\mathbf{X}^\top\mathbf{y}$ 时会用到逆矩阵，但实际工程中通常通过解线性方程组而非显式求逆来实现，因为直接求逆数值不稳定且代价高。

迹（Trace）和行列式（Determinant）是刻画方阵性质的标量。迹为对角线元素之和 $\text{tr}(\mathbf{A}) = \sum_i a_{ii}$，具有循环置换不变性 $\text{tr}(\mathbf{A}\mathbf{B}) = \text{tr}(\mathbf{B}\mathbf{A})$。行列式衡量列向量围成平行多面体的有向体积——$\det(\mathbf{A}) = 0$ 意味着 $\mathbf{A}$ 将空间"压扁"了，信息不可逆地丢失。对 $2 \times 2$ 矩阵：
$$\det\begin{pmatrix} a & b \\ c & d \end{pmatrix} = ad - bc$$

# 线性变换

线性变换为理解神经网络的计算过程提供了几何直觉：每一层网络就是先做线性变换（矩阵乘法），再做非线性激活。线性变换保持向量加法和标量乘法结构，即 $T(\mathbf{u} + \mathbf{v}) = T(\mathbf{u}) + T(\mathbf{v})$ 且 $T(c\mathbf{v}) = cT(\mathbf{v})$。

任何线性变换都可用矩阵表示：$T(\mathbf{v}) = \mathbf{A}\mathbf{v}$。不同类型的矩阵对应不同类型的几何操作：

旋转矩阵保持向量长度不变但改变其方向

缩放矩阵（对角阵）沿坐标轴拉伸或压缩向量

投影矩阵将向量映射到某个子空间上

反射矩阵将向量关于某个超平面做镜像

矩阵的四个基本子空间揭示了线性变换的内部结构：列空间是变换的"可达范围"（值域），零空间是被变换"消灭"为零向量的所有输入的集合。秩-零化度定理 $\text{rank}(\mathbf{A}) + \dim(\text{Null}(\mathbf{A})) = n$ 将这两个概念联系起来——信息量（秩）与信息损失（零空间维数）之和恒等于输入维数。

# 线性方程组与高斯消元

线性方程组 $\mathbf{A}\mathbf{x} = \mathbf{b}$ 的求解是许多 ML 算法的基础步骤。线性回归的闭式解需要解法方程 $\mathbf{X}^\top\mathbf{X}\mathbf{w} = \mathbf{X}^\top\mathbf{y}$；SVM 的对偶问题求解涉及线性方程组；贝叶斯线性回归的后验均值也是线性方程组的解。

当方程个数等于未知数个数且 $\mathbf{A}$ 可逆时，唯一解为 $\mathbf{x} = \mathbf{A}^{-1}\mathbf{b}$。但更常见的情况是超定系统（方程多于未知数），如训练样本数超过特征数时，此时需要最小二乘解：
$$\mathbf{x}^* = (\mathbf{A}^\top\mathbf{A})^{-1}\mathbf{A}^\top\mathbf{b}$$

高斯消元法通过三种初等行变换（交换行、行乘以常数、行加减）将增广矩阵化为阶梯形，然后回代求解。虽然算法复杂度为 $O(n^3)$，但对于中小规模问题是最常用的直接求解方法。

# 范数与内积空间

范数和内积定义了"距离""长度""角度"这些几何概念在高维空间中的推广。在机器学习中，距离度量直接决定了一个模型如何理解"相似"——k-近邻的邻居选择、聚类算法的簇分配、损失函数中的误差度量，无一不依赖于范数。

L1 范数（曼哈顿距离）：$\|\mathbf{x}\|_1 = \sum_i |x_i|$。它的特殊之处在于诱导稀疏性——作为正则化项时，优化的最优解会有部分参数恰好为零，实现自动特征选择。这在 Lasso 回归和高维稀疏建模中至关重要。

L2 范数（欧氏距离）：$\|\mathbf{x}\|_2 = \sqrt{\sum_i x_i^2}$。这是最常用的距离度量，对应我们熟悉的"直线距离"。作为正则化项时，L2 惩罚将参数均匀地收缩向零但不置零，是岭回归（Ridge）的基础。

L∞ 范数：$\|\mathbf{x}\|_\infty = \max_i |x_i|$，只关注最大分量的绝对值。

内积 $\langle \mathbf{u}, \mathbf{v} \rangle = \mathbf{u}^\top\mathbf{v} = \sum_i u_i v_i = \|\mathbf{u}\|\|\mathbf{v}\|\cos\theta$ 同时包含了长度和角度信息。当内积为零时两向量正交；内积为负时夹角大于 90 度。余弦相似度 $\frac{\mathbf{u}^\top\mathbf{v}}{\|\mathbf{u}\|\|\mathbf{v}\|}$ 正是内积的归一化形式，在文本相似度和推荐系统中广泛使用。

# 特征值与特征向量

特征值分解在 AI 中的价值在于捕捉系统"内在的、不变的模式"。FaceBook 的 Face Recognition 用特征脸（协方差矩阵的特征向量）来表示人脸，Google 的 PageRank 用马尔可夫矩阵的主特征向量来衡量网页重要性——这些都是特征分解的具体应用。

对于方阵 $\mathbf{A}$，若非零向量 $\mathbf{v}$ 满足 $\mathbf{A}\mathbf{v} = \lambda\mathbf{v}$，则 $\mathbf{v}$ 是特征向量，$\lambda$ 是对应的特征值。几何意义上，特征向量是经矩阵变换后方向保持不变的向量方向——变换只是将其沿自身方向拉伸或压缩 $\lambda$ 倍。

特征值的代数意义：所有特征值之和等于矩阵的迹 $\sum \lambda_i = \text{tr}(\mathbf{A})$，所有特征值之积等于行列式 $\prod \lambda_i = \det(\mathbf{A})$。若所有特征值为正，矩阵是正定的；所有非负则为半正定——协方差矩阵天然是半正定的。

对称矩阵的特征值均为实数，且存在一组正交的特征向量基。这一性质使对称矩阵（如协方差矩阵）可以被完美对角化：$\mathbf{A} = \mathbf{Q}\mathbf{\Lambda}\mathbf{Q}^\top$，其中 $\mathbf{Q}$ 的列为标准正交特征向量，$\mathbf{\Lambda}$ 为特征值对角阵。

# 奇异值分解（SVD）

SVD 被称为"线性代数的瑞士军刀"——因为它适用于任意形状的矩阵（不必是方阵）且数值稳定。在 AI 中，SVD 是 PCA 降维、隐语义分析、矩阵补全等技术的数学引擎。

对任意矩阵 $\mathbf{A} \in \mathbb{R}^{m \times n}$，SVD 分解为三个矩阵的乘积：
$$\mathbf{A} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^\top$$

其中 $\mathbf{U} \in \mathbb{R}^{m \times m}$ 和 $\mathbf{V} \in \mathbb{R}^{n \times n}$ 是正交矩阵，$\mathbf{\Sigma} \in \mathbb{R}^{m \times n}$ 是对角矩阵，其对角元素 $\sigma_1 \geq \sigma_2 \geq \dots \geq \sigma_{\min(m,n)} \geq 0$ 称为奇异值。奇异值越大，对应方向上的"信息强度"越大。

几何直觉：任何线性变换等价于依次做三次操作：先旋转 $\mathbf{V}^\top$，再沿坐标轴缩放 $\mathbf{\Sigma}$，最后再旋转 $\mathbf{U}$。缩放因子就是奇异值。

Eckart-Young 定理：保留前 $k$ 个最大的奇异值（其余置零），得到的是在 Frobenius 范数下与原矩阵最接近的秩-$k$ 矩阵。这构成了 PCA 的理论保证——前 $k$ 个主成分捕获了数据中最大的方差方向。保留方差比例为 $\frac{\sum_{i=1}^k \sigma_i^2}{\sum_i \sigma_i^2}$。

SVD 的典型应用场景：

PCA 降维：数据中心化后做 SVD，$\mathbf{V}$ 的列即主成分方向

推荐系统：将用户-物品评分矩阵做 SVD 分解为低秩表示，捕捉用户偏好和物品属性之间的潜在关联

伪逆计算：$\mathbf{A}^+ = \mathbf{V}\mathbf{\Sigma}^+\mathbf{U}^\top$（非零奇异值取倒数），给出最小范数最小二乘解

# 矩阵分解（Cholesky, QR, LU）

除了 SVD，多种矩阵分解技术在不同的 ML 场景中各司其职。它们将复杂矩阵拆解为简单矩阵的乘积，使得原本困难的计算（求逆、解方程）变得可行。

Cholesky 分解将正定矩阵分解为 $\mathbf{A} = \mathbf{L}\mathbf{L}^\top$（$\mathbf{L}$ 为下三角阵）。其重要性在于计算量仅为 LU 分解的一半，且数值稳定。在高斯过程中，协方差矩阵的求逆——计算量往往是瓶颈——通过 Cholesky 分解后只需解两个三角系统。此外，从多元高斯分布 $\mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ 采样时，先做 Cholesky 分解 $\boldsymbol{\Sigma} = \mathbf{L}\mathbf{L}^\top$，再生成标准正态噪声 $\mathbf{z} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$，则 $\mathbf{x} = \boldsymbol{\mu} + \mathbf{L}\mathbf{z}$ 即为服从目标分布的样本。

QR 分解将矩阵分解为 $\mathbf{A} = \mathbf{Q}\mathbf{R}$（$\mathbf{Q}$ 正交，$\mathbf{R}$ 上三角）。在求解最小二乘问题时，由于 $\mathbf{Q}^\top\mathbf{Q} = \mathbf{I}$，法方程简化为只需回代的三角系统 $\mathbf{R}\mathbf{x} = \mathbf{Q}^\top\mathbf{b}$，避免了直接求逆带来的数值病态。

LU 分解：$\mathbf{A} = \mathbf{L}\mathbf{U}$（$\mathbf{L}$ 单位下三角，$\mathbf{U}$ 上三角）。求解 $\mathbf{A}\mathbf{x} = \mathbf{b}$ 时先解 $\mathbf{L}\mathbf{y} = \mathbf{b}$（前向代入），再解 $\mathbf{U}\mathbf{x} = \mathbf{y}$（回代），总复杂度 $O(n^2)$ 远低于直接求逆的 $O(n^3)$。

# 张量运算

张量是向量和矩阵的高维推广，是现代深度学习框架（PyTorch、TensorFlow）的数据基本单位。在深度学习中，一个批量的 RGB 图像存储为四阶张量——形状为 (批量大小, 通道数, 高度, 宽度)。理解张量运算是理解 conv2d、batch_norm、multi-head attention 等操作的前提。

张量的阶数：标量是零阶，向量是一阶，矩阵是二阶，图像批量是四阶。深度学习框架中的"维度"（dim）就是张量的阶数。

关键张量操作：

广播机制允许不同形状的张量自然地执行逐元素运算。例如，将一个形状为 (64, 1) 的偏置加到形状为 (64, 256) 的矩阵上，偏置自动沿第二维复制扩展。Batch Normalization 中的 $\frac{x - \mu}{\sigma}$ 正是依赖广播。

收缩是沿指定轴求和的操作。矩阵乘法 $\mathbf{C}_{ij} = \sum_k \mathbf{A}_{ik}\mathbf{B}_{kj}$ 本质上是二阶张量在轴 $k$ 上的收缩。现代框架用 einsum 记号（如 `ik,kj->ij`）简洁地表达任意复杂度的张量收缩。

Hadamard 积（逐元素积）：$(\mathbf{A} \odot \mathbf{B})_{ij} = a_{ij} b_{ij}$。LSTM 的门控机制——遗忘门、输入门、输出门——都是通过 Hadamard 积来选择性控制信息流的。

# 应用：数据降维与嵌入表示

PCA 是线性代数在 ML 中最经典的降维应用。当数据维度成百上千时，PCA 能将数据投影到一个低维子空间，同时保留尽可能多的方差（信息）。

PCA 的步骤：第一，将数据矩阵 $\mathbf{X} \in \mathbb{R}^{n \times d}$ 中心化（每列减均值），消除位置偏移的影响；第二，计算协方差矩阵 $\mathbf{S} = \frac{1}{n}\mathbf{X}^\top\mathbf{X}$，它捕获了特征之间两两的线性关联程度；第三，对 $\mathbf{S}$ 做特征分解（或直接对 $\mathbf{X}$ 做 SVD），取前 $k$ 大特征值对应的特征向量构成投影矩阵 $\mathbf{W} \in \mathbb{R}^{d \times k}$；第四，变换后的低维表示为 $\mathbf{Z} = \mathbf{X}\mathbf{W} \in \mathbb{R}^{n \times k}$。解释方差比例 $\frac{\sum_{i=1}^k \lambda_i}{\sum_{i=1}^d \lambda_i}$ 用作"解释了多少信息"的报告。

词嵌入是线性代数赋能自然语言处理的典范。Word2Vec、GloVe 等模型将词汇表中的每个词映射为一个几十到几百维的稠密实数向量。这些向量经过大规模语料训练后，呈现出令人惊讶的语义结构：语义相近的词在向量空间中彼此靠近，语义关系表现为一致的向量方向。"国王" - "男人" + "女人" ≈ "女王" 这种向量算术表明嵌入空间编码了独立的语义维度。这背后全部是线性代数——相似度是向量内积，类比是向量加减。
