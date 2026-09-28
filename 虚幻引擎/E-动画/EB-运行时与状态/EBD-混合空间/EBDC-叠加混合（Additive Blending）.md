**叠加混合（Additive Blending）不在两个姿势之间取中间值，而是先算出一段动画相对于某个参考姿势的“差值”，再把这个差值按权重加到任意基础姿势上。它让一段小动作（点头、呼吸、受击抖动、瞄准方向）可以叠在走、跑、蹲等任何动作之上。UE 里在动画序列的 Additive Settings 中把动画设为 Local Space 或 Mesh Space 叠加类型，在 AnimGraph 里用 `Apply Additive` / `Apply Mesh Space Additive` 应用，`Make Dynamic Additive` 可以在运行时临时生成差值。**

> 参考：《Game Engine Architecture》动画混合一章（additive blending、difference clip）。UE 部分对照 Epic 5.8 文档 *Blend Nodes*、*Aim Offset* 和 API 枚举 `EAdditiveAnimationType`、`EAdditiveBasePoseType`。

## 普通混合为什么不够

普通混合是插值：$\operatorname{blend}(A, B, \alpha)$ 的结果永远在 $A$ 和 $B$ 之间，权重和为 1。想让角色“走路时点头”，用普通混合只能把走路和一个点头的全身姿势按比例混，点头权重越大，走路的腿部动作就被冲淡得越多。要让点头只影响头部，就得用骨骼遮罩（见 [[EBDD-分部混合（Skeleton Masked Blending）]]），但遮罩是替换，点头动画里那根脖子的朝向是在站立姿势下做的，放到弯腰奔跑的身体上就对不上了。

叠加混合换了一种表述：点头动画不记录“脖子朝哪”，只记录“脖子相对参考姿势多转了多少”。这个增量加到任何基础姿势上都成立，站着点头、跑着点头、蹲着点头用的是同一份数据。美术只需做一套差值，就能和一整套移动动画组合，这是它最大的价值：组合数从乘法变成加法。

## 数学

记参考姿势（reference）为 $R$，源动画为 $S$，基础姿势为 $B$。对每根骨骼的局部变换，先求差值 $D$：

$$
\Delta\mathbf{t} = \mathbf{t}_S - \mathbf{t}_R,\qquad
\Delta\mathbf{q} = \mathbf{q}_S\,\mathbf{q}_R^{-1},\qquad
\Delta\mathbf{s} = \frac{\mathbf{s}_S}{\mathbf{s}_R} - 1
$$

再按权重 $\alpha$ 应用到基础姿势上：

$$
\mathbf{t} = \mathbf{t}_B + \alpha\,\Delta\mathbf{t},\qquad
\mathbf{q} = \operatorname{slerp}(\mathbf{1}, \Delta\mathbf{q}, \alpha)\,\mathbf{q}_B,\qquad
\mathbf{s} = \mathbf{s}_B \odot (\mathbf{1} + \alpha\,\Delta\mathbf{s})
$$

旋转的“差”是相对旋转而不是分量相减；按权重缩放差值时，是从单位四元数 $\mathbf{1}$ 往 $\Delta\mathbf{q}$ 插值。缩放按比例存储，是为了让多个缩放差值之间能线性混合。当 $B = R$、$\alpha = 1$ 时，结果精确还原 $S$；基础姿势偏离参考姿势越远，叠加结果就越“外推”，这是叠加动画出错的根源。UE 里对应的实现在 `FAnimationRuntime::ConvertPoseToAdditive`（生成差值）和 `FTransform::BlendFromIdentityAndAccumulate`（带权重应用），四元数乘法的左右顺序以源码为准。

几个性质：差值之间可以直接普通混合（瞄准偏移就是在若干差值之间做混合空间插值，再整体叠加）；$\alpha$ 可以大于 1，用来夸张动作；多个叠加层可以依次叠上去，但顺序会影响旋转结果，因为四元数乘法不交换。

## Local Space 和 Mesh Space

差值在哪个空间里求，决定了它叠到别的姿势上时“往哪个方向转”：

| 叠加类型（`Additive Anim Type`） | 枚举值 | 差值的含义 | 适合 |
| --- | --- | --- | --- |
| No additive | `AAT_None` | 普通动画 | |
| Local Space | `AAT_LocalSpaceBase` | 每根骨骼相对父骨骼的局部变换差值，包含平移、旋转、缩放 | 呼吸、受击、表情这类依附身体局部的动作 |
| Mesh Space | `AAT_RotationOffsetMeshSpace` | 骨骼在组件（网格）空间下的旋转偏移 | 瞄准偏移、需要朝绝对方向转的动作 |

局部空间差值跟着父骨骼走。角色侧身倾斜时，脊椎局部坐标系已经歪了，局部空间的“向上抬枪”叠上去，枪口就会朝倾斜后的“上”方向偏，不再是真正的上方。Mesh Space 把旋转偏移放在骨骼网格组件的坐标系里应用，不管前面的骨骼怎么歪，都朝同一个绝对方向转。Epic 的 Aim Offset 文档用的正是这个例子，并且规定 Aim Offset 的样本必须设为 Mesh Space。

## 在 UE 里设置

叠加类型是动画序列资产上的属性，在资产的 Asset Details 面板 Additive Settings 分类下：

| 属性 | 作用 |
| --- | --- |
| `Additive Anim Type` | No additive / Local Space / Mesh Space |
| `Base Pose Type` | 参考姿势从哪来：Skeleton Reference Pose（骨骼参考姿势）、Selected animation scaled（另一段动画，按时间缩放逐帧对应）、Selected animation frame（另一段动画的某一帧）、Frame from this animation（本动画的某一帧） |
| `Base Pose Animation` | 参考动画 |
| `Ref Frame Index` | 参考帧序号 |

以瞄准偏移为例，Epic 文档给出的设置是：每个瞄准样本 `Additive Anim Type` 设为 Mesh Space，`Base Pose Type` 设为 Selected animation frame，`Base Pose Animation` 设为向前瞄准的中性动画，`Ref Frame Index` 为 0。

AnimGraph 里的节点：

| 节点 | 作用 |
| --- | --- |
| `Apply Additive` | 把局部空间叠加姿势按 Alpha 加到 Base 上 |
| `Apply Mesh Space Additive` | 同上，用于 Mesh Space 叠加姿势 |
| `Make Dynamic Additive` | 输入 Base 和 Additive 两个普通姿势，运行时求差值；Details 里可以勾 Mesh Space |
| Aim Offset 节点 | 在 Mesh Space 叠加样本之间做混合空间插值后叠到输入姿势上 |
| `Blend Multi`（勾 `Additive Node`） | 多个叠加姿势之间混合 |

两个 Apply 节点都受 LOD Threshold 控制：网格 LOD 超过阈值时跳过叠加，远处角色省掉这部分开销。叠加姿势的引脚和普通姿势不能混接，蒙太奇也能播放叠加动画，叠加蒙太奇应该播到接在 `Apply Additive` 上的 Slot 里。

## 常见用法

**瞄准偏移。** 以向前瞄准为参考，做上下左右若干方向的差值，放进 Aim Offset，按 Pitch、Yaw 插值后叠到任何持枪移动动画上。

**待机变化和呼吸。** 一段轻微起伏的叠加循环，叠在所有状态之上，让角色不再僵硬。

**受击反馈。** 受击方向各做一段短叠加，叠在当前任何动作上，不必打断移动。

**倾斜。** 转弯时按角速度叠一个侧倾差值。

**用 `Make Dynamic Additive` 做姿势修正。** 例如把“持重物的姿势”与“空手姿势”求差，叠到各种移动动画上，省掉一整套持重物移动动画。

## 容易踩的坑

**参考姿势选错。** 叠加动画是“相对参考姿势”的差值，参考姿势和实际基础姿势差得越远，结果越怪。参考帧应该是这组动画真正会叠上去的那类姿势，而不是随手用 T-pose。

**瞄准偏移用了 Local Space。** 角色倾斜或弯腰时瞄准方向跑偏。Aim Offset 的样本一律设为 Mesh Space。

**同一个混合空间里混用叠加类型。** 混合空间要求所有样本的叠加类型一致，否则编辑器会报错拒绝。

**顺序依赖。** 多个叠加层叠在一起时，旋转部分的先后会影响结果。决定好顺序并固定下来。

**把普通动画接到叠加节点上。** 动画没设成叠加类型时，引擎把整个姿势当作“差值”叠上去，角色会扭成一团。

**缩放叠加。** 缩放差值按比例存储，参考姿势里骨骼缩放为 0 或极小的时候会出问题；一般让叠加动画不带缩放变化。

## 相关

[[EBDD-分部混合（Skeleton Masked Blending）]] [[EBCE-分层动画状态机]] [[EBAC-线性插值]] [[EAADK-球面线性插值]] [[EAADO-姿势的插值]]
