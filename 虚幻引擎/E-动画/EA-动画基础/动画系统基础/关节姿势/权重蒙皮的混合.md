在权重蒙皮中，混合指的是将多个骨骼变换矩阵按各自的权重进行线性插值，以计算出顶点的最终变换。这个过程通常在顶点着色器中实现，通过使用每个顶点的骨骼索引和权重，从蒙皮矩阵调色板中取出对应的骨骼变换矩阵进行混合。

### 权重蒙皮的混合步骤

1. 获取顶点的骨骼索引和权重：每个顶点存储了影响它的骨骼索引和对应的权重。
2. 从蒙皮矩阵调色板中取出骨骼变换矩阵：根据顶点的骨骼索引，从调色板中取出对应的骨骼变换矩阵。
3. 按权重进行线性混合：将取出的骨骼变换矩阵按权重进行线性混合，得到顶点的最终变换矩阵。
4. 应用最终变换矩阵：将最终变换矩阵应用到顶点上，计算出顶点的最终位置。

### 示例代码

以下是一个完整的例子，包括 C++ 代码和 GLSL 着色器代码，展示如何实现权重蒙皮的混合。

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
    // 初始化最终变换矩阵
    mat4 skinningMatrix = mat4(0.0);

    // 按权重混合骨骼变换矩阵
    skinningMatrix += a_BoneWeights.x * u_SkinningPalette[a_BoneIndices.x];
    skinningMatrix += a_BoneWeights.y * u_SkinningPalette[a_BoneIndices.y];
    skinningMatrix += a_BoneWeights.z * u_SkinningPalette[a_BoneIndices.z];
    skinningMatrix += a_BoneWeights.w * u_SkinningPalette[a_BoneIndices.w];

    // 应用最终变换矩阵
    vec4 skinnedPosition = skinningMatrix * vec4(a_Position, 1.0);

    // 设置顶点的最终位置
    gl_Position = /* 投影矩阵 * 视图矩阵 * */ skinnedPosition;
}
```

### 关键点解释

1. 顶点权重和骨骼索引：在顶点数据结构中，每个顶点都有 `boneIndices` 和 `boneWeights`，分别表示影响顶点的骨骼索引和对应的权重。
2. 蒙皮矩阵计算：在 CPU 端计算每个骨骼的蒙皮矩阵（当前变换矩阵乘以绑定姿势的逆矩阵），并将这些矩阵传递到 GPU。
3. 混合变换矩阵：在顶点着色器中，使用顶点的骨骼索引和权重，从蒙皮矩阵调色板中取出对应的骨骼变换矩阵，按权重进行线性混合，计算出顶点的最终变换矩阵。
4. 应用最终变换矩阵：将最终变换矩阵应用到顶点上，计算出顶点的最终位置。

### 优点

- 自然变形：通过按权重混合多个骨骼的变换矩阵，可以实现更加平滑和自然的变形效果。
- 高效计算：在 GPU 上进行顶点变换，可以大幅提高计算效率，适用于复杂的角色动画。

### 结论

权重蒙皮的混合技术通过对顶点进行线性混合变形，实现了高质量的角色动画效果。通过在顶点着色器中使用蒙皮矩阵调色板，可以高效地处理复杂的骨骼动画系统，使角色动画更加自然和流畅。