**分部混合（Skeleton Masked Blending，也叫局部混合、per-bone blending）给每根骨骼分配各自的混合权重，让一段动画只作用于骨骼树的一部分，例如上半身播换弹、下半身继续跑。本质是把普通混合的一个全局权重 $\alpha$ 换成逐骨骼的权重 $\alpha \cdot m_j$，$m_j$ 来自骨骼遮罩。UE 里对应 AnimGraph 的 `Layered blend per bone` 节点，遮罩用 Branch Filter 或骨骼资产上的 Blend Mask 定义。**

> UE 部分对照 Epic 5.8 文档 *Blend Nodes*、*Blend Masks and Blend Profiles*、*Using Layered Animations*。

## 从全局权重到逐骨骼权重

普通混合对所有骨骼用同一个权重：

$$
\mathbf{x}_j = \operatorname{blend}(\mathbf{x}^{A}_j, \mathbf{x}^{B}_j, \alpha)\quad\text{对所有骨骼 } j
$$

分部混合给每根骨骼一个遮罩值 $m_j \in [0, 1]$：

$$
\mathbf{x}_j = \operatorname{blend}(\mathbf{x}^{A}_j, \mathbf{x}^{B}_j, \alpha\, m_j)
$$

$m_j = 0$ 的骨骼完全保留基础姿势 $A$，$m_j = 1$ 的骨骼在 $\alpha = 1$ 时完全取 $B$。blend 仍是平移、缩放 Lerp 加旋转 NLerp（[[EBAC-线性插值]]）。这就是它和叠加混合的根本区别：分部混合是在指定骨骼上“替换”，叠加混合是在所有骨骼上“加差值”（[[EBDC-叠加混合（Additive Blending）]]）。两者经常一起用：上半身用分部混合换成持枪动作，再叠加瞄准偏移。

## 遮罩边界的问题

遮罩在骨骼树上从 0 突变到 1 时，边界处的两根骨骼分别来自两个动画。局部空间下每根骨骼的旋转是相对父骨骼的，如果下半身在跑步中骨盆左右扭动，而上半身完全取换弹动画的局部旋转，上半身就会被骨盆带着一起扭：换弹动画里的“胸口朝前”是相对于换弹动画自己的骨盆，放到跑步的骨盆上就歪了。

有两种处理：

- **渐变遮罩。** 让权重沿脊椎逐节增加，例如 `spine_01` 0.3、`spine_02` 0.6、`spine_03` 1.0，把扭动分摊到几节脊椎上。
- **在网格空间混合旋转。** 对上半身骨骼，不混合局部旋转，而是混合它们在组件空间里的朝向，再换算回局部。这样上半身在世界里的朝向由上层动画决定，不受下半身骨盆扭动影响。UE 的 `Mesh Space Rotation Blend` 就是这个开关，另有 `Mesh Space Scale Blend` 对缩放做同样处理。

上半身持枪瞄准时，枪口必须指向准星，通常要开 Mesh Space Rotation Blend；而像挥手这种希望跟着身体一起动的动作，局部空间混合反而更自然。

## 在 UE 里：Layered blend per bone

节点输入是一个 `Base Pose` 和若干个 `Blend Poses N`（右键 `Add Blend Pin` 增加），每个混合姿势有自己的 `Blend Weights N`。Details 面板里的主要属性：

| 属性 | 作用 |
| --- | --- |
| `Blend Mode` | `Branch Filter` 或 `Blend Mask` |
| `Layer Setup` → `Branch Filters` | 每项一个 `Bone Name` 和 `Blend Depth` |
| `Blend Masks` | `Blend Mode` 为 Blend Mask 时，选骨骼资产上定义的遮罩 |
| `Mesh Space Rotation Blend` | 旋转在组件空间混合 |
| `Mesh Space Scale Blend` | 缩放在组件空间混合 |
| `Curve Blend Option` | 动画曲线怎么合并 |
| `Blend Root Motion Based on Root Bone` | 根运动按根骨骼的权重混合 |

**Branch Filter 的 `Blend Depth`：**

| 取值 | 效果 |
| --- | --- |
| 0 | `Bone Name` 及其所有子骨骼权重都为 1 |
| 正数 | 从 `Bone Name` 开始经过若干根骨骼逐渐增加到 1。文档的例子：Blend Depth 为 2 时，`Bone Name` 权重 0.5，下一根子骨骼权重 1 |
| 负数 | 禁用混合姿势、偏向 Base Pose；小于 -1 时同样在若干根骨骼内渐变 |

正的 Blend Depth 就是上一节说的“渐变遮罩”的快捷写法。Epic 的分层动画教程用的配置是 `Bone Name` = `spine_01`、`Blend Depth` = 1、勾选 `Mesh Space Rotation Blend`。

**Curve Blend Option 的取值：** Override、Do Not Override、Normalize by Weight、Blend by Weight、Use Base Pose、Use Min Value、Use Max Value。动画曲线（比如驱动面部或材质的曲线）不属于任何骨骼，遮罩管不到它们，需要单独决定怎么合并。

## Blend Mask

Branch Filter 写在节点上，每个用到的节点都要重复配置。Blend Mask 把遮罩存到骨骼资产里：在骨骼编辑器的 Skeleton Tree 中点 `Options > Add Blend Mask`，Skeleton Tree 会多出一列 Blend，每根骨骼填 0～1 的值，右键骨骼选 `Recursively Set Blend Scales` 可以一次给所有子骨骼设同一个值。节点里把 `Blend Mode` 设为 `Blend Mask` 并选择这张遮罩即可。多个动画蓝图共享同一套上半身遮罩时，改一处全部生效。修改 Blend Mask 就是在修改骨骼资产。

Blend Mask 和 Blend Profile 都放在同一列、同一个菜单里，名字要起清楚：

| | Blend Mask | Blend Profile |
| --- | --- | --- |
| 控制 | 每根骨骼的混合权重（参不参与、参与多少） | 每根骨骼的混合速度（多快完成过渡） |
| 用在 | `Layered blend per bone` | 状态机过渡、`Blend Poses by bool/int/enum`、蒙太奇 Blend In/Out |
| 类型 | 一种 | Time Blend Profile、Weight Blend Profile |

## 和蒙太奇配合

最常见的用法是“移动中播上半身动作”：

1. 在蒙太奇里把动画放进一个自定义 Slot，例如 `UpperBody`；
2. AnimGraph 里移动状态机输出先 `Save Cached Pose`；
3. 缓存姿势一路接 `Layered blend per bone` 的 `Base Pose`，另一路接 `Slot 'UpperBody'` 节点再接到 `Blend Poses 0`；
4. 按上面的方式配置 Branch Filter。

没有蒙太奇在播时，Slot 节点直接透传输入的移动姿势，混合结果等于纯移动；播放蒙太奇时，上半身被替换。所有使用 `UpperBody` Slot 的蒙太奇都会自动走这条路径。

需要更细粒度的控制时，`Blend Bone by Channel` 可以指定具体某根骨骼从另一根骨骼取平移、旋转或缩放中的某些通道，并选择在世界、组件、父骨骼或骨骼空间计算，适合单根骨骼的修正而不是整块身体的替换。

## 容易踩的坑

**上半身跟着骨盆扭。** 局部空间混合的典型症状，勾 `Mesh Space Rotation Blend` 或者用渐变遮罩。

**边界处骨骼断裂或抖动。** Blend Depth 为 0 时权重在一根骨骼处突变，两套动画节奏差别大时会很明显，用正的 Blend Depth 做过渡。

**只遮了骨骼，没管曲线。** 面部、材质曲线可能被下层或上层意外覆盖，检查 `Curve Blend Option`。

**根运动被上半身动画影响。** 上半身蒙太奇带根运动时，默认也可能被混进来，用 `Blend Root Motion Based on Root Bone` 或在动画上关掉根运动。

**直接连同一个状态机两次。** Base Pose 和 Slot 输入都来自移动状态机时，用缓存姿势，避免同一子图被求值两遍。

## 相关

[[EBDC-叠加混合（Additive Blending）]] [[EBCE-分层动画状态机]] [[EBCB-Cross Fades]] [[EBAC-线性插值]] [[EAADO-姿势的插值]]
