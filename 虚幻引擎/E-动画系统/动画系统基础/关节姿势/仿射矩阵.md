仿射矩阵（Affine Matrix）是线性代数中的一种矩阵，用于表示仿射变换。仿射变换包括平移、旋转、缩放和剪切等操作，这些操作在计算机图形学和游戏开发中广泛使用。仿射矩阵能够通过单一矩阵运算实现多个变换的组合，提供了计算上的简便性和高效性。

### 仿射矩阵的形式

在三维空间中，仿射变换通常使用 4x4 矩阵来表示。一个标准的 4x4 仿射矩阵形式如下：

```
| a11 a12 a13 a14 |
| a21 a22 a23 a24 |
| a31 a32 a33 a34 |
|  0   0   0   1  |
```

其中：
- 上左 3x3 子矩阵（a11 到 a33）用于表示旋转、缩放和剪切变换。
- 最右列（a14, a24, a34）用于表示平移变换。
- 最后一行（0, 0, 0, 1）使得矩阵能够进行齐次坐标变换，便于组合多个仿射变换。

### 常见仿射变换

#### 平移（Translation）

平移变换将对象在空间中移动。平移矩阵如下：

```
| 1 0 0 Tx |
| 0 1 0 Ty |
| 0 0 1 Tz |
| 0 0 0  1 |
```

其中 (Tx, Ty, Tz) 是平移向量。

#### 旋转（Rotation）

旋转变换绕坐标轴旋转对象。例如，绕 z 轴旋转的矩阵为：

```
| cosθ -sinθ 0 0 |
| sinθ  cosθ 0 0 |
|  0     0   1 0 |
|  0     0   0 1 |
```

θ 是旋转角度。

#### 缩放（Scaling）

缩放变换改变对象的大小。缩放矩阵如下：

```
| Sx  0  0 0 |
|  0 Sy  0 0 |
|  0  0 Sz 0 |
|  0  0  0 1 |
```

其中 Sx, Sy, Sz 是缩放因子。

#### 剪切（Shear）

剪切变换改变对象形状，使其在某一方向上倾斜。例如，x 方向上的剪切矩阵为：

```
| 1 Shx 0 0 |
| 0  1  0 0 |
| 0  0  1 0 |
| 0  0  0 1 |
```

其中 Shx 是剪切因子。

### 仿射矩阵的组合

多个仿射变换可以通过矩阵乘法进行组合。例如，首先平移然后旋转的复合变换可以表示为：

```
M = R * T
```

其中 R 是旋转矩阵，T 是平移矩阵。

### 示例代码

以下是一个简单的 C++ 示例，展示如何使用仿射矩阵进行变换：

```cpp
#include <iostream>
#include <Eigen/Dense>

int main() {
    Eigen::Matrix4f transform = Eigen::Matrix4f::Identity();

    // 平移
    Eigen::Vector3f translation(1.0f, 2.0f, 3.0f);
    transform.block<3, 1>(0, 3) = translation;

    // 旋转（绕 Z 轴 45 度）
    float angle = 45.0f * M_PI / 180.0f;
    Eigen::Matrix3f rotation;
    rotation = Eigen::AngleAxisf(angle, Eigen::Vector3f::UnitZ());
    transform.block<3, 3>(0, 0) = rotation;

    // 打印变换矩阵
    std::cout << "Transformation Matrix:\n" << transform << std::endl;

    return 0;
}
```

这个示例使用了 Eigen 库来处理矩阵运算。通过平移和旋转的组合，生成了一个复合仿射变换矩阵。

仿射矩阵是图形学和动画中的基本工具，它提供了一种统一的方式来处理复杂的空间变换，使得对象的移动、旋转和缩放变得简单而高效。