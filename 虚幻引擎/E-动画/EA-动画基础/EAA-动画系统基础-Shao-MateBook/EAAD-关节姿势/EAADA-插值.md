“撬值”实际上是指“插值”（Interpolation）。在计算机图形学和动画中，插值是一种在两个已知数值之间计算出一个或多个中间数值的技术，用于生成平滑的过渡效果。插值在动画中尤为重要，因为它能使角色动作从一个姿势平滑过渡到另一个姿势。

### 插值类型

在动画和图形学中，常用的插值方法包括：

#### 线性插值（LERP - Linear Interpolation）

线性插值是最简单的一种插值方法，它在两个已知数值之间按照直线进行过渡。其公式如下：

\[ LERP(a, b, t) = a + t \cdot (b - a) \]

其中：
- \( a \) 和 \( b \) 是已知数值
- \( t \) 是插值参数，范围在 [0, 1] 之间

#### 球形线性插值（SLERP - Spherical Linear Interpolation）

球形线性插值常用于四元数插值，以避免旋转的非线性失真。其公式较为复杂，但基本思想是沿球面大圆弧插值。SLERP 的公式如下：

\[ SLERP(q_1, q_2, t) = \frac{\sin((1 - t) \theta)}{\sin \theta} q_1 + \frac{\sin(t \theta)}{\sin \theta} q_2 \]

其中：
- \( q_1 \) 和 \( q_2 \) 是四元数
- \( t \) 是插值参数，范围在 [0, 1] 之间
- \( \theta \) 是 \( q_1 \) 和 \( q_2 \) 之间的夹角

#### 归一化线性插值（NLERP - Normalized Linear Interpolation）

NLERP 是 LERP 和 SLERP 之间的一种折中方法。首先进行线性插值，然后对结果进行归一化。其公式如下：

\[ NLERP(q_1, q_2, t) = \frac{LERP(q_1, q_2, t)}{\| LERP(q_1, q_2, t) \|} \]

其中 \( \| \cdot \| \) 表示向量的长度（或模）。

### 插值在动画中的应用

在骨骼动画中，插值用于生成关键帧之间的平滑过渡。关键帧是动画中的重要姿态，通过在这些关键帧之间进行插值，可以生成流畅的动画效果。

#### 关键帧插值

假设我们有两个关键帧 \( K1 \) 和 \( K2 \)，它们分别表示角色在两个不同时间点的姿态。通过在 \( K1 \) 和 \( K2 \) 之间进行插值，可以生成中间帧 \( K_t \)，表示角色在这两个时间点之间的姿态。

```cpp
struct KeyFrame {
    Vector3 position;
    Quaternion rotation;
    Vector3 scale;
};

// 线性插值示例
KeyFrame interpolate(const KeyFrame& k1, const KeyFrame& k2, float t) {
    KeyFrame result;
    result.position = LERP(k1.position, k2.position, t);
    result.rotation = SLERP(k1.rotation, k2.rotation, t); // 用 SLERP 插值四元数
    result.scale = LERP(k1.scale, k2.scale, t);
    return result;
}
```

### 示例代码

以下是一个 C++ 示例，展示如何使用线性插值（LERP）和球形线性插值（SLERP）在两个关键帧之间进行插值：

```cpp
#include <iostream>
#include <Eigen/Dense>
#include <Eigen/Geometry>

using Vector3 = Eigen::Vector3f;
using Quaternion = Eigen::Quaternionf;

struct KeyFrame {
    Vector3 position;
    Quaternion rotation;
    Vector3 scale;
};

// 线性插值
Vector3 LERP(const Vector3& a, const Vector3& b, float t) {
    return a + t * (b - a);
}

// 球形线性插值
Quaternion SLERP(const Quaternion& q1, const Quaternion& q2, float t) {
    return q1.slerp(t, q2);
}

KeyFrame interpolate(const KeyFrame& k1, const KeyFrame& k2, float t) {
    KeyFrame result;
    result.position = LERP(k1.position, k2.position, t);
    result.rotation = SLERP(k1.rotation, k2.rotation, t);
    result.scale = LERP(k1.scale, k2.scale, t);
    return result;
}

int main() {
    // 定义两个关键帧
    KeyFrame k1 = { Vector3(0, 0, 0), Quaternion::Identity(), Vector3(1, 1, 1) };
    KeyFrame k2 = { Vector3(10, 10, 10), Quaternion(Eigen::AngleAxisf(M_PI, Vector3::UnitY())), Vector3(2, 2, 2) };

    // 插值参数 t，范围在 [0, 1] 之间
    float t = 0.5f;

    // 进行插值
    KeyFrame interpolatedFrame = interpolate(k1, k2, t);

    // 输出结果
    std::cout << "Interpolated Position: " << interpolatedFrame.position.transpose() << std::endl;
    std::cout << "Interpolated Scale: " << interpolatedFrame.scale.transpose() << std::endl;

    return 0;
}
```

在这个示例中，我们定义了两个关键帧 `k1` 和 `k2`，并在它们之间进行了插值，得到了中间帧的姿态。这种插值技术在动画中广泛应用，使得角色的动作平滑自然。