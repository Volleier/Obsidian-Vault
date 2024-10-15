姿势的插值（Pose Interpolation）是动画系统中的关键技术，用于在关键帧之间平滑过渡角色姿势，从而实现连续的动画。常见的插值方法有线性插值（LERP）、球面线性插值（SLERP）和球面非线性插值（NLERP）。

### 基本概念

1. 关键帧（Keyframe）：定义在特定时间点上的角色姿势，包括位置、旋转和缩放信息。
2. 插值：在两个关键帧之间计算中间姿势，使角色动作平滑过渡。

### 插值方法

#### 线性插值（LERP）

线性插值是一种简单的插值方法，适用于位置和缩放。它根据时间参数 t 在两个关键帧之间计算中间值。

公式：
\[ \text{LERP}(A, B, t) = (1 - t) \cdot A + t \cdot B \]

#### 球面线性插值（SLERP）

球面线性插值适用于旋转插值，通过四元数实现。在单位四元数的球面上进行线性插值，保证插值结果的旋转是平滑和一致的。

公式：
\[ \text{SLERP}(q_1, q_2, t) = \frac{\sin((1 - t) \cdot \Omega) \cdot q_1 + \sin(t \cdot \Omega) \cdot q_2}{\sin(\Omega)} \]
其中，\(\Omega\) 是四元数 \(q_1\) 和 \(q_2\) 之间的夹角。

#### 球面非线性插值（NLERP）

球面非线性插值也是针对旋转的插值方法，比 SLERP 计算更简单，但在一些情况下可能会产生非均匀速度。

公式与 LERP 类似，但应用于单位四元数：
\[ \text{NLERP}(q_1, q_2, t) = \frac{(1 - t) \cdot q_1 + t \cdot q_2}{||(1 - t) \cdot q_1 + t \cdot q_2||} \]

### 示例代码

以下示例代码展示了如何在 C++ 和 GLSL 中实现姿势插值。

#### C++ 插值函数

```cpp
#include <iostream>
#include <Eigen/Dense>
#include <Eigen/Geometry> // 用于四元数
#include <vector>

using namespace Eigen;

// 线性插值
Vector3f lerp(const Vector3f& A, const Vector3f& B, float t) {
    return (1.0f - t) * A + t * B;
}

// 球面线性插值 (SLERP)
Quaternionf slerp(const Quaternionf& q1, const Quaternionf& q2, float t) {
    return q1.slerp(t, q2);
}

// 球面非线性插值 (NLERP)
Quaternionf nlerp(const Quaternionf& q1, const Quaternionf& q2, float t) {
    Quaternionf result = (1.0f - t) * q1.coeffs() + t * q2.coeffs();
    return result.normalized();
}

int main() {
    Vector3f posA(0.0f, 0.0f, 0.0f);
    Vector3f posB(10.0f, 10.0f, 10.0f);
    float t = 0.5f;

    Vector3f interpolatedPos = lerp(posA, posB, t);
    std::cout << "Interpolated Position: " << interpolatedPos.transpose() << std::endl;

    Quaternionf rotA = Quaternionf::Identity();
    Quaternionf rotB = AngleAxisf(M_PI / 2, Vector3f::UnitY());

    Quaternionf interpolatedRotSlerp = slerp(rotA, rotB, t);
    std::cout << "SLERP Interpolated Rotation: " << interpolatedRotSlerp.coeffs().transpose() << std::endl;

    Quaternionf interpolatedRotNlerp = nlerp(rotA, rotB, t);
    std::cout << "NLERP Interpolated Rotation: " << interpolatedRotNlerp.coeffs().transpose() << std::endl;

    return 0;
}
```

#### 顶点着色器代码（GLSL）

```glsl
#version 330 core

layout(location = 0) in vec3 a_Position;
layout(location = 1) in vec3 a_Normal;
layout(location = 2) in vec4 a_BoneWeights;
layout(location = 3) in ivec4 a_BoneIndices;

uniform mat4 u_SkinningPalette[100]; // 假设最多有 100 个关节
uniform float u_InterpolationFactor; // 插值因子

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

### 插值方法的应用

1. 线性插值 (LERP)：适用于位置和缩放插值，计算简单，但在旋转插值中可能导致不平滑。
2. 球面线性插值 (SLERP)：适用于旋转插值，保证旋转的平滑和一致，但计算较复杂。
3. 球面非线性插值 (NLERP)：适用于旋转插值，计算简单，但在某些情况下可能会产生非均匀速度。

### 优点

- 平滑动画：通过插值在关键帧之间平滑过渡角色姿势，实现连续动画。
- 灵活性强：多种插值方法适用于不同类型的动画需求。

### 结论

姿势的插值技术通过在关键帧之间计算中间姿势，使角色动画更加自然和流畅。线性插值、球面线性插值和球面非线性插值是常见的插值方法，各有优缺点，适用于不同的应用场景。通过合理选择插值方法，可以实现高质量的角色动画效果。