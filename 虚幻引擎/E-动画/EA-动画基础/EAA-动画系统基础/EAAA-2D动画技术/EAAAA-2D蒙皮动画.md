**2D 蒙皮动画（2D Skinned Animation）是把 2D 图片切成部件、挂到一套骨骼上，再让图片对应的多边形网格顶点按权重跟随多根骨骼变形的动画方式。它和 3D 的线性混合蒙皮是同一套数学，只是降到了平面上；代表工具是 Spine、Unity 的 2D Animation 包，Live2D 走的是另一条以变形器为主的路线。**

> 参考：Spine 用户手册与价格页（esotericsoftware.com）、spine-ue 运行时文档；3D 蒙皮部分的数学见《Game Engine Architecture》第 3 版动画系统一章。UE 相关内容对照 5.x。

## 从帧动画到骨骼，再到蒙皮

传统的 2D 动画是逐帧画出来的（见 [[EAAAC-帧动画]]）：角色走一步要画八张图，换一件衣服要把八张图全部重画。游戏里一个角色有几十个动作，每个动作几十帧，美术量和贴图内存都跟着帧数线性增长。

2D 骨骼动画先解决了"重复画"的问题：把角色拆成头、躯干、上臂、前臂、大腿、小腿等若干张图片，每张图片刚性地挂在一根骨骼上，动画只记录骨骼的旋转、平移、缩放。这其实就是 2D 版的[[EAABC-刚性层级动画]]，存储量从"每帧一张图"降到"每帧每根骨骼几个浮点数"，同一套部件还能被所有动作复用。它的毛病也和 3D 刚性层级动画一样：关节处两张图片是硬接的，手肘一弯就露出缝隙或重叠，只能靠美术把接缝处画成圆头、互相遮挡来掩饰。

蒙皮是在这个基础上再走一步：图片不再是一个矩形，而是被剖分成一张多边形网格（Spine 里叫 Mesh 附件），网格的每个顶点可以同时绑定到多根骨骼，各有一个权重。手肘附近的顶点一半跟上臂、一半跟前臂，弯曲时这块区域就被平滑地拉伸，而不是断开。代价是美术要额外做网格剖分和权重绘制，运行时要逐顶点做矩阵混合。

## 和 3D 蒙皮是同一个公式

2D 情况下每根骨骼的世界变换是一个 2D 仿射变换，用齐次坐标写成 $3\times3$ 矩阵（或者 Spine 内部用的 $a, b, c, d$ 加平移 $x, y$ 六个数）。设顶点在绑定姿势（Spine 里叫 Setup Pose）下相对骨骼 $j$ 的局部坐标为 $\mathbf{p}_j$，当前帧骨骼 $j$ 的世界变换为 $\mathbf{W}_j$，权重为 $w_j$，则

$$
\mathbf{p}' = \sum_{j} w_j \, \mathbf{W}_j \, \mathbf{p}_j ,\qquad \sum_j w_j = 1
$$

这和 3D 的线性混合蒙皮（LBS）完全一样，只是矩阵小一号。一个实现细节上的差别值得记住：3D 引擎通常存"绑定姿势下的模型空间顶点"加"逆绑定矩阵"，每帧算蒙皮矩阵 $K_j = B_j^{-1} C_j$（推导见 [[EAADH-蒙皮矩阵]]）；Spine 导出的加权网格则直接为每个"顶点—骨骼"对存一份骨骼局部坐标 $\mathbf{p}_j$，相当于把 $B_j^{-1}$ 预先乘进了顶点数据里。数据量更大，但运行时省掉了一次矩阵乘。

线性混合在 2D 下同样会出问题：两根骨骼夹角接近 180° 时，混合出的点会向关节中心塌缩，平面上表现为"扭麻花"或局部变窄。2D 角色的旋转幅度一般不大，美术通常靠多加一根过渡骨骼或调整权重来绕开。

## 主流工具的做法

| 工具 | 核心思路 | 网格 / 权重 | 与引擎的关系 |
| --- | --- | --- | --- |
| Spine | 骨骼 + 插槽（Slot）+ 附件（Attachment），动画曲线驱动骨骼 | Mesh、Deformation、Weights 只在 Professional 版提供；Essential 版不能保存或导出含网格、IK 的项目 | 导出 JSON 或二进制 `.skel` + `.atlas`，官方提供各引擎运行时 |
| Unity 2D Animation 包 | 在 Sprite Editor 的 Skinning Editor 里建骨骼、剖分网格、刷权重 | 支持 | 运行时由 `SpriteSkin` 组件做变形，直接进 Unity 动画系统 |
| Live2D Cubism | 以参数驱动的变形器（弯曲变形器、旋转变形器）为主，并非以骨骼为中心 | 网格变形为主 | 常用于立绘、VTuber，提供 SDK |
| DragonBones | 思路接近 Spine 的免费工具 | 支持 | 多家运行时，维护状态以官方仓库为准（未核实） |

Spine 的"插槽"值得单独说一下：插槽决定绘制顺序，也决定这个位置当前显示哪个附件。换装、换武器就是换附件，骨骼和动画完全不动；再配合 Skins（一整套附件映射），同一副骨骼能套出外观完全不同的角色。这就是 GEA 里说的"多套网格共享一副骨骼和一套动画"在 2D 里的对应物。

## 在 UE 里

UE 自带的 2D 模块 Paper2D 只有 Sprite 和 Flipbook（逐帧动画），**没有**原生的 2D 骨骼或 2D 蒙皮支持。要在 UE 里做 2D 蒙皮角色，常见的有两条路：

第一条是用 Spine 官方的 **spine-ue** 运行时插件。它把 Spine 导出的 `.json`/`.skel` 和 `.atlas` 导入成自定义资产，给 Actor 加 `Spine Skeleton Animation` 组件和 `Spine Skeleton Renderer Component` 即可显示。渲染组件用程序化网格绘制，每帧在 CPU 上算好顶点，这和 UE 骨骼网格体的 GPU 蒙皮是两套东西，角色数量很多时要留意开销。插件是 C++ 写的，纯蓝图项目用不了；5.3 起导入的 `.skel` 和 `.atlas` 文件名不能共用前缀（例如 `skeleton.skel` + `skeleton.atlas` 不行，要改成 `skeleton-data.skel`），这是 spine-ue 文档里专门写明的坑。

```cpp
// spine-ue 插件的真实 API（Build.cs 需依赖 "SpinePlugin"）
#include "SpineSkeletonAnimationComponent.h"

void AMySpineActor::BeginPlay()
{
	Super::BeginPlay();

	USpineSkeletonAnimationComponent* Anim = FindComponentByClass<USpineSkeletonAnimationComponent>();
	if (Anim)
	{
		// 轨道 0 循环播放 walk，2 秒后接 run
		Anim->SetAnimation(0, FString(TEXT("walk")), true);
		Anim->AddAnimation(0, FString(TEXT("run")), true, 2.0f);
	}
}
```

动画名是 Spine 里起的名字，写错了不会报编译错误，只会什么都不播，调试时先在 Skeleton Data 资产的详情面板里核对动画列表。

第二条是把 2D 角色当成"扁平的 3D 角色"：在 DCC 里把切好的图片贴到平面网格上，绑定骨骼、刷权重，导出 FBX 作为普通的 `USkeletalMesh` 导入。这样就能用 UE 完整的动画蓝图、状态机、IK 和 GPU 蒙皮，代价是要自己处理排序（半透明的绘制顺序）和像素风格的对齐问题。

社区常用的 PaperZD 插件给 Paper2D 加了动画蓝图式的状态机，但它驱动的仍是 Flipbook，不是蒙皮。

## 容易踩的坑

**把 2D 骨骼动画当成 2D 蒙皮。** 只有"图片挂骨骼"而没有网格和权重的方案，关节处照样会断开。Spine Essential 版能做前者，做不了后者。

**网格剖分太密。** 2D 网格的顶点数直接决定蒙皮开销，而 Spine 这类运行时往往是在 CPU 上变形的；剖分只要足够覆盖弯曲区域即可，平坦区域用少量大三角形。

**权重不归一。** 手动刷权重后如果总和不为 1，顶点会被整体缩放或偏移，工具一般会自动归一，但导入第三方数据时要检查。

**运行时和编辑器版本不匹配。** Spine 的导出格式随大版本变化，运行时分支要和编辑器导出版本一致，否则导入失败或动画错乱。

## 相关

[[EAAAC-帧动画]] [[EAABC-刚性层级动画]] [[EAABH-3D蒙皮动画]] [[EAADD-多骨骼的蒙皮权重]] [[EAADH-蒙皮矩阵]]
