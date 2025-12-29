# 变换的数学定义与分类

## 作为映射的变换
二维几何变换是从二维空间到自身的映射：
$$f: \mathbb{R}^2 \rightarrow \mathbb{R}^2, \quad (x, y) \mapsto (x', y')$$

矩阵表示提供该映射的可计算形式。对于点 $P = (x, y)$，变换后点 $P' = (x', y')$ 可通过矩阵乘法获得。

## 线性变换与仿射变换
线性变换满足：
1. 可加性：$f(u + v) = f(u) + f(v)$
2. 齐次性：$f(\alpha u) = \alpha f(u)$

其矩阵形式为：
$$P' = A \cdot P, \quad A \in \mathbb{R}^{2 \times 2}$$
包含旋转、缩放、错切、反射等。

仿射变换扩展线性变换，增加平移项：
$$P' = A \cdot P + t, \quad t \in \mathbb{R}^2$$
为统一表示为矩阵乘法，引入齐次坐标。

# 齐次坐标系统

## 动机与定义
平移变换 $P' = P + t$ 无法表示为 2×2 矩阵乘法。齐次坐标通过升维解决此问题：
- 点：$(x, y) \mapsto (x, y, 1)$
- 向量：$(v_x, v_y) \mapsto (v_x, v_y, 0)$

## 数学原理
二维欧氏空间嵌入三维投影空间：
$$\mathbb{R}^2 \hookrightarrow \mathbb{P}^2, \quad (x, y) \mapsto [x : y : 1]$$

仿射变换在齐次坐标下成为线性变换：
$$\begin{bmatrix}
x' \\
y' \\
1
\end{bmatrix}
=
\begin{bmatrix}
a & b & t_x \\
c & d & t_y \\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
x \\
y \\
1
\end{bmatrix}$$

## 归一化与投影
齐次坐标 $(x, y, w)$ 对应欧氏点：
$$\left( \frac{x}{w}, \frac{y}{w} \right), \quad w \neq 0$$

向量 $(v_x, v_y, 0)$ 表示无穷远点，不受平移影响。

# 仿射矩阵的几何解释

## 块结构分析
仿射矩阵分解为：
$$M =
\begin{bmatrix}
A & t \\
0 & 1
\end{bmatrix},
\quad
A = \begin{bmatrix}
a & b \\
c & d
\end{bmatrix},
\quad
t = \begin{bmatrix}
t_x \\
t_y
\end{bmatrix}$$


**几何意义**：
1. 平移部分 $t$：决定坐标原点的像
	$M \begin{bmatrix} 0 \\ 0 \\ 1 \end{bmatrix} = \begin{bmatrix} t_x \\ t_y \\ 1 \end{bmatrix}$
2. 线性部分 A：决定标准基向量的像
   - $e_x = (1, 0)^T$ 映射为 $(a, c)^T$
   - $e_y = (0, 1)^T$ 映射为 $(b, d)^T$

## 基变换视角
仿射变换 $P' = A P + t$ 可解释为：
1. 在原坐标系下，点坐标为 $P$
2. 线性部分 $A$ 改变坐标轴的缩放、旋转和倾斜
3. 平移部分 $A$ 改变坐标系原点的位置

# 基本变换的矩阵表示

## 平移变换
数学定义
$$T_t: \mathbb{R}^2 \rightarrow \mathbb{R}^2, \quad P \mapsto P + t$$

齐次矩阵
$$M_T =
\begin{bmatrix}
1 & 0 & t_x \\
0 & 1 & t_y \\
0 & 0 & 1
\end{bmatrix}$$

性质：
- 保持距离：$||T(P_1) - T(P_2)|| = ||P_1 - P_2||$
- 保持角度：$\angle(T(\vec{u}), T(\vec{v})) = \angle(\vec{u}, \vec{v})$
- 行列式 $det(A) = 1$

## 缩放变换
数学定义
$$S_{s_x, s_y}: (x, y) \mapsto (s_x x, s_y y)$$

齐次矩阵
$$M_S =
\begin{bmatrix}
s_x & 0 & 0 \\
0 & s_y & 0 \\
0 & 0 & 1
\end{bmatrix}$$

性质分析：
- 长度缩放：沿 $x$ 轴缩放 $|s_x|$，沿 $y$ 轴缩放 $|s_y|$
- 面积缩放因子：$det(A) = s_x s_y$
- 手性变化：当 $s_x s_y < 0$ 时发生镜像

## 旋转变换
数学定义（绕原点）
$$R_θ: (x, y) \mapsto (x\cosθ - y\sinθ, x\sinθ + y\cosθ)$$

齐次矩阵
$$M_R =
\begin{bmatrix}
\cosθ & -\sinθ & 0 \\
\sinθ & \cosθ & 0 \\
0 & 0 & 1
\end{bmatrix}$$

正交性证明
$$R_θ^T R_θ =
\begin{bmatrix}
\cosθ & \sinθ \\
-\sinθ & \cosθ
\end{bmatrix}
\begin{bmatrix}
\cosθ & -\sinθ \\
\sinθ & \cosθ
\end{bmatrix}
= I$$

性质：
- 保距性：$||R(P)|| = ||P||$
- 保角性：$\angle(R(\vec{u}), R(\vec{v})) = \angle(\vec{u}, \vec{v})$
- 行列式 $det(R) = 1$，保持手性

## 反射变换
### 关于 $x$ 轴的反射：$(x, y) \mapsto (x, -y)$
$$F_x = \begin{bmatrix}
1 & 0 & 0 \\
0 & -1 & 0 \\
0 & 0 & 1
\end{bmatrix}$$

### 关于 $y$ 轴：$(x, y) \mapsto (-x, y)$
$$F_y = \begin{bmatrix}
-1 & 0 & 0 \\
0 & 1 & 0 \\
0 & 0 & 1
\end{bmatrix}$$

性质：
- 等距变换：保持两点间距离
- 行列式 $\det(F) = -1$，表示手性翻转
- 幂等性：$F^2 = I$

## 错切变换
数学定义
- $-x$ 方向错切：$(x, y) \mapsto (x + sh_x \cdot y, y)$
- - $y$ 方向错切：$(x, y) \mapsto (x, sh_y \cdot x + y)$

矩阵表示
$$H_x(sh_x) =
\begin{bmatrix}
1 & sh_x & 0 \\
0 & 1 & 0 \\
0 & 0 & 1
\end{bmatrix},
\quad
H_y(sh_y) =
\begin{bmatrix}
1 & 0 & 0 \\
sh_y & 1 & 0 \\
0 & 0 & 1
\end{bmatrix}$$

性质：
- 改变角度但不改变面积：$det(H) = 1$
- 非正交变换

# 变换组合的数学理论

## 函数复合视角
变换序列 $T_1, T_2, \dots, T_n$ 的复合：
$$T = T_n \circ T_{n-1} \circ \cdots \circ T_1$$

矩阵表示：
$$M = M_n \cdot M_{n-1} \cdot ... \cdot M_1$$


## 顺序的数学解释
对于点 $P$，应用变换 $T_1 $后 $T_2$：
$$P' = T_2(T_1(P)) = (T_2 \circ T_1)(P)$$

对应矩阵乘法：
```
P' = M₂ · (M₁ · P) = (M₂ · M₁) · P
```
故 T₁ 对应的矩阵 M₁ 在乘法中位于右侧。

## 5.3 矩阵乘法的不可交换性
一般地，M₁M₂ ≠ M₂M₁，反映变换顺序的重要性。

# 6. 复合变换的构造方法

## 6.1 绕任意点旋转
设旋转中心 C = (c_x, c_y)，旋转角 θ。

**分解步骤**：
1. 平移使 C 与原点重合：T(-c_x, -c_y)
2. 绕原点旋转 θ：R(θ)
3. 平移恢复：T(c_x, c_y)

**复合矩阵**：
```
M = T(c_x, c_y) · R(θ) · T(-c_x, -c_y)
```

## 6.2 关于任意直线反射
设直线 L: ax + by + c = 0。

**构造方法**：
1. 平移使 L 过原点：平移向量 t 满足 a t_x + b t_y + c = 0
2. 旋转使 L 与 x 轴对齐：旋转角 α = arctan(a/b)
3. 关于 x 轴反射
4. 逆旋转：R(-α)
5. 逆平移：T(-t)

**数学表达式**：
```
M = T(-t) · R(-α) · F_x · R(α) · T(t)
```

# 7. 逆变换与坐标系变换

## 7.1 仿射矩阵的逆
对于 M = \begin{bmatrix} A & t \\ 0 & 1 \end{bmatrix}，若 det(A) ≠ 0，则：
```
M^{-1} = \begin{bmatrix}
A^{-1} & -A^{-1}t \\
0 & 1
\end{bmatrix}
```

**几何解释**：
1. 减去平移：P - t
2. 施加线性逆变换：A⁻¹(P - t)

## 7.2 坐标系变换的两种视角
1. **点变换**：固定坐标系，移动点
2. **坐标系变换**：固定点，改变坐标系

二者数学形式相同，解释不同。逆矩阵常用于坐标系变换。
# 8. 变换的性质与判别

## 8.1 行列式的几何意义
- det(A) > 0：保持手性（orientation）
- det(A) < 0：翻转手性（镜像）
- |det(A)|：面积缩放因子

## 8.2 正交变换的特征
矩阵 Q 为正交矩阵当且仅当：
```
Q^T Q = I
```

性质：
- 保持内积：(Qu)ᵀ(Qv) = uᵀv
- 保持长度：‖Qu‖ = ‖u‖
- 行列式 det(Q) = ±1

## 8.3 仿射变换的分类
根据线性部分 A 的性质：

| 变换类型 | $det(A)$ | $A^TA$ | 保持性质    |
| ---- | -------- | ------ | ------- |
| 刚体变换 | ±1       | $I$    | 距离、角度   |
| 相似变换 | ≠0       | λI     | 角度、比例   |
| 仿射变换 | ≠0       | 任意     | 平行性、共线性 |
