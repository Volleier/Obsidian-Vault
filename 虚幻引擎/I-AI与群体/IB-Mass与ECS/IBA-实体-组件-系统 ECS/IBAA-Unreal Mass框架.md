**Mass（MassEntity）是虚幻引擎自带的一套基于 Archetype 的 ECS 框架：实体只是一个句柄，数据放在按组合分组、按 Chunk 连续存放的 Fragment 里，逻辑写在无状态的 Processor 中批量处理。它最早由 Epic 的 AI 团队为大规模人群模拟开发，《The Matrix Awakens》/ City Sample 里的行人和车辆就是用它跑的。**

> 版本说明：Mass 的 API 在 5.x 各版本之间一直在变。本文代码按 **5.6～5.8** 的写法，对照 MassSample 仓库（已升级到 5.8）的源码和 Epic 5.8 API 文档核对过。凡是和旧版本不一样的地方都在文中单独标出来了；遇到对不上的，以你手上引擎的头文件为准。

## 为什么要有 Mass

Actor 模型的问题并不在于“慢”，而在于它的内存布局对批量处理不友好。每个 Actor 都是一个独立的 UObject，身上挂着若干 Component，这些对象散落在堆上各处。一帧里要更新一万个 NPC 的位置时，CPU 会在内存里来回跳，每读一个对象就可能碰上一次 cache miss。再算上 Tick 的调度开销、虚函数调用、GC 引用追踪，数量一上去，开销就不是逻辑本身，而是“把数据找到”这件事。

Mass 的思路是把问题反过来：先问“哪些实体拥有 Transform 和 Velocity”，再把这些实体的 Transform 和 Velocity 一段一段连续地交给同一个函数处理。数据在内存里挨着放，处理函数只碰它声明过的那几种数据，CPU 预取和缓存都能吃满。代价是写法变了：没有继承、没有“这个对象是什么”，只有“这个实体身上有什么数据”。

Mass 的术语和通用 ECS 基本一一对应，只是为了不和 UE 已有的 Component、System 撞名，换了叫法：

| 通用 ECS | Mass | 在 UE 里对应的类型 |
| --- | --- | --- |
| Entity | Entity | `FMassEntityHandle` |
| Component | Fragment | `FMassFragment` 及其变体 |
| System | Processor | `UMassProcessor` |
| World / Registry | Entity Manager | `FMassEntityManager`（由 `UMassEntitySubsystem` 持有） |
| 无数据组件 | Tag | `FMassTag` |
| Archetype | Archetype | 内部是 `FMassArchetypeData` |

Epic 官方文档里特别强调了两点：Fragment 和 Entity 都只是数据，不含逻辑；Processor 是无状态的。后者很容易被忽略，Processor 里可以缓存配置，但不该存任何“某个实体的状态”，那种东西应该放进 Fragment。

## 模块与插件分布

Mass 分散在好几个模块和插件里，写 `Build.cs` 时经常找不到类型，先把位置理清楚：

| 名称 | 类型与位置 | 内容 |
| --- | --- | --- |
| `MassEntity` | 5.4 及以前是插件（`Engine/Plugins/Runtime/MassEntity`）；5.5 起移进引擎，变成 `Engine/Source/Runtime/MassEntity` 模块，原插件标为废弃 | EntityManager、Archetype、Query、Processor、CommandBuffer、Observer |
| `MassCore` | 5.8 新增的引擎模块（`Engine/Source/Runtime/Mass/MassCore`） | `FMassEntityHandle`、`FMassFragment`、`FMassTag` 等基础元素类型从 MassEntity 挪到了这里 |
| `MassGameplay` 插件 | `Engine/Plugins/Runtime/MassGameplay` | 下含 `MassCommon`（`FTransformFragment`）、`MassMovement`、`MassSpawner`（Trait、`UMassEntityConfigAsset`、`AMassSpawner`）、`MassRepresentation`、`MassLOD`、`MassActors`、`MassSignals`、`MassReplication`、`MassSmartObjects` 等模块 |
| `MassAI` 插件 | 引擎插件 | StateTree 集成、导航、ZoneGraph 寻路等，AI 相关的 Mass 功能 |
| `MassCrowd` | 引擎插件，Experimental | City Sample 行人的通用行为 |
| `ZoneGraph` | 引擎插件 | 用样条和车道描述人行道、道路，MassCrowd 靠它移动 |
| `MassTraffic` | **不是**引擎插件，是 City Sample 工程里的 `Traffic` 插件 | 车辆交通，只为那个 Demo 写的，官方不单独支持 |

另外 `StructUtils`（`FInstancedStruct`、`FConstSharedStruct` 这些 Mass 大量用到的类型）也已经并进了 CoreUObject，不用再单独启用插件。一个 5.8 工程最少需要在 `Build.cs` 里依赖 `MassEntity`、`MassCore`，用到 Transform 就加 `MassCommon`，用 Trait 和 Spawner 就加 `MassSpawner`；5.7 及以前没有 `MassCore`，不要加。

## Entity 和 Archetype

Entity 本身什么都不存。`FMassEntityHandle` 里只有两个 `int32`：`Index` 和 `SerialNumber`。Index 是实体在 EntityManager 内部表里的槽位，SerialNumber 用来区分同一个槽位的前后两任主人：实体销毁后槽位会被复用，新实体拿到的 SerialNumber 不同，于是旧句柄一对就知道失效了。这和 UE 里 `FWeakObjectPtr` 的 ObjectIndex + SerialNumber 是同一个思路。`FMassEntityHandle::IsSet()` 只检查这两个数有没有被赋值，不代表实体还活着，要判断实体是否有效得去问 EntityManager。

一个实体身上所有 Fragment 类型和 Tag 类型的集合叫它的**组合（composition）**，组合完全相同的实体归到同一个 **Archetype**。比如 `[Transform, Velocity]` 是一个 Archetype，`[Transform, Velocity] + FDeadTag` 就是另一个。Tag 虽然不带数据，但它是组合的一部分：每个 Archetype 用一个 bitset 记录自己有哪些 Tag，Tag 不同就是不同的 Archetype。

每个 Archetype 下面是一串固定大小的 **Chunk**（`FMassArchetypeChunk`）。一个 Chunk 里装着若干个实体，内存按“类 SoA”排布：先是这批实体的全部 Transform 连续排开，然后是全部 Velocity，依此类推。之所以不把整个 Archetype 的所有 Transform 排成一条长数组，是因为处理时通常要同时读好几种 Fragment。如果 Transform 和 Velocity 各自是一条几万长的数组，读第 i 个实体的两种数据就要跨越很远的两段内存；切成 Chunk 之后，同一批实体的几种 Fragment 都在同一块内存里，一起进缓存。MassSample 文档里提到 Chunk 大小（`UE::Mass::ChunkSize`）是按 128 字节缓存行 × 1024 行来定的，所以单个实体数据越多，一个 Chunk 能装下的实体就越少。

这套布局直接带来一个后果：**改变实体的组合就是搬家**。给实体加一个 Fragment 或 Tag，它就不再属于原来的 Archetype，EntityManager 得把它所有数据从旧 Chunk 拷到新 Archetype 的 Chunk 里，旧位置再由别的实体填上。频繁开关 Tag 不是免费的，这点后面讲 Command Buffer 时还会提到。

## Fragment 的几种类型

所有 Fragment 都是继承自对应基类的 `USTRUCT`，差别在于数据归属于谁：

| 基类 | 数据归属 | 查询时的声明 | 读取方式 |
| --- | --- | --- | --- |
| `FMassFragment` | 每个实体一份 | `AddRequirement<T>(Access, Presence)` | `GetFragmentView<T>()` / `GetMutableFragmentView<T>()` |
| `FMassTag` | 无数据，只看有没有 | `AddTagRequirement<T>(Presence)` | `DoesArchetypeHaveTag<T>()` |
| `FMassSharedFragment` | 多个实体共享一份，可写 | `AddSharedRequirement<T>(Access, Presence)` | `GetSharedFragment<T>()` / `GetMutableSharedFragment<T>()` |
| `FMassConstSharedFragment` | 多个实体共享一份，只读 | `AddConstSharedRequirement<T>(Presence)` | `GetConstSharedFragment<T>()` |
| `FMassChunkFragment` | 每个 Chunk 一份 | `AddChunkRequirement<T>(Access, Presence)` | `GetChunkFragment<T>()` / `GetMutableChunkFragment<T>()` |

Shared Fragment 适合放一组实体共用的配置，比如 LOD 距离、使用哪个网格。共享值也是组合的一部分：两个实体 Fragment 类型完全一样，但引用了不同的共享值，它们就会被分进不同的 Chunk。所以一次 `ForEachEntityChunk` 回调里拿到的共享 Fragment 一定是同一份，直接在循环外取一次就行。Chunk Fragment 是给“管理性”数据用的，官方举的例子是 LOD 计算：同一 Chunk 的实体共用一个 LOD 结果，可以整块跳过。

```cpp
// MyMassFragments.h
#pragma once

#include "MassEntityTypes.h" // 5.8 起基础类型实际定义在 MassCore 的 Mass/EntityElementTypes.h，这个头仍然可用
#include "MyMassFragments.generated.h"

// 每个实体一份的速度
USTRUCT()
struct MYGAME_API FMyVelocityFragment : public FMassFragment
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere)
	FVector Value = FVector::ZeroVector;
};

// 一组实体共用的只读配置
USTRUCT()
struct MYGAME_API FMySpeedLimitSharedFragment : public FMassConstSharedFragment
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere)
	float MaxSpeed = 600.f;
};

// Tag 不能有成员
USTRUCT()
struct MYGAME_API FMyMoverTag : public FMassTag
{
	GENERATED_BODY()
};

USTRUCT()
struct MYGAME_API FMyFrozenTag : public FMassTag
{
	GENERATED_BODY()
};
```

5.7 开始 Mass 要求 Fragment 是 C++ 意义上的 trivially copyable，否则直接断言。确实需要在 Fragment 里放 `TSharedPtr` 之类的成员时，要显式告诉 Mass 你知道自己在干什么：

```cpp
// 5.7+：非 trivially copyable 的 Fragment 需要声明 trait
template<>
struct TMassFragmentTraits<FMyFragmentWithSharedPtr> final
{
	enum
	{
		AuthorAcceptsItsNotTriviallyCopyable = true
	};
};
```

## Processor

Processor 是写逻辑的地方，继承 `UMassProcessor`，至少覆写 `ConfigureQueries` 和 `Execute`。**不需要手动注册**：任何非抽象的 `UMassProcessor` 子类默认都会被自动加进对应的处理阶段（`bAutoRegisterWithProcessingPhases` 默认为 `true`），这也是为什么只要写好类、编译通过，它就开始每帧跑了。

调度相关的设置都在构造函数里写：

| 成员 | 作用 |
| --- | --- |
| `ProcessingPhase` | 在哪个阶段执行，`EMassProcessingPhase`：`PrePhysics`（默认）、`StartPhysics`、`DuringPhysics`、`EndPhysics`、`PostPhysics`、`FrameEnd`，与同名 `ETickingGroup` 对应（`FrameEnd` 对应 `TG_LastDemotable`） |
| `ExecutionOrder.ExecuteInGroup` | 放进哪个处理组，引擎内置组名在 `UE::Mass::ProcessorGroupNames`（如 `Movement`、`Behavior`） |
| `ExecutionOrder.ExecuteBefore` / `ExecuteAfter` | `TArray<FName>`，填组名或 Processor 的类名，用来约束先后顺序 |
| `ExecutionFlags` | 在哪种网络/世界模式下执行，`EProcessorExecutionFlags`：`Standalone`、`Server`、`Client`、`Editor`、`EditorWorld`，以及组合值 `AllNetModes`、`AllWorldModes`、`All` |
| `bRequiresGameThreadExecution` | 为 `true` 时强制在游戏线程执行，要碰 Actor、UObject 就得开 |
| `bAutoRegisterWithProcessingPhases` | 设为 `false` 就不自动注册，适合手动驱动或只给 Observer 用的情况 |

`EditorWorld` 和 `AllWorldModes` 是 5.5 加的。初始化时 Mass 会根据这些排序规则和每个 Processor 声明的数据读写需求建一张依赖图，按图执行。

下面是一个完整的按速度移动实体的 Processor（5.6+ 写法）：

```cpp
// MyMoveProcessor.h
#pragma once

#include "MassProcessor.h"
#include "MassEntityQuery.h"
#include "MyMoveProcessor.generated.h"

UCLASS()
class MYGAME_API UMyMoveProcessor : public UMassProcessor
{
	GENERATED_BODY()

public:
	UMyMoveProcessor();

protected:
	virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
	virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;

private:
	FMassEntityQuery MoveQuery;
};
```

```cpp
// MyMoveProcessor.cpp
#include "MyMoveProcessor.h"

#include "MassCommonFragments.h"   // FTransformFragment（MassCommon 模块）
#include "MassCommonTypes.h"       // UE::Mass::ProcessorGroupNames
#include "MassExecutionContext.h"
#include "MyMassFragments.h"

// 用 *this 构造 Query，会自动注册到本 Processor
UMyMoveProcessor::UMyMoveProcessor()
	: MoveQuery(*this)
{
	ProcessingPhase = EMassProcessingPhase::PrePhysics;
	ExecutionOrder.ExecuteInGroup = UE::Mass::ProcessorGroupNames::Movement;
	ExecutionFlags = (int32)EProcessorExecutionFlags::AllNetModes;
	bRequiresGameThreadExecution = false; // 只碰 Fragment，可以放到工作线程
}

void UMyMoveProcessor::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	MoveQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadWrite);
	MoveQuery.AddRequirement<FMyVelocityFragment>(EMassFragmentAccess::ReadOnly);
	MoveQuery.AddConstSharedRequirement<FMySpeedLimitSharedFragment>(EMassFragmentPresence::All);
	MoveQuery.AddTagRequirement<FMyMoverTag>(EMassFragmentPresence::All);
	MoveQuery.AddTagRequirement<FMyFrozenTag>(EMassFragmentPresence::None); // 冻结的实体不参与
}

void UMyMoveProcessor::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	MoveQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
	{
		// 一次回调 = 一个 Chunk
		const int32 NumEntities = Context.GetNumEntities();
		const TArrayView<FTransformFragment> Transforms = Context.GetMutableFragmentView<FTransformFragment>();
		const TConstArrayView<FMyVelocityFragment> Velocities = Context.GetFragmentView<FMyVelocityFragment>();
		const FMySpeedLimitSharedFragment& SpeedLimit = Context.GetConstSharedFragment<FMySpeedLimitSharedFragment>();
		const float DeltaTime = Context.GetDeltaTimeSeconds();

		for (int32 EntityIndex = 0; EntityIndex < NumEntities; ++EntityIndex)
		{
			const FVector Velocity = Velocities[EntityIndex].Value.GetClampedToMaxSize(SpeedLimit.MaxSpeed);
			Transforms[EntityIndex].GetMutableTransform().AddToTranslation(Velocity * DeltaTime);
		}
	});
}
```

`ConfigureQueries` 的签名换过一次。5.5 及以前是无参的 `virtual void ConfigureQueries()`；5.6 起改成带 `const TSharedRef<FMassEntityManager>&` 参数，无参版本仍在头文件里但已不是该覆写的那个。同一批改动里，`virtual void Initialize(UObject& Owner)` 被标为废弃，需要自定义初始化（比如订阅 Signal）时改为覆写 `InitializeInternal(UObject& Owner, const TSharedRef<FMassEntityManager>& EntityManager)`，记得调 `Super`。

## Query

`FMassEntityQuery` 描述“我要处理哪些实体、要怎么用它们的数据”。它会缓存所有符合条件的 Archetype，新 Archetype 出现时再增量更新，所以每帧执行时不用重新筛选。

Query 必须和 Processor 关联，否则依赖图里没有它，执行时也会出问题。关联方式有两种：在 Processor 构造函数初始化列表里用 `MoveQuery(*this)` 构造（`FMassEntityQuery(UMassProcessor& Owner)`），或者在 `ConfigureQueries` 里先 `Query.Initialize(EntityManager)` 再在末尾 `Query.RegisterWithProcessor(*this)`。MassSample 两种都用，效果一样，挑一种坚持用就好。另外官方文档写明：一个有效的 Query 至少要有一个 `All`、`Any` 或 `Optional` 的 Fragment 需求，只有 Tag 需求的 Query 是不合法的。

**访问需求**（`EMassFragmentAccess`）决定能不能写：

| 值 | 含义 | 对应取数据的函数 |
| --- | --- | --- |
| `None` | 不绑定数据，只参与筛选 | 无 |
| `ReadOnly` | 只读 | `GetFragmentView`，返回 `TConstArrayView` |
| `ReadWrite` | 读写 | `GetMutableFragmentView`，返回 `TArrayView` |

ReadOnly 和 ReadWrite 的区分不只是 const 正确性的问题，它是 Mass 做并行调度的依据。依赖图知道每个 Processor 读哪些类型、写哪些类型：两个 Processor 都只读 Transform，可以同时跑；其中一个要写 Transform，另一个就得等它。什么都声明成 ReadWrite 的话，代码照样能跑，只是 Mass 会认为这些 Processor 互相冲突，只能串行执行。5.5 起 `FMassExecutionContext` 会检查：对只声明了 ReadOnly 的 Fragment 调 `GetMutableFragmentView` 会被拦下来。

世界子系统也按同样的方式声明：`Query.AddSubsystemRequirement<UMySubsystem>(EMassFragmentAccess::ReadWrite)`，在回调里用 `Context.GetMutableSubsystemChecked<UMySubsystem>()` 或 `GetSubsystemChecked` 取。如果是在 `Execute` 里、Query 回调外面用到子系统，则声明在 Processor 自带的 `ProcessorRequirements` 上。自定义子系统要让 Mass 知道它是否线程安全，需要特化 `TMassExternalSubsystemTraits`，设置 `ThreadSafeRead` / `ThreadSafeWrite`。

**存在需求**（`EMassFragmentPresence`）决定筛选：

| 值 | 含义 |
| --- | --- |
| `All` | 必须全部具备（默认） |
| `Any` | 标了 Any 的里面至少有一个 |
| `None` | 一个都不能有 |
| `Optional` | 有就用，没有也不影响匹配 |

`Optional` 和 `Any` 的 Fragment 在当前 Chunk 里可能不存在，此时 `GetFragmentView` 返回空的 view，要先判断 `Num() > 0` 再按下标访问。Tag 则用 `Context.DoesArchetypeHaveTag<T>()` 判断。

执行 Query 用 `ForEachEntityChunk(Context, Lambda)`，回调粒度是 Chunk，不是单个实体。回调里 `GetNumEntities()` 是当前 Chunk 的实体数，循环可以写普通 for，也可以用 `for (const int32 EntityIndex : Context.CreateEntityIterator())`。`EntityIndex` 只是这个 Chunk 里的下标，下一帧实体可能已经换了位置，需要长期记住某个实体时必须存 `Context.GetEntity(EntityIndex)` 返回的 `FMassEntityHandle`。

想把一个 Query 的 Chunk 分给多个线程，可以改用 `ParallelForEachEntityChunk`，它默认给每个任务分配独立的 Command Buffer；Processor 级别的并行则由控制台变量 `mass.FullyParallel` 控制。并行相关的开关和默认值在各版本间变化较多，用之前先查一下自己版本的源码。

较新的版本还提供了一套基于模板的简化写法 `UE::Mass::FQueryExecutor`，用 `FQueryDefinition<FConstFragmentAccess<...>, ...>` 在类型层面声明访问需求，再挂到 Processor 的 `AutoExecuteQuery` 上。MassSample 是在升级 5.8 时才加了示例，作者本人也说目前主流还是上面这种写法。

## Command Buffer 与结构变更

在 `ForEachEntityChunk` 里不能直接增删 Fragment、Tag，也不能直接销毁实体。原因前面已经埋下了：改组合就是把实体从一个 Chunk 搬到另一个 Archetype 的 Chunk，而你此刻正拿着这个 Chunk 的 `TArrayView` 在遍历。搬走一个实体，后面的实体会被挪过来补位，手里的 view 和下标全部失效。多线程下更糟，别的线程可能也在读同一个 Archetype。

所以结构变更一律**延迟**：通过 `Context.Defer()` 拿到 `FMassCommandBuffer`，把命令记下来，等当前处理批次结束后再统一执行。Epic 文档对这一点的描述是：命令会在当前处理批次末尾批量执行。批量还有额外好处，同一类命令可以合并，一次性搬一批实体比一个个搬便宜得多。

常用的便捷接口：

```cpp
Context.Defer().AddTag<FMyFrozenTag>(Entity);
Context.Defer().RemoveTag<FMyFrozenTag>(Entity);
Context.Defer().SwapTags<FOldTag, FNewTag>(Entity);
Context.Defer().AddFragment<FMyFragment>(Entity);
Context.Defer().RemoveFragment<FMyFragment>(Entity);
Context.Defer().DestroyEntity(Entity);
Context.Defer().DestroyEntities(Entities);
```

需要带数据添加时用 `PushCommand` 配合预定义的命令类型，比如 `FMassCommandAddFragmentInstances`（加上带初值的 Fragment，已存在则覆盖）、`FMassCommandBuildEntity`（用预留的句柄一次性建出带数据的实体）。

一个完整的例子：生命值归零的实体打上死亡标记，同时停止移动。

```cpp
// MyDeathProcessor.h
#pragma once

#include "MassProcessor.h"
#include "MassEntityQuery.h"
#include "MyDeathProcessor.generated.h"

USTRUCT()
struct MYGAME_API FMyHealthFragment : public FMassFragment
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere)
	float Health = 100.f;
};

USTRUCT()
struct MYGAME_API FMyDeadTag : public FMassTag
{
	GENERATED_BODY()
};

UCLASS()
class MYGAME_API UMyDeathProcessor : public UMassProcessor
{
	GENERATED_BODY()

public:
	UMyDeathProcessor();

protected:
	virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
	virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;

private:
	FMassEntityQuery DeathQuery;
};
```

```cpp
// MyDeathProcessor.cpp
#include "MyDeathProcessor.h"

#include "MassCommandBuffer.h"
#include "MassExecutionContext.h"
#include "MyMassFragments.h"

UMyDeathProcessor::UMyDeathProcessor()
	: DeathQuery(*this)
{
	ExecutionFlags = (int32)EProcessorExecutionFlags::AllNetModes;
}

void UMyDeathProcessor::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	DeathQuery.AddRequirement<FMyHealthFragment>(EMassFragmentAccess::ReadOnly);
	DeathQuery.AddTagRequirement<FMyDeadTag>(EMassFragmentPresence::None); // 已死的不再处理
}

void UMyDeathProcessor::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	DeathQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
	{
		const TConstArrayView<FMyHealthFragment> Healths = Context.GetFragmentView<FMyHealthFragment>();

		for (int32 EntityIndex = 0; EntityIndex < Context.GetNumEntities(); ++EntityIndex)
		{
			if (Healths[EntityIndex].Health > 0.f)
			{
				continue;
			}

			const FMassEntityHandle Entity = Context.GetEntity(EntityIndex);
			// 这里只是记录命令，实体此刻还在原来的 Archetype 里
			Context.Defer().AddTag<FMyDeadTag>(Entity);
			Context.Defer().RemoveTag<FMyMoverTag>(Entity);
		}
	});
}
```

要做的事不在现成命令里，比如必须在游戏线程改 Actor，可以推一个 lambda，它会在命令刷新时执行：

```cpp
// lambda 参数必须带 FMassEntityManager&
Context.Defer().PushCommand<FMassDeferredSetCommand>([Entity](FMassEntityManager& Manager)
{
	// 刷新命令时在这里执行，可以安全地调用 EntityManager 的直接接口
});
```

刷新时命令按操作类型分批执行，顺序是 Create → Add → Remove → ChangeComposition → Set → None，也就是先建实体、再改组合、最后改数据，保证“先建再设值”这类依赖成立。

处理流程之外（比如在子系统或 Actor 里）也可以用 `EntityManager.Defer()` 拿同一套接口。EntityManager 也有直接改组合的函数，但只有在游戏线程、且当前没有 Processor 在跑的时候调用才安全。

## Trait 与生成

直接在 C++ 里拼 Archetype 很繁琐，Mass 提供了数据驱动的方式：**Trait**。一个 Trait 是一组 Fragment、Tag 和配置的打包，官方文档的说法是“为某项功能提供所需 Fragment 的集合”，比如避障、LookAt、StateTree 各自是一个 Trait。策划在编辑器里新建一个 `UMassEntityConfigAsset`（数据资产），往里添加 Trait 并填参数；Config 还可以指定 Parent，继承父资产的 Trait。运行时这些资产被编译成 Entity Template，Spawner 按模板批量创建实体。内置的 `Assorted Fragments` Trait 可以直接在编辑器里往实体上塞任意 Fragment，不写 C++ 也能拼组合。

`UMassEntityTraitBase` 和 `UMassEntityConfigAsset` 位于 MassGameplay 插件的 `MassSpawner` 模块里，不在 MassEntity。

```cpp
// MyMoverTrait.h
#pragma once

#include "MassEntityTraitBase.h"
#include "MyMoverTrait.generated.h"

UCLASS(meta = (DisplayName = "My Mover"))
class MYGAME_API UMyMoverTrait : public UMassEntityTraitBase
{
	GENERATED_BODY()

protected:
	virtual void BuildTemplate(FMassEntityTemplateBuildContext& BuildContext, const UWorld& World) const override;

	UPROPERTY(EditAnywhere, Category = "Mass")
	FVector InitialVelocity = FVector(100.f, 0.f, 0.f);

	UPROPERTY(EditAnywhere, Category = "Mass")
	float MaxSpeed = 600.f;
};
```

```cpp
// MyMoverTrait.cpp
#include "MyMoverTrait.h"

#include "MassCommonFragments.h"
#include "MassEntityTemplateRegistry.h"
#include "MassEntityUtils.h"
#include "MyMassFragments.h"

void UMyMoverTrait::BuildTemplate(FMassEntityTemplateBuildContext& BuildContext, const UWorld& World) const
{
	FMassEntityManager& EntityManager = UE::Mass::Utils::GetEntityManagerChecked(World);

	// Require：要求别的 Trait 提供，自己不负责添加
	BuildContext.RequireFragment<FTransformFragment>();

	// 添加并设初值
	BuildContext.AddFragment_GetRef<FMyVelocityFragment>().Value = InitialVelocity;
	BuildContext.AddTag<FMyMoverTag>();

	// 相同参数的 Config 会拿到同一份共享 Fragment
	FMySpeedLimitSharedFragment SpeedLimit;
	SpeedLimit.MaxSpeed = MaxSpeed;
	const FConstSharedStruct SharedSpeedLimit = EntityManager.GetOrCreateConstSharedFragment(SpeedLimit);
	BuildContext.AddConstSharedFragment(SharedSpeedLimit);
}
```

`BuildTemplate` 的 `World` 参数在当前版本是 `const UWorld&`，早期版本（MassSample 旧 README 里的写法）是 `UWorld&`，覆写时签名要和自己引擎版本的基类一致。`GetOrCreateConstSharedFragment` 按内容哈希去重，所以共享 Fragment 的成员要能被正确哈希。旧版本可以自己传哈希值，MassSample 的 5.7 更新记录提到这个用法已经不行了。Trait 还可以覆写 `ValidateTemplate` 做校验，它在所有 Trait 的 `BuildTemplate` 之后调用。

生成实体最省事的是在关卡里放 `AMassSpawner`：Entity Types 里填 Config 资产和数量，Spawn Data Generators 里选位置生成方式（内置 EQS 和 ZoneGraph 两种），勾上 `bAutoSpawnOnBeginPlay` 就在开局生成，也可以运行时调 `DoSpawning()` / `DoDespawning()`。Spawner 适合开局就铺好的东西，比如人群、树木；像子弹这种运行时按需生成、初始数据由外部决定的实体，更适合在代码里用 `FMassEntityManager` 的批量创建接口或 `FMassCommandBuildEntity` 命令。

## Observer

有些逻辑只在“某个 Fragment 刚加上”或“刚被移除”时执行一次，比如初始化、清理外部资源。每帧用 Query 去扫一遍“有 A 没有 B”的实体很浪费，Observer 就是干这个的。`UMassObserverProcessor` 是 Processor 的子类，它不每帧跑，而是在一批实体发生了它关心的组合变化后被触发，执行的实体范围也只限于这批刚变化的实体。

Observer 的声明方式每个版本都在变：

| 版本 | 观察的类型 | 观察的操作 |
| --- | --- | --- |
| 5.6 及以前 | `ObservedType = FMyFragment::StaticStruct();` | `Operation = EMassObservedOperation::Add;` |
| 5.7 | `ObservedType = ...;` | `ObservedOperations = EMassObservedOperationFlags::Add;` |
| 5.8 | `ObservedTypes = { FMyFragment::StaticStruct() };`（可以一次观察多个类型） | `ObservedOperations = EMassObservedOperationFlags::Add;` |

5.8 的 `EMassObservedOperationFlags` 把操作拆细了：`AddElement`、`RemoveElement`、`CreateEntity`、`DestroyEntity`，其中 `Add` = `AddElement | CreateEntity`，`Remove` = `RemoveElement | DestroyEntity`。也就是说观察 `Add` 时，“直接带着这个 Fragment 创建出来的实体”也会触发。

```cpp
// MyVelocityInitObserver.h
#pragma once

#include "MassObserverProcessor.h"
#include "MassEntityQuery.h"
#include "MyVelocityInitObserver.generated.h"

UCLASS()
class MYGAME_API UMyVelocityInitObserver : public UMassObserverProcessor
{
	GENERATED_BODY()

public:
	UMyVelocityInitObserver();

protected:
	virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
	virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;

private:
	FMassEntityQuery EntityQuery;
};
```

```cpp
// MyVelocityInitObserver.cpp
#include "MyVelocityInitObserver.h"

#include "MassExecutionContext.h"
#include "MyMassFragments.h"

UMyVelocityInitObserver::UMyVelocityInitObserver()
	: EntityQuery(*this)
{
	// 5.8 写法，旧版本见上表
	ObservedTypes = { FMyVelocityFragment::StaticStruct() };
	ObservedOperations = EMassObservedOperationFlags::Add;
	ExecutionFlags = (int32)EProcessorExecutionFlags::AllNetModes;
}

void UMyVelocityInitObserver::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	// Query 里除了被观察的类型，还可以要别的数据
	EntityQuery.AddRequirement<FMyVelocityFragment>(EMassFragmentAccess::ReadWrite);
}

void UMyVelocityInitObserver::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	EntityQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
	{
		const TArrayView<FMyVelocityFragment> Velocities = Context.GetMutableFragmentView<FMyVelocityFragment>();
		for (const int32 EntityIndex : Context.CreateEntityIterator())
		{
			// 只处理刚加上速度的实体：给个随机朝向
			Velocities[EntityIndex].Value = FMath::VRand() * Velocities[EntityIndex].Value.Size();
		}
	});
}
```

Observer 要有人“通知”才会触发。通过 Command Buffer 做的组合变更会触发 Observer；早期版本里有一些 EntityManager 的直接调用会绕过 Observer，MassSample 作者说到 5.5 时几乎所有改组合的路径都已经覆盖了。5.5 的更新日志里还修了一个问题：通过 `CreateEntity`、`BuildEntity` 创建的实体，Observer 被通知时 Fragment 的值还没设好。如果你在老版本上，Observer 里读到的初值不对，多半是这个原因。

## Signal

Mass 的主流模式是“拉”：Processor 每帧自己去查。但有些事件很稀疏，比如“这个实体被击中了”，每帧为它扫一遍全体不划算。`MassSignals` 模块提供了命名信号：往一批实体身上发一个 `FName` 信号，订阅了这个信号的 Processor 下一次只处理收到信号的实体。官方文档把它描述为“没有负载的事件”，MassAI 的 StateTree 集成就靠它在需要时唤醒实体的状态树，而不是每帧都跑。

发信号用 `UMassSignalSubsystem`，比如 `SignalSubsystem->SignalEntities(SignalName, Entities)`，在 Processor 里也有延迟版本 `SignalEntitiesDeferred(Context, SignalName, Entities)`。收信号的一方继承 `UMassSignalProcessorBase`，在 `InitializeInternal` 里调用 `SubscribeToSignal(*SignalSubsystem, SignalName)` 订阅，再覆写 `SignalEntities(FMassEntityManager&, FMassExecutionContext&, FMassSignalNameLookup&)` 写处理逻辑，基类自带一个 `EntityQuery` 成员可以直接配置。

## 表现层：LOD 与 Representation

实体本身没有任何可见的东西，画出来要靠 MassGameplay 里的 `MassLOD` 和 `MassRepresentation`。

**MassLOD** 给每个实体算出四档 LOD：`High`、`Medium`、`Low`、`Off`，每档可以配距离和该档的最大实体数，也能输出一个 0.0（High）到 3.0（Off）的浮点重要度。它有三个使用方：

- 表现 LOD：除了距离还考虑是否在视锥内，视锥内外可以配不同的距离；
- 模拟 LOD（`MassSimulationLOD`）：把同 LOD 的实体分到同一批 Chunk，Query 可以按 LOD 过滤，也支持按 LOD 降低更新频率，远处的实体几帧才算一次；
- 复制 LOD：给每个连接的客户端分别算 LOD，用来控制网络带宽。

**MassRepresentation** 按表现 LOD 决定实体用什么来画。官方文档列的是四种：高精度 Actor、低精度 Actor、Instanced Static Mesh（ISM）和不显示；5.8 的 `EMassRepresentationType` 枚举里是 `HighResSpawnedActor`、`LowResSpawnedActor`、`SkinnedMeshInstance`、`StaticMeshInstance`、`None`。ISM 是最便宜的方式，配合顶点动画可以让几千个人动起来（City Sample 的做法），但官方把 ISM 动画标为 Experimental。离镜头近的实体换成真正的 Actor，就能用完整的骨骼动画、物理和碰撞；表现子系统负责两种形态之间的切换，并自动池化、复用生成出来的 Actor。

配置方式是在 Config 资产里加 `UMassVisualizationTrait`（或者它的子类），在 `Params.LODRepresentation[EMassLOD::High]` 这类字段里给每档 LOD 指定表现类型。Actor 和实体之间的数据同步由 `MassActors` 模块负责，它提供双向拷贝 Transform 等数据的 Translator Processor。

5.5 起还有一个建立在这套机制上的功能 Instanced Actors：把关卡里大量相同网格的 Actor（石头、树）平时换成 Mass 实体，靠近时再按 Mass LOD 或物理检测“水合”回 Actor。

## 容易踩的坑

**把 Chunk 下标当实体 ID。** `ForEachEntityChunk` 里的 `EntityIndex` 下一帧就可能指向别的实体。跨帧引用只能存 `FMassEntityHandle`，用之前还要让 EntityManager 确认它仍然有效；需要读别的实体的数据时，用 `FMassEntityView`，它会缓存实体所在 Archetype 的信息，比每次重新查快。

**在遍历中直接改组合。** 哪怕是单线程，也会让手里的 view 失效。增删 Fragment、Tag、销毁实体一律走 `Defer()`。

**什么都声明成 ReadWrite。** 能跑，但并行调度全废，而且 5.5 起反过来的错误（声明 ReadOnly 却取 Mutable view）会触发检查。

**Query 没注册，或者只有 Tag 需求。** 没用 `*this` 构造也没 `RegisterWithProcessor` 的 Query 不会进依赖图；只有 Tag 需求的 Query 不合法，至少要有一个 All/Any/Optional 的 Fragment 需求。

**Optional 的 Fragment 直接按下标取。** 当前 Chunk 没有这个 Fragment 时 view 是空的，先判断 `Num()`。

**在 Processor 里碰 UObject 却没开 `bRequiresGameThreadExecution`。** Processor 默认可以在工作线程执行，访问 Actor、组件、大部分子系统都必须在游戏线程，要么开这个标志，要么把操作包进 `FMassDeferredSetCommand`。

**Processor 自己就注册了。** 继承现有 Processor 做扩展时，父类也还在跑，两个一起改同一份数据，执行顺序又不固定，调起来很痛苦。不想要的 Processor 要把 `bAutoRegisterWithProcessingPhases` 关掉（Processor 的配置也会出现在 Project Settings 的 Mass 分类里）。

**频繁开关 Tag。** Tag 改变就是换 Archetype、搬数据。每帧都在变的状态用 Fragment 里的字段表示，Tag 留给变化不频繁、主要用于筛选的状态。

**照抄旧教程。** `ConfigureQueries()` 无参（≤5.5）、`Initialize(UObject&)`（≤5.5）、Observer 的 `Operation`（≤5.6）、`ObservedType`（≤5.7）、5.4 及以前 MassEntity 还是插件、5.8 多了 `MassCore` 模块，网上相当多的文章停在 5.0～5.3 的写法上。MassSample 的 README 自己也说部分示例还是 5.6 以前的代码，以仓库里的源码为准。

## 相关

[[IBA-实体-组件-系统 ECS]] [[IBAAA-实体Entity实现]] [[IBAAB-组件component实现]] [[IBAB-基于组件的设计]]
