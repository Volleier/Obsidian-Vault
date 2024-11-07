虚幻引擎中的Controller是用于控制角色或Pawn的对象。它们分为两种主要类型：[[PlayerController玩家角色控制器|PlayerController]]和[[AIControllerAI角色控制器|AIController]]。`PlayerController`用于处理玩家输入并控制角色，而`AIController`则用于控制由人工智能驱动的角色。
Controller是Actor的子类，这意味着它具有Actor的所有属性和功能，同时Controller能将输入转换为角色的动作，并管理角色的状态和行为。
![[Controller控制器-1.png]]
Controller和Pawn是平级关系（都是继承自Actor），但Controller可以控制Pawn，并且Controller只在运行的时候关联Pawn。
Controller本身还可以继续派生下去（如AIController和PlayerController），也可以容纳Components；也带着一个SceneComponent所以可以摆放在世界中；自身也可以添加成员变量来记忆存储游戏状态；自身也有一个FName StateName（Playing、Spectating、Inactive），切换自身的状态（运行，观察，非激活）；因为跟Pawn是平级的关系，只在运行的时候引用关联，所以对彼此独立存在不做强制约束，提高了灵活性。
一个Pawn自身上也可以配置策略，虚幻引擎可以根据Pawn来配置Controller：
```cpp
namespace EAutoReceiveInput
{
    enum Type
    {
        Disabled,
        Player0,
        Player1,
        Player2,
        Player3,
        Player4,
        Player5,
        Player6,
        Player7,
    };
}
TEnumAsByte<EAutoReceiveInput::Type> AutoPossessPlayer;
enum class EAutoPossessAI : uint8
{
    /** Feature is disabled (do not automatically possess AI). */
    Disabled,
    /** Only possess by an AI Controller if Pawn is placed in the world. */
    PlacedInWorld,
    /** Only possess by an AI Controller if Pawn is spawned after the world has loaded. */
    Spawned,
    /** Pawn is automatically possessed by an AI Controller whenever it is created. */
    PlacedInWorldOrSpawned,
};
EAutoPossessAI AutoPossessAI;
TSubclassOf<AController> AIControllerClass;
```
Controller是特殊化的Actor，因此Controller本质上来说是可以像Actor一样层级嵌套和渲染的。但Controller主要是要表达一个逻辑的概念，目的是用来表达游戏逻辑的，不需要在“大控制”下嵌套“小控制”以及添加Mesh在场景中显示，因此将层级嵌套和渲染隐藏了，可以在虚幻殷勤的源码中更改可见性：
```cpp
    bHidden = true;
#if WITH_EDITORONLY_DATA
    bHiddenEd = true;
#endif // WITH_EDITORONLY_DATA
```