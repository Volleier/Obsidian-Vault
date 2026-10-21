**正向运动学（FK，Forward Kinematics）由各关节的旋转沿骨骼链逐级累乘，求出末端（手、脚）在哪里；反向运动学（IK，Inverse Kinematics）反过来，给定末端要到达的位置（和朝向），求各关节该怎么转。动画数据本身就是 FK：每帧存的是每根骨骼的局部变换。IK 在游戏里通常是运行时对 FK 结果的修正，UE 里有 Two Bone IK、FABRIK、CCDIK 等 AnimGraph 节点，以及 Control Rig 和 IK Rig 里的求解器。**

> 中文里常把 kinematics 译作“动力学”，严格说应为“运动学”：它只研究位置和角度，不涉及力和质量。涉及力的是 dynamics（物理动画、布娃娃）。UE 部分对照 Epic 5.8 文档。

## FK：从根到末端

骨骼链上第 $i$ 根骨骼的局部变换 $L_i$（相对父骨骼）由动画给出，它在模型空间里的变换是从根开始一路乘下来：

$$
M_i = M_{i-1}\,L_i = L_0\,L_1 \cdots L_i
$$

末端位置 $\mathbf{p} = f(\theta_1, \dots, \theta_n)$ 是关节角的确定函数，只有一个答案，计算量是链长的线性函数。FK 的特点是改一个关节会带动它下面的所有子骨骼：转肩膀，整条手臂跟着转。这对动画师做“甩手臂”“挥剑弧线”这类以关节为主导的动作很自然，但要让手恰好按在桌面某一点，就得反复调肩、肘、腕三个关节去凑。

## IK：从末端到根

IK 要解的是 $\boldsymbol\theta = f^{-1}(\mathbf{p}_{\text{target}})$。难点在于 $f^{-1}$ 通常不是函数：

- **无解。** 目标超出链的总长度，够不到。
- **多解。** 人的手臂从肩到腕有 7 个自由度，而手的位置和朝向只有 6 个约束，肘部可以绕肩腕连线转一圈，手不动。
- **奇异。** 链完全伸直时，某些方向的微小位移需要极大的关节转动。

所以 IK 求解器除了“到达目标”，还要额外决定选哪个解：用极向量（pole vector）或关节目标（Joint Target）指定肘膝朝向，用关节角度限制排除不合生理的解，用“离当前姿势最近”作为偏好。

## 求解方法

| 方法 | 思路 | 特点 | 见 |
| --- | --- | --- | --- |
| 解析法（两骨骼） | 三角形余弦定理直接算出肘/膝角度，再用极向量确定弯曲平面 | 精确、恒定开销，只适用于三关节两段链 | 本文下节 |
| CCD | 从末端往根逐个关节旋转，让末端朝向目标 | 简单，每步几何直观，容易加角度限制；链尾关节动得多 | [[ECAH-循环星标下降算法]] |
| FABRIK | 在位置空间里前后两遍拖拽关节点，保持骨长 | 收敛快、姿势自然，旋转需要从位置反推 | [[ECAA-FABRIK]] |
| 雅可比法 | 线性化 $f$，用转置、伪逆或阻尼最小二乘求关节增量 | 通用、可处理多末端和多种约束，计算量大，要处理奇异 | [[ECAI-雅可比矩阵法]] |
| 基于位置的全身 IK | 把骨骼当约束粒子系统迭代求解 | UE 的 Full Body IK 用这一类 | 本文“在 UE 里” |

## 两骨骼解析解

大臂长 $a$，小臂长 $b$，肩到目标距离 $d$（先夹到 $|a-b| \le d \le a+b$）。由余弦定理，肘部内角 $\gamma$ 和肩部相对“肩→目标”方向需要偏开的角度 $\beta$ 为

$$
\cos\gamma = \frac{a^2 + b^2 - d^2}{2ab},\qquad
\cos\beta = \frac{a^2 + d^2 - b^2}{2ad}
$$

肩、肘、目标三点确定了一个三角形，但这个三角形可以绕“肩→目标”轴任意转动，所以还需要一个极向量把它钉在某个平面上，这就是 Two Bone IK 节点 `Joint Target Location` 的作用。UE 的两骨骼解析求解实现在 AnimationCore 模块的 `AnimationCore::SolveTwoBoneIK`，蓝图里也有 `Two Bone IK` 函数可以直接调用。

## 游戏里怎么组合 FK 和 IK

DCC 软件里，IK/FK 是给动画师用的两种操控方式，常见“IK/FK 切换”和“IK/FK 匹配”功能：腿部行走用 IK 控制脚锁在地面，手臂甩动用 FK。导出到引擎后，动画数据一律是烘焙好的 FK 局部变换。

引擎运行时，IK 作为后处理修正 FK 结果，处理动画制作时无法预知的环境：

| 需求 | 做法 |
| --- | --- |
| 脚踩在斜坡、台阶上 | 射线检测地面高度，用 IK 把脚拉到接触点，同时下压骨盆 |
| 双手握枪 | 以主手为准，用 IK 把副手拉到枪上的握把 socket |
| 伸手按按钮、开门 | IK 把手拉到交互点，权重随交互进度升降 |
| 看向目标 | Look At 类节点旋转头颈 |
| 不同体型共用动画 | 重定向后用 IK 修正手脚位置 |

IK 在 AnimGraph 里通常放在最后面：先由状态机、混合、叠加得出 FK 姿势，再用 IK 修正末端。IK 节点工作在组件空间（Component Space），UE 5 会在局部空间和组件空间节点之间自动插入转换。

## 在 UE 里

| 工具 | 说明 |
| --- | --- |
| `Two Bone IK` 节点 | 三关节链的解析解，`IK Bone` 往上数两根骨骼构成链；`Effector Location`、`Joint Target Location`；可允许拉伸（`Allow Stretching`、`Start Stretch Ratio`、`Max Stretch Scale`） |
| `FABRIK` 节点 | 任意长度链，`Root Bone` 到 `Tip Bone`，`Effector Transform`、`Precision`、`Max Iterations` |
| `CCDIK` 节点 | 任意长度链，支持每个关节的旋转限制；官方标为 Experimental |
| Control Rig | 在 Rig Graph 里手写 FK 控制器、Basic IK、FABRIK、Full Body IK 等节点，可以用于运行时也可以用于在 Sequencer 里制作动画 |
| IK Rig | 资产化的 IK 配置：Solver Stack 里叠加 Body Mover、Limb IK、Full Body IK、Pole Solver、Set Transform 等求解器，AnimGraph 里用 IK Rig 节点驱动，也是 IK Retargeter 的基础 |

Control Rig 和 IK Rig 里的 Full Body IK 据 Epic 文档建立在“基于位置的 IK”框架上，支持每骨骼刚度、偏好角度、角度限制和拉伸。

原笔记里的 IK 示例是给 `AnimGraphNode_Fabrik` 赋值，那个类是编辑器里的节点包装，运行时并不存在，正确的做法是在动画实例里准备好目标位置变量，接到节点的引脚上：

```cpp
// MyAnimInstance.h（节选）
UPROPERTY(Transient, BlueprintReadOnly, Category = "IK")
FVector HandIKTarget = FVector::ZeroVector;   // 世界空间

UPROPERTY(Transient, BlueprintReadOnly, Category = "IK")
float HandIKAlpha = 0.f;
```

```cpp
// MyAnimInstance.cpp（节选）
void UMyAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
	Super::NativeUpdateAnimation(DeltaSeconds);
	if (const AMyCharacter* Character = Cast<AMyCharacter>(TryGetPawnOwner()))
	{
		// 目标由游戏逻辑决定，比如当前交互物体上的 socket
		HandIKAlpha = Character->IsInteracting() ? 1.f : 0.f;
		HandIKTarget = Character->GetInteractionHandLocation();
	}
}
```

AnimGraph 里把 `HandIKTarget` 接到 FABRIK 节点的 `Effector Transform`（`Effector Transform Space` 设为 World Space），`HandIKAlpha` 接到 `Alpha`。`AMyCharacter`、`IsInteracting`、`GetInteractionHandLocation` 是示意用的游戏代码。Alpha 最好平滑过渡，否则 IK 开关时手会瞬移。

## 容易踩的坑

**IK 放在混合之前。** IK 修正完的姿势再被后面的混合冲淡，末端就又不准了。

**没给极向量。** 两骨骼 IK 缺少 Joint Target 时，膝盖、手肘可能朝奇怪的方向翻。

**目标够不到时抖动。** 目标在可达边缘来回时，链在伸直与弯曲间跳变。给目标加距离限制，或用拉伸选项。

**只改了脚没改骨盆。** 下坡时脚被拉低，骨盆还在原高度，腿就被拉直甚至够不到地面。脚部 IK 通常要配合骨盆下压。

## 相关

[[ECAA-FABRIK]] [[ECAH-循环星标下降算法]] [[ECAI-雅可比矩阵法]] [[EAABAA-IK]] [[EAABA-基于物理的动画]] [[EAABAC-布娃娃系统]]
