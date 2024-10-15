球面非线性插值（NLERP, Normalized Linear Interpolation）是一种用于插值旋转的算法。虽然 NLERP 的计算简单，但在某些情况下会导致非均匀速度变化。为了保证插值结果的最短路径，我们通常需要对插值的四元数进行归一化处理。

### 基本概念

1. 四元数：用于表示旋转。四元数 \(q\) 有四个分量 \(w, x, y, z\)，能够避免欧拉角的万向节锁问题。
2. 插值参数 \(t\)：在 [0, 1] 之间的参数，用于表示插值的进度。\(t=0\) 对应起始位置，\(t=1\) 对应结束位置。

### NLERP 算法

给定两个四元数 \(q_1\) 和 \(q_2\)，NLERP 插值公式为：

\[ \text{NLERP}(q_1, q_2, t) = \frac{(1 - t) \cdot q_1 + t \cdot q_2}{||(1 - t) \cdot q_1 + t \cdot q_2||} \]

其中，\(\| \cdot \|\) 表示向量的模长。

### 最短路径插值

为了保证插值的最短路径，通常需要检查两个四元数的点积是否为负。如果点积为负，则取四元数的反方向（即取负值）来进行插值。

### 示例代码

以下是如何在 C++ 中实现最短路径的 NLERP 插值示例代码：

#### C++ 代码

```cpp
#include <iostream>
#include <Eigen/Dense>
#include <Eigen/Geometry>

using namespace Eigen;

// 计算四元数之间的球面非线性插值 (NLERP)
Quaternionf nlerp(const Quaternionf& q1, const Quaternionf& q2, float t) {
    Quaternionf q2Modified = q2;
    // 检查是否需要取反方向以确保最短路径
    if (q1.dot(q2) < 0.0f) {
        q2Modified = -q2;
    }
    // 计算插值
    Quaternionf result = (1.0f - t) * q1.coeffs() + t * q2Modified.coeffs();
    return result.normalized();
}

int main() {
    // 定义两个四元数
    Quaternionf rotA = Quaternionf::Identity();
    Quaternionf rotB = AngleAxisf(M_PI / 2, Vector3f::UnitY());

    // 插值因子 t
    float t = 0.5f;

    // 计算插值结果
    Quaternionf interpolatedRot = nlerp(rotA, rotB, t);
    std::cout << "Interpolated Rotation: " << interpolatedRot.coeffs().transpose() << std::endl;

    return 0;
}
```

### GLSL 代码

在 GLSL 中进行 NLERP 插值时，也需要确保插值的最短路径。假设我们在 C++ 中计算了四元数插值结果，并将其传递到着色器中：

```glsl
#version 330 core

layout(location = 0) in vec3 a_Position;
layout(location = 1) in vec3 a_Normal;
layout(location = 2) in vec4 a_BoneWeights;
layout(location = 3) in ivec4 a_BoneIndices;

uniform mat4 u_SkinningPalette[100]; // 假设最多有 100 个关节

void main() {
    mat4 skinningMatrix = mat4(0.0);

    skinningMatrix += a_BoneWeights.x * u_SkinningPalette[a_BoneIndices.x];
    skinningMatrix += a_BoneWeights.y * u_SkinningPalette[a_BoneIndices.y];
    skinningMatrix += a_BoneWeights.z * u_SkinningPalette[a_BoneIndices.z];
    skinningMatrix += a_BoneWeights.w * u_SkinningPalette[a_BoneIndices.w];

    vec4 skinnedPosition = skinningMatrix * vec4(a_Position, 1.0);

    gl_Position = /* 投影矩阵 * 视图矩阵 * */ skinnedPosition;
}
```

### NLERP 的应用

1. 动画系统：在关键帧动画系统中，NLERP 用于在关键帧之间平滑插值角色的旋转姿态。
2. 摄像机运动：在游戏和虚拟现实应用中，NLERP 用于平滑插值摄像机的旋转，使视角切换更加平滑自然。
3. 机械臂和机器人：在机器人学中，NLERP 用于平滑插值机械臂的关节旋转，实现精确的运动控制。

### 优点

- 计算简单：NLERP 的计算量比 SLERP 小，适用于实时应用。
- 平滑旋转：虽然不保证恒定速度，但可以产生平滑的旋转效果。
- 避免万向节锁：使用四元数表示旋转，可以避免欧拉角表示中的万向节锁问题。

### 结论

NLERP 是一种计算简单且有效的旋转插值算法，适用于实时动画和运动控制。通过确保插值过程中的最短路径，可以进一步提高插值结果的质量，使角色动画和视角切换更加自然流畅。


球面非线性插值（NLERP）的最短路径插值在计算旋转时确保插值的路径是最短的。为了实现这一点，需要检查两个四元数的点积。如果点积为负，说明两个四元数之间的夹角大于 180 度，此时需要将其中一个四元数取负值，这样插值路径就会是最短的。

### NLERP 最短路径插值

NLERP 插值公式：
\[ \text{NLERP}(q_1, q_2, t) = \frac{(1 - t) \cdot q_1 + t \cdot q_2}{||(1 - t) \cdot q_1 + t \cdot q_2||} \]

为了确保插值路径是最短的，我们需要在插值前检查四元数的点积：

\[ \text{if} (q_1 \cdot q_2 < 0) \text{ then } q_2 = -q_2 \]

### 示例代码

以下是实现最短路径 NLERP 插值的示例代码：

#### C++ 代码

```cpp
#include <iostream>
#include <Eigen/Dense>
#include <Eigen/Geometry>

using namespace Eigen;

// 计算四元数之间的球面非线性插值 (NLERP) 并确保最短路径
Quaternionf nlerp(const Quaternionf& q1, const Quaternionf& q2, float t) {
    Quaternionf q2Modified = q2;
    // 检查是否需要取反方向以确保最短路径
    if (q1.dot(q2) < 0.0f) {
        q2Modified = -q2;
    }
    // 计算插值
    Quaternionf result = (1.0f - t) * q1.coeffs() + t * q2Modified.coeffs();
    return result.normalized();
}

int main() {
    // 定义两个四元数
    Quaternionf rotA = Quaternionf::Identity();
    Quaternionf rotB = AngleAxisf(M_PI / 2, Vector3f::UnitY());

    // 插值因子 t
    float t = 0.5f;

    // 计算插值结果
    Quaternionf interpolatedRot = nlerp(rotA, rotB, t);
    std::cout << "Interpolated Rotation: " << interpolatedRot.coeffs().transpose() << std::endl;

    return 0;
}
```

### GLSL 代码

在 GLSL 中进行最短路径 NLERP 插值时，也需要检查四元数的点积以确保插值路径最短。假设我们在 C++ 中计算了四元数插值结果，并将其传递到着色器中：

```glsl
#version 330 core

layout(location = 0) in vec3 a_Position;
layout(location = 1) in vec3 a_Normal;
layout(location = 2) in vec4 a_BoneWeights;
layout(location = 3) in ivec4 a_BoneIndices;

uniform mat4 u_SkinningPalette[100]; // 假设最多有 100 个关节

void main() {
    mat4 skinningMatrix = mat4(0.0);

    skinningMatrix += a_BoneWeights.x * u_SkinningPalette[a_BoneIndices.x];
    skinningMatrix += a_BoneWeights.y * u_SkinningPalette[a_BoneIndices.y];
    skinningMatrix += a_BoneWeights.z * u_SkinningPalette[a_BoneIndices.z];
    skinningMatrix += a_BoneWeights.w * u_SkinningPalette[a_BoneIndices.w];

    vec4 skinnedPosition = skinningMatrix * vec4(a_Position, 1.0);

    gl_Position = /* 投影矩阵 * 视图矩阵 * */ skinnedPosition;
}
```

### 关键步骤

1. 检查点积：如果两个四元数的点积为负，说明它们之间的夹角大于 180 度。此时需要将其中一个四元数取反。
2. 插值：在插值计算时，将四元数的点积检查结果应用到插值公式中，以确保插值路径最短。
3. 归一化：插值结果需要归一化，以确保结果仍然是一个有效的四元数。

### 结论

通过使用球面非线性插值（NLERP）并确保插值路径的最短，我们可以实现平滑且计算简单的旋转插值。适用于动画系统、摄像机运动和机器人学中的旋转插值。通过检查四元数的点积并进行适当的处理，可以确保插值结果的质量，使得动画和运动更加自然和流畅。