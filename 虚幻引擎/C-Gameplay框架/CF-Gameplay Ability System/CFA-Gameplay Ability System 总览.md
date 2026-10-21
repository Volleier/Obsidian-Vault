**Gameplay Ability System（GAS）是虚幻引擎自带的 GameplayAbilities 插件：用 AbilitySystemComponent（ASC）统一管理技能（GameplayAbility）、数值（AttributeSet）、效果（GameplayEffect）、标签（GameplayTag）与表现提示（GameplayCue），并在多人下提供可预测的激活与属性修改。Fortnite、Paragon、Lyra 都建立在同一套框架上。**

> 版本说明：本文按 **UE 5.4～5.6** 的 GameplayAbilities 插件写法，对照过 Epic Python API（5.4/5.5）、`AbilitySystemComponent.h` / `GameplayAbility.h` / `GameplayEffect.h` 公开 API，以及社区整理的 Lyra / GAS 实践。5.3 起 GameplayEffect 大量标签逻辑迁到 GE Component；对不上时以你手上引擎的头文件为准。

## 为什么要用 GAS，而不是「蓝图事件堆技能」

单机里用 Character 上的自定义事件、布尔和定时器也能做出「按 Q 放火球」。一进多人，问题立刻变成：谁有权改血量、客户端能不能先播动作、冷却和消耗怎么防作弊、眩晕要取消哪些技能、同一套逻辑怎么复用到 AI。把这些规则散落在 Actor 蓝图里，最终会得到一堆互不相通的 RPC、复制属性和状态机。

GAS 把问题拆成几块固定职责：技能是可授予、可预测的逻辑单元；数值只通过 GameplayEffect 改；状态用 GameplayTag 查询；表现用 GameplayCue 解耦。代价是前期接入成本——必须开插件、写 C++ 挂 ASC 和 AttributeSet、理解 Spec/Handle——但换来的是可网络化的技能骨架，而不是每个技能重写一遍同步。

| 角色 | 类型 / 入口 | 干什么 |
| --- | --- | --- |
| 中枢 | `UAbilitySystemComponent` | 授予/激活技能、应用效果、持有 Tag、驱动 Cue |
| 技能 | `UGameplayAbility` | 一次可激活的玩法逻辑（可含 Cost/Cooldown） |
| 数值容器 | `UAttributeSet` + `FGameplayAttributeData` | 生命、法力等可复制浮点属性 |
| 效果 | `UGameplayEffect` → `FGameplayEffectSpec` | 改属性、授 Tag、叠层、周期结算 |
| 标签 | `FGameplayTag` / `FGameplayTagContainer` | 状态查询、激活条件、免疫、取消 |
| 表现 | GameplayCue（Tag 驱动） | 粒子、音效等非权威表现 |
| 异步步骤 | `UAbilityTask`（依赖 GameplayTasks） | 播 Montage、等事件、等目标等潜伏节点 |

## 模块、插件与编辑器入口

GAS 以插件形式随引擎发布，路径通常是 `Engine/Plugins/Runtime/GameplayAbilities`。工程侧至少要做两件事：在 `.uproject` 启用 `GameplayAbilities` 插件；在模块 `Build.cs` 依赖相关模块。漏掉 `GameplayTasks` 时，Ability Task 相关符号会在链接阶段才爆。

```csharp
// YourGame.Build.cs
PublicDependencyModuleNames.AddRange(new string[]
{
    "GameplayAbilities",
    "GameplayTags",
    "GameplayTasks"
});
```

```json
// YourGame.uproject 的 Plugins 片段
{ "Name": "GameplayAbilities", "Enabled": true }
```

编辑器里常见入口：

| 入口 | 用途 |
| --- | --- |
| Content Browser → Blueprint Class → 父类选 `GameplayAbility` / `GameplayEffect` | 做具体技能与效果资产 |
| Project Settings → Gameplay Tags | 维护层级 Tag |
| 运行时控制台 `showdebug abilitysystem` | 查看 ASC 上的属性、效果、技能（调试用） |
| Actor 上挂 `AbilitySystemComponent` | 需要 GAS 的角色/单位必须有一个 ASC |

OwnerActor 与 AvatarActor 都可以（且通常应该）实现 `IAbilitySystemInterface::GetAbilitySystemComponent()`。ASC 通过 `InitAbilityActorInfo(Owner, Avatar)` 绑定二者：Owner 管复制归属，Avatar 是世界里的表现体。二者可以是同一个 Actor（简单 AI），也可以 Owner 在 `PlayerState`、Avatar 在 `Character`（需要跨重生保留属性时常用）。

## ASC 如何把几块拼在一起

ASC 继承 `UGameplayTasksComponent`，是技能、效果、属性的运行时枢纽。授予技能写入 `ActivatableAbilities`（`FGameplayAbilitySpecContainer`）；激活时按 `NetExecutionPolicy` 走本地预测或服务器路径；应用效果时生成 Spec，成功后得到 `FActiveGameplayEffectHandle`。属性本身由 AttributeSet 复制，ASC 负责把效果结算推到 AttributeSet 的回调链。

最小挂接（概念示意，细节见各子条目）：

```cpp
// 构造函数里创建并标记复制（必须 C++）
AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
AbilitySystemComponent->SetIsReplicated(true);

// 服务器 Possess / 客户端拿到 PlayerState 后
AbilitySystemComponent->InitAbilityActorInfo(/*Owner*/ this, /*Avatar*/ this);

// 权威端授予
FGameplayAbilitySpec Spec(UMyFireballAbility::StaticClass(), /*Level*/ 1, /*InputID*/ INDEX_NONE, this);
FGameplayAbilitySpecHandle Handle = AbilitySystemComponent->GiveAbility(Spec);

// 尝试激活
AbilitySystemComponent->TryActivateAbility(Handle);
```

Blueprint 侧 ASC 还提供按类授予的便捷封装（如 `GiveAbility(AbilityClass, Level, InputID)`），底层仍是 Spec。技能内部用 AbilityTask 做「播动画并等待」「等待 GameplayEvent」；需要改血量时不要直接 `SetHealth`，而是 `ApplyGameplayEffect*`，这样才能进预测与聚合。

GameplayCue 用 Tag（常见前缀 `GameplayCue.`）触发本地或复制的表现，不参与权威数值；Tag 则贯穿激活条件（Required/Blocked）、效果授予、技能取消等查询。这几块可以单独深入，但查阅时记住：**改状态走 GE + Tag，改表现走 Cue，改流程走 Ability + Task**。


## Tag、Cue、AbilityTask 各管一层

GameplayTag 是层级 `FName`（如 `State.Debuff.Stun`），挂在 ASC 上可计数。技能用 Required/Blocked Tags 决定能不能放，效果用 Granted Tags 描述「身上有什么状态」，驱散则按 Asset Tags 或 Granted Tags 查询。比起布尔枚举，Tag 可以跨系统查询且支持容器匹配，但要提前在 Project Settings 里注册，复制多的项目通常打开 Fast Replication。

GameplayCue 不改权威数值，只响应 Cue Tag 播表现。Instant GE Execute Cue；持续 GE Add/Remove Cue。技能也可直接 `ExecuteGameplayCue` / `AddGameplayCue`。把「掉血」和「播受击特效」拆开，才能在预测回滚时只撤效果、或只补表现。

AbilityTask 继承自 GameplayTasks，让技能在不阻塞游戏线程的前提下等待 Montage 结束、WaitGameplayEvent、WaitTargetData 等。Task 必须挂在已实例化的技能上，因此 `NonInstanced` 技能基本用不了潜伏 Task。Task 结束或技能 End/Cancel 时要清干净，否则会留下悬空回调。

## 复制模式怎么选

ASC 对 GameplayEffect / Tag / Cue 有 `Full` / `Mixed` / `Minimal` 等复制模式（属性仍由 AttributeSet 复制）。单机可用 Full；玩家角色多人常用 Mixed（拥有者看得全，模拟代理主要靠 Tag/Cue）；大量 AI 常用 Minimal 降带宽。Mixed 期望 Owner 的 Owner 是 Controller——ASC 在 PlayerState 时天然满足，ASC 在 Pawn 时要注意 `SetOwner`。细节与预测交互见 [[CFE-预测与网络同步]]。


## 和蓝图事件的分工

GAS 不禁止蓝图：Ability / Effect 都可以是蓝图资产，AttributeSet 与 ASC 挂接则必须 C++。合理分工是——设计师在编辑器里配 GE 修饰、技能图表、Cue；程序员提供 AttributeSet、技能基类、输入路由、伤害 ExecCalc。继续用蓝图事件做关卡脚本、UI、与 GAS 无关的演出即可；**不要**再为「技能能不能放」平行维护一套布尔。

调试时 `showdebug abilitysystem` 与 Gameplay Debugger 的 Ability 分类能看到当前 Tag、活跃 GE、可激活技能，比在角色蓝图里打日志更快定位「授没授上」「Tag 卡住」问题。

## 容易踩的坑

- 只开了插件、没在 `Build.cs` 加 `GameplayAbilities` / `GameplayTags` / `GameplayTasks`，编译期缺头文件或链接 AbilityTask 失败。
- 忘了 `InitAbilityActorInfo`：会出现 `Can't activate LocalOnly or LocalPredicted ability when not local` 一类警告，技能「授了却激活不了」。
- ASC 放在 `PlayerState` 却不提高 `NetUpdateFrequency`：客户端属性/Tag 更新显得「钝」。
- 用蓝图自定义事件直接改血并手写 RPC：绕过 GE 后，Cost/Cooldown 预测、Tag 免疫、叠层都接不上。

## 相关

[[CFB-GameplayAbility]] [[CFC-AttributeSet]] [[CFD-GameplayEffect]] [[CFE-预测与网络同步]]
