动画状态机（Animation State Machine）是一种用于管理和控制角色动画的工具。它通过定义不同的动画状态和状态之间的转换规则，帮助设计师和开发者在游戏或应用中实现复杂的动画逻辑。动画状态机使得角色能够在不同的动画状态之间平滑过渡，从而表现出自然的动作和行为。

### 动画状态机的基本概念

1. 状态（State）：动画状态机中的每个状态对应一个特定的动画剪辑。例如，站立、行走、跑步、跳跃等。
2. 过渡（Transition）：定义从一个状态切换到另一个状态的条件和规则。过渡可以是基于时间、输入事件或其他条件。
3. 参数（Parameter）：用于控制状态之间的过渡，例如速度、方向、动作触发器等。这些参数可以是布尔值、整数、浮点数等。
4. 混合树（Blend Tree）：在状态之间过渡时的动画混合系统，通常用于在多个状态之间进行平滑过渡，例如从走路到跑步的平滑过渡。

### 动画状态机的工作流程

1. 定义状态：创建并配置不同的动画状态，每个状态对应一个动画剪辑。
2. 设置过渡：定义状态之间的过渡规则，设置条件和触发事件。
3. 配置参数：设定控制状态过渡的参数，并在游戏逻辑中更新这些参数。
4. 运行时管理：在游戏运行时，根据当前状态和参数，自动在状态机中进行状态切换和动画播放。

### 应用场景

1. 角色动画控制：用于控制角色的不同动作状态，如行走、跑步、跳跃等。
2. 复杂行为表现：管理具有多个行为和动作的角色，例如在战斗中切换不同的攻击动作和防御动作。
3. 动画过渡：处理动画之间的平滑过渡，避免动画切换时的突兀感。

### 示例代码

以下是一个简化的 C++ 示例代码，展示如何实现一个基本的动画状态机。假设我们有两个动画状态：站立和行走。

#### C++ 示例代码

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>

using namespace std;

// 动画状态机状态
enum class AnimationState {
    Idle,
    Walking,
};

// 动画状态机类
class AnimationStateMachine {
public:
    AnimationStateMachine() : currentState(AnimationState::Idle) {}
    
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
            cout << "Transitioned to state: " << StateToString(currentState) << endl;
        }
    }
    
    string StateToString(AnimationState state) {
        switch (state) {
            case AnimationState::Idle: return "Idle";
            case AnimationState::Walking: return "Walking";
            default: return "Unknown";
        }
    }
    
private:
    AnimationState currentState;
    unordered_map<AnimationState, unordered_map<string, AnimationState>> transitions;
    string parameter;
};

int main() {
    AnimationStateMachine asm;
    
    // 添加状态过渡
    asm.AddTransition(AnimationState::Idle, AnimationState::Walking, "StartWalking");
    asm.AddTransition(AnimationState::Walking, AnimationState::Idle, "StopWalking");
    
    // 设置参数并更新状态
    asm.SetParameter("StartWalking");
    asm.SetParameter("StopWalking");
    
    return 0;
}
```

### 示例场景说明

1. 定义状态：状态机有两个状态：Idle（站立）和 Walking（行走）。
2. 添加过渡：设置从 Idle 到 Walking 的过渡条件为 “StartWalking”，从 Walking 到 Idle 的过渡条件为 “StopWalking”。
3. 设置参数：通过设置参数并调用 `UpdateState` 函数，状态机根据参数值进行状态转换。

### 结论

动画状态机是一种强大的工具，帮助开发者管理和控制角色的动画状态。通过定义状态、过渡和参数，可以实现复杂的动画逻辑，使得角色在不同动作之间平滑过渡。无论是简单的状态转换还是复杂的动画行为，动画状态机都能提供灵活且可扩展的解决方案。