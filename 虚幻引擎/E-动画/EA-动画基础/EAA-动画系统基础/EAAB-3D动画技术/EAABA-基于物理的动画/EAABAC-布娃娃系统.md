**布娃娃（Ragdoll）是把角色骨架近似成一组用关节约束连起来的刚体，交给物理引擎模拟，再把刚体的变换写回骨骼、驱动蒙皮网格的做法。它最常用于死亡和击飞，也可以和动画按权重混合做受击反应。UE 里一个骨骼网格体的布娃娃由 Physics Asset（`UPhysicsAsset`）定义，运行时通过 `USkeletalMeshComponent::SetSimulatePhysics` 等接口开关。**

> 参考：Thomas Jakobsen, *Advanced Character Physics*（GDC 2001，《杀手：代号 47》的布娃娃方案）；《Game Engine Architecture》第 3 版物理一章；UE 部分对照 5.x API 与 Physics Asset Editor 文档。

## 为什么要用布娃娃

在布娃娃普及之前，角色死亡靠播放预制的倒地动画。问题是倒地动画不知道周围的环境：角色死在楼梯上会悬空躺在台阶之间，死在墙边会一半身子插进墙里，被爆炸从侧面炸到却按固定方向倒下。做更多的死亡动画只能缓解，不能解决。

布娃娃把"怎么倒"交给物理：身体会顺着台阶滚下去，靠墙时滑坐下来，冲量从哪边来就往哪边飞。2000 年前后的《杀手：代号 47》用 Verlet 积分加距离约束实现了早期的布娃娃，Jakobsen 在 GDC 2001 的文章里把这套方法公开，之后布娃娃很快成了动作游戏的标配。

它的代价也很明确：纯布娃娃没有"意志"，角色像一袋沙子一样瘫下去，缺少人在失去平衡时会有的挣扎和自我保护动作；关节限制设得不好还会扭成不可能的姿势。所以现代游戏更多用的是"动画 + 物理"的混合方案。

## 结构：刚体、约束和骨骼映射

一套布娃娃由三部分组成。

**刚体。** 并不是每根骨骼都有刚体。一个人形骨架可能有上百根骨骼（含手指、面部、辅助骨），但布娃娃通常只给十几根主要骨骼配刚体：骨盆、两到三节脊柱、头、上臂、前臂、手、大腿、小腿、脚。碰撞形状用胶囊体为主，因为胶囊体的碰撞检测最便宜，而且滚动时不会卡住。质量一般按体积自动估算，再手动微调，躯干要比四肢重得多，否则四肢会甩着躯干跑。

**约束。** 相邻刚体之间用关节约束连接，限制它们的相对运动。肩和髋是球窝关节，需要一个锥形的摆动限制加一个扭转限制；肘和膝接近铰链，只允许一个方向弯曲。UE 的物理约束把这三个自由度叫 Swing1、Swing2、Twist，每个可以设成 Free、Limited、Locked。

**映射。** 每一帧，有刚体的骨骼直接取刚体的世界变换；没有刚体的骨骼（手指、辅助骨）保持相对父骨骼的局部变换，跟着走。这样整条骨架都有了姿势，再照常做蒙皮。

## 进入和退出布娃娃

**从动画进入**相对简单：物理身体在开启模拟的那一刻取当前动画姿势作为初始状态。如果还想让身体保持原有的动量（比如跑动中被击毙），需要把动画中各骨骼的速度传给刚体，否则角色会原地垂直瘫倒。再在击中点施加一个冲量，布娃娃就会向合理的方向倒下。

**从布娃娃回到动画**（起身）要麻烦得多。物理姿势是任意的，而起身动画是从固定姿势开始的。通常的做法是：等布娃娃基本静止后，把当前的物理姿势截取成一个快照；根据骨盆朝向判断角色是仰面还是俯卧，选择对应的起身动画；把角色胶囊体移到骨盆位置；关闭物理，然后在几帧内从快照混合到起身动画的第一帧。

**混合模式**（powered ragdoll）介于两者之间：物理一直开着，但每个关节都有一个马达（PD 控制器，见 [[EAABA-基于物理的动画]]）把刚体拉向当前动画姿势。被打中时临时降低马达强度，身体就会被冲量推歪，然后慢慢被拉回动画姿势。NaturalMotion 的 Euphoria 更进一步，用行为控制器实时生成保持平衡、伸手护头等主动动作。

## 在 UE 里

Physics Asset 在 Physics Asset Editor 里编辑，可以从骨骼网格体一键生成初始的身体和约束，再手动调整。资产通过骨骼网格体的 `PhysicsAsset` 属性关联，组件上也可以用 `SetPhysicsAsset` 覆盖。

下面是一个角色死亡时进入布娃娃、并在击中骨骼上施加冲量的例子，用到的都是 `USkeletalMeshComponent` / `UPrimitiveComponent` 的真实接口：

```cpp
#include "Components/CapsuleComponent.h"
#include "Components/SkeletalMeshComponent.h"
#include "GameFramework/CharacterMovementComponent.h"

void AMyCharacter::EnterRagdoll(const FVector& Impulse, const FVector& HitLocation, FName HitBone)
{
	// 胶囊体不再参与碰撞，移动组件停止
	GetCapsuleComponent()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	GetCharacterMovement()->DisableMovement();

	USkeletalMeshComponent* MeshComp = GetMesh();
	MeshComp->SetCollisionProfileName(TEXT("Ragdoll")); // 引擎自带的碰撞预设
	MeshComp->SetSimulatePhysics(true);
	MeshComp->WakeAllRigidBodies();

	// 冲量施加到被击中的那个刚体上
	MeshComp->AddImpulseAtLocation(Impulse, HitLocation, HitBone);
}
```

最容易漏的是碰撞预设：骨骼网格体在角色上默认往往不参与物理碰撞，不改成 `Ragdoll`（或其他启用了 Physics 的预设），开了模拟也会直接穿过地面掉下去。

只让上半身受物理影响、下半身继续播放动画时，用 `SetAllBodiesBelowSimulatePhysics(TEXT("spine_01"), true, true)` 打开某根骨骼以下的模拟，再用 `SetAllBodiesBelowPhysicsBlendWeight` 控制物理和动画的混合比例。想要身体被推歪后能自己回正，就加 `UPhysicalAnimationComponent`，调用 `SetSkeletalMeshComponent` 绑定网格体，再用 `ApplyPhysicalAnimationSettingsBelow` 给相应骨骼设置追随动画的强度。

起身时，先在动画实例上调用 `SavePoseSnapshot(TEXT("RagdollPose"))` 截取当前物理姿势，关闭模拟，然后在 AnimGraph 里用 Pose Snapshot 节点读出这个快照，混合到起身动画。

布娃娃默认不做网络同步。多人游戏里一般只同步"进入布娃娃"这个事件和初始冲量，各客户端各自模拟，结果不完全一致但通常可以接受；如果尸体位置影响玩法（比如拾取战利品），要单独同步最终位置。

## 容易踩的坑

**刚体配得太多。** 每根手指都配刚体既贵又不稳定，小刚体之间的约束很容易抖。只给影响整体轮廓的骨骼配刚体。

**关节限制太松。** 默认生成的约束有时限制过宽，布娃娃会扭出反关节姿势。重点检查膝、肘（单向）和脖子（扭转范围）。

**相邻刚体互相碰撞。** 胶囊体重叠又开着相互碰撞，一模拟就互相弹开。Physics Asset Editor 默认禁用相邻身体碰撞，但自己加的身体需要手动检查。

**质量比失衡。** 手、脚刚体很小却和躯干质量接近时，约束求解会不稳定，表现为高频抖动。保持相连刚体的质量比在一个合理范围内。

**起身时姿势跳变。** 直接关物理、直接播放起身动画，角色会瞬间"弹"到起身姿势。一定要用快照做过渡，并先把胶囊体移到身体所在的位置。

## 相关

[[EAABA-基于物理的动画]] [[EAABAA-IK]] [[EAABAB-布料与流体模拟]] [[EAABC-刚性层级动画]] [[EAABH-3D蒙皮动画]]
