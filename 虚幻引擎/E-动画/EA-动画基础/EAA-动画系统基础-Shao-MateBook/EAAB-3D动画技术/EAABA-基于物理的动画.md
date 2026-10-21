**基于物理的动画（Physics-Based Animation）是让角色或物体的一部分运动由物理模拟算出来，而不是由动画师逐帧指定的做法。游戏里它几乎从不单独存在，而是和关键帧动画按比例混合：从完全由动画驱动（运动学，kinematic）到完全由物理驱动（动力学，dynamic）之间有一整条光谱，布娃娃、布料、头发、次级抖动都是这条光谱上的点。UE 里对应的入口是 Physics Asset（`UPhysicsAsset`）、`USkeletalMeshComponent` 的物理模拟接口、Physical Animation 组件、动画蓝图里的 RigidBody / AnimDynamics 节点，以及 Chaos Cloth。**

> 参考：《Game Engine Architecture》第 3 版物理与动画两章；UE 部分对照 5.x 官方文档（Physics Asset Editor、Skeletal Controls 节点参考）。

## 为什么要让物理参与动画

关键帧动画的内容是预先做好的，而游戏里的情况无穷无尽：角色可能在楼梯上被爆炸掀飞，可能倒在斜坡上，披风可能被任意方向的风吹。为每种情况做一段动画不可能，于是把"预先不知道"的那部分交给物理。

另一类需求是次级运动（secondary motion）：主体动作由动画师控制，但耳环、马尾、腰带、肚腩这些附属物的晃动，本质上是被主体带动的惯性和弹性响应。它们手动 K 帧既费时又容易和主体动作脱节，用物理算反而更自然，也更便宜。

代价在于**失控**。物理模拟的结果不受美术直接控制，参数稍有不对就会抖动、穿插、爆炸；而且同样的输入在不同帧率下可能得到不同的结果。所以工程上的主要功夫不在"怎么模拟"，而在"怎么把模拟结果和动画拼在一起，并把它限制在可接受的范围里"。

## 从运动学到动力学的光谱

| 形式 | 谁在驱动 | 典型用途 | 对应笔记 |
| --- | --- | --- | --- |
| 纯关键帧 | 动画数据 | 绝大部分动作 | [[EAABH-3D蒙皮动画]] |
| 程序化 / IK 修正 | 数学约束，不涉及力 | 脚贴地、手抓物体、看向目标 | [[EAABAA-IK]] |
| 次级动力学 | 动画为主，少数骨骼物理跟随 | 头发、饰品、尾巴、布料 | [[EAABAB-布料与流体模拟]] |
| 物理动画（powered ragdoll） | 物理为主，关节马达把身体拉向动画姿势 | 受击反应、推搡、跌倒过程 | [[EAABAC-布娃娃系统]] |
| 纯布娃娃 | 物理 | 死亡、击飞 | [[EAABAC-布娃娃系统]] |
| 物理角色控制 | 控制器输出关节力矩，没有"目标动画"或只作为参考 | NaturalMotion 的 Euphoria（《GTA IV》等）、DeepMimic 类研究 | — |

要注意 IK 虽然常和物理动画放在一起讨论（原笔记也把它归在这里），但它本身是纯几何的求解，不涉及质量、力和积分，属于程序化动画。

## 模拟的基本环节

不论是刚体还是布料，一步模拟大致都是：累加外力（重力、风、冲量）→ 积分出新的速度和位置 → 处理约束（关节限制、碰撞、布料的距离约束）→ 输出变换给渲染。

游戏里常用半隐式欧拉（symplectic Euler），先更新速度再用新速度更新位置：

$$
\mathbf{v}_{t+\Delta t} = \mathbf{v}_t + \frac{\mathbf{F}}{m}\Delta t,\qquad
\mathbf{x}_{t+\Delta t} = \mathbf{x}_t + \mathbf{v}_{t+\Delta t}\Delta t
$$

它比显式欧拉稳定得多，计算量却几乎一样。布料和软体更常用基于位置的方法（PBD / XPBD），直接修正位置去满足约束，再由位置差反推速度，对大步长更宽容，Chaos Cloth 就属于这一类。

让物理"追随"动画，最常见的是比例—微分（PD）控制，也就是一个带阻尼的弹簧：

$$
\boldsymbol{\tau} = k_p\,(\boldsymbol{\theta}_{\text{target}} - \boldsymbol{\theta}) - k_d\,\dot{\boldsymbol{\theta}}
$$

$\boldsymbol{\theta}_{\text{target}}$ 来自当前帧的动画姿势，$k_p$ 决定拉回去的力度，$k_d$ 抑制振荡。$k_p$ 调大就更像动画、更僵硬，调小就更像布娃娃、更软。UE 的 Physical Animation 组件和物理约束上的 Angular Drive 本质上都是这个模型，参数名叫 Strength / Damping。

## 在 UE 里

UE 的角色物理全部建立在 **Physics Asset** 上：它为骨骼网格体的若干骨骼各配一个简单碰撞体（胶囊、球、盒），并用物理约束（Constraint）把它们连起来，约束上设置 Swing1 / Swing2 / Twist 三个角度限制。编辑器是 Physics Asset Editor。一个骨骼网格体资产通过 `PhysicsAsset` 属性引用它。

在这之上，按"物理参与多少"有几种用法：

| 需求 | UE 里的做法 |
| --- | --- |
| 整体变布娃娃 | `SetSimulatePhysics(true)`，通常同时把碰撞预设改成 `Ragdoll` |
| 只让某根骨骼以下模拟 | `SetAllBodiesBelowSimulatePhysics(BoneName, true, bIncludeSelf)` |
| 物理和动画按比例混合 | `SetPhysicsBlendWeight` / `SetAllBodiesBelowPhysicsBlendWeight` |
| 物理身体追随动画姿势（powered ragdoll） | `UPhysicalAnimationComponent`：`SetSkeletalMeshComponent` 后调用 `ApplyPhysicalAnimationSettingsBelow` 或 `ApplyPhysicalAnimationProfileBelow` |
| 动画蓝图内的次级物理 | RigidBody 节点（在动画图里按 Physics Asset 做局部空间模拟，适合多骨骼、需要碰撞的情况）；AnimDynamics 节点（不需要 Physics Asset 的轻量求解器，适合单根骨骼或短链） |
| 布料 | Chaos Cloth，见 [[EAABAB-布料与流体模拟]] |

官方文档给 RigidBody 节点和 AnimDynamics 节点的定位不同：前者依赖 Physics Asset，能和世界碰撞，适合一串骨骼一起晃；后者只在动画线程里算自己那几根骨骼，更可控、更便宜，适合马尾、挂件。两者都跑在动画求值阶段，不经过游戏线程的物理场景，这也是它们比整体开物理便宜的原因。

较新的版本还有 Physics Control 插件（`UPhysicsControlComponent`），提供更细的"控制 + 身体修饰器"模型来驱动物理角色，定位是取代 Physical Animation 组件的更灵活方案；它在 5.x 早期是实验性功能，具体状态以手上的引擎版本为准。软体方面有 Chaos Flesh 插件，同样是实验性的。

## 容易踩的坑

**碰撞体之间互相碰撞。** 相邻骨骼的碰撞体如果重叠又没关掉相互碰撞，一开模拟就会互相推开导致抖动甚至炸开。Physics Asset Editor 里默认会禁用相邻身体的碰撞，但手动加的身体要自己检查。

**把 IK 当物理。** IK 不会产生惯性，也不会被推开；需要"被撞了会晃"的效果时要用物理动画，而不是 IK。

**模拟结果依赖帧率。** 低帧率下步长变大，约束容易失效。UE 物理有子步（substepping）选项，布料也有独立的迭代次数和子步设置，出问题先看这些。

**开关物理时姿势跳变。** 直接 `SetSimulatePhysics(true)` 的那一帧，物理身体从当前动画姿势开始模拟，这通常没问题；反过来从物理回到动画时，如果直接关掉，姿势会瞬间跳回动画。要用 Physics Blend Weight 渐变，或者先截取物理姿势（Pose Snapshot）再过渡到起身动画。

## 相关

[[EAABAA-IK]] [[EAABAB-布料与流体模拟]] [[EAABAC-布娃娃系统]] [[EAABH-3D蒙皮动画]] [[ECAB-IK和FK]]
