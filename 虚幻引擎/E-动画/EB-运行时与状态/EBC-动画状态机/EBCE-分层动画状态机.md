**分层动画状态机（Layered Animation State Machine）指同时运行多个状态机，每个状态机（层）负责身体的一部分或一类动作，最后按层的顺序、权重和骨骼范围把它们的输出合成一个姿势。Unity 的 Animator Layers 是直接做成这种形式的；UE 没有“状态机层”这个单独的资产，而是在 AnimGraph 里用多个 State Machine 节点，加上 Layered blend per bone、Apply Additive、Slot、Linked Anim Layers 这些节点组合出同样的结构。**

> 参考：《Game Engine Architecture》动画一章对 layered state machine 的描述；UE 部分对照 Epic 5.8 文档 *Blend Nodes*、*Animation Blueprint Linking*、*Blend Masks and Blend Profiles*。

## 单个状态机为什么不够

设想一个能边走边换弹、边跑边开枪、蹲着也能扔手雷的角色。如果只有一个状态机，每种组合都要一个状态：走+换弹、跑+换弹、蹲+换弹、走+开枪……状态数是各维度的乘积，过渡线更是爆炸。更糟的是，下半身的移动和上半身的动作本来互不相干，却被迫共享一个“当前状态”：换弹期间想从走切到跑，就得同时处理换弹进度该怎么保持。

分层的办法是承认这些维度互相独立，各自用一个状态机：一层管移动（待机、走、跑、跳），一层管上半身动作（空闲、瞄准、换弹、投掷），可能还有一层管叠加的呼吸、受击抖动。每层内部状态数只和自己那一维相关，组合交给最后的合成步骤。

## 层之间怎么合成

每一层输出一个完整姿势，合成方式决定这一层怎样影响最终结果：

| 合成方式 | 效果 | UE 里的节点 |
| --- | --- | --- |
| 覆盖（override），限定骨骼范围 | 这一层在指定骨骼上取代下层的姿势 | `Layered blend per bone`（Branch Filter 或 Blend Mask） |
| 叠加（additive） | 这一层的差值加到下层姿势上 | `Apply Additive`、`Apply Mesh Space Additive` |
| 按权重整体混合 | 两层姿势整体插值 | `Blend`、`Blend Multi` |
| 由游戏逻辑临时插入 | 蒙太奇播放时覆盖某一段 | `Slot` |

覆盖型分层依赖骨骼遮罩：上半身层只作用于从 `spine_01` 往下的子骨骼，下半身保持移动层的姿势。遮罩的写法和 Blend Depth 的含义见 [[EBDD-分部混合（Skeleton Masked Blending）]]。叠加型分层适合“在任何基础动作上都成立”的细节，例如呼吸、瞄准偏移、受击晃动，原理见 [[EBDC-叠加混合（Additive Blending）]]。

层的顺序有意义：先合成的层会被后面的层覆盖或叠加。一个常见的顺序是

1. 全身移动状态机，输出基础姿势；
2. 用 `Save Cached Pose` 缓存下来，后面多处复用而不重复求值；
3. 上半身动作状态机，通过 `Layered blend per bone` 从脊椎开始覆盖；
4. 蒙太奇 `Slot`，让一次性动作能临时接管；
5. 叠加层（瞄准偏移、呼吸）；
6. IK 和其他骨骼控制，放在最后修正脚和手的位置。

## 在 UE 里搭一个两层结构

AnimGraph 里放两个 State Machine 节点，一个叫 Locomotion，一个叫 UpperBody。Locomotion 的输出接 `Save Cached Pose`，命名为 `LocoCache`。然后：

- `Use Cached Pose 'LocoCache'` 接到 `Layered blend per bone` 的 `Base Pose`；
- UpperBody 状态机的输出接到 `Blend Poses 0`；
- `Layer Setup` 里加一个 Branch Filter，`Bone Name` 填 `spine_01`，`Blend Depth` 按需要设置；勾选 `Mesh Space Rotation Blend`，让上半身的朝向在组件空间里保持，而不是跟着下半身的骨盆扭动；
- `Blend Weights 0` 用变量控制，例如没有武器时设为 0，整层失效。

UpperBody 状态机内部的某些状态（比如“空闲”）可以直接引用 `LocoCache`，这样上半身没事可做时，输出和下半身完全一致，混合结果就是纯移动动画。这种“上层的默认状态透传下层姿势”的写法很常见，能避免上半身在空闲时被强行冻结在某个姿势。

`Layered blend per bone` 还可以改用 Blend Mask 模式：在骨骼资产的 Skeleton Tree 里通过 `Options > Add Blend Mask` 定义一张每骨骼 0～1 的权重表，节点里把 `Blend Mode` 设为 `Blend Mask` 并选中它。效果和 Branch Filter 类似，好处是遮罩存在骨骼资产上，可以在多个动画蓝图里复用。

## 用 Linked Anim Layers 把层拆出去

层多了之后，一个动画蓝图会变得很大，而且上半身层往往随武器变化：步枪、手枪、弓的上半身动画完全不同。UE 的 Linked Anim Layers 把“层”变成可以在运行时替换的模块：

1. 新建一个 Animation Layer Interface 资产，里面声明若干层，例如 `FullBody`、`UpperBody`、`IK`，需要的话给层加姿势输入；
2. 主动画蓝图在 Class Settings 里实现这个接口，把这几个层节点按顺序放进 AnimGraph，作为结构骨架，里面可以不写具体逻辑；
3. 每种武器各做一个动画蓝图，同样实现这个接口，在各层里写自己的状态机、瞄准偏移、IK；
4. 运行时调用 `Link Anim Class Layers`，`Target` 接角色的 Skeletal Mesh 组件，`In Class` 填武器的动画蓝图类，这些层的逻辑就替换进来。

武器 Actor 只引用自己的动画蓝图，于是只有装备了这把武器时，它那套动画资源才会被加载。Epic 文档里举的 Fortnite 例子同时用了两个接口，一个给武器、一个给载具：开车时载具层覆盖全身；坐在副驾时载具层管下半身坐姿，武器层管上半身，换武器只换上半身那部分。主图里的移动状态机（跳、落、落地、滑索）也不必为每种武器复制一份，武器只需覆盖对应状态里的层；不想覆盖的层留空即可。

层的切换只能通过惯性化混合，所以使用 Linked Anim Layers 的图里，层节点之后要有 Inertialization（或 Dead Blending）节点。同一个非默认 Group 名下的层会共享同一个动画实例，官方建议给接口里的层起一个非默认的 Group 名。

## 分层和嵌套的区别

| | 分层（多个状态机并行） | 嵌套（状态里再放状态机） |
| --- | --- | --- |
| 同时活跃的状态 | 每层一个，多个同时活跃 | 父状态机一个，子状态机在父状态活跃时才运行 |
| 解决的问题 | 相互独立的维度组合爆炸 | 单个维度内的状态过多、需要分组 |
| 典型例子 | 下半身移动 + 上半身持枪动作 | “空中”状态内部再分起跳、滞空、落地 |
| UE 里的实现 | 多个 State Machine 节点 + 混合节点，或 Linked Anim Layers | State 里放一个 State Machine 节点 |

两者经常一起用：移动层本身是嵌套的，上半身层与它并行。

## 容易踩的坑

**上半身跟着骨盆乱转。** 局部空间混合时，上半身骨骼的旋转相对于父骨骼，下半身的骨盆扭动会带着上半身一起转。在 `Layered blend per bone` 上勾 `Mesh Space Rotation Blend`。

**同一个状态机被求值两次。** 在两个地方直接连同一个子图，会求值两遍。先 `Save Cached Pose`，再在需要的地方 `Use Cached Pose`。

**层权重直接跳变。** 把上半身层的权重从 0 直接改成 1 会跳。权重本身要平滑变化，或者让上半身状态机内部从“透传下层”的状态过渡到动作状态。

**Linked Anim Layers 切换时姿势一跳。** 图里没有 Inertialization 节点，层切换的惯性化请求无人处理。

**根运动混乱。** 上半身层的动画带根运动时，默认也会被混进来。`Layered blend per bone` 有 `Blend Root Motion Based on Root Bone` 选项，按根骨骼的权重混合根运动。

## 相关

[[EBC-动画状态机]] [[EBCB-Cross Fades]] [[EBDD-分部混合（Skeleton Masked Blending）]] [[EBDC-叠加混合（Additive Blending）]]
