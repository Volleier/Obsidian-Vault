在角色动画系统中，骨骼（Skeleton）的数据结构需要高效地存储在内存中，以便在动画运行时快速访问和更新。一个典型的骨骼数据结构需要包含关节的层次关系、关节的初始姿势（绑定姿势）、关节的当前姿势、关节的变换矩阵等信息。

### 骨骼数据结构

以下是一个用于在内存中存储骨骼的常见数据结构示例：

1. 关节数据结构：存储每个关节的基本信息。
2. 骨骼数据结构：存储所有关节，并维护关节的层次关系。

### 关节数据结构

关节数据结构通常包括：
- 关节名称：方便调试和识别。
- 父关节索引：用于维护层次关系。
- 绑定姿势变换矩阵：关节在绑定姿势下的变换矩阵。
- 当前变换矩阵：关节在当前动画姿势下的变换矩阵。

### 骨骼数据结构

骨骼数据结构通常包括：
- 关节列表：存储所有关节的信息。
- 层次关系：通过父关节索引维护关节的层次关系。

### 示例代码

以下是一个 C++ 示例，展示如何在内存中存储骨骼数据结构：

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <Eigen/Dense>

using Matrix4 = Eigen::Matrix4f;
using Vector3 = Eigen::Vector3f;

// 关节数据结构
struct Joint {
    std::string name;
    int parentIndex;         // 父关节索引
    Matrix4 bindPoseMatrix;  // 绑定姿势矩阵
    Matrix4 currentMatrix;   // 当前姿势矩阵
};

// 骨骼数据结构
class Skeleton {
public:
    Skeleton(int jointCount) : joints(jointCount) {}

    // 设置关节信息
    void setJoint(int index, const std::string& name, int parentIndex, const Matrix4& bindPoseMatrix) {
        joints[index].name = name;
        joints[index].parentIndex = parentIndex;
        joints[index].bindPoseMatrix = bindPoseMatrix;
        joints[index].currentMatrix = bindPoseMatrix; // 初始时，当前姿势与绑定姿势相同
    }

    // 更新关节的当前姿势矩阵
    void updateJointMatrix(int index, const Matrix4& newMatrix) {
        joints[index].currentMatrix = newMatrix;
    }

    // 递归更新全局变换矩阵
    void updateGlobalMatrices() {
        updateGlobalMatrix(0, Matrix4::Identity());
    }

    // 输出骨骼信息
    void printSkeletonInfo() const {
        for (const auto& joint : joints) {
            std::cout << "Joint: " << joint.name << ", Parent: " << joint.parentIndex << std::endl;
        }
    }

private:
    std::vector<Joint> joints;

    // 递归更新关节的全局变换矩阵
    void updateGlobalMatrix(int index, const Matrix4& parentMatrix) {
        Matrix4 globalMatrix = parentMatrix * joints[index].currentMatrix;
        // 如果有子关节，递归更新子关节的全局变换矩阵
        for (size_t i = 0; i < joints.size(); ++i) {
            if (joints[i].parentIndex == index) {
                updateGlobalMatrix(i, globalMatrix);
            }
        }
    }
};

int main() {
    // 创建一个骨骼，包含3个关节
    Skeleton skeleton(3);

    // 设置关节信息
    skeleton.setJoint(0, "root", -1, Matrix4::Identity()); // 根关节，没有父关节
    skeleton.setJoint(1, "spine", 0, Eigen::Translation3f(0.0f, 1.0f, 0.0f).matrix());
    skeleton.setJoint(2, "head", 1, Eigen::Translation3f(0.0f, 1.0f, 0.0f).matrix());

    // 输出骨骼信息
    skeleton.printSkeletonInfo();

    // 更新骨骼的全局变换矩阵
    skeleton.updateGlobalMatrices();

    return 0;
}
```

### 关键点解释

1. 关节名称和父关节索引：`name` 和 `parentIndex` 用于维护关节的层次结构。根关节的 `parentIndex` 通常设置为 `-1`。
2. 绑定姿势矩阵和当前姿势矩阵：`bindPoseMatrix` 和 `currentMatrix` 分别存储关节在绑定姿势和当前姿势下的变换矩阵。
3. 更新全局变换矩阵：`updateGlobalMatrices` 函数递归地更新每个关节的全局变换矩阵，以确保子关节的变换矩阵正确地应用了父关节的变换。

### 结论

通过将骨骼数据结构存储在内存中，可以高效地管理和更新角色的动画姿势。关节的层次关系和变换矩阵是骨骼动画系统的核心，通过递归更新全局变换矩阵，可以确保角色动画的准确性和一致性。