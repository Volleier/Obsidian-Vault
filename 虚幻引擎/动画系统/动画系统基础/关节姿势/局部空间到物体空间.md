在计算机图形学和游戏开发中，“局部空间”（Local Space）和“物体空间”（Object Space）是两个常用的术语，分别指代对象在其自身坐标系和对象相对于其父对象或全局坐标系的位置和姿态。将一个对象从局部空间变换到物体空间通常涉及应用一系列仿射变换，包括平移、旋转和缩放。

### 局部空间

局部空间指的是对象自身的坐标系。在局部空间中，对象的原点通常位于自身中心或某个定义的位置，所有的变换（如平移、旋转和缩放）都相对于这个局部原点进行。

### 物体空间

物体空间指的是对象相对于其父对象（如果有父对象）或世界坐标系的坐标系。物体空间是对象在场景中的最终位置和姿态，通过将局部空间中的点或向量应用变换矩阵获得。

### 局部空间到物体空间的变换

要将对象从局部空间变换到物体空间，需要应用一个仿射变换矩阵，这个矩阵结合了平移、旋转和缩放信息。通常，这个变换矩阵称为“模型矩阵”或“世界矩阵”。

### 变换矩阵的构建

变换矩阵通常由三个主要部分组成：平移矩阵、旋转矩阵和缩放矩阵。这些矩阵可以组合在一起形成一个单一的变换矩阵。以下是这些矩阵的形式：

#### 平移矩阵

```
| 1 0 0 Tx |
| 0 1 0 Ty |
| 0 0 1 Tz |
| 0 0 0  1 |
```

其中 (Tx, Ty, Tz) 是平移向量。

#### 旋转矩阵

例如，绕 z 轴旋转的矩阵：

```
| cosθ -sinθ 0 0 |
| sinθ  cosθ 0 0 |
|  0     0   1 0 |
|  0     0   0 1 |
```

#### 缩放矩阵

```
| Sx  0  0 0 |
|  0 Sy  0 0 |
|  0  0 Sz 0 |
|  0  0  0 1 |
```

其中 Sx, Sy, Sz 是缩放因子。

### 组合变换矩阵

可以将平移、旋转和缩放矩阵相乘，得到一个综合的变换矩阵：

```
M = T * R * S
```

其中 T 是平移矩阵，R 是旋转矩阵，S 是缩放矩阵。

### 示例代码

以下是一个示例代码，展示如何在 C++ 中使用 Eigen 库构建变换矩阵，并将一个点从局部空间变换到物体空间：

```cpp
#include <iostream>
#include <Eigen/Dense>

int main() {
    // 定义平移、旋转和缩放
    Eigen::Vector3f translation(1.0f, 2.0f, 3.0f);
    float angle = 45.0f * M_PI / 180.0f;
    Eigen::Vector3f scale(2.0f, 2.0f, 2.0f);

    // 构建平移矩阵
    Eigen::Matrix4f T = Eigen::Matrix4f::Identity();
    T.block<3, 1>(0, 3) = translation;

    // 构建旋转矩阵（绕Z轴）
    Eigen::Matrix3f R;
    R = Eigen::AngleAxisf(angle, Eigen::Vector3f::UnitZ());
    Eigen::Matrix4f R4 = Eigen::Matrix4f::Identity();
    R4.block<3, 3>(0, 0) = R;

    // 构建缩放矩阵
    Eigen::Matrix4f S = Eigen::Matrix4f::Identity();
    S(0, 0) = scale.x();
    S(1, 1) = scale.y();
    S(2, 2) = scale.z();

    // 组合变换矩阵
    Eigen::Matrix4f M = T * R4 * S;

    // 定义局部空间中的点
    Eigen::Vector4f localPoint(1.0f, 1.0f, 1.0f, 1.0f);

    // 变换到物体空间
    Eigen::Vector4f objectPoint = M * localPoint;

    // 输出结果
    std::cout << "Object Space Point: " << objectPoint.transpose() << std::endl;

    return 0;
}
```

在这个示例中，我们定义了平移、旋转和缩放，然后将这些变换组合成一个变换矩阵，最终将局部空间中的点变换到物体空间中。通过这种方式，可以将对象在其局部坐标系中的位置和姿态准确地转换到全局坐标系中，用于渲染和其他计算。