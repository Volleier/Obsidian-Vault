叠加混合节点（Additive Blend Node）是动画系统中一种非常有用的工具，特别适合在基础动画的基础上叠加额外的动画效果。通过这种方式，可以在不破坏基础动画的情况下，添加诸如微小动作、情感表现等细节。下面详细介绍 Additive Blend Node 的概念、使用方法以及在虚幻引擎中的具体操作。

### 叠加混合节点的基本概念

- 叠加动画
  Additive Blend Node 允许你在一个基础动画上叠加另一个动画效果。叠加动画通常用于添加细微的动作，如呼吸、表情变化等，不会改变基础动画的主要运动轨迹。

- 基础动画和叠加动画
  基础动画是角色的主要动作，如走路、跑步等。叠加动画是在基础动画上添加的附加效果，例如角色在走路时增加呼吸的动作。

- 权重值
  Additive Blend Node 使用权重值来控制叠加动画的影响程度。权重值为0时，只显示基础动画；权重值为1时，叠加动画完全应用在基础动画上；权重值在0到1之间时，显示的是两者的混合效果。

### 使用叠加混合节点的步骤

1. 准备动画资源
   确保已经有一个基础动画和一个叠加动画。叠加动画通常需要以基础动画的静止姿势为参考进行制作，以保证叠加效果的正确性。

2. 创建动画蓝图（Animation Blueprint）
   打开或创建一个角色的动画蓝图，并进入 AnimGraph 编辑界面。

3. 添加 Additive Blend Node
   在 AnimGraph 中，搜索并添加 `Layered blend per bone` 或 `Blend poses by bool`、`Blend poses by float` 等节点，这些节点支持叠加动画的功能。

4. 设置输入动画
   将基础动画连接到 Additive Blend Node 的第一个输入端口，将叠加动画连接到第二个输入端口。

5. 配置权重值
   定义一个浮点类型的参数（如 `AdditiveWeight`），并将其连接到 Additive Blend Node 的权重输入端口。通过蓝图或代码动态调整这个权重值。

6. 测试和调试
   在编辑器中运行和测试动画叠加效果，调整权重值和输入动画，确保实现预期的叠加效果。

### 在虚幻引擎中的具体操作

以下是在虚幻引擎中使用 Additive Blend Node 的具体操作步骤：

1. 创建基础动画和叠加动画
   在 Content Browser 中，确保已有基础动画和叠加动画。

2. 创建动画蓝图
   在 Content Browser 中右键点击角色骨骼网格（Skeletal Mesh），选择 `Create > Animation Blueprint`，然后打开新创建的动画蓝图。

3. 添加 Layered Blend Per Bone 节点
   在 AnimGraph 中，搜索并添加 `Layered Blend Per Bone` 节点。此节点可以按骨骼层次结构进行混合。

4. 设置输入动画
   将基础动画连接到 `Layered Blend Per Bone` 节点的基础输入端口（Base Pose），将叠加动画连接到叠加输入端口（Blend Poses[0]）。

5. 配置骨骼层次结构
   在 `Layered Blend Per Bone` 节点的设置中，配置需要进行叠加的骨骼层次。例如，可以只在上半身叠加动画，而下半身保持基础动画。

6. 配置权重值
   定义一个浮点类型的参数（如 `AdditiveWeight`），并在 `Layered Blend Per Bone` 节点的权重设置中使用该参数。可以通过蓝图事件或代码动态调整这个权重值。

### 示例代码

以下是一个简单的蓝图示例，展示如何在角色的动画蓝图中实现 Additive Blend 动画：

```cpp
// 在角色类中定义权重值
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Animation")
float AdditiveWeight;

// 在Tick函数中动态调整权重值
void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 根据游戏状态调整AdditiveWeight
    // 例如：当角色疲劳时增加呼吸动画权重
    AdditiveWeight = FMath::Clamp(GetFatigueLevel() / MaxFatigue, 0.0f, 1.0f);
}

// 在动画蓝图中设置Additive Blend Node
AnimGraph.AdditiveWeight = AdditiveWeight;
```

### 应用场景

- 呼吸动作
  在角色的基础运动（如行走、跑步）上叠加呼吸动作，使角色更具生命力。

- 表情变化
  在基础动画上叠加面部表情变化，实现角色在不同情境下的情感表现。

- 受伤状态
  通过叠加受伤动作，使角色在基础运动中表现出受伤的状态，例如跛行。

通过 Additive Blend Node，可以显著提高角色动画的细腻程度和表现力，使游戏角色更具真实感和生命力。如果你需要更具体的示例或进一步的解释，请告诉我。