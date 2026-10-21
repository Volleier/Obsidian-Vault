**布料模拟和流体模拟是基于物理的动画里处理"可变形连续介质"的两类方法：布料是二维曲面，用质点加约束（弹簧或位置约束）来近似；流体是三维体积，用网格（欧拉法）或粒子（拉格朗日法）离散 Navier–Stokes 方程。游戏里布料已经是角色的标配，UE 对应 Chaos Cloth；实时流体仍然昂贵，多数时候靠预计算或着色器近似，UE 里有 Niagara Fluids 插件和 Water 插件。**

> 参考：Müller et al., *Position Based Dynamics*（2007）；Macklin et al., *XPBD*（2016）；Bridson《Fluid Simulation for Computer Graphics》；UE 部分对照 5.x 的 Chaos Cloth 与 Niagara Fluids 官方教程。

## 布料：从质点弹簧到位置约束

最早的实时布料模型是**质点—弹簧系统**：把布料网格的每个顶点当作一个质点，相邻顶点之间连弹簧。弹簧按作用分三类：结构弹簧连接横竖相邻的点，抵抗拉伸；剪切弹簧连接对角点，抵抗斜向变形；弯曲弹簧跨一个点连接，抵抗折叠。每根弹簧的力按胡克定律加阻尼：

$$
\mathbf{F}_{ij} = -\left[k_s\left(\lVert\mathbf{x}_i-\mathbf{x}_j\rVert - L_{ij}\right) + k_d\,(\mathbf{v}_i-\mathbf{v}_j)\cdot\hat{\mathbf{d}}\right]\hat{\mathbf{d}},\qquad \hat{\mathbf{d}} = \frac{\mathbf{x}_i-\mathbf{x}_j}{\lVert\mathbf{x}_i-\mathbf{x}_j\rVert}
$$

问题在于真实布料几乎不可拉伸，对应非常大的 $k_s$。显式积分下刚度越大，允许的时间步越小，否则系统会发散；游戏帧率下为了稳定只能把布料调得像橡皮筋。

**基于位置的动力学（PBD）** 换了思路：不算力，而是先按惯性预测位置，再直接把位置"投影"回满足约束的状态。对长度为 $L$ 的距离约束，两个质点（逆质量 $w_1, w_2$）的修正量是

$$
\Delta\mathbf{x}_1 = -\frac{w_1}{w_1+w_2}\left(\lVert\mathbf{x}_1-\mathbf{x}_2\rVert - L\right)\hat{\mathbf{d}},\qquad
\Delta\mathbf{x}_2 = +\frac{w_2}{w_1+w_2}\left(\lVert\mathbf{x}_1-\mathbf{x}_2\rVert - L\right)\hat{\mathbf{d}}
$$

所有约束依次投影若干遍，最后用位置差更新速度。PBD 在大步长下也不会炸，代价是刚度和迭代次数、步长耦合在一起，参数没有明确的物理意义。XPBD 引入柔度（compliance）解决了这个问题，让刚度与迭代次数无关。

角色布料还有一个游戏特有的问题：布料必须"跟着角色走"，但不能完全自由。常见的约束手段是给每个顶点刷一个最大距离（Max Distance）：顶点最多偏离蒙皮位置多远，值为 0 就完全由蒙皮驱动（运动学顶点）。再加上 Backstop（防止顶点穿到身体内侧的单侧约束）和长程附着（Long Range Attachment / tether：动态顶点到最近固定区域的距离不超过测地线长度，防止低迭代时布料下垂拉长），再和身体的胶囊体做碰撞，基本就是一套游戏布料的全部。

## 流体：欧拉网格与拉格朗日粒子

不可压缩流体由 Navier–Stokes 方程描述：

$$
\rho\left(\frac{\partial\mathbf{u}}{\partial t} + \mathbf{u}\cdot\nabla\mathbf{u}\right) = -\nabla p + \mu\nabla^{2}\mathbf{u} + \mathbf{f},\qquad \nabla\cdot\mathbf{u} = 0
$$

左边是加速度（含对流项），右边依次是压力梯度、粘性和外力；第二个式子是不可压缩条件。离散方式有两大类：

| 方式 | 代表方法 | 思路 | 游戏里的适用性 |
| --- | --- | --- | --- |
| 欧拉法（网格） | Stable Fluids（Stam 1999） | 在固定网格上存速度、密度，每步做对流、外力、压力投影 | 适合烟、火、气体；2D 网格可以实时，3D 网格开销大 |
| 拉格朗日法（粒子） | SPH（光滑粒子流体动力学） | 每个粒子携带质量和速度，用核函数从邻居估计密度和压力 | 适合水花、飞溅；粒子数一多邻域搜索很贵，不可压缩性难保证 |
| 混合 | PIC / FLIP | 粒子携带速度，在网格上解压力再传回粒子 | 离线特效的主流水体方法，也有实时 GPU 实现 |
| 界面追踪 | VOF、Level Set | 在网格法之上追踪液面位置 | 离线为主 |

原笔记里把 VOF 写成"体素流体模拟"，其实 VOF 是 Volume of Fluid，指用每个网格单元的液体体积分数来表示液面，是界面追踪手段，不是独立的求解方法。

真正在游戏里跑完整流体求解的情况很少。常见的做法是：海洋和湖面用 Gerstner 波或 FFT 频谱在顶点着色器里生成；河流用流向贴图（flow map）滚动法线；烟火爆炸用序列帧或预先模拟好的体积缓存；只有局部、小范围的交互（角色蹚水的涟漪、局部烟雾）才用实时模拟，而且多是 2D 网格或高度场。

## 在 UE 里

**Chaos Cloth** 是 UE5 的布料系统，基于位置的求解器，官方教程里可以在 PBD 和 XPBD 两种模型间选择。传统工作流是在骨骼网格体编辑器里选中一个材质 Section，从它创建 Clothing Data，然后在布料绘制工具里刷 Max Distance、Backstop 等权重图，配置写在 `UChaosClothConfig` 和共享配置里。Max Distance 为 0 的顶点是运动学的，Long Range Attachment 的固定端通常就取这些顶点。

5.3 前后，Epic 开始用新的 **Cloth Asset**（配合 Dataflow 图和 Panel Cloth Editor）替代旧的 Clothing Data 编辑流程，并在 5.5～5.8 持续更新（Outfit 资产、重拟合等）。这套新流程标为实验性，节点在版本之间有过废弃和替换，按官方的说法迁移时要以当前版本的默认模板为准。

**Niagara Fluids** 是一个需要手动启用的插件，提供基于网格的 2D / 3D 气体模拟、FLIP 液体和浅水模拟的模板发射器（例如 Grid 3D Gas Master Emitter），跑在 GPU 上，适合局部的烟火和水花效果。大面积水体则用 **Water** 插件（水体 Actor、河流样条、海岸线），它的波浪是程序化生成的，不是流体求解。

角色身上的柔体（肌肉、脂肪）有 Chaos Flesh 插件，目前是实验性的；更常见的做法是用 [[EAABI-变形目标动画]] 或 ML Deformer 近似。

## 容易踩的坑

**布料和身体一起穿模。** 刷了 Max Distance 却没设 Backstop，快速动作时布料会跑进身体内侧。先刷 Max Distance 确定活动范围，再用 Backstop 挡住内侧。

**低迭代时布料被拉长。** 位置约束迭代次数不够时，长裙、披风会越来越长。优先用 Long Range Attachment 这种一次就能纠正整体长度的约束，而不是一味加迭代。

**角色瞬移后布料乱飞。** 角色被传送或动画突变时，布料会以为身体瞬间高速移动。`USkeletalMeshComponent` 上有 `ClothTeleportDistThreshold` / `ClothTeleportRotThreshold` 两个阈值，超过就按传送处理；代码里瞬移角色时也可以主动调用 `ForceClothNextUpdateTeleportAndReset()`。

**把实时流体当默认方案。** 3D 网格的分辨率每翻一倍，单元数乘 8。先问效果能否用序列帧、体积缓存或着色器近似，确实需要交互再上模拟。

## 相关

[[EAABA-基于物理的动画]] [[EAABAC-布娃娃系统]] [[EAABF-逐顶点动画]] [[EAABH-3D蒙皮动画]]
