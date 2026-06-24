
线性代数是人工智能和机器学习的计算骨架。它研究向量、矩阵及其运算，为数据表示、特征变换和模型计算提供数学框架。在机器学习中，数据集被表示为矩阵，模型参数被组织为权重矩阵，几乎所有涉及高维数据的操作——从简单的线性回归到复杂的深度神经网络——都依赖于线性代数的高效计算。

向量与向量空间

向量是一组有序的数值，可视为高维空间中的一个点。$n$ 维向量 $\mathbf{v} \in \mathbb{R}^n$ 记为 $\mathbf{v} = (v_1, v_2, \dots, v_n)^\top$。向量空间是由向量构成的集合，对加法和标量乘法封闭。向量空间的性质包括：

加法结合性：$\mathbf{u} + (\mathbf{v} + \mathbf{w}) = (\mathbf{u} + \mathbf{v}) + \mathbf{w}$

加法交换性：$\mathbf{u} + \mathbf{v} = \mathbf{v} + \mathbf{u}$

标量分配性：$a(\mathbf{u} + \mathbf{v}) = a\mathbf{u} + a\mathbf{v}$

零向量 $\mathbf{0}$ 满足 $\mathbf{v} + \mathbf{0} = \mathbf{v}$

每个向量 $\mathbf{v}$ 存在唯一的相反向量 $-\mathbf{v}$，满足 $\mathbf{v} + (-\mathbf{v}) = \mathbf{0}$

线性无关（Linear Independence）：一组向量 $\{\mathbf{v}_1, \dots, \mathbf{v}_k\}$ 称为线性无关，当且仅当 $a_1\mathbf{v}_1 + \dots + a_k\mathbf{v}_k = \mathbf{0}$ 只有零解 $a_1 = \dots = a_k = 0$。线性相关意味着某些向量可以被其他向量线性组合表示，存在冗余信息。

基（Basis）：向量空间 $V$ 的一组基是线性无关且张成 $V$ 的向量集合。标准基是最常见的基选择，如 $\mathbb{R}^3$ 的标准基为 $\{(1,0,0), (0,1,0), (0,0,1)\}$。

维数（Dimension）：向量空间的维数是其任意一组基中向量的个数，刻画了该空间的"自由度"。例如 $\mathbb{R}^n$ 的维数为 $n$。

矩阵及其基本运算

矩阵是由 $m \times n$ 个数排列成的矩形阵列，记作 $\mathbf{A} \in \mathbb{R}^{m \times n}$。矩阵的每一项 $a_{ij}$ 表示第 $i$ 行第 $j$ 列的元素。

矩阵加法：两个同型矩阵对应的元素逐元素相加。$(\mathbf{A} + \mathbf{B})_{ij} = a_{ij} + b_{ij}$

标量乘法：将矩阵的每个元素乘以标量。$(c\mathbf{A})_{ij} = c \cdot a_{ij}$

矩阵乘法：$\mathbf{C} = \mathbf{A}\mathbf{B}$，其中 $\mathbf{A} \in \mathbb{R}^{m \times n}, \mathbf{B} \in \mathbb{R}^{n \times p}$，结果 $\mathbf{C} \in \mathbb{R}^{m \times p}$ 的每个元素为：
$$c_{ij} = \sum_{k=1}^{n} a_{ik} b_{kj}$$

矩阵乘法满足结合律 $(\mathbf{A}\mathbf{B})\mathbf{C} = \mathbf{A}(\mathbf{B}\mathbf{C})$，也满足分配律 $\mathbf{A}(\mathbf{B} + \mathbf{C}) = \mathbf{A}\mathbf{B} + \mathbf{A}\mathbf{C}$。但矩阵乘法一般不满足交换律——$\mathbf{A}\mathbf{B} \neq \mathbf{B}\mathbf{A}$ 是常态而非特例。

转置（Transpose）：将矩阵的行列互换。$(\mathbf{A}^\top)_{ij} = a_{ji}$。转置具有性质 $(\mathbf{A}\mathbf{B})^\top = \mathbf{B}^\top\mathbf{A}^\top$。

逆矩阵（Inverse）：方阵 $\mathbf{A} \in \mathbb{R}^{n \times n}$ 的逆 $\mathbf{A}^{-1}$ 满足 $\mathbf{A}\mathbf{A}^{-1} = \mathbf{A}^{-1}\mathbf{A} = \mathbf{I}$。只有满秩方阵才是可逆的（非奇异矩阵）。实际应用中，直接求逆矩阵的计算代价较高（$O(n^3)$），常见做法是解线性方程组 $\mathbf{A}\mathbf{x} = \mathbf{b}$ 来避免显式求逆。

迹（Trace）：方阵对角线元素之和，$\text{tr}(\mathbf{A}) = \sum_i a_{ii}$。迹具有循环置换不变性：$\text{tr}(\mathbf{A}\mathbf{B}\mathbf{C}) = \text{tr}(\mathbf{C}\mathbf{A}\mathbf{B})$。

行列式（Determinant）：方阵 $\mathbf{A}$ 的行列式 $\det(\mathbf{A})$ 衡量由矩阵列向量围成的平行多面体的有向体积。$\det(\mathbf{A}) = 0$ 等价于 $\mathbf{A}$ 奇异（不可逆）。对于 $2 \times 2$ 矩阵：
$$\det\begin{pmatrix} a & b \\ c & d \end{pmatrix} = ad - bc$$

线性方程组与高斯消元

线性方程组 $\mathbf{A}\mathbf{x} = \mathbf{b}$ 是机器学习中求解线性回归闭式解、最小二乘问题等的基础。高斯消元法通过行变换（交换行、行乘以非零常数、行加上另一行的倍数）将增广矩阵化为行阶梯形，从而回代求解。算法复杂度为 $O(n^3)$。

当方程个数多于未知数时（超定系统），通常无精确解，需求使 $\|\mathbf{A}\mathbf{x} - \mathbf{b}\|$ 最小的最小二乘解：
$$\mathbf{x}^* = (\mathbf{A}^\top\mathbf{A})^{-1}\mathbf{A}^\top\mathbf{b}$$

范数与距离

范数（Norm）将向量映射到非负标量，衡量"长度"或"大小"。常见范数包括：

L1 范数（曼哈顿范数）：$\|\mathbf{x}\|_1 = \sum_i |x_i|$，诱导稀疏性，广泛用于特征选择和 Lasso 正则化

L2 范数（欧氏范数）：$\|\mathbf{x}\|_2 = \sqrt{\sum_i x_i^2}$，最为常用，用于岭回归（Ridge）正则化

L∞ 范数（最大范数）：$\|\mathbf{x}\|_\infty = \max_i |x_i|$

Lp 范数通式：$\|\mathbf{x}\|_p = \left(\sum_i |x_i|^p\right)^{1/p}, \quad p \geq 1$

Frobenius 范数（矩阵范数）：$\|\mathbf{A}\|_F = \sqrt{\sum_{i,j} a_{ij}^2}$，相当于将矩阵拉成向量后的 L2 范数

内积给出了两个向量之间的夹角信息。标准欧几里得内积（点积）为：
$$\langle \mathbf{u}, \mathbf{v} \rangle = \mathbf{u}^\top\mathbf{v} = \sum_i u_i v_i = \|\mathbf{u}\|\|\mathbf{v}\|\cos\theta$$

当 $\mathbf{u}^\top\mathbf{v} = 0$ 时，两向量正交。内积还自然导出正交投影：向量 $\mathbf{v}$ 在方向 $\mathbf{u}$ 上的投影为 $\frac{\mathbf{u}^\top\mathbf{v}}{\mathbf{u}^\top\mathbf{u}}\mathbf{u}$。

特征值与特征向量

对于方阵 $\mathbf{A} \in \mathbb{R}^{n \times n}$，若非零向量 $\mathbf{v}$ 满足：
$$\mathbf{A}\mathbf{v} = \lambda\mathbf{v}$$
则 $\mathbf{v}$ 是 $\mathbf{A}$ 的特征向量，$\lambda$ 为对应的特征值。几何上，特征向量是经矩阵变换后方向保持不变的向量方向。

特征值之和等于迹：$\sum_i \lambda_i = \text{tr}(\mathbf{A})$

特征值之积等于行列式：$\prod_i \lambda_i = \det(\mathbf{A})$

在机器学习中，特征分解（Eigendecomposition）被用于主成分分析（PCA）的协方差矩阵分解、谱聚类中的拉普拉斯矩阵分析等。

对称矩阵的特征值均为实数，且总能找到一组正交的特征向量基。半正定矩阵（$\mathbf{x}^\top\mathbf{A}\mathbf{x} \geq 0$ 对所有 $\mathbf{x}$ 成立，记作 $\mathbf{A} \succeq 0$）的特征值均非负，协方差矩阵就是典型的半正定矩阵。

奇异值分解（SVD）

奇异值分解是线性代数中最重要的矩阵分解方法之一。对任意矩阵 $\mathbf{A} \in \mathbb{R}^{m \times n}$，SVD 将 $\mathbf{A}$ 分解为：
$$\mathbf{A} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^\top$$

其中 $\mathbf{U} \in \mathbb{R}^{m \times m}$ 是正交矩阵（左奇异向量），$\mathbf{\Sigma} \in \mathbb{R}^{m \times n}$ 是对角矩阵（奇异值 $\sigma_1 \geq \sigma_2 \geq \dots \geq 0$），$\mathbf{V} \in \mathbb{R}^{n \times n}$ 是正交矩阵（右奇异向量）。

SVD 的核心应用：

降维：保留前 $k$ 个最大的奇异值，能得到矩阵的最佳 $k$ 秩逼近（Eckart-Young 定理）。这正是 PCA 的数学基础——对数据中心化后的矩阵做 SVD，右奇异向量即为 PCA 的主成分方向。

伪逆（Pseudo-inverse）：非方阵或奇异矩阵的"广义逆"，由 $\mathbf{A}^+ = \mathbf{V}\mathbf{\Sigma}^+\mathbf{U}^\top$ 给出，用于求解最小范数最小二乘解。

推荐系统：SVD 用于矩阵补全，将用户-物品评分矩阵分解为低秩表示，捕捉隐语义因子。

矩阵分解族

除了 SVD，多种矩阵分解在机器学习中各有用途。

Cholesky 分解：对正定矩阵 $\mathbf{A}$，可分解为 $\mathbf{A} = \mathbf{L}\mathbf{L}^\top$，其中 $\mathbf{L}$ 为下三角阵。计算量是 LU 分解的一半，常用于高斯过程的协方差矩阵求逆和高效采样。

QR 分解：将 $\mathbf{A}$ 分解为 $\mathbf{A} = \mathbf{Q}\mathbf{R}$，其中 $\mathbf{Q}$ 为正交矩阵，$\mathbf{R}$ 为上三角矩阵。用于求解最小二乘问题和正交化。

LU 分解：$\mathbf{A} = \mathbf{L}\mathbf{U}$，其中 $\mathbf{L}$ 为下三角矩阵，$\mathbf{U}$ 为上三角矩阵。用于高效求解线性方程组和计算行列式。

张量

张量（Tensor）是向量和矩阵的高维推广。标量是零阶张量，向量是一阶张量，矩阵是二阶张量，更高阶的张量则用于深度学习中的多通道图像数据（批量 × 通道 × 高度 × 宽度，即四阶张量）。

张量运算的核心操作包括：

广播（Broadcasting）：在不同形状的张量之间执行元素运算时，自动扩展较小的张量维度

收缩（Contraction）：在指定的轴上求和，矩阵乘法本质上是二阶张量在一个轴上的收缩

Hadamard 积（元素积）：$(\mathbf{A} \odot \mathbf{B})_{ij} = a_{ij} \cdot b_{ij}$，在 LSTM 的门控机制等场景中常用

Kronecker 积：$\mathbf{A} \otimes \mathbf{B}$ 生成更大的分块矩阵，用于构造复杂概率模型和某些循环神经网络的参数化

线性代数在机器学习中的典型应用

PCA（主成分分析）：计算数据中心化后协方差矩阵 $\frac{1}{n}\mathbf{X}^\top\mathbf{X}$ 的特征向量，前 $k$ 个特征向量张成最佳 $k$ 维投影子空间，保留最大方差方向。

线性回归闭式解：最小化均方误差 $J(\mathbf{w}) = \|\mathbf{X}\mathbf{w} - \mathbf{y}\|^2$ 得到法方程 $\mathbf{X}^\top\mathbf{X}\mathbf{w} = \mathbf{X}^\top\mathbf{y}$。若 $\mathbf{X}^\top\mathbf{X}$ 可逆，闭式解为 $\mathbf{w}^* = (\mathbf{X}^\top\mathbf{X})^{-1}\mathbf{X}^\top\mathbf{y}$。

神经网络前向传播：一层全连接网络的前向计算 $\mathbf{z} = \mathbf{W}\mathbf{x} + \mathbf{b}$ 本质是矩阵-向量乘法，多层堆叠构成一系列线性变换和非线性激活的复合。

词嵌入：Word2Vec 等模型将词汇映射到低维向量空间，使得语义相似的词在向量空间中靠近，$\text{cosine}(\mathbf{v}_{\text{king}} - \mathbf{v}_{\text{man}} + \mathbf{v}_{\text{woman}}, \mathbf{v}_{\text{queen}}) \approx 1$。

图拉普拉斯矩阵：$\mathbf{L} = \mathbf{D} - \mathbf{W}$（$\mathbf{D}$ 为度矩阵，$\mathbf{W}$ 为邻接矩阵），其谱性质用于谱聚类和图神经网络的消息传递设计。
