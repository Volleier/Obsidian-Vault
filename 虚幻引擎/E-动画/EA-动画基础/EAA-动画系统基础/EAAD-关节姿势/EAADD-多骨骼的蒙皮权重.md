在多骨骼的蒙皮权重（Skin Weights for Multiple Bones）技术中，每个顶点可以受到多个骨骼（关节）的影响，每个骨骼对顶点的影响程度由一个权重值表示。这个权重值通常在 [0, 1] 范围内，所有权重的总和为 1。通过这种方式，可以实现更加平滑和自然的变形效果。

### 基本概念

1. 顶点权重（Skin Weights）：每个顶点与多个骨骼关联，每个关联都有一个权重值。权重值表示该骨骼对顶点的影响程度。
2. 骨骼索引（Bone Indices）：每个顶点存储多个骨骼的索引，用于在蒙皮矩阵调色板中查找对应的变换矩阵。
3. 顶点变换：在顶点着色器中，根据顶点的骨骼索引和权重，从蒙皮矩阵调色板中取出对应的变换矩阵，进行线性混合变形。

### 示例代码

以下是一个示例，展示如何在 C++ 中处理多骨骼的蒙皮权重，并在 GLSL 中实现顶点变换：

#### CPU 端代码（C++）

```cpp
#include <iostream>
#include <vector>
#include <Eigen/Dense>
#include <GL/glew.h>
#include <GLFW/glfw3.h>

using Matrix4 = Eigen::Matrix4f;
using Vector4 = Eigen::Vector4f;

// 关节数据结构
struct Joint {
    Matrix4 bindPoseInverse;  // 绑定姿势下的逆变换矩阵
    Matrix4 currentTransform; // 当前动画姿势的变换矩阵
};

// 顶点数据结构
struct Vertex {
    Eigen::Vector3f position;
    Eigen::Vector4i boneIndices; // 骨骼索引
    Eigen::Vector4f boneWeights; // 骨骼权重
};

// 计算蒙皮矩阵
void computeSkinningMatrices(const std::vector<Joint>& joints, std::vector<Matrix4>& skinningMatrices) {
    for (size_t i = 0; i < joints.size(); ++i) {
        skinningMatrices[i] = joints[i].currentTransform * joints[i].bindPoseInverse;
    }
}

// 更新蒙皮矩阵调色板
void updateSkinningPalette(GLuint shaderProgram, const std::vector<Matrix4>& skinningMatrices) {
    GLuint paletteLocation = glGetUniformLocation(shaderProgram, "u_SkinningPalette");
    glUniformMatrix4fv(paletteLocation, skinningMatrices.size(), GL_FALSE, skinningMatrices[0].data());
}

int main() {
    // 初始化 GLFW 和 GLEW 代码省略...

    // 假设我们有 3 个关节
    std::vector<Joint> joints(3);
    joints[0].bindPoseInverse = Matrix4::Identity().inverse();
    joints[0].currentTransform = Matrix4::Identity();

    joints[1].bindPoseInverse = Eigen::Translation3f(0.0f, 1.0f, 0.0f).matrix().inverse();
    joints[1].currentTransform = Eigen::Translation3f(0.0f, 1.0f, 0.0f).matrix();

    joints[2].bindPoseInverse = Eigen::Translation3f(0.0f, 2.0f, 0.0f).matrix().inverse();
    joints[2].currentTransform = Eigen::Translation3f(0.0f, 2.0f, 0.0f).matrix();

    // 计算蒙皮矩阵
    std::vector<Matrix4> skinningMatrices(joints.size());
    computeSkinningMatrices(joints, skinningMatrices);

    // 更新着色器中的蒙皮矩阵调色板
    GLuint shaderProgram = /* 创建并编译着色器程序 */;
    updateSkinningPalette(shaderProgram, skinningMatrices);

    // 渲染循环代码省略...

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

void main() {
    mat4 skinningMatrix = a_BoneWeights.x * u_SkinningPalette[a_BoneIndices.x] +
                          a_BoneWeights.y * u_SkinningPalette[a_BoneIndices.y] +
                          a_BoneWeights.z * u_SkinningPalette[a_BoneIndices.z] +
                          a_BoneWeights.w * u_SkinningPalette[a_BoneIndices.w];

    vec4 skinnedPosition = skinningMatrix * vec4(a_Position, 1.0);

    gl_Position = /* 投影矩阵 * 视图矩阵 * */ skinnedPosition;
}
```

### 关键点解释

1. 顶点权重和骨骼索引：顶点数据结构包含 `boneIndices` 和 `boneWeights`，分别表示影响该顶点的骨骼索引和对应的权重。
2. 蒙皮矩阵计算：在 CPU 端，根据关节的当前变换矩阵和绑定姿势的逆矩阵计算蒙皮矩阵，并将这些矩阵传递到 GPU。
3. 着色器中的线性混合变形：在顶点着色器中，使用顶点的骨骼索引和权重，从蒙皮矩阵调色板中取出对应的矩阵进行线性混合变形，计算顶点的最终位置。

### 优点

- 自然变形：顶点受到多个骨骼的影响，可以实现更加平滑和自然的变形效果。
- 高效计算：通过在 GPU 上进行顶点变换，可以大幅提高计算效率，适用于复杂的角色动画。

### 结论

多骨骼的蒙皮权重技术通过对顶点进行线性混合变形，实现了高质量的角色动画效果。通过在顶点着色器中使用蒙皮矩阵调色板，可以高效地处理复杂的骨骼动画系统，使角色动画更加自然和流畅。