**基于组件的设计（Component-Based Design）把一个游戏对象拆成“容器 + 若干组件”：容器提供身份和生命周期，每个组件封装一项相对独立的能力（渲染、碰撞、移动、生命值），对象具备什么能力由它挂了哪些组件决定，而不是由继承自哪个类决定。UE 的 Actor 与 `UActorComponent` 体系、Unity 的 GameObject 与 MonoBehaviour 都是这种设计。它和 ECS 的区别在于组件自己带逻辑，ECS 则把逻辑全部移到系统里，见 [[IBA-实体-组件-系统 ECS]]。**

> 参考：Robert Nystrom《游戏编程模式》Component 一章；Scott Bilas, *A Data-Driven Game Object System*（GDC 2002）；Mick West, *Evolve Your Hierarchy*（2007）。UE 部分对照 Epic 5.x 文档 *Components* 与 API 参考。

## 为什么要拆成组件

继承表达的是“是一个”（is-a），组件表达的是“有一个”（has-a）。游戏对象的能力组合极其灵活：一扇门有网格、有碰撞、能被交互，可能还会播放声音；一个敌人有网格、有碰撞、会移动、有生命值、有 AI。用继承树描述这些组合，要么出现菱形继承，要么把一堆能力堆到公共基类里，让每个子类都背着用不上的东西。

组件把每项能力做成独立的类，对象只是把需要的组件装在一起。这带来几件事：同一个组件可以在完全不相关的对象上复用（生命值组件既能挂在敌人身上，也能挂在可破坏的箱子上）；能力可以在运行时增删；不同的人可以并行开发不同的组件，互不影响；配合编辑器，策划可以不写代码就拼出新对象。《游戏编程模式》把它概括为“允许一个实体跨越多个领域，而这些领域彼此不耦合”。

代价是对象的行为被拆散到多个类里，组件之间需要通信，而通信方式选得不好，耦合会以另一种形式回来。

## 组件之间怎么通信

| 方式 | 做法 | 耦合程度 | UE 里的对应 |
| --- | --- | --- | --- |
| 直接引用 | 组件查找兄弟组件并直接调用 | 高，编译期依赖对方类型 | `GetOwner()->FindComponentByClass<T>()` |
| 通过容器 | 组件把共享状态放在所属对象上，其他组件读取 | 中 | Actor 上的属性、Actor 的 Root 变换 |
| 事件 / 委托 | 组件广播事件，关心的一方订阅 | 低，只依赖事件签名 | `DECLARE_DYNAMIC_MULTICAST_DELEGATE_*` |
| 接口 | 通过接口调用，不关心实现者是谁 | 低 | `UINTERFACE` + `Execute_` 调用 |
| 消息 / 标签 | 通过字符串、标签或消息总线传递 | 最低，但失去类型检查 | Gameplay Tags、Lyra 的 Gameplay Message Subsystem |

多数项目混用：同一个 Actor 内部的紧密协作用直接引用，跨对象、跨系统的通知用委托或消息。

## UE 的 Actor/Component 体系

| 类 | 能力 | 例子 |
| --- | --- | --- |
| `UActorComponent` | 最基本的组件：生命周期、Tick、复制，没有空间位置 | `UCharacterMovementComponent`、自定义的生命值组件 |
| `USceneComponent` | 加上相对变换和挂接（attachment）层级 | 空的定位点、`USpringArmComponent`、`UCameraComponent` |
| `UPrimitiveComponent` | 加上几何表示：渲染、碰撞、物理 | `UStaticMeshComponent`、`USkeletalMeshComponent`、`UBoxComponent` |

Actor 的位置由它的 `RootComponent`（一个 `USceneComponent`）决定，其他场景组件通过 `SetupAttachment` / `AttachToComponent` 挂在根下面，形成一棵变换树。没有位置概念的逻辑组件直接挂在 Actor 上，不参与这棵树。

组件的创建分两种时机：

- **构造函数里**用 `CreateDefaultSubobject<T>(Name)` 创建默认子对象，它们成为这个类的一部分，在蓝图子类的组件面板里可见、可编辑，并且被序列化。
- **运行时**用 `NewObject<T>(Owner)` 创建，再调用 `RegisterComponent()` 注册到世界，否则不会渲染、不会 Tick、不会碰撞。

生命周期里常用的几个回调：`OnRegister`（注册到世界时）、`InitializeComponent`（需要 `bWantsInitializeComponent = true`）、`BeginPlay`、`TickComponent`（需要 `PrimaryComponentTick.bCanEverTick = true`）、`EndPlay`、`OnComponentDestroyed`。组件的 Tick 可以单独开关、单独设置 Tick 组和间隔，不需要每帧运行的组件应该关掉 Tick。

下面是原笔记里的 Actor 示例，补全了头文件里缺少的前向声明，并改用 `TObjectPtr`：

```cpp
// MyActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

class UStaticMeshComponent;
class UPointLightComponent;
class UBoxComponent;

UCLASS()
class MYGAME_API AMyActor : public AActor
{
	GENERATED_BODY()

public:
	AMyActor();

protected:
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
	TObjectPtr<UStaticMeshComponent> MeshComponent;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
	TObjectPtr<UPointLightComponent> PointLightComponent;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
	TObjectPtr<UBoxComponent> BoxComponent;
};
```

```cpp
// MyActor.cpp
#include "MyActor.h"

#include "Components/BoxComponent.h"
#include "Components/PointLightComponent.h"
#include "Components/StaticMeshComponent.h"

AMyActor::AMyActor()
{
	// 这个 Actor 自己不需要每帧逻辑
	PrimaryActorTick.bCanEverTick = false;

	MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
	RootComponent = MeshComponent;

	PointLightComponent = CreateDefaultSubobject<UPointLightComponent>(TEXT("PointLightComponent"));
	PointLightComponent->SetupAttachment(RootComponent);

	BoxComponent = CreateDefaultSubobject<UBoxComponent>(TEXT("BoxComponent"));
	BoxComponent->SetupAttachment(RootComponent);
}
```

`SetupAttachment` 只能在构造函数里（组件注册之前）使用；运行时挂接用 `AttachToComponent`。原示例里空的 `BeginPlay`、`Tick` 覆写已删去，它们什么都不做，反而让 Actor 默认参与每帧 Tick。

## 写一个可复用的逻辑组件

组件式设计真正的价值在自定义逻辑组件上。一个生命值组件不关心自己挂在谁身上，只负责数值和事件：

```cpp
// HealthComponent.h
#pragma once

#include "Components/ActorComponent.h"
#include "HealthComponent.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnHealthChanged, float, NewHealth, float, Delta);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnDeath);

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class MYGAME_API UHealthComponent : public UActorComponent
{
	GENERATED_BODY()

public:
	UHealthComponent();

	UFUNCTION(BlueprintCallable, Category = "Health")
	void ApplyDamage(float Amount);

	UPROPERTY(BlueprintAssignable, Category = "Health")
	FOnHealthChanged OnHealthChanged;

	UPROPERTY(BlueprintAssignable, Category = "Health")
	FOnDeath OnDeath;

protected:
	virtual void BeginPlay() override;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Health", meta = (ClampMin = "1"))
	float MaxHealth = 100.f;

	UPROPERTY(VisibleInstanceOnly, BlueprintReadOnly, Category = "Health")
	float Health = 0.f;
};
```

```cpp
// HealthComponent.cpp
#include "HealthComponent.h"

UHealthComponent::UHealthComponent()
{
	PrimaryComponentTick.bCanEverTick = false; // 纯事件驱动，不需要 Tick
}

void UHealthComponent::BeginPlay()
{
	Super::BeginPlay();
	Health = MaxHealth;
}

void UHealthComponent::ApplyDamage(float Amount)
{
	if (Amount <= 0.f || Health <= 0.f)
	{
		return;
	}
	const float Old = Health;
	Health = FMath::Clamp(Health - Amount, 0.f, MaxHealth);
	OnHealthChanged.Broadcast(Health, Health - Old);
	if (Health <= 0.f)
	{
		OnDeath.Broadcast();
	}
}
```

敌人、可破坏箱子、炮塔都可以挂这个组件，各自在蓝图里订阅 `OnDeath` 决定死亡表现。组件对宿主一无所知，宿主也不需要继承任何“可受伤基类”。`BlueprintSpawnableComponent` 让它出现在编辑器的“添加组件”列表里。网络游戏里还需要 `SetIsReplicatedByDefault(true)` 并把 `Health` 标记为复制属性，这里省略。

## 模块化玩法：运行时往 Actor 上挂组件

UE 5 的 Game Features 插件和 Modular Gameplay 插件把组件式设计推进了一步：`UGameFrameworkComponentManager` 允许在 Actor 注册为“接收者”之后，由外部的 Game Feature 通过 “Add Components” 动作把组件挂到指定类的 Actor 上。Lyra 示例用这套机制让不同的游戏模式给同一个角色挂上不同的能力组件，角色类本身不需要知道这些组件的存在。

## 组件式设计和 ECS

组件式设计解决的是“怎么组合能力”，ECS 在此基础上还要解决“怎么高效处理大量对象”。两者的分界线是逻辑放在哪里、数据怎么存：

| | 组件式（UE Actor/Component） | ECS（UE Mass） |
| --- | --- | --- |
| 组件 | 带方法和状态的对象（UObject） | 纯数据结构体（Fragment） |
| 逻辑 | 组件的方法、TickComponent | 系统（Processor）批量处理 |
| 内存 | 每个组件单独分配 | 按原型分块连续存放 |
| 适合的数量级 | 成百上千 | 数万以上 |
| 编辑器、蓝图、复制支持 | 完整 | 有限，需要专门的 Trait 和表现层 |

UE 里两者通常并存：主角、交互物、关卡里摆放的对象用 Actor，大规模人群和交通用 Mass，详见 [[IBAA-Unreal Mass框架]]。

## 容易踩的坑

**所有组件都开着 Tick。** 组件默认的 Tick 设置不一定符合需要，事件驱动的组件应该关掉 `bCanEverTick`，需要周期性处理的也可以设置 `TickInterval`。

**运行时创建组件忘了注册。** `NewObject` 之后不调 `RegisterComponent`，组件存在但不参与世界。

**在构造函数之外调用 `CreateDefaultSubobject`。** 只能在构造函数里用，运行时要用 `NewObject`。

**组件之间到处直接引用。** 每个组件都 `FindComponentByClass` 一串兄弟组件，删掉任何一个都会崩。跨组件通知优先用委托或接口。

**把组件当成 ECS 用。** 给一万个 Actor 各挂一个移动组件并各自 Tick，开销主要花在对象调度而不是逻辑上，这时该考虑 Mass 或者由一个管理器统一更新。

## 相关

[[IBA-实体-组件-系统 ECS]] [[IBAA-Unreal Mass框架]] [[IBAAA-实体Entity实现]] [[IBAAB-组件component实现]] [[ACAFB-策略模式（Strategy Pattern）]] [[ACAFJ-中介者模式（Mediator Pattern）]]
