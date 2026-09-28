**动画状态机（Animation State Machine）用“状态 + 过渡”的有限状态机来组织角色动画：每个状态输出一个姿势，过渡规则决定何时离开当前状态，过渡期间两个状态的姿势按时间混合。在 UE 里它是动画蓝图 AnimGraph 中的 State Machine 节点，内部每个状态本身又是一张完整的动画图。**

> 版本说明：本文按 UE 5.x 的动画蓝图编辑器描述，属性名对照 Epic 5.7～5.8 文档 *State Machines*、*Transition Rules*。

## 为什么动画需要状态机

角色的动画选择天然带着“上下文”：同样是松开移动键，在地面上应该播停步动画，在空中就该继续下落；跳跃动画必须先起跳、再滞空循环、再落地，不能随便从中间切入。只用一堆 `if` 或按参数直接选动画，很快会变成一张谁都不敢改的判断表，而且丢掉了“从哪里来”这个信息。

状态机把这些信息显式化：当前处于哪个状态是一份持久的记忆；每条过渡只描述“从 A 到 B 需要什么条件”，没有画出来的路径就不可能发生。动画师看图就能知道角色可能的行为序列，程序只需要提供条件所需的变量。它和软件设计里的状态模式（[[ACAFK-状态模式（State Pattern）]]）是同一种思想，区别在于动画状态机的切换不是瞬时的，而是一段有时长的混合。

## 结构

| 元素 | 在 UE 里 | 说明 |
| --- | --- | --- |
| 入口（Entry） | 每个状态机都有 | 连向默认状态，状态机变为相关（relevant）时从这里开始 |
| 状态（State） | 右键 `Add State` | 内部是一张独立的动画图，最终连到自己的 Output Pose |
| 过渡（Transition） | 从状态边框拖到另一状态 | 单向；来回切换需要两条 |
| 过渡规则（Transition Rule） | 双击过渡图标打开 | 一张输出 bool 的图，为 true 时触发过渡 |
| 导管（Conduit） | 右键 `Add Conduit` | 一对多、多对一的共享分叉点，本身有规则，默认返回 false |
| 状态别名（State Alias） | 右键 `Add State Alias` | 把若干状态“视为”同一个起点，合并重复的过渡线 |
| 子状态机 | 状态里再放一个 State Machine | 用来分层组织，例如把“空中”整体做成一个状态 |

状态里可以放任何动画逻辑：一个 Sequence Player、一个混合空间，或者瞄准偏移加上层叠混合。状态机不关心状态内部怎么算，只关心它最后输出的姿势。

导管最常见的用法是做“入口分叉”：勾选状态机的 `Allow Conduit Entry States` 后，入口可以连到一个导管，再由导管的多条过渡决定这次从哪个状态开始，蒙太奇打断状态机后重新进入时很有用。状态别名则用来收拾“很多状态都能跳到同一个状态”的连线：比如地面移动和落地两个状态都要能转到起跳和下落，建一个别名勾上这两个状态，再从别名连出去即可；勾 `Global Alias` 相当于所有状态（包括以后新增的）都能走这条过渡，官方建议只用在攻击、交互这类单次触发、时长有限的状态上。别名合并的过渡共享同一套规则和混合设置，需要不同参数时还是得单独连线。

## 过渡规则怎么写

规则图里可以用任意蓝图逻辑，最终输出一个 bool。常见的条件来源是角色移动组件的速度、是否在空中，以及一些只在规则图里可用的函数：

| 函数 | 返回 |
| --- | --- |
| `Current State Time` | 当前状态已经持续的秒数 |
| `Get Relevant Anim Time Remaining` / `Time Remaining` | 状态里最相关的动画（或指定的动画）剩余时间 |
| `Get Relevant Anim Time` / `Current Time` | 已播放时间 |
| `Was Anim Notify Name Triggered…` 等 | 上一帧是否触发了某个 Notify，可以限定在源状态或某个状态机里查找 |

“播完就走”的过渡不必手写剩余时间判断：勾选过渡的 `Automatic Rule Based on Sequence Player in State`，引擎会根据最相关的资产播放节点的剩余时间自动触发；`AutomaticRuleTriggerTime` 小于 0 时按过渡的混合时长提前触发，大于等于 0 时按指定秒数提前触发。想让某个动画不参与“最相关”的判断，在它的节点上勾 `Ignore for Relevancy Test`。

多条过渡同时为 true 时，`Priority Order` 数值最小的胜出。过渡属性里还有一个 `Bidirectional`，文档明确写着目前不受支持、不起作用，不要指望它替你生成反向过渡。

## 过渡怎么混合

过渡一旦触发，就进入一段混合期，由 `Blend Logic` 决定混合方式：

| Blend Logic | 做法 | 要求 |
| --- | --- | --- |
| Standard Blend（默认） | 按 `Duration` 和曲线 `Mode` 交叉淡化，两个状态在混合期内都要求值 | 无 |
| Inertialization | 立即切到新状态，用切换瞬间的速度、加速度把旧姿势的偏移衰减掉，旧状态不再求值 | AnimGraph 里状态机之后必须有 Inertialization（或 Dead Blending）节点 |
| Custom | 在一张自定义混合图里自己拼混合逻辑 | 时长和曲线仍来自 Blend Settings |

Standard Blend 就是经典的交叉淡化，细节（`Duration`、`Mode`、自定义曲线、`Blend Profile` 让不同骨骼以不同速度过渡）见 [[EBCB-Cross Fades]]。

## 状态机的运行时属性

| 属性 | 作用 |
| --- | --- |
| `Max Transitions Per Frame` | 一帧内最多走几次过渡。多条规则可能连环成立时设为 1，避免一帧里跳过好几个状态 |
| `Skip First Update Transition` | 状态机刚变为相关时，如果某条过渡已经成立，是否直接跳过去而不做混合 |
| `Reinitialize on Becoming Relevant` | 变为相关时重置第一个进入的状态 |
| `Create Notify Meta Data` | 规则里用到 Notify 查询函数时必须开启 |
| `Allow Conduit Entry States` | 允许入口连到导管 |

每个状态还有 `Always Reset on Entry`：开启时每次进入都让里面的动画从头播放，关闭时回到这个状态会接着上次的进度。状态和过渡都能配置 `Entered State Event`、`Left State Event`、`Fully Blended State Event`、`Start/End/Interrupt Transition Event` 这些自定义事件，名字会生成对应的 Skeleton Notify，在事件图里接收。过渡被打断时还可以通过 `Transition Interrupt` 绑定 Notify。

## 给状态机喂数据

过渡规则只应该读变量，不应该在规则图里调用复杂的游戏逻辑。变量在动画实例里每帧更新一次，状态机读取。动画蓝图开启多线程更新时，AnimGraph 的求值在工作线程上进行，所以更新变量的位置也要注意线程安全。C++ 里常见的写法是在游戏线程取数据、在线程安全的回调里算派生量：

```cpp
// MyAnimInstance.h
#pragma once

#include "Animation/AnimInstance.h"
#include "MyAnimInstance.generated.h"

class UCharacterMovementComponent;

UCLASS()
class MYGAME_API UMyAnimInstance : public UAnimInstance
{
	GENERATED_BODY()

protected:
	virtual void NativeInitializeAnimation() override;
	virtual void NativeUpdateAnimation(float DeltaSeconds) override;            // 游戏线程
	virtual void NativeThreadSafeUpdateAnimation(float DeltaSeconds) override;  // 可能在工作线程

	UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
	float GroundSpeed = 0.f;

	UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
	bool bIsFalling = false;

	UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
	bool bShouldMove = false;

private:
	UPROPERTY(Transient)
	TObjectPtr<UCharacterMovementComponent> Movement;

	FVector CachedVelocity = FVector::ZeroVector;
};
```

```cpp
// MyAnimInstance.cpp
#include "MyAnimInstance.h"

#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"

void UMyAnimInstance::NativeInitializeAnimation()
{
	Super::NativeInitializeAnimation();
	if (const ACharacter* Character = Cast<ACharacter>(TryGetPawnOwner()))
	{
		Movement = Character->GetCharacterMovement();
	}
}

void UMyAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
	Super::NativeUpdateAnimation(DeltaSeconds);
	// 访问组件只在游戏线程做，拷贝成普通值
	if (Movement)
	{
		CachedVelocity = Movement->Velocity;
		bIsFalling = Movement->IsFalling();
	}
}

void UMyAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
	Super::NativeThreadSafeUpdateAnimation(DeltaSeconds);
	// 只用上面拷贝好的数据
	GroundSpeed = CachedVelocity.Size2D();
	bShouldMove = GroundSpeed > 3.f;
}
```

过渡规则里直接读 `bShouldMove`、`bIsFalling` 即可。蓝图里的对应做法是把逻辑放进 `Blueprint Thread Safe Update Animation`，通过 Property Access 读取角色数据。

## 状态机不适合做的事

状态机擅长描述“离散的行为阶段”，不擅长连续变化的参数。走、慢跑、冲刺如果各做一个状态，速度在阈值附近抖动就会来回过渡；这类连续量应该交给混合空间，状态机里只留一个“地面移动”状态。一次性、由游戏逻辑触发的动作（攻击、受击、开门）通常用蒙太奇播放到 Slot 上，而不是给每个动作都画进状态机。上下半身各自有状态的情况，用分层的方式组织，见 [[EBCE-分层动画状态机]]。

## 容易踩的坑

**规则连环成立，一帧跳过多个状态。** A→B 和 B→C 的规则同时为 true 时，默认可能一帧内连续过渡。确实不想要时把 `Max Transitions Per Frame` 设为 1。

**Inertialization 过渡没效果或报错。** 状态机之后没有 Inertialization / Dead Blending 节点时，惯性化请求无人处理，运行时会在消息日志里报错。

**以为 `Bidirectional` 能省一条线。** 它目前不起作用。

**状态机刚激活时闪一下。** 默认状态和实际应在的状态不一致时，会先进入默认状态再混合过去。考虑 `Skip First Update Transition` 或导管入口。

**在规则图里做重逻辑或访问 Actor。** 多线程更新下这既不安全也浪费，把计算挪到动画实例的更新函数里。

## 相关

[[EBCB-Cross Fades]] [[EBCE-分层动画状态机]] [[EBAC-线性插值]] [[EBDD-分部混合（Skeleton Masked Blending）]] [[ACAFK-状态模式（State Pattern）]]
