**帧动画（Frame-by-Frame Animation）是把一段动作画成一串完整的静态图像，运行时按固定节奏依次切换显示的动画方式。在游戏里它通常叫精灵动画（Sprite Animation），《Game Engine Architecture》把它看作传统赛璐珞（cel）动画的电子版本；UE 里对应的是 Paper2D 的 Flipbook（`UPaperFlipbook` / `UPaperFlipbookComponent`），特效里对应材质和 Niagara 的 SubUV 动画。**

> 参考：《Game Engine Architecture》第 3 版动画系统一章（Cel Animation 一节）；UE 部分对照 5.x 的 Paper 2D Flipbooks 文档与 Python API 参考。

## 原理和它的代价

帧动画的原理没有任何技巧：每一帧都是一张画好的图，播放器只需要知道"现在该显示第几张"。人眼在每秒十几张以上的切换速度下就会把它看成连续运动，传统手绘动画常用每秒 24 格、"一拍二"（每张画停两格，即 12 张/秒），游戏里像素风角色 8～12 帧/秒的动作很常见。

它的全部优点和缺点都来自"每帧一张完整的图"：

美术对每一帧有完全的控制权。挤压拉伸、夸张的变形、烟雾火焰这种没有骨架可言的东西，都可以直接画出来，不受任何骨骼结构限制。像素风游戏尤其依赖这一点，因为像素画的每个点都是手摆的，任何插值或变形都会破坏它。

代价是数据量和帧数成正比，而且不能复用。GEA 在讨论动画技术时的一个视角是"各种动画方法本质上都是数据压缩手段"：帧动画是压缩率最低的一种，它什么都不压缩。粗算一下：一帧 256×256 的 RGBA8 图是 256 KB，一个 12 帧的跑步循环就是 3 MB；一个角色十几个动作、八个朝向，未压缩的贴图很容易上百 MB。用 BC7 这类块压缩格式能降到每像素 1 字节，但数量级不变。而且换一件衣服就要全部重画，这正是后来出现骨骼动画和 [[EAAAA-2D蒙皮动画]] 的原因。

另一个代价是不能插值。两张图之间没有"中间状态"，播放速度调慢时只能让每张图停留更久，动作会变得一顿一顿的；想要更流畅只能多画几张。

## 数据组织：精灵图集与帧时长

为了减少贴图切换和内存碎片，帧通常打包进一张大图集（sprite sheet / atlas），每帧记录它在图集里的 UV 矩形、裁剪后的尺寸和锚点（pivot）。锚点很重要：同一个动作的各帧如果锚点不一致，角色播放时会原地抖动。打包工具（TexturePacker、Spine 自带的打包器、Aseprite 导出等）通常会把每帧的透明边裁掉以节省空间，同时记录原始偏移，引擎按偏移还原。

时间上有两种常见表示：一种是"全局帧率 + 每帧持续几拍"，另一种是"每帧持续多少毫秒"。前者更接近动画师的思维方式（一拍一、一拍二），UE 的 Flipbook 就是这种。

## 在 UE 里

UE 的 2D 模块 Paper2D 提供了完整的帧动画链路，基本概念只有三个：

| 资产 / 类型 | 作用 |
| --- | --- |
| Texture | 原始图集贴图，导入时建议用 Paper2D 的纹理设置（关闭 mip、使用 Nearest 过滤等，像素风尤其需要） |
| Sprite（`UPaperSprite`） | 图集上的一个矩形区域加锚点，右键贴图 → Sprite Actions → Extract Sprites 可以批量切出来 |
| Flipbook（`UPaperFlipbook`） | 一串关键帧，每个关键帧是一个 Sprite 加一个以"帧"为单位的持续时长，外加一个 Frames Per Second 属性 |

按官方文档的说法，Flipbook 由一系列关键帧组成，每个关键帧包含要显示的 Sprite 和显示时长（以帧计），Frames Per Second 决定一秒有多少"拍"。创建方式有两种：内容浏览器里新建空的 Paper Flipbook 再手动填帧；或者选中一组 Sprite，右键 Create Flipbook 自动生成。后者按资产名的**字母顺序**排帧，所以 Sprite 命名要像 `Idle_01`、`Idle_02` 这样补零，否则 `Idle_10` 会排到 `Idle_2` 前面。Paper2D 还能直接导入 JSON 格式的精灵图集描述（如 TexturePacker 导出的），一次生成贴图、Sprite 和 Flipbook。

运行时由 `UPaperFlipbookComponent` 播放，`APaperCharacter` 自带一个。常用接口都是真实的成员函数：

```cpp
// 需在 Build.cs 中依赖 "Paper2D"
#include "PaperFlipbookComponent.h"
#include "PaperFlipbook.h"

void AMyPaperActor::PlayAttack(UPaperFlipbook* AttackFlipbook)
{
	UPaperFlipbookComponent* Sprite = FindComponentByClass<UPaperFlipbookComponent>();
	if (!Sprite || !AttackFlipbook)
	{
		return;
	}

	// SetFlipbook 换成新的 Flipbook 时会把播放时间重置为 0
	Sprite->SetFlipbook(AttackFlipbook);
	Sprite->SetLooping(false);
	Sprite->PlayFromStart();

	// 非循环 Flipbook 播完时触发，委托无参数
	Sprite->OnFinishedPlaying.AddUniqueDynamic(this, &AMyPaperActor::HandleAttackFinished);
}
```

`HandleAttackFinished` 需要声明成 `UFUNCTION()`，否则动态委托绑不上。其他常用的还有 `SetPlayRate`、`Reverse`、`SetPlaybackPositionInFrames(Frame, bFireEvents)`、`GetFlipbookLengthInFrames` 等。组件的 Mobility 如果不是 Movable，运行时换 Flipbook 会受限，这在构造函数以外调用时容易碰到。

Paper2D 本身近几年基本没有新功能（社区的普遍看法，Epic 没有正式声明），缺少动画状态机。社区插件 PaperZD 给 Flipbook 加了类似动画蓝图的状态机和通知系统，做纯 2D 项目的人大多会用它。

帧动画在 3D 项目里也没有消失，只是换了位置：粒子特效的火焰、烟雾、爆炸几乎都是序列帧。材质里有 `FlipBook` 材质函数按时间切换 SubUV，Niagara 里用 Sub UV Animation 模块驱动粒子的帧序号，贴图同样是一张多帧图集。

## 容易踩的坑

**锚点不统一。** 各帧的 pivot 不一致会导致角色原地抖动。在 Sprite 编辑器里统一设置 Pivot Mode，或者在打包时保留原始画布尺寸。

**过滤方式不对。** 像素风素材用默认的双线性过滤和 mipmap 会糊成一团，要对贴图应用 Paper2D 的纹理设置（Nearest 过滤、无 mip）。

**用 Tick 手动换 Sprite。** 自己在 Tick 里按时间切 `UPaperSprite` 可以工作，但会丢掉 Flipbook 的帧时长、播放完成事件和倒放功能，没有理由不用 Flipbook。

**帧数和帧率混用。** Flipbook 里关键帧时长的单位是"帧"，总时长 = 各关键帧帧数之和 ÷ Frames Per Second。改了 FPS 整段动画的速度都会变。

## 相关

[[EAAAA-2D蒙皮动画]] [[EAABF-逐顶点动画]] [[EAABC-刚性层级动画]]
