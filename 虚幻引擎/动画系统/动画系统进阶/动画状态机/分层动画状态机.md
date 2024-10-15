分层动画状态机（Layered Animation State Machine，简称 Layered ASM）是一种动画系统设计技术，用于在复杂的动画系统中管理和混合多个动画状态机层。分层状态机允许在不同的层上独立管理动画状态，并在这些层之间进行协调，从而实现更复杂的动画表现。

### 分层动画状态机的基本概念

1. 层（Layer）：在分层状态机中，每个层可以包含独立的动画状态机。每个层的动画状态机可以控制角色的不同方面，例如上半身和下半身的动画。

2. 状态（State）：每个层中的状态机具有自己的动画状态，例如行走、跑步、攻击等。

3. 过渡（Transition）：定义在同一层或不同层之间状态的切换条件和规则。每个层的过渡可以是独立配置的，也可以通过协调来实现更复杂的动画效果。

4. 权重（Weight）：每个层可以有不同的权重值，控制各个层在最终动画中的影响力。权重值可以动态调整，以平滑混合不同层的动画。

5. 混合（Blend）：在不同层之间进行混合，使得最终的动画输出是所有层的动画的加权组合。混合可以根据层的权重和状态进行计算。

### 分层状态机的工作流程

1. 定义层：创建多个层，每个层包含一个独立的动画状态机。每个层可以控制角色的不同部分或不同类型的动作。
   
2. 配置状态：在每个层中定义动画状态和过渡规则。例如，一个层可以管理角色的上半身动画，另一个层可以管理下半身动画。

3. 设置过渡：配置每个层内的状态过渡规则，并定义如何在不同层之间协调这些过渡。

4. 调整权重：根据需要调整各个层的权重，以决定每个层在最终动画中的贡献度。

5. 计算混合：在运行时，根据当前状态和权重计算最终的动画结果，并应用到角色模型上。

### 应用场景

1. 复杂角色动画：例如，一个角色在行走时可以同时进行手部攻击，使用分层状态机可以将行走和攻击动画分别管理。
   
2. 动作组合：处理多个动作组合的情况，例如角色在行走时抬起手臂，或在跳跃时执行旋转动作。

3. 动画同步：在多个层之间协调动画，使得角色的不同部分能够同步运动。

### 示例代码

以下是一个简化的 C++ 示例代码，展示了如何实现分层动画状态机的基本结构。假设我们有两个层：一个用于控制角色的上半身动画，另一个用于控制下半身动画。

#### C++ 示例代码

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>

using namespace std;

// 动画状态
enum class AnimationState {
    Idle,
    Walking,
    Running,
    Attacking,
};

// 动画层
class AnimationLayer {
public:
    AnimationLayer() : currentState(AnimationState::Idle), weight(1.0f) {}
    
    void AddTransition(AnimationState from, AnimationState to, const string& trigger) {
        transitions[from][trigger] = to;
    }
    
    void SetParameter(const string& parameter) {
        this->parameter = parameter;
        UpdateState();
    }
    
    void UpdateState() {
        if (transitions[currentState].find(parameter) != transitions[currentState].end()) {
            currentState = transitions[currentState][parameter];
            cout << "Layer State transitioned to: " << StateToString(currentState) << endl;
        }
    }
    
    void SetWeight(float w) {
        weight = w;
    }
    
    float GetWeight() const {
        return weight;
    }
    
    string StateToString(AnimationState state) {
        switch (state) {
            case AnimationState::Idle: return "Idle";
            case AnimationState::Walking: return "Walking";
            case AnimationState::Running: return "Running";
            case AnimationState::Attacking: return "Attacking";
            default: return "Unknown";
        }
    }

    AnimationState GetCurrentState() const {
        return currentState;
    }
    
private:
    AnimationState currentState;
    float weight;
    unordered_map<AnimationState, unordered_map<string, AnimationState>> transitions;
    string parameter;
};

// 分层动画状态机
class LayeredAnimationStateMachine {
public:
    void AddLayer(AnimationLayer layer) {
        layers.push_back(layer);
    }
    
    void SetParameter(int layerIndex, const string& parameter) {
        if (layerIndex < layers.size()) {
            layers[layerIndex].SetParameter(parameter);
        }
    }
    
    void SetLayerWeight(int layerIndex, float weight) {
        if (layerIndex < layers.size()) {
            layers[layerIndex].SetWeight(weight);
        }
    }
    
    void Update() {
        // 在这里可以根据各层的权重和状态进行动画混合
        cout << "Updating Layered Animation State Machine:" << endl;
        for (int i = 0; i < layers.size(); ++i) {
            cout << "Layer " << i << " State: " << layers[i].StateToString(layers[i].GetCurrentState()) << ", Weight: " << layers[i].GetWeight() << endl;
        }
    }

private:
    vector<AnimationLayer> layers;
};

int main() {
    LayeredAnimationStateMachine layeredASM;
    
    // 创建两个动画层
    AnimationLayer upperBodyLayer;
    AnimationLayer lowerBodyLayer;
    
    // 添加过渡
    upperBodyLayer.AddTransition(AnimationState::Idle, AnimationState::Attacking, "Attack");
    lowerBodyLayer.AddTransition(AnimationState::Idle, AnimationState::Walking, "Walk");
    
    // 添加层到状态机
    layeredASM.AddLayer(upperBodyLayer);
    layeredASM.AddLayer(lowerBodyLayer);
    
    // 设置参数并更新状态
    layeredASM.SetParameter(0, "Attack"); // 设置上半身层的参数
    layeredASM.SetParameter(1, "Walk");   // 设置下半身层的参数
    
    // 设置权重
    layeredASM.SetLayerWeight(0, 0.8f); // 上半身层权重
    layeredASM.SetLayerWeight(1, 1.0f); // 下半身层权重
    
    // 更新状态机
    layeredASM.Update();
    
    return 0;
}
```

### 示例场景说明

1. 定义动画层：创建了两个动画层，分别控制角色的上半身和下半身动画。
   
2. 添加过渡：设置上半身层和下半身层的状态过渡规则。
   
3. 设置参数和权重：在运行时设置参数，并调整每个层的权重。

4. 更新状态机：根据当前状态和权重输出各层的状态信息。

### 结论

分层动画状态机是处理复杂动画需求的强大工具。通过将动画状态机分为多个层，并在这些层之间进行协调和混合，能够实现更复杂、更自然的动画效果。这种技术在需要同时处理多个动作或角色部分时尤为重要，使得动画系统更加灵活和可控。