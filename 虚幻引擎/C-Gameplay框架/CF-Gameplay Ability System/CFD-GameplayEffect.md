**UGameplayEffect（GE）是 GAS 里「改属性、授 Tag、叠层、周期结算」的数据定义：运行时由 ASC 做成 FGameplayEffectSpec 再应用，成功则得到 FActiveGameplayEffectHandle。GE 本身应是数据类（可 Blueprint 子类做配置），不应往里堆事件图逻辑。**

> 版本说明：Duration / Modifier / Stacking / Spec 按 **UE 5.4～5.5** Python API 与 `GameplayEffect.h` 核对。**5.3 起** 大量标签与条件字段迁到 `GEComponents`（`UGameplayEffectComponent` 子类）；旧的 Inheritable*Tags、ApplicationTagRequirements 等在 5.5 Python API 里标为 deprecated。新资产优先用 Component；读老项目时两者可能并存。

## 为什么改数值要走 Effect，而不是 Setter

AttributeSet 的 Setter 能改数，但不会自动带上来源、等级、预测键、免疫查询、叠层与 Cue。技能、子弹、Buff 药如果各自 `SetHealth`，网络预测与「驱散带某 Tag 的效果」都无从谈起。GE 把一次修改描述成可配置资产：多久、改哪个属性、怎么叠、附带哪些 Tag/Cue；运行时只生成 Spec、填 Context/SetByCaller，再 `ApplyGameplayEffectSpecToSelf/Target`。

代价是配置面大，且 Instant 与 Duration 对 Base/Current 的语义不同，混用会算出「看起来 Buff 没生效」的假 bug。

## DurationPolicy：三种寿命

| `EGameplayEffectDurationType` | 行为 | 典型用途 |
| --- | --- | --- |
| `Instant` | 立即 Execute，改 **BaseValue**；不挂持续 Tag | 伤害、治疗、永久加点 |
| `Infinite` | 一直挂着直到移除；Modifier 影响 **CurrentValue** | 装备被动、光环、沉默 |
| `HasDuration` | 有限时长后过期；同上改 Current | 临时加速、护盾 Buff |

`DurationMagnitude` 在 HasDuration 时给出秒数；Periodic 的 `Period`（`FScalableFloat`）让 Duration/Infinite 按间隔再 Execute 一次（DoT）。周期执行按 Instant 语义改 Base，且**不能预测**。`ExecutePeriodicEffectOnApplication` 控制应用当下是否立刻跳一次周期。

## Modifier 与 Execution

| 机制 | 能否预测 | 能力 |
| --- | --- | --- |
| `Modifiers`（`FGameplayModifierInfo`） | 可以 | 每个 Modifier 改一个属性；Op 为 Add / Multiply / Divide / Override |
| `Executions`（`UGameplayEffectExecutionCalculation`） | **否**（服务器算） | 捕获多属性、复杂伤害公式，一次改多项 |

Modifier 量级来源常见：ScalableFloat、Attribute Based、`UGameplayModMagnitudeCalculation`（MMC，可预测）、SetByCaller（按 Tag 在 Spec 上塞 float）。聚合大致按 `((Base + Additive) * Multiplicitive) / Division`（引擎 Aggregator 实现；Multiply 多项是先偏置再加和，和直觉「连乘」不完全一样，改公式需动引擎或换 MMC/Exec）。

应用入口（ASC）：

```cpp
FGameplayEffectContextHandle Context = ASC->MakeEffectContext();
Context.AddSourceObject(this);

FGameplayEffectSpecHandle SpecHandle =
	ASC->MakeOutgoingSpec(UGE_Damage::StaticClass(), /*Level=*/1.f, Context);

if (SpecHandle.IsValid())
{
	// 可选：SpecHandle.Data->SetSetByCallerMagnitude(DamageTag, 42.f);
	FActiveGameplayEffectHandle Active =
		ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
}
```

技能里也有 `MakeOutgoingGameplayEffectSpec` / `ApplyGameplayEffectToTarget` 等封装。移除用 `RemoveActiveGameplayEffect(Handle)` 或按 Tag/源效果类查询移除。

## Spec、Handle 与 Stacking

`UGameplayEffect` 是定义（CDO/蓝图默认对象）；`FGameplayEffectSpec` 是带等级、Context、动态 Tag、SetByCaller 的运行时实例；应用成功后的活跃项是 `FActiveGameplayEffect`，对外句柄为 `FActiveGameplayEffectHandle`。

叠层只对 Duration / Infinite 有意义：

| 属性 | 作用 |
| --- | --- |
| `StackingType` | 按 Source 聚合或按 Target 聚合等 |
| `StackLimitCount` | 层数上限 |
| `StackDurationRefreshPolicy` | 成功叠层时是否刷新剩余时间（`RefreshOnSuccessfulApplication` / `NeverRefresh`） |
| `StackPeriodResetPolicy` | 是否重置周期计时 |
| `StackExpirationPolicy` | 到期清全部层、还是掉一层并刷新等 |
| `OverflowEffects` / `DenyOverflowApplication` | 溢出时额外应用或拒绝再叠 |

## 与 Tag、Cue、授予技能的关系

逻辑上 GE 仍区分几类 Tag（旧属性名或 5.3+ Component 等价物）：

| 类别 | 含义 |
| --- | --- |
| Asset Tags | 描述此 GE 自身，用于驱散查询（「移除带某 Asset Tag 的效果」） |
| Granted Tags（Owned） | 应用期间授给目标 ASC，移除时带走 |
| Application / Ongoing Requirements | 应用条件；Ongoing 不满足时效果可被抑制再恢复 |
| Remove Gameplay Effects with Tags | 应用成功时驱散目标上匹配的效果 |

`GameplayCues` 数组用 Cue Tag 触发表现：Instant 走 Execute，持续效果走 Add/Remove。`GrantedAbilities` 可在持续 GE 期间授予技能，并带移除策略（立刻取消、结束时移除、或留下）。免疫相关查询在新旧 API 间有迁移，以当前编辑器里的 GE Component 列表为准。


## SetByCaller 与 Context

`FGameplayEffectSpec::SetSetByCallerMagnitude(FGameplayTag, float)`（也有 FName 重载）把运行时才知道的数塞进 Spec，Modifier 配成 SetByCaller 时按同一 Tag 取值。蓄力伤害、动态治疗量都靠它。读取可用 `GetSetByCallerMagnitude`，缺省时可给默认值并选择是否警告。

`FGameplayEffectContext` / `FGameplayEffectContextHandle` 记录 Instigator、Causer、可选 HitResult 等。项目常子类化 Context，在 `UAbilitySystemGlobals::AllocGameplayEffectContext` 里替换，以便 ExecCalc 与 Cue 读到自定义字段。Context 在 `MakeEffectContext` / `MakeOutgoingSpec` 时带上，贯穿应用全程。

## 5.3+ Gameplay Effect Component

旧版把 Asset Tags、Granted Tags、Application Requirements、免疫等平铺在 `UGameplayEffect` 上。5.3 引入 `GEComponents` 数组后，行为拆成可组合组件（如 Asset Tags、Target Tag Requirements、Grant Abilities 等，具体类名以编辑器添加列表为准）。新项目应在组件里配 Tag 逻辑；只改父类里已 deprecated 的字段，运行时可能完全不生效。Duration、Modifiers、Executions、Period、Stacking 仍在 GE 本体上，没有搬进 Component。


## 应用与移除的入口

最终都会落到目标 ASC 的 `ApplyGameplayEffectSpecToSelf`。便捷封装包括：ASC 的 `ApplyGameplayEffectToSelf/ToTarget`、技能的 `ApplyGameplayEffectToOwner/ToTarget`、以及先 `MakeOutgoingSpec` 再把 Spec 交给子弹，击中时再 Apply。成功应用 Duration/Infinite 后，可绑 `OnActiveGameplayEffectAddedDelegateToSelf`；移除可绑 `OnAnyGameplayEffectRemovedDelegate`。复制模式下，模拟代理不一定能收到完整活跃 GE 列表，UI 若依赖「读自己的 Buff 列表」要确认 ReplicationMode。

Cooldown 类 GE 常配合查询剩余时间：`ASC` / 技能上的 cooldown 查询 API（如技能的 `GetCooldownTimeRemaining`）读的是仍活跃的冷却效果。预测冷却时，高延迟玩家可能比低延迟玩家更吃亏——这是 GAS 已知的 reconciliation 课题，短 CD 武器尤其明显。

## 容易踩的坑

- Instant 用来加「临时移速」：改的是 Base，Buff 不会自动结束回滚。
- 伤害公式放进 Execution，又期望客户端预测掉血：Exec 不算预测，血量要等复制；反馈应靠 Cue/动画掩盖或改用可预测 Modifier。
- SetByCaller Modifier 配了 Tag 却忘了在 Spec 上赋值：量级当 0，Divide 时尤其危险。
- 5.3+ 仍只改已 deprecated 的 Tag 字段：编辑器里看起来有 Tag，运行时走的是空 Component。

## 相关

[[CFA-Gameplay Ability System 总览]] [[CFB-GameplayAbility]] [[CFC-AttributeSet]] [[CFE-预测与网络同步]]
