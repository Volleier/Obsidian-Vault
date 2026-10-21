**UGameplayAbility 是 GAS 里一次可激活的玩法逻辑单元：由 AbilitySystemComponent 授予后以 FGameplayAbilitySpec 形式存在，激活时走 CanActivate → Activate →（可选）Commit Cost/Cooldown → End/Cancel。技能本身可以是 C++ 或蓝图子类，但授予、实例策略与网络执行策略都由引擎在 ASC 侧统一调度。**

> 版本说明：生命周期与枚举按 **UE 5.4** Python API / `GameplayAbility.h`、`GameplayAbilityTypes.h` 核对。`EGameplayAbilityInstancingPolicy::NonInstanced` 在 5.4 仍存在；新项目更常用 InstancedPerActor。以你手上引擎头文件为准。

## 为什么技能要做成 Ability，而不是 Character 上的函数

把「开火」「冲刺」「眩晕」写成 Character 成员函数，单机够用；多人下每个函数都要自己处理：能不能放、扣不扣蓝、冷却、被沉默时怎么办、客户端要不要先播 Montage。GAS 把这些问题收敛到同一套检查点：激活前查 Tag 与 Cost；Commit 时真正扣资源并上冷却；End/Cancel 统一收尾任务与 Owned Tags。

代价是你必须接受 Spec/Handle、实例策略和 NetExecutionPolicy，而不能「调一下函数就完事」。换来的是同一技能类可授予给玩家与 AI，并由 ASC 负责复制与预测窗口。

## 生命周期：从 TryActivate 到 End

客户端或服务器调用 `UAbilitySystemComponent::TryActivateAbility`（或按 Tag/类的变体）后，内部会走到 `InternalTryActivateAbility`：先 `CanActivateAbility`（无副作用检查），再按实例策略拿到实例，处理网络与预测，最后 `CallActivateAbility` → `PreActivate` → `ActivateAbility`。

| 阶段 | 关键 API | 作用 |
| --- | --- | --- |
| 可否激活 | `CanActivateAbility` / BP `CanActivateAbility` | 查 Tag、冷却、消耗等，失败则填 RelevantTags |
| 真正执行 | `ActivateAbility` / BP `ActivateAbility` | 子类覆写；蓝图图应调用 Commit 与 End |
| 提交 | `CommitAbility` / `CommitAbilityCost` / `CommitAbilityCooldown` | 扣 Cost、上 Cooldown；覆写 C++ Activate 时需自己调用 |
| 正常结束 | `EndAbility`（BP 常暴露为 End Ability） | 结束任务、清 ActivationOwnedTags、通知 ASC |
| 取消 | `CancelAbility` | 被打断或主动取消；`OnEndAbility(bWasCancelled)` |

蓝图注释写得很清楚：Activate 图里应调用 CommitAbility，并在逻辑结束时调用 EndAbility；潜伏节点可以先返回，但「逻辑上结束」时仍应 Commit/End。C++ 覆写 `ActivateAbility` 时引擎不会自动 Commit。

```cpp
// 典型 C++ 技能骨架（签名以 GameplayAbility.h 为准）
void UGA_Fireball::ActivateAbility(
	const FGameplayAbilitySpecHandle Handle,
	const FGameplayAbilityActorInfo* ActorInfo,
	const FGameplayAbilityActivationInfo ActivationInfo,
	const FGameplayEventData* TriggerEventData)
{
	if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
	{
		EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
		return;
	}

	// 用 AbilityTask 播 Montage / 等命中；命中后再 ApplyGameplayEffect*
	// ...

	// 即时技能可在此直接 End；有潜伏任务则等任务回调里 End
	EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}
```

Cost 与 Cooldown 不是硬编码字段，而是技能上的两个 `TSubclassOf<UGameplayEffect>`：`CostGameplayEffectClass`、`CooldownGameplayEffectClass`。Commit 时 ASC 应用它们；Cooldown 通常靠 GE 授予的 Tag（或查询剩余时间）阻止再次激活。

## 实例策略与网络执行策略

| 枚举 | 值 | 含义 |
| --- | --- | --- |
| `EGameplayAbilityInstancingPolicy` | `NonInstanced` | 跑在 CDO 上，不能安全存状态 |
| | `InstancedPerActor` | 每个 Actor 一份实例，可存状态、可复制 |
| | `InstancedPerExecution` | 每次激活新实例；复制支持受限 |
| `EGameplayAbilityNetExecutionPolicy` | `LocalPredicted` | 本地先跑，服务器确认/拒绝 |
| | `LocalOnly` | 仅本地控制端 |
| | `ServerInitiated` | 服务器发起，本地有控制端时也会跑 |
| | `ServerOnly` | 仅服务器 |

需要 AbilityTask、成员变量或复制时，几乎总是 `InstancedPerActor`。纯跳一下之类无状态逻辑才考虑 NonInstanced。玩家主动技能默认多选 `LocalPredicted`；纯服务器逻辑（掉落、结算）用 `ServerOnly` / `ServerInitiated`。

另有 `ReplicationPolicy`（技能状态是否复制给其他人）和 `NetSecurityPolicy`（客户端能否请求改执行），与「在哪执行」不是同一件事：预测不要求技能状态全复制。

## 授予、激活与输入绑定

权威端授予：

```cpp
// C++：构造 Spec 再 GiveAbility
FGameplayAbilitySpec Spec(
	UGA_Fireball::StaticClass(),
	/*Level=*/1,
	/*InputID=*/0,   // 与 ASC AbilityInputID 约定；无输入可用 INDEX_NONE
	/*SourceObject=*/this);
FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(Spec);

// 一次性授予并激活，结束后移除
ASC->GiveAbilityAndActivateOnce(Spec);
```

工程里常见「StartupAbilities」数组：在 `PossessedBy` / 初始化 ASC 后遍历 `GiveAbility`。Lyra 则用 AbilitySet 资产批量授予——模式不同，底层仍是 Spec。激活可用 `TryActivateAbility(Handle)`、`TryActivateAbilityByClass`、`TryActivateAbilitiesByTag`。

输入侧概览：传统路径是给 Spec 填 `InputID`，本地调用 `ASC->AbilityLocalInputPressed(InputID)` / `Released`（蓝图封装为 Press/Release Input ID）。新项目常在技能基类上挂 `FGameplayTag`（如 InputTag），由 Enhanced Input 根据 Tag 找到 Spec 再 `TryActivateAbility`。无论哪种，**绑定只负责触发 TryActivate，真正能否放仍走 CanActivate**。

技能上的 Tag 容器（AbilityTags、ActivationRequired/BlockedTags、Cancel/BlockAbilitiesWithTag、ActivationOwnedTags）决定互斥与状态；细节与 GE 授予的 Tag 一起查，见效果条目。


## Cost、Cooldown 与 Commit 的细节

`CommitAbility` 会依次尝试提交 Cost 与 Cooldown（也可分开调用 `CommitAbilityCost` / `CommitAbilityCooldown`）。Cost GE 多为 Instant，从 Mana/Stamina 等属性扣值；Cooldown GE 多为 Duration，靠授予的冷却 Tag 让后续 `CanActivateAbility` 失败。`CheckAbilityCost` / `CheckAbilityCooldown` 只检查不提交，适合 UI 灰显。

若技能在 Commit 之前就播了不可逆表现，服务器拒绝时会出现「假动作」。更稳的做法是：先过 CanActivate，预测窗口内 Commit，再开 Task；或把昂贵表现放在服务器确认之后（手感会差一截，看项目取舍）。

`GiveAbilityAndActivateOnce` 适合一次性逻辑（如被击退时对方授予并自动激活的位移技能）：授 → 激活 → 结束后从 ActivatableAbilities 移除。被动技能则常在授予时用 Tag Trigger 或在 `OnGiveAbility` 路径里自动 `TryActivate`（具体钩子以项目基类为准）。

## AbilityTask 在生命周期中的位置

`ActivateAbility` 可以立刻 End（开火瞬间结算），也可以拉起多个 Task 后先返回。技能在 Task 未完成期间保持 Active，ActivationOwnedTags 一直挂着，Block/Cancel 规则继续生效。`EndAbility` 会结束残留 Task；`CancelAbility` 走取消语义，`OnEndAbility(true)`。不要在 Task 回调里假设技能一定还活着——先查 `IsActive()` 或缓存的 Handle 是否仍有效。


## 技能上的 Tag 字段

| 字段 | 作用 |
| --- | --- |
| AbilityTags | 描述本技能，供查询与取消匹配 |
| ActivationOwnedTags | 激活期间授给 Owner，结束时移除（复制与否受 AbilitySystemGlobals 配置影响） |
| ActivationRequiredTags / ActivationBlockedTags | 激活门闩 |
| Source/Target Required/Blocked | 对源或目标 ASC 的额外门闩 |
| CancelAbilitiesWithTag / BlockAbilitiesWithTag | 激活时取消或屏蔽其他技能 |
| AbilityTriggers | 响应 GameplayEvent 自动尝试激活 |

这些与 GE 授予的 Tag 共用 ASC 上的计数容器：同一 Tag 多来源会叠计数，计数到 0 才算真正去掉。调试「明明取消了还在 Block」时先查还有没有别的 GE/技能在撑着计数。

## 容易踩的坑

- C++ 覆写了 `ActivateAbility` 却忘记 `CommitAbility`：蓝图节点不会替你扣蓝上冷却。
- `InstancedPerExecution` 却依赖复制状态：文档标明 replication currently unsupported。
- 冷却 GE 没授予可查询的 Tag：UI 和 `CheckAbilityCooldown` 行为会和设计预期不一致。
- 在非本地控制端激活 `LocalPredicted`/`LocalOnly`：ASC 未正确 `InitAbilityActorInfo` 时最常见。

## 相关

[[CFA-Gameplay Ability System 总览]] [[CFC-AttributeSet]] [[CFD-GameplayEffect]] [[CFE-预测与网络同步]]
