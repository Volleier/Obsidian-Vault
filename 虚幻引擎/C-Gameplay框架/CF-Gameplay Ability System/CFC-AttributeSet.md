**UAttributeSet 是 GAS 里存放数值属性的 UObject：属性以 FGameplayAttributeData（BaseValue / CurrentValue）声明，经 ASC 注册后只能可靠地由 GameplayEffect 修改，并在 PreAttributeChange、PostGameplayEffectExecute 等回调里做钳制与结算。复制由 AttributeSet 自己的 GetLifetimeReplicatedProps 负责，而不是 ASC 直接推属性。**

> 版本说明：宏与回调按 **UE 5.4～5.6** 常见写法，对照 `AttributeSet.h`（`FGameplayAttributeData`、`InitFromMetaDataTable`、`GAMEPLAYATTRIBUTE_*`）及 Epic/社区 AttributeSet 样例。下文 `ATTRIBUTE_ACCESSORS` 是项目侧对引擎宏的薄包装，不是引擎另发明的宏。

## 为什么属性不能只是 float 成员

直接在 Character 上放 `float Health` 并复制，改血只能靠 RPC 或自定义逻辑，无法接入 GE 的修饰符聚合、预测 delta、以及「先写进 Damage 元属性再分到护盾/生命」这类管道。GAS 要求可玩法修改的数值进 AttributeSet：效果系统认的是 `FGameplayAttribute`（指向某个 AttributeSet 上的属性属性），修改路径统一，UI 也能绑 `GetGameplayAttributeValueChangeDelegate`。

代价是属性必须 C++ 声明，蓝图只能读/用 GE 改，不能在蓝图里「加一个新属性字段」。

## BaseValue 与 CurrentValue

| 字段 | 含义 | 谁改 |
| --- | --- | --- |
| `BaseValue` | 相对持久的底值 | Instant / Periodic 类「Execute」效果写入 |
| `CurrentValue` | Base + 持续修饰后的当前值 | Duration / Infinite 的 Modifier 叠加在聚合器上 |

移速底值 600，吃 +50 的 5 秒 Buff 时 Base 仍 600、Current 变 650；Buff 结束 Current 回到 Base。初学者常把 Base 当成 MaxHealth——错。最大值应另做 `MaxHealth` 属性，或在回调里用常量钳制。`FAttributeMetaData` 的 Min/Max 在表里是元数据，**引擎不会自动按它们钳制**（头文件仍标 work in progress 语气），真正钳制写在 `PreAttributeChange` / `PreAttributeBaseChange` / `PostGameplayEffectExecute`。

## 声明属性与引擎宏

引擎在 `AttributeSet.h` 提供 `GAMEPLAYATTRIBUTE_PROPERTY_GETTER`、`GAMEPLAYATTRIBUTE_VALUE_GETTER`、`GAMEPLAYATTRIBUTE_VALUE_SETTER`、`GAMEPLAYATTRIBUTE_VALUE_INITTER`、`GAMEPLAYATTRIBUTE_REPNOTIFY`。项目里通常再包一层，避免每个属性抄四行：

```cpp
// 项目头文件里常见包装（展开后仍是引擎宏）
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

UCLASS()
class UMyAttributeSet : public UAttributeSet
{
	GENERATED_BODY()
public:
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital")
	FGameplayAttributeData Health;
	ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)

	UFUNCTION()
	void OnRep_Health(const FGameplayAttributeData& OldHealth);

	virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
	virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
	virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;
};
```

```cpp
void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);
	// REPNOTIFY_Always：本地预测值已等于服务器值时仍触发 OnRep，便于预测对账
	DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}

void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
	Super::PreAttributeChange(Attribute, NewValue);
	if (Attribute == GetHealthAttribute())
	{
		NewValue = FMath::Clamp(NewValue, 0.f, GetMaxHealth());
	}
}

void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
	Super::PostGameplayEffectExecute(Data);
	// Instant 效果改 Base 之后：做伤害管道、死亡、最终钳制
	if (Data.EvaluatedData.Attribute == GetHealthAttribute())
	{
		SetHealth(FMath::Clamp(GetHealth(), 0.f, GetMaxHealth()));
	}
}
```

在 Owner 构造函数里 `CreateDefaultSubobject<UMyAttributeSet>` 会注册到 ASC。也可随后 `InitHealth(100.f)`（INITTER 生成）或对 Set 调用 `InitFromMetaDataTable(const UDataTable*)`：按属性名匹配 `FAttributeMetaData` 行，写入 Base/Current。ASC 还有 `InitStats(AttributeSetClass, DataTable)` 便捷封装。Lyra 等项目更多用 CurveTable + `FAttributeSetInitter` 按等级灌默认值——与 MetaDataTable 是另一条官方示例路径。

## 回调该干什么、不该干什么

| 回调 | 时机 | 适合 |
| --- | --- | --- |
| `PreGameplayEffectExecute` | Instant 执行前 | 可返回 false 拒绝本次修改 |
| `PostGameplayEffectExecute` | Instant 执行后（Base 已改） | 元属性伤害结算、死亡、钳 Base |
| `PreAttributeChange` | 任意导致 Current 将变之前 | 钳 Current（Health∈[0,Max]） |
| `PreAttributeBaseChange` | Base 将变且存在聚合器时 | 钳 Base |
| `PostAttributeChange` 等 | 变更后 | 视版本头文件；玩法事件优先绑 ASC 委托 |

头文件强调：`PreAttributeChange` 用来强制数值规则，不要在这里抛复杂玩法事件；UI/死亡监听用 `ASC->GetGameplayAttributeValueChangeDelegate(Attribute)`。Meta 属性（如 Damage/PendingDamage）通常不复制，只在服务器 Execute 管道里短暂存在。


## 初始化的几条路

| 方式 | API / 做法 | 备注 |
| --- | --- | --- |
| INITER | `InitHealth(100.f)` | 宏生成，构造或 BeginPlay 调用 |
| Instant GE | 应用只改属性的 Instant 效果 | Epic 常推荐，便于策划改表 |
| MetaDataTable | `InitFromMetaDataTable(DataTable)` | 行名匹配属性名；Min/Max 不自动钳制 |
| ASC InitStats | `ASC->InitStats(UMySet::StaticClass(), DataTable)` | 内部创建/取得 Set 再 InitFromMetaDataTable |
| CurveTable Init | `FAttributeSetInitter::InitAttributeSetDefaults` | 按 Group + Level 灌曲线，Lyra 类项目常用 |

AttributeSet 作为子对象创建时 Outer 是 OwnerActor；`GetOwningAbilitySystemComponent()` 从 Outer 找到 ASC。不要在未注册到 ASC 之前就依赖效果回调。

## Meta 属性做伤害管道

常见做法是定义不复制的 `Damage` 或 `PendingDamage`：GE/Exec 把算好的伤害写进该属性，`PostGameplayEffectExecute` 里读取后清零，再从护盾、生命依次扣。这样「伤害量计算」和「角色怎么吃伤害」分离，子类 AttributeSet 可以改分配规则而不改所有 GE。Meta 属性不要期望在客户端预测里漂亮对齐——尤其配合 ExecCalc 时，客户端往往只能等 Health 复制。


## 派生属性与聚合

若某属性要由其他属性算出（例如攻击力部分来自力量），可用 Infinite GE + Attribute Based Modifier 或 MMC：依赖属性变化时，未 snapshot 的捕获会刷新 CurrentValue。聚合顺序与 Multiply 偏置规则见 [[CFD-GameplayEffect]]。注意：从聚合器重算 CurrentValue 时**不会**再走一遍 AttributeSet 的 `PreAttributeChange`，若钳制写在那里，MMC/Exec 捕获路径要自己再钳一次。

监听变化：`ASC->GetGameplayAttributeValueChangeDelegate(GetHealthAttribute())` 返回的委托在数值变化时广播；`FOnAttributeChangeData` 在服务器上才带完整 `FGameplayEffectModCallbackData`。HUD 用它刷条即可，不必 Tick 轮询。

## 容易踩的坑

- 用 `SetHealth` 在技能里直接改血，绕过 GE：预测与修饰符聚合失效，多人易不同步。
- 只在 `PreAttributeChange` 钳制：Instant 改 Base 的路径仍可能越界，需在 `PostGameplayEffectExecute` 再钳。
- `DOREPLIFETIME` 不用 `CONDITION_NOTIFY` + `REPNOTIFY_Always`：预测对账时 OnRep 可能不触发。
- 同一 ASC 挂两个相同类的 AttributeSet：按属性查找时只认其中一个，行为未定义。

## 相关

[[CFA-Gameplay Ability System 总览]] [[CFB-GameplayAbility]] [[CFD-GameplayEffect]] [[CFE-预测与网络同步]]
