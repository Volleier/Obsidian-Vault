**单关节蒙皮（Single-Joint Skinning）是每个顶点只绑定一个关节、权重恒为 1 的蒙皮。它是推导蒙皮矩阵最简单的情形：《Game Engine Architecture》正是用"只有一个关节的骨架"来引出蒙皮矩阵 $\mathbf{K}_j = \mathbf{B}_{j\to M}^{-1}\,\mathbf{C}_{j\to M}$ 的。从效果上看，它等价于[[EAABC-刚性层级动画]]，只是网格在数据上是一整张，而不是分开的几块。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（Skinning and Matrix Palette Generation 中的 One-Jointed Skeleton 例子）。本文统一采用 GEA 和 UE 的行向量约定：点写在左边，$\mathbf{v}' = \mathbf{v}\mathbf{M}$。

## 问题是什么

蒙皮网格的顶点坐标是在绑定姿势下、以模型空间给出的。GEA 强调了一点：无论骨架处于绑定姿势还是任何其他姿势，蒙皮顶点的位置始终用模型空间表示。我们要找的是一个矩阵，把顶点从"绑定姿势下的模型空间位置"变到"当前姿势下的模型空间位置"，这就是蒙皮矩阵。

## 推导

关键的观察是：**顶点在它所绑定关节的局部空间里的坐标是不变的**。关节怎么转、怎么动，顶点跟着一起动，所以相对关节的位置永远相同。

于是计算分三步：

1. 把绑定姿势下的模型空间顶点 $\mathbf{v}_M^{B}$ 变到关节 $j$ 的局部空间。绑定姿势下关节的全局矩阵 $\mathbf{B}_{j\to M}$ 把关节空间变到模型空间，它的逆就把模型空间变到关节空间：

$$
\mathbf{v}_j = \mathbf{v}_M^{B}\,\mathbf{B}_{M\to j} = \mathbf{v}_M^{B}\,\left(\mathbf{B}_{j\to M}\right)^{-1}
$$

2. 关节移动到当前姿势，$\mathbf{v}_j$ 保持不变。

3. 用关节当前的全局矩阵 $\mathbf{C}_{j\to M}$ 把它变回模型空间：

$$
\mathbf{v}_M^{C} = \mathbf{v}_j\,\mathbf{C}_{j\to M} = \mathbf{v}_M^{B}\,\left(\mathbf{B}_{j\to M}\right)^{-1}\mathbf{C}_{j\to M}
$$

把两个矩阵合起来就是蒙皮矩阵：

$$
\mathbf{K}_j = \left(\mathbf{B}_{j\to M}\right)^{-1}\mathbf{C}_{j\to M},\qquad \mathbf{v}_M^{C} = \mathbf{v}_M^{B}\,\mathbf{K}_j
$$

用列向量约定的教材会写成 $\mathbf{K}_j = \mathbf{C}_j\,\mathbf{B}_j^{-1}$、$\mathbf{v}' = \mathbf{K}_j\mathbf{v}$，内容完全相同，只是乘法顺序反过来。

这个式子有一个很好的检查点：骨架处于绑定姿势时 $\mathbf{C}_{j\to M} = \mathbf{B}_{j\to M}$，于是 $\mathbf{K}_j = \mathbf{I}$，网格保持原样。实现蒙皮时如果绑定姿势下网格就变形了，多半是逆绑定矩阵算错或乘法顺序反了。

原笔记里写的是 $\mathbf{v}' = \mathbf{M}_B\,\mathbf{v}$，直接用关节的变换矩阵乘顶点，漏掉了逆绑定矩阵。这只在顶点坐标本来就定义在关节局部空间时才成立（那其实是刚性层级动画的数据组织方式）；对于以模型空间存储的蒙皮网格，缺了 $\mathbf{B}^{-1}$ 顶点会被变换两次。

## 各项的更新频率

$\left(\mathbf{B}_{j\to M}\right)^{-1}$ 在整个游戏过程中不变，通常和骨架一起预先算好存下来（GEA 给出的骨架数据结构里，每个关节除了名字和父索引，就只存了一个逆绑定姿势矩阵）。$\mathbf{C}_{j\to M}$ 每帧随姿势变化，由局部姿势逐级相乘得到。所以每帧的工作量是：为每个关节做一次矩阵乘法，然后为每个顶点做一次矩阵—向量乘法。

如果最终要的是世界空间坐标，还可以把模型到世界的矩阵 $\mathbf{M}_{M\to W}$ 预先乘进去：$\mathbf{K}_j' = \mathbf{K}_j\,\mathbf{M}_{M\to W}$，每个顶点省一次矩阵乘法。GEA 提到的例外是动画实例化（animation instancing），一大群角色共用一份矩阵调色板，此时模型到世界的矩阵必须单独保留。

法线和切线也要变换。只有旋转和均匀缩放时，直接用 $\mathbf{K}_j$ 的左上 3×3 部分变换再归一化即可；有非均匀缩放时，法线要用该部分的逆转置。

## 算法示意

下面用 UE 的矩阵类型写出单关节蒙皮的计算。UE 的 `FMatrix` 同样是行向量约定，`A * B` 表示先 A 后 B，所以乘法顺序和上面的公式一一对应。这是说明原理的示意代码，引擎本身并不这样逐顶点在 CPU 上算：

```cpp
// 单关节蒙皮示意：Vertex 为绑定姿势下的模型空间坐标
FVector3f SkinSingleJoint(const FVector3f& Vertex, const FMatrix44f& InvBindPose, const FMatrix44f& CurrentGlobal)
{
	const FMatrix44f SkinningMatrix = InvBindPose * CurrentGlobal; // K = B^-1 · C
	return SkinningMatrix.TransformPosition(Vertex);
}
```

在 UE 里，`InvBindPose` 对应 `USkeletalMesh::GetRefBasesInvMatrix()` 中的一项，`CurrentGlobal` 对应骨骼网格体组件的组件空间骨骼变换（`GetComponentSpaceTransforms()`），两者相乘的结果在引擎里叫 ReferenceToLocal 矩阵，由渲染模块上传到 GPU（详见 [[EAADH-蒙皮矩阵]]）。

## 单关节蒙皮的局限和用处

每个顶点只受一个关节影响，关节弯曲时相邻两块区域的顶点分别跟两个关节刚性运动，交界处的三角形被拉伸或压扁，视觉上和刚性层级动画的开裂问题类似，只是表现为三角形被拉长，而不是出现缝隙。要平滑过渡，必须让交界处的顶点同时受两个关节影响，这就是[[EAADD-多骨骼的蒙皮权重]]。

单关节蒙皮仍然有实际用途：机械角色、载具、硬表面道具本来就是刚性的，每个顶点一个关节完全够用，而且 GPU 上每个顶点只做一次矩阵乘法，数据里也不用存权重。

## 容易踩的坑

**漏掉逆绑定矩阵。** 最常见的错误，症状是绑定姿势下网格就已经偏移或变形。

**用列向量公式配行向量库。** 公式里的 $\mathbf{B}^{-1}\mathbf{C}$ 在列向量库里要写成 $\mathbf{C}\,\mathbf{B}^{-1}$，否则只有在变换可交换的特殊情况下（比如纯平移）结果才碰巧正确，调试时很容易被这种特例骗过。

**逆绑定矩阵在运行时重算。** 它是常量，应该在导入时算好。每帧求逆既浪费又可能引入数值误差。

## 相关

[[EAADD-多骨骼的蒙皮权重]] [[EAADH-蒙皮矩阵]] [[EAADI-蒙皮矩阵调色板]] [[EAABC-刚性层级动画]] [[EAABH-3D蒙皮动画]]
