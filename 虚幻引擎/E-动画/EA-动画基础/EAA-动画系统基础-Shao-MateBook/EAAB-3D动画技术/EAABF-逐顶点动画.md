**逐顶点动画（Per-Vertex Animation）是直接记录网格每个顶点在每一帧的位置（有时连同法线），播放时逐帧替换或插值顶点位置的动画方式。它不依赖骨骼，能表现任意形变，代价是数据量和顶点数、帧数同时成正比。《Game Engine Architecture》认为它数据量太大，在实时游戏里用途有限；如今它主要以两种形式出现：Alembic 导入的几何缓存（UE 的 Geometry Cache），以及把顶点位置烘焙进贴图、在顶点着色器里回放的顶点动画贴图（VAT，UE 里可用 AnimToTexture 插件生成）。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（Per-Vertex Animation and Morph Targets）；UE 部分对照 Alembic 导入文档和 AnimToTexture 插件（5.4 起随引擎发布，位于 Experimental 插件目录）。

## 为什么数据量是主要问题

骨骼动画每帧存的是几十到几百个关节的变换，逐顶点动画每帧存的是几千到几十万个顶点的位置。算一笔账：一个 1 万顶点的网格，每个顶点位置 3 个 float（12 字节），30 帧/秒播放 5 秒共 150 帧，光位置数据就是

$$
10^4 \times 12\ \text{B} \times 150 \approx 18\ \text{MB}
$$

加上法线翻倍。同样的动作用 60 根骨骼的骨骼动画存，每根骨骼每帧 10 个 float（四元数 + 平移 + 缩放），只需要 $60 \times 40\ \text{B} \times 150 \approx 0.36\ \text{MB}$，还没算压缩。GEA 把各种动画方法看作不同程度的数据压缩：骨骼动画之所以成为主流，就是因为它用少量关节参数"压缩"了大量顶点的运动；逐顶点动画则完全没有压缩。

所以逐顶点动画适合的是骨骼表达不了、或者表达起来代价太大的形变：布料和流体的模拟结果、破碎和融化、植物生长、大量群众角色的廉价播放。原笔记里提到的面部表情，游戏里通常用 [[EAABI-变形目标动画]]（只存少量关键形状、运行时混合）或面部骨骼，而不是逐帧存储全部顶点。

## 两种典型形态

**几何缓存。** DCC 或模拟软件（Houdini、Maya、Marvelous Designer）把每帧的网格导出成 Alembic（`.abc`）文件，引擎按帧读取。它能完整保留模拟结果，甚至支持拓扑随帧变化（流体表面每帧顶点数都不同），但数据只能靠通用压缩和流式加载来控制，适合过场动画和影视级的一次性效果。

**顶点动画贴图（VAT）。** 把"第 $f$ 帧第 $i$ 个顶点的位置"写进一张贴图：横轴是顶点编号，纵轴是帧号，RGB 存位置（通常是相对于静止姿势的偏移，并归一化到包围盒内）。网格本身是一个普通的静态网格，每个顶点在 UV 通道里记录自己的编号，材质在顶点着色器里按时间采样贴图，把结果加到顶点位置上。

$$
\mathbf{p}_i(t) = \mathbf{p}_i^{\text{rest}} + \text{lerp}\left(\mathbf{d}_{i,\lfloor f\rfloor},\ \mathbf{d}_{i,\lfloor f\rfloor+1},\ f-\lfloor f\rfloor\right),\qquad f = t \cdot \text{fps}
$$

VAT 的优势在于：播放完全在 GPU 上，不需要 CPU 参与动画求值，每个实例只需要一个时间偏移参数，所以能配合实例化渲染同时画成千上万个。City Sample 的人群就是这样渲染的，远处的行人是实例化的静态网格加顶点动画贴图，近处再切换成真正的骨骼网格体。

VAT 也有一个变体，贴图里存的不是顶点位置而是骨骼变换（每帧每根骨骼一个变换），顶点着色器再按权重做蒙皮。这样贴图尺寸只和骨骼数有关，不随顶点数增长，适合高面数角色。

## 在 UE 里

**Alembic 导入。** UE 的 Alembic 导入器（需启用 Alembic Importer 插件）可以把 `.abc` 导入为三种形式：静态网格（只取一帧）、几何缓存（`UGeometryCache` 资产，由 `UGeometryCacheComponent` 播放，也可以在 Sequencer 里放 Geometry Cache 轨道），以及骨骼网格体（导入器用 PCA 把逐帧顶点数据压缩成若干基础形状，以变形目标的形式存储）。几何缓存保真度最高，适合过场动画；骨骼网格体形式体积更小，但会损失细节。

**AnimToTexture 插件。** 5.4 起随引擎发布，位于 `Engine/Plugins/Experimental/AnimToTexture`，默认不启用。它以一个数据资产为配置，把骨骼网格体的动画序列烘焙成贴图，支持逐顶点和逐骨骼两种模式，并自动更新对应的材质实例参数（起止帧等）。使用时的几个限制：它只为一个 LOD 生成数据，目标静态网格要把 LOD 数设为 1，否则切到其他 LOD 时动画会错乱；每个网格需要独立的材质实例，因为烘焙会把动画相关参数写进实例。

**材质回放。** 顶点动画在材质里通过 World Position Offset 输出。下面是 VAT 采样逻辑的示意，写成材质 Custom 节点里的 HLSL 风格，**不是**插件自带材质的源码：

```hlsl
// VAT 采样示意：VertexIndexUV 来自网格额外的 UV 通道，Time 为实例时间
float Frame = Time * FramesPerSecond;
float F0 = floor(Frame);
float Alpha = Frame - F0;
float V0 = (fmod(F0, NumFrames) + 0.5) / NumFrames;
float V1 = (fmod(F0 + 1.0, NumFrames) + 0.5) / NumFrames;

// 位置偏移归一化在 [0,1]，需还原到包围盒范围
float3 D0 = PositionTex.SampleLevel(PositionTexSampler, float2(VertexIndexUV.x, V0), 0).rgb;
float3 D1 = PositionTex.SampleLevel(PositionTexSampler, float2(VertexIndexUV.x, V1), 0).rgb;
float3 Offset = lerp(D0, D1, Alpha) * (BoundsMax - BoundsMin) + BoundsMin;
return Offset; // 接到 World Position Offset
```

采样必须用 `SampleLevel` 指定 mip 0，并且位置贴图要关闭 mip、用 Nearest 过滤、不做 sRGB 转换和有损压缩，否则相邻顶点的数据会被混在一起，网格表面出现毛刺。另外 WPO 移动的顶点不会更新网格的包围盒，动画幅度大时需要调大组件的 Bounds Scale，否则会在视野边缘被错误剔除。

## 容易踩的坑

**贴图压缩和过滤。** 这是 VAT 最常见的问题。位置、法线贴图必须是无损格式（如 16 位浮点 HDR），关闭 sRGB 和 mipmap，过滤用 Nearest。

**顶点顺序被改变。** VAT 依赖"顶点编号 ↔ 贴图列"的对应关系，导入时如果引擎重新排序或拆分了顶点（硬边、UV 接缝会拆分顶点），数据就对不上。导入设置里要保持顶点顺序，或者在烘焙时就以最终的顶点布局为准。

**拿 VAT 做主角。** VAT 没有混合、没有 IK、切换动作生硬，适合远景群众和固定循环的环境动画，不适合玩家角色。

**几何缓存体积失控。** 长时间、高面数的 Alembic 缓存很容易达到 GB 级，要提前规划流式加载，或者只把必须保留的片段做成几何缓存。

## 相关

[[EAABI-变形目标动画]] [[EAABH-3D蒙皮动画]] [[EAABC-刚性层级动画]] [[EAABAB-布料与流体模拟]] [[EAAAC-帧动画]]
