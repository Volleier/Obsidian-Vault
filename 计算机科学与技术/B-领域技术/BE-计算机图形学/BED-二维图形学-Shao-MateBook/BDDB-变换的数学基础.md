# 变换的数学基础
**二维图形变换是对图形的几何坐标进行的操作，其最终结果表现为每个像素点的颜色被重新计算和放置。**
二维图形的变换是计算机图形学中的核心内容，其数学基础主要建立在线性代数、向量几何和齐次坐标系统之上，主要以下几个部分组成：基本变换类型、齐次坐标、变换的组合和级联、几何基础与向量运算


# 基本变换类型
## 平移变换
将图形中的每个点沿指定方向移动固定距离

### 数学表示
#### 非齐次坐标
$$
\begin{gather}
x'=x+t_x \\
y'=y+t_y
\end{gather}
​$$
#### 齐次坐标矩阵（3×3）
$$
M_{\text{trans}}(t_x, t_y) = \begin{bmatrix}
1 & 0 & t_x \\
0 & 1 & t_y \\
0 & 0 & 1
\end{bmatrix}
$$

### 几何意义
- 保持图形形状、大小、方向不变
- 仅改变图形位置
- 无法用2×2线性变换矩阵表示
## 缩放变换
沿坐标轴方向拉伸或压缩图形。

### 数学表示
#### 非齐次坐标

$$
\begin{gather}
    x'=s_x \cdot x \\ 
    y'=s_y\cdot y
\end{gather}
​$$

    
#### 齐次坐标矩阵
$$

M_{\text{scale}}{(S_y,S_y)} = \begin{bmatrix}
s_x & 0 & 0 \\
0 & s_y & 0 \\
0 & 0 & 1
\end{bmatrix}
$$
类型：
- 均匀缩放：$s_x=s_y$​，保持形状比例
- 非均匀缩放：$sx \neq sy​$，产生拉伸或压缩变形

### 几何性质
- 当 $s_x,s_y>1$ 时，图形放大
- 当 $0<s_x,s_y<1$ 时，图形缩小
- 当缩放因子为负时，产生反射效果

## 旋转变换
绕指定点（通常为原点）旋转图形一定角度

### 数学表示
#### 非齐次坐标：  
$$
\begin{gather}
    x'=x\cos ⁡\theta − y\sin⁡ \theta \\ 
    y'=x\sin⁡ \theta + y\cos⁡ \theta
\end{gather}
​$$
#### 齐次坐标矩阵：
$$
M_{\text{rotate}}(\theta) = \begin{bmatrix}
\cos \theta & -\sin \theta & 0 \\
\sin \theta & \cos \theta & 0 \\
0 & 0 & 1
\end{bmatrix}
$$

方向约定：
- 逆时针旋转：$\theta > 0$（标准约定）
- 顺时针旋转：$\theta < 0$

### 重要性质：
- 旋转矩阵是正交矩阵：$R^{−1}(\theta)=R^T(\theta)=R(−\theta)$
- 保持向量长度和角度不变
- 旋转中心为原点

## 反射变换
生成图形关于某条直线（轴）的镜像
常见反射矩阵：
- 关于X轴反射：
$$
F_{x} = \begin{bmatrix}
1 & 0 & 0 \\
0 & -1 & 0 \\
0 & 0 & 1
\end{bmatrix}
$$
- 关于Y轴反射：
$$
F_{y} = \begin{bmatrix}
-1 & 0 & 0 \\
0 & 1 & 0 \\
0 & 0 & 1
\end{bmatrix}
$$

- 关于直线 $y=x$ 反射：
$$
F_{x=y} = \begin{bmatrix}
0 & 1 & 0 \\
1 & 0 & 0 \\
0 & 0 & 1
\end{bmatrix}
$$

### 几何特性
- 行列式值为 $-1$
- 属于等距变换（保持距离不变）
- 改变图形的手性（左右手性互换）

## 错且变换
使图形沿某一方向发生倾斜变形

### 数学表示
#### X方向错切：
$$
H_{x}{(sh_x)} = \begin{bmatrix}
1 & sh_x & 0 \\
0 & 1 & 0 \\
0 & 0 & 1
\end{bmatrix}
$$


$x'=x+sh_x \cdot y, y'=y$
#### Y方向错切：
$$
H_{y}{(sh_y)} = \begin{bmatrix}
1 & 0 & 0 \\
sh_y & 1 & 0 \\
0 & 0 & 1
\end{bmatrix}
$$

$x'=x,y'=sh_y \cdot x+y$

### 几何效果
- 保持面积不变
- 将矩形变为平行四边形
- 在字体倾斜、三维投影中有应用

# 齐次坐标（Homogeneous Coordinates）
齐次坐标，英文名Homogeneous Coordinates，在平移变换无法用2×2矩阵统一表示，导致变换组合复杂化的情况下使用，齐次坐标可将二维点从$(x,y)$扩展为三维齐次坐标 $(x,y,w)$。

## 定义与表示
- 基本定义
	- 二维点 $P=(x,y)$ 的齐次坐标：$(x,y,1)$
	- 二维向量 $v⃗=(v_x,v_y)$ 的齐次坐标：$(v_x,v_y,0)$
- 归一化处理
	- 齐次坐标 $(x,y,w)$ 对应二维点 $(x/w,y/w)$，其中 $w \neq 0$
	- 所有 $(k_x,k_y,k_w)（k \neq 0）$表示同一个二维点

## 核心优势

### 统一表示所有仿射变换
- 平移、旋转、缩放、反射、错切均可用3×3矩阵表示
- 变换组合简化为矩阵乘法

### 区分点与向量
- 点：w=1w=1，受平移影响
- 向量：w=0w=0，不受平移影响
- 几何意义明确：向量 = 点 - 点

### 支持透视变换扩展
- 为三维图形学的透视投影提供基础
- 可实现灭点、近大远小等效果

### 简化变换组合与求逆
- 复合变换：$M=M_n⋯M_2 \cdot M_1$
- 逆变换：$M−1$ 直接计算

## 变换矩阵的一般形式
仿射变换的齐次矩阵：
$$
M = \begin{bmatrix}
a & b & t_x \\
c & d & t_y \\
0 & 0 & 1
\end{bmatrix}
$$

子矩阵分解：
- 线性变换部分：$\begin{bmatrix}a & b \\c & d \end{bmatrix}$（旋转、缩放、错切、反射）
- 平移部分：$\begin{bmatrix}t_x \\ t_y \end{bmatrix}$
- 底部行：$\begin{bmatrix} 0 & 0 & 1 \end{bmatrix}$ 保证仿射性
# 变换的组合与级联
## 基本原理
多个变换按顺序应用，等价于单个复合变换
数学原理：
- 对点 $P$ 依次应用变换 $T_1,T_2,…,Tn$​
- 最终结果：$P'=T_n(⋯ T_2(T_1(P)) ⋯ )$
-  矩阵形式：$P'=M_n ⋯ M_2 \cdot M_1 \cdot P$

## 矩阵乘法顺序
变换应用顺序与矩阵乘法顺序**相反**。
示例：先缩放 $S$，后旋转 $R$，再平移 $T$
- 正确：$M=T \cdot R \cdot S$
- 点变换：$P'=T(R(S(P)))=(T \cdot R \cdot S) \cdot P$

## 复合变换示例

### 绕任意点旋转
步骤
1. 平移使旋转中心 $C(cx,cy)$ 移至原点：$T(−c_x,−c_y)$
2. 绕原点旋转 $\theta$ 角度：$R(\theta)$
3. 平移回原位置：$T(c_x,c_y)$

复合矩阵：

$$M=T(c_x,c_y) \cdot R(\theta) \cdot T(−c_x,−c_y)$$

### 关于任意直线反射
设直线方程为：$ax+by+c=0$

步骤：
1. 平移使直线过原点：$T(0,−c/b)$（假设 $b \neq 0$）
2. 旋转使直线与X轴对齐：$R(− \alpha )$，其中 $\alpha=\arctan ⁡(a/b)$
3. 关于X轴反射：$F_x$
4. 旋转回原方向：$R(\alpha)$
5. 平移回原位置：$T(0,c/b)$

# 几何基础与向量运算

## 点与向量的数学区分
### 点的特性
- 表示空间中的绝对位置
- 齐次坐标 $w=1$
- 受平移变换影响
- 可进行：点 - 点 = 向量

### 向量的特性
- 表示方向和大小
- 齐次坐标 $w=0$
- 不受平移影响
- 可进行：向量加减、数乘、点积、叉积

## 基本向量运算

### 向量加法与减法
- 几何意义：三角形法则/平行四边形法则
- 代数形式：$\vec{u} \pm \vec{v} = (u_x \pm v_x, u_y \pm v_y)$
- 应用：计算位移、相对位置

### 数乘运算
- 定义：$k\vec{v} = (kv_x, kv_y)$
- 几何意义：缩放向量长度，保持/反转方向
- $k>0$：同向缩放
- $k<0$：反向缩放
- $k=0$：零向量

### 点积（内积）
定义
$$\vec{u} \cdot \vec{v} = u_x v_x + u_y v_y = \|\vec{u}\|\|\vec{v}\|\cos\theta $$
几何意义与应用
1. 计算夹角：$\cos\theta = \frac{\vec{u} \cdot \vec{v}}{\|\vec{u}\|\|\vec{v}\|}$
2. 向量投影：$\text{proj}_{\vec{v}}\vec{u} = \frac{\vec{u} \cdot \vec{v}}{\vec{v} \cdot \vec{v}} \vec{v}$
3. 判断垂直：$\vec{u} \cdot \vec{v} = 0 \iff \vec{u} \perp \vec{v}$
4. 计算长度平方：$\vec{v} \cdot \vec{v} = \|\vec{v}\|^2$

### 二维叉积（外积）
定义：
$$\vec{u} \times \vec{v} = u_x v_y - u_y v_x = \|\vec{u}\|\|\vec{v}\|\sin\theta$$

几何意义与应用
1. 计算平行四边形/三角形面积：$\text{Area} = |\vec{u} \times \vec{v}|$
2. 判断方向/旋转：
    - $\vec{u} \times\vec{v}>0$：$\vec{v}$ 在 $\vec{u}$ 逆时针方向
    -  $\vec{u} \times\vec{v}<0$：$\vec{v}$ 在 $\vec{u}$ 顺时针方向
    - $\vec{u} \times\vec{v}=0$：两向量共线
3. 多边形凸性测试
4. 光线与线段求交

## 坐标系变换

### 局部坐标到世界坐标
对象定义在自己的局部坐标系中，需要放置到场景的世界坐标系。

变换矩阵
$$M_{\text{local→world}} = T_{\text{world}} \cdot R_{\text{world}} \cdot S_{\text{world}}$$

### 世界坐标到视图坐标
从世界坐标系转换到相机/观察者坐标系

构造视图矩阵
1. 计算相机坐标系基向量
    - $\hat{z} = \frac{\text{相机位置} - \text{目标位置}}{\|\text{相机位置} - \text{目标位置}\|}$
    - $\hat{x} = \frac{\text{上方向} \times \hat{z}}{\|\text{上方向} \times \hat{z}\|}$
    - $\hat{y} = \hat{z} \times \hat{x}$
2. 视图矩阵
$$V = \begin{bmatrix}
\hat{x}_x & \hat{x}_y & -(\hat{x} \cdot \text{相机位置}) \\
\hat{y}_x & \hat{y}_y & -(\hat{y} \cdot \text{相机位置}) \\
\hat{z}_x & \hat{z}_y & -(\hat{z} \cdot \text{相机位置}) \\
0 & 0 & 1
\end{bmatrix}$$

### 坐标变换链
完整图形管线中的坐标变换：
$$P_{screen}=M_{projection} \cdot M_{view} \cdot M_{world} \cdot P_{local}$$
