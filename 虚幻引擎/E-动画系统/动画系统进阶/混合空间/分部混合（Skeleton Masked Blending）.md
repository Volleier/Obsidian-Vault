分部混合（Skeleton Masked Blending）是一种动画混合技术，用于在骨骼动画中实现局部混合。这种方法允许在角色的不同骨骼部分应用不同的动画，同时保持其他部分的动画不变。它非常适合需要对角色不同部位进行独立动画处理的场景，比如在角色的上半身做攻击动作，而下半身保持行走状态。

### Skeleton Masked Blending 的基本概念

1. **骨骼掩码（Skeleton Mask）**：定义哪些骨骼部分参与混合，哪些骨骼部分保持原样。通常使用布尔掩码或权重掩码来指定。
2. **动画剪辑**：包含全身或局部的动画剪辑。
3. **混合因子**：控制动画之间的混合比例。
4. **混合计算**：根据骨骼掩码和混合因子，将不同的动画剪辑应用到指定的骨骼部分。

### 应用场景

1. **复杂角色动画**：例如，在战斗中角色的上半身进行攻击动画，而下半身保持移动动画。
2. **互动动画**：角色与环境互动时，角色的某些部分（如手部）应用特殊动画，而其他部分保持正常动画。
3. **动画过渡**：在不同的动画状态之间进行平滑过渡，同时保持某些骨骼的动画不变。

### 示例代码

以下是一个简单的 C++ 示例代码，展示如何实现骨骼掩码混合。假设我们有两个动画剪辑，并且我们希望在一个剪辑中对角色的上半身进行混合，而下半身保持原样。

#### C++ 示例代码

```cpp
#include <iostream>
#include <vector>
#include <Eigen/Dense>
#include <Eigen/Geometry>

using namespace Eigen;

// 骨骼结构体
struct Bone {
    Vector3f position;
    Quaternionf rotation;
    Vector3f scale;
};

// 动画剪辑结构体
struct AnimationClip {
    std::vector<Bone> keyframes;
};

// 插值函数
Quaternionf slerp(const Quaternionf& q1, const Quaternionf& q2, float t) {
    return q1.slerp(t, q2);
}

Vector3f lerp(const Vector3f& v1, const Vector3f& v2, float t) {
    return (1.0f - t) * v1 + t * v2;
}

// 骨骼掩码混合
std::vector<Bone> skeletonMaskedBlend(const std::vector<Bone>& basePose, const std::vector<Bone>& blendPose, float blendFactor, const std::vector<bool>& mask) {
    std::vector<Bone> blendedPose = basePose;
    for (size_t i = 0; i < blendedPose.size(); ++i) {
        if (mask[i]) {
            blendedPose[i].position = lerp(basePose[i].position, blendPose[i].position, blendFactor);
            blendedPose[i].rotation = slerp(basePose[i].rotation, blendPose[i].rotation, blendFactor);
            blendedPose[i].scale = lerp(basePose[i].scale, blendPose[i].scale, blendFactor);
        }
    }
    return blendedPose;
}

int main() {
    // 定义两个动画剪辑
    AnimationClip walkClip = {/* keyframes for walking */};
    AnimationClip attackClip = {/* keyframes for attacking */};
    
    // 基础姿势和混合姿势
    std::vector<Bone> basePose = walkClip.keyframes;
    std::vector<Bone> blendPose = attackClip.keyframes;
    
    // 混合因子（例如，0.5表示50%混合）
    float blendFactor = 0.5f;
    
    // 定义骨骼掩码（假设上半身骨骼需要混合）
    std::vector<bool> mask = {true, true, false, false}; // 根据具体骨骼数量设置
    
    // 计算骨骼掩码混合后的姿势
    std::vector<Bone> blendedPose = skeletonMaskedBlend(basePose, blendPose, blendFactor, mask);
    
    // 输出混合姿势
    for (const auto& bone : blendedPose) {
        std::cout << "Position: " << bone.position.transpose() << std::endl;
        std::cout << "Rotation: " << bone.rotation.coeffs().transpose() << std::endl;
        std::cout << "Scale: " << bone.scale.transpose() << std::endl;
    }
    
    return 0;
}
```

### 示例场景说明

1. **定义动画剪辑**：在示例中，我们定义了两个动画剪辑：行走和攻击。
2. **设置基础姿势和混合姿势**：基础姿势设为行走姿势，混合姿势设为攻击姿势。
3. **计算混合因子**：设置混合因子为 `0.5`，表示 50% 的混合。
4. **定义骨骼掩码**：指定哪些骨骼需要进行混合，例如，上半身骨骼参与混合，下半身骨骼保持原样。
5. **计算骨骼掩码混合后的姿势**：使用 `skeletonMaskedBlend` 函数计算最终的混合姿势。

### 结论

Skeleton Masked Blending 允许在骨骼动画中实现复杂的局部混合效果。通过定义骨骼掩码，可以控制哪些骨骼部分参与混合，哪些骨骼部分保持不变。这种方法特别适合需要对角色的不同部分应用不同动画的场景，提升了动画系统的灵活性和表现力。