## 梯度与一阶差分

对连续函数 $I(x,y)$，梯度为：

$$
\nabla I = \left[\frac{\partial I}{\partial x},\ \frac{\partial I}{\partial y}\right]
$$

常用的边缘强度（梯度幅值）：

$$
\|\nabla I\| = \sqrt{G_x^2 + G_y^2}
$$

在离散图像上，$G_x,G_y$ 由卷积核近似实现。

## Sobel 算子（经典一阶梯度核）

Sobel 在 x 方向的核（检测垂直边缘）：

$$
K_x=
\begin{bmatrix}
-1&0&1\\
-2&0&2\\
-1&0&1
\end{bmatrix}
$$

对应的 y 方向核：

$$
K_y=
\begin{bmatrix}
-1&-2&-1\\
0&0&0\\
1&2&1
\end{bmatrix}
$$

使用方式：

- $G_x = I * K_x$
- $G_y = I * K_y$
- $G = \sqrt{G_x^2 + G_y^2}$（或用 $|G_x|+|G_y|$ 近似）

## 三、拉普拉斯（Laplacian，二阶变化）
拉普拉斯算子近似二阶导数，对“变化的变化”敏感：
$$
\nabla^2 I = \frac{\partial^2 I}{\partial x^2} + \frac{\partial^2 I}{\partial y^2}
$$

离散核：
$$
K_{lap}=
\begin{bmatrix}
0&-1&0\\
-1&4&-1\\
0&-1&0
\end{bmatrix}
$$

拉普拉斯常用于边缘增强或作为锐化的一部分（见去噪与增强）。