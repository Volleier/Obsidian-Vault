**交叉淡化（Cross Fade）是在一段时间内把旧动画的权重从 1 降到 0、同时把新动画的权重从 0 升到 1 的过渡方式，两段动画在这段时间里都在播放、逐帧按权重混合。UE 状态机过渡的默认混合方式 Standard Blend、`Blend Poses by bool/int/enum` 的 Blend Time、蒙太奇的 Blend In/Out 都是交叉淡化。**

> 参考：Jason Gregory《Game Engine Architecture》动画混合一章（smooth / frozen transition 的说法来自这里）。UE 属性名对照 Epic 5.8 文档 *Transition Rules*、*Blend Masks and Blend Profiles*、*Sync Groups*。

## 为什么不能直接切

动画之间硬切，骨骼在一帧之内从一个姿势跳到另一个姿势，屏幕上就是明显的“跳帧”：手臂瞬移、脚滑、身体抖一下。交叉淡化用一段时间 $D$ 把跳变摊开。设过渡开始后经过的时间为 $\tau$，归一化进度 $u = \tau / D$，则

$$
w_B = f(u),\qquad w_A = 1 - f(u),\qquad \text{Pose} = \operatorname{blend}(\text{Pose}_A, \text{Pose}_B, w_B)
$$

$f$ 是缓动曲线，最简单的是 $f(u) = u$（线性）。blend 就是逐骨骼的平移、缩放 Lerp 加旋转 NLerp，见 [[EBAC-线性插值]]。

## 平滑过渡和冻结过渡

《Game Engine Architecture》把交叉淡化分成两种：

| 类型 | 旧动画在过渡期间 | 适用 |
| --- | --- | --- |
| 平滑过渡（smooth transition） | 继续播放 | 两段动画节奏能对上，例如走和跑都是循环步态 |
| 冻结过渡（frozen transition） | 停在切换那一帧不动 | 两段动画毫无关系，例如从跑步转到受击 |

平滑过渡要求两段动画在时间上“对齐”。如果走路循环的左脚落地在 0.3 秒，跑步循环的左脚落地在 0.1 秒，直接混合就会出现两条腿互相打架的中间姿势。UE 用同步组（Sync Groups）解决这个问题：参与混合的动画放进同一个同步组，权重最大的成为领导者（leader），其余作为跟随者（follower）调整播放速率去匹配领导者的周期，跟随者的 Notify 会被抑制。更精确的做法是在动画里打同步标记（Sync Markers），例如每次左脚、右脚落地各一个，同步时按标记对齐相位，而不是只按总长度缩放。同步组的角色可以设为 `Can be Leader`、`Always Follower`、`Always Leader`、`Transition Leader`、`Transition Follower`。

状态机过渡还有一个相关选项 `Sync Group Name to Require Valid Markers Rule`：只有当前状态的动画带有指定同步组的有效标记时，这条过渡才会被使用。

## 缓动曲线

线性交叉淡化在开始和结束两个时刻权重的变化率突变，看起来有点“机械”。常用的做法是换成两端导数为 0 的曲线，比如三次 Hermite $f(u) = 3u^2 - 2u^3$。UE 过渡的 `Mode`（类型 `EAlphaBlendOption`）提供了一组现成曲线：

| 分类 | 选项 |
| --- | --- |
| 线性 | `Linear` |
| 三次 | `Cubic`、`HermiteCubic`、`CubicInOut` |
| 正弦 | `Sinusoidal` |
| 多项式缓入缓出 | `QuadraticInOut`、`QuarticInOut`、`QuinticInOut` |
| 圆弧 | `CircularIn`、`CircularOut`、`CircularInOut` |
| 指数 | `ExpIn`、`ExpOut`、`ExpInOut` |
| 自定义 | `Custom`，配合 `Custom Blend Curve` 指定一个曲线资产 |

编辑器里按住 Ctrl + Alt 悬停在选项上能预览曲线形状。

## 在 UE 里的设置位置

选中状态机里的一条过渡，Details 面板中：

| 属性 | 作用 |
| --- | --- |
| `Blend Logic` | `Standard Blend` 就是交叉淡化；另有 `Inertialization`、`Custom` |
| `Duration` | 过渡时长（秒） |
| `Mode` | 缓动曲线 |
| `Custom Blend Curve` | `Mode` 为 `Custom` 时使用的曲线资产 |
| `Blend Profile` | 让不同骨骼以不同速度完成过渡 |
| `Transition Crossfade Sharing` | 把这组混合设置提升为共享设置，多条过渡共用，改一处全改 |

`Blend Profile` 存放在骨骼资产里，分两种：Time Blend Profile 的值是时长乘数（1 为正常，越小越快，0 表示瞬间完成）；Weight Blend Profile 的值是速度倍数（1 为正常，2 表示快一倍）。典型用法是从待机转到移动时让腿部比上半身更快完成过渡，起步时脚先动、停步时脚先站稳。Blend Profile 也能用在 `Blend Poses by bool/int/enum` 节点和蒙太奇的 `Blend Profile In/Out` 上。

`Custom` 混合逻辑会打开一张单独的混合图，在里面可以用 `State Weight`（源状态权重，从 1 降到 0）、`Get Transition Time Elapsed (ratio)`（从 0 升到 1）、`Get Transition Crossfade Duration` 等函数自己拼混合。用一个普通 `Blend` 节点接 `Get Transition Time Elapsed (ratio)`，就等于复刻了 Standard Blend。

## 交叉淡化的代价与惯性化

交叉淡化期间两个源姿势都要完整求值：旧状态里的混合空间、IK、各种节点都照跑。过渡时间越长、同时进行的过渡越多，开销越大。过渡中如果又被打断转向第三个状态，状态机会同时维护多个正在淡出的状态。

惯性化（Inertialization）是另一条思路：切换那一刻就只求值新姿势，把“旧姿势与新姿势的差”连同切换瞬间的速度作为一个偏移，在短时间内平滑衰减到零。它是一种后处理，省掉了旧姿势的求值，但旧动画上的 Notify 在过渡开始后也不再触发。UE 里要在状态机之后放 Inertialization 节点（实验性的 Dead Blending 节点是另一种实现，它预测旧动画的后续运动再与新动画混合，并且会使用过渡曲线，Inertialization 节点则忽略曲线）。官方建议惯性化时长控制在 0.4 秒以内，两个姿势差别极大时不要用。

| | 交叉淡化 | 惯性化 |
| --- | --- | --- |
| 过渡期间求值 | 新旧两个姿势 | 只有新姿势 |
| 旧动画的 Notify | 继续触发 | 不再触发 |
| 适合的时长 | 可长可短 | 短（< 0.4 s） |
| 姿势差别很大时 | 中间姿势可能难看，但可控 | 效果可能不好 |
| 额外要求 | 无 | 图里要有 Inertialization / Dead Blending 节点 |

## 容易踩的坑

**步态动画不做同步。** 走跑混合时脚打架，多半是没放进同一个同步组，或者动画长度差别过大导致跟随者播放速率变化明显。

**过渡时长配得太长。** 两段动画长时间“叠影”，角色显得软、反应迟钝；同时两个状态都在求值，开销也翻倍。

**混合设置到处复制。** 几十条过渡各写一遍 0.2 秒，改起来很痛苦。用 `Transition Crossfade Sharing` 提升为共享设置。

**切到惯性化后 Notify 丢了。** 依赖旧动画末尾 Notify 的游戏逻辑（比如攻击判定结束）需要改到别处触发。

## 相关

[[EBC-动画状态机]] [[EBAC-线性插值]] [[EAADO-姿势的插值]] [[EBCE-分层动画状态机]]
