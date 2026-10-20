**GAS 的客户端预测让本地控制端在等待服务器确认前先执行技能副作用（激活、部分 GE、Cue、Montage），再用 FPredictionKey 把这些副作用绑成可确认或可回滚的一组。真正的数值权威仍在服务器；复制属性、拒绝激活、以及不可预测的路径（Execution、周期 Tick、GE 移除）决定了你能「先演」到什么程度。**

> 版本说明：预测窗口、`FPredictionKey`、`FScopedPredictionWindow`、以及 Server 成功/失败客户端回调的行为，对照引擎 `GameplayPrediction.h` 注释传统、Epic ASC 激活路径，以及 GAS 社区对 5.x 的整理。具体 RPC 名以你引擎 `AbilitySystemComponent` 源码为准；下文对 Confirm/Reject 只作流程级描述。

## 为什么技能要预测

100ms RTT 下若每次开火都等服务器回包再播 Montage，手感会像「按下后顿一下」。`NetExecutionPolicy = LocalPredicted` 时，本地先跑 `ActivateAbility`，服务器并行校验；同意则权威结果覆盖预测，拒绝则回滚。代价是两套世界短暂并存，设计必须接受：有些事不能预测，预测错了会闪一下。

这和 CharacterMovement 的预测是两条线：移动由 CMC 管，GAS 管技能与效果侧。ASC 的 OwnerActor / AvatarActor 决定复制归属——Owner 常在 PlayerState、Avatar 在 Pawn 时，初始化错误会直接导致「本地不能预测激活」。

## PredictionKey 与预测窗口

`FPredictionKey` 是客户端生成的标识，随激活请求送到服务器，用来对齐「这次预测产生的所有副作用」。键的复制是非对称的：发起预测的客户端需要它做对账，其他客户端通常看不到有效键。

窗口在一次 `ActivateAbility` 调用栈内打开（引擎用 `FScopedPredictionWindow` RAII）。**栈返回后窗口关闭**：Timer、Delay、下一帧再 Apply 的 GE，默认不在本次预测里。需要后续再预测时，要显式开新的 scoped window / 依赖键（进阶用法，见 `GameplayPrediction.h`）。

| 内容 | 通常能否预测 | 备注 |
| --- | --- | --- |
| 技能激活本身 | 是 | LocalPredicted 核心路径 |
| Modifier 型 GE（含 Cost/Cooldown Tag） | 是 | 属性用 delta 方式对账 |
| GameplayCue | 是 | 确认后避免播两次 |
| Montage（经 ASC/技能接口） | 是 | 仍受 Anim 复制限制 |
| Execution Calculation | 否 | 只在服务器算 |
| GE Periodic 跳字 | 否 | |
| GE 移除 | 否 | 已知限制；有时用「反向 GE」近似 |
| 生成 Actor | 部分 | 需 `UAbilityTask_SpawnActor` 等与键配合 |

属性预测不是「客户端把 Health 写成 90 再覆盖」，而是预测相对服务器权威值的修饰；确认后复制下来的属性与预测 delta 对齐，预测侧效果再卸掉。

## 确认、拒绝与 Owner/Avatar

流程概览（名字以源码为准，逻辑稳定）：

1. 本地 `TryActivateAbility` → 生成 PredictionKey → 发服务器尝试激活。
2. 不等待，本地进入预测窗口执行 `ActivateAbility`（扣 Cost、上 Buff、播 Cue/Montage）。
3. 服务器跑同一套 CanActivate / Activate；成功则通知客户端成功并对齐键；失败则通知失败。
4. 失败：杀掉本地技能实例，回滚带该键的预测 GE/Cue/Montage。
5. 成功：短暂可能同时存在「预测份」与「复制份」活跃效果；当复制侧的预测键追上后，预测份移除，权威份留下。

`InitAbilityActorInfo(Owner, Avatar)` 必须在服务器与本地控制端都就绪。Owner 决定 ASC 挂谁、复制给谁；Avatar 是技能取 Mesh/位置的 Actor。ASC 在 PlayerState、Pawn 重生换身体时，只更新 Avatar、保留 Owner 上的属性与技能，是 MOBA/英雄射击常见布局。`Mixed` 复制模式下，完整 GE 列表往往只给拥有者，模拟代理主要靠 Tag 与 Cue——预测主要发生在拥有者客户端，而不是每个旁观者都预测别人的技能。

## 和「普通 RPC」的边界

GAS 激活/确认走的是 ASC 内部 RPC 与预测键协议，不是业务里随手写的 `Server_DoSkill`。把技能逻辑改成自定义 RPC 绕过 ASC，等于丢掉 Cost 预测、GE 对账和 Tag 取消。若项目网络章节讲的是 Actor RPC 基础，与这里互补但不互相替代；本条目不链到泛化 RPC 笔记，避免概念串味。


## 复制什么、预测什么

可以粗分为三条带：

1. **预测带（拥有者客户端）**：LocalPredicted 技能激活、窗口内 Modifier GE、Cue、经 ASC 的 Montage。
2. **复制带（AttributeSet / ASC）**：属性 `DOREPLIFETIME_*`；按 ReplicationMode 复制的活跃 GE、Tag、Cue 状态。
3. **权威带（仅服务器）**：ExecCalc 结果、周期伤害跳字、大多数 Actor 生成、GE 移除、反作弊相关的净空检查。

UI 读属性时，拥有者客户端在预测窗口内可能看到「临时 CurrentValue」；`GetGameplayAttributeValue` 一类查询在客户端注释里提醒过：不一定计入所有预测修饰。做血条时要么接受短暂偏差，要么监听确认后再刷关键数字。

服务器拒绝后的回滚是「按键撤副作用」，不是通用状态快照。若你在预测窗口外改了角色自定义变量，GAS 不会帮你回滚——那些仍要自己写补偿。

## NetExecutionPolicy 怎么选（网络视角）

| 策略 | 本地手感 | 服务器角色 |
| --- | --- | --- |
| LocalPredicted | 立刻反馈 | 校验并 Confirm/Reject |
| LocalOnly | 仅本地 | 不当局权威玩法 |
| ServerInitiated | 等服务器开跑后再在本地演 | 发起方 |
| ServerOnly | 本地可能只见复制结果 | 全程权威 |

近战受击、AI 技能多用 ServerInitiated/ServerOnly；玩家主动技能才值得付预测复杂度。`NetSecurityPolicy` 另管「客户端能不能请求改这条技能的执行」，和执行地点正交。

## 容易踩的坑

- 在 `ActivateAbility` 里 `SetTimer` 半秒后再 Apply 伤害 GE：窗口已关，客户端不预测，表现为「先假动作后跳血」或只服务器掉血。
- 伤害走 ExecCalc 却做本地血条预测：血条要等属性复制；用 Cue/UI 动画过渡，或可预测段改 Modifier。
- 高延迟下冷却预测与权威冷却不一致：短 CD 技能射速会被 RTT 拖累（社区长期指出的 reconciliation 问题）。
- 客户端 Tag 状态与服务器不一致导致本地预测成功、服务器拒绝：回滚闪烁；激活条件宜简单、可预期。
- 以为 `RemoveActiveGameplayEffect` 会本地立刻与服务器一致：移除不预测，权威慢一拍。
- Montage 用非 GAS 路径只在本地 Play：其他人看不到，预测对账也接不上。

## 相关

[[CFA-Gameplay Ability System 总览]] [[CFB-GameplayAbility]] [[CFC-AttributeSet]] [[CFD-GameplayEffect]]
