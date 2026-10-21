**实体-组件-系统（Entity-Component-System，ECS）是一种把游戏对象拆成三部分的架构：实体只是一个 ID，组件是挂在实体上的纯数据，系统是按“拥有哪些组件”批量查询实体并处理它们的逻辑。它和面向对象的区别在于身份、数据、行为三者彻底分开，数据按类型集中存放，便于缓存友好的批量处理和多线程调度。UE 的 Actor/Component 不是 ECS，UE 自带的 ECS 是 Mass 框架，见 [[IBAA-Unreal Mass框架]]。**

> 参考：Scott Bilas, *A Data-Driven Game Object System*（GDC 2002，《地牢围攻》的组件系统）；Mick West, *Evolve Your Hierarchy*（2007）；Adam Martin 2007 年起的 *Entity Systems are the future of MMOG development* 系列博文；Timothy Ford, *Overwatch Gameplay Architecture and Netcode*（GDC 2017）；Robert Nystrom《游戏编程模式》Component 与 Data Locality 两章。

## 从继承到组合，再到 ECS

早期游戏对象多用深继承树：`GameObject` → `MovableObject` → `Character` → `Enemy` → `FlyingEnemy`。需求一变就出问题：会飞的宝箱该继承谁？能力被锁死在树的某一支上，公共逻辑不断上移，基类越来越胖。

第一步演化是**组合**：对象本身是个容器，能力拆成组件挂上去，这就是基于组件的设计（[[IBAB-基于组件的设计]]），Unity 的 GameObject/MonoBehaviour、UE 的 Actor/ActorComponent 都属于这一类。这一步解决了“能力组合”的问题，但组件仍然是带虚函数、各自 Tick 的对象，散落在堆上，逻辑也仍然分散在各个组件里。

ECS 是第二步：把组件里的逻辑也拿出来。组件退化成没有方法的结构体，逻辑集中到系统里，一个系统只处理拥有特定组件组合的实体。对象“是什么”不再由类决定，只由它身上有哪些组件决定：有 `Position` + `Velocity` 就会被移动系统处理，再加上 `Health` 就会被伤害系统处理。

| | 继承树 | 组件式（Actor/Component） | ECS |
| --- | --- | --- | --- |
| 对象身份 | 类的实例 | 容器对象 | 一个整数 ID |
| 数据位置 | 对象内部 | 各组件对象内部，分散在堆上 | 按组件类型（或组合）集中连续存放 |
| 逻辑位置 | 类的方法 | 组件的方法、各自 Tick | 系统，按查询批量处理 |
| 新增能力 | 改继承关系 | 挂组件 | 加组件类型 + 系统 |
| 组件之间通信 | 直接调用 | 查找兄弟组件、委托 | 系统同时读写多种组件，或通过事件/标签 |

## 三个部分

**实体（Entity）。** 只是一个标识符，通常是“索引 + 版本号”：索引定位存储槽位，版本号在槽位复用时递增，旧句柄一比对就知道已失效。实体不存数据，也没有方法。

**组件（Component）。** 纯数据的结构体，最好是平凡可拷贝的（POD）。组件类型的粒度决定了查询的粒度：`Transform` 和 `Velocity` 分开，才能让“只有位置的静态物体”不被移动系统处理。没有数据、只用来做筛选的组件叫标签（Tag）。

**系统（System）。** 声明自己需要哪些组件、对每种组件是读还是写，框架据此找出匹配的实体交给它。系统本身应当无状态，跨帧的状态放回组件里。读写声明也是调度的依据：两个系统只读同一种组件可以并行，一个写另一个读就必须排先后。

## 存储方式

ECS 的性能来自存储布局。主流实现分两派：

| | Archetype（原型 / 表） | Sparse Set（稀疏集） |
| --- | --- | --- |
| 组织方式 | 组件组合完全相同的实体放在同一张表里，表内每种组件一列连续存放，常再切成固定大小的 Chunk | 每种组件一个稠密数组，外加一个“实体 ID → 稠密下标”的稀疏数组 |
| 多组件查询 | 找到所有包含这些组件的表，逐表线性遍历，所有列天然对齐 | 以最小的那个组件集合为驱动，对每个实体去其他集合查找 |
| 增删组件 | 实体要搬到另一张表，拷贝所有组件数据，开销大 | 只动那一种组件的数组，很便宜 |
| 适合 | 组合稳定、大批量迭代 | 组件频繁增删 |
| 代表 | Unity Entities（DOTS）、flecs、Bevy、UE Mass | EnTT |

Archetype 布局的核心收益是：系统遍历时，它要的几种组件在内存里都是连续的，CPU 预取和缓存行几乎不浪费；每个 Chunk 里的数据也可以直接交给 SIMD 或分到不同线程。代价是“改变组件组合就是搬家”，所以在这类框架里，频繁开关的状态更适合用组件里的字段表示，而不是加删组件。

下面是稀疏集的最小实现，它能说明 ECS 存储的基本思路：组件在一个稠密数组里连续存放，删除时用末尾元素补洞，保持连续。

```cpp
#include <cstdint>
#include <vector>

using Entity = std::uint32_t;
constexpr std::uint32_t Invalid = UINT32_MAX;

// 一种组件类型一个 SparseSet
template <typename T>
class SparseSet
{
public:
	void Add(Entity E, const T& Value)
	{
		if (E >= Sparse.size()) { Sparse.resize(E + 1, Invalid); }
		Sparse[E] = static_cast<std::uint32_t>(Dense.size());
		Dense.push_back(Value);
		Owners.push_back(E);
	}

	void Remove(Entity E)
	{
		const std::uint32_t Index = Sparse[E];
		const Entity Last = Owners.back();
		// 用最后一个元素填洞，保持 Dense 连续
		Dense[Index] = Dense.back();
		Owners[Index] = Last;
		Sparse[Last] = Index;
		Dense.pop_back();
		Owners.pop_back();
		Sparse[E] = Invalid;
	}

	bool Has(Entity E) const { return E < Sparse.size() && Sparse[E] != Invalid; }
	T& Get(Entity E) { return Dense[Sparse[E]]; }

	std::vector<T> Dense;          // 连续存放的组件数据
	std::vector<Entity> Owners;    // Dense[i] 属于哪个实体
private:
	std::vector<std::uint32_t> Sparse; // 实体 → Dense 下标
};

struct Position { float X, Y; };
struct Velocity { float X, Y; };

// “移动系统”：以 Velocity 集合为驱动，查找对应的 Position
void MoveSystem(SparseSet<Position>& Positions, SparseSet<Velocity>& Velocities, float Dt)
{
	for (std::size_t i = 0; i < Velocities.Dense.size(); ++i)
	{
		const Entity E = Velocities.Owners[i];
		if (!Positions.Has(E)) { continue; }
		Position& P = Positions.Get(E);
		P.X += Velocities.Dense[i].X * Dt;
		P.Y += Velocities.Dense[i].Y * Dt;
	}
}
```

删除时用末尾补洞会打乱顺序，遍历中删除元素会跳过被换过来的那个。真正的框架会把结构变更延迟到遍历结束后统一执行（命令缓冲），Mass 的 `Context.Defer()` 就是这个用途。

## 代价

ECS 不是免费的。一些在 OOP 里很自然的东西在 ECS 里变得别扭：

- **关系和层级。** 父子挂接、“我的目标是谁”这类实体间引用，只能存另一个实体的 ID，访问时再去查，层级变换的传播要专门的系统来做。flecs 为此专门引入了关系（relationship）作为一等概念。
- **单个实体的逻辑。** 独一无二的玩家角色、只有一个的 Boss 行为，写成“查询所有带 PlayerTag 的实体”显得绕。很多项目里 ECS 只用在数量大的部分（子弹、人群、粒子式单位），主角仍然用传统对象。
- **调试。** 断点看不到“这个对象”，只能看到一堆数组里的某一格；需要专门的实体查看器。
- **执行顺序。** 逻辑分散在多个系统里，系统之间的顺序和数据依赖要显式管理，否则同一帧内数据可能读到上一步还是下一步的值。

《守望先锋》的 GDC 演讲是 ECS 用于完整游戏玩法而不仅是性能热点的代表案例：他们把几乎所有玩法逻辑写成系统，并提到单例组件（整个世界只有一份的数据，例如输入）是处理全局状态的主要手段。

## 在 UE 里

UE 的 Actor/Component 是组件式设计，不是 ECS：`UActorComponent` 是完整的 UObject，有自己的方法和 Tick，组件对象分散在堆上，由 GC 管理。它的强项是编辑器集成、蓝图、网络复制和易用性，而不是一次处理几万个对象。

需要 ECS 的场景（大规模人群、交通、大量简单单位），UE 提供 Mass（MassEntity）：实体是 `FMassEntityHandle`，组件叫 Fragment，系统叫 Processor，按 Archetype + Chunk 存储，详见 [[IBAA-Unreal Mass框架]]。Mass 实体和 Actor 之间可以双向同步（MassActors 模块），近处用 Actor 表现、远处用实例化网格表现，是两种模型共存的典型方式。

## 容易踩的坑

**把 ECS 当成代码组织方式而忽略存储。** 组件仍然是 new 出来的对象、系统里通过指针跳转访问，就只得到了 ECS 的写法，没有得到它的性能。

**组件切得过细或过粗。** 太细，每个系统都要查一大串组件，查询和结构变更变贵；太粗，无关数据跟着进缓存，查询也失去区分能力。按“哪些数据总是被一起读写”来切。

**在遍历中直接增删组件或销毁实体。** 迭代器、数组下标全部失效。用命令缓冲延迟执行。

**系统里存实体状态。** 系统应无状态，否则并行调度和存档回放都会出问题。

## 相关

[[IBAA-Unreal Mass框架]] [[IBAB-基于组件的设计]] [[IBAAA-实体Entity实现]] [[IBAAB-组件component实现]]
