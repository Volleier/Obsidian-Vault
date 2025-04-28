>**Actor** 是可以放置在关卡中的任意对象，如摄像机、静态网格体或玩家出生点。Actor支持3D变换，如平移、旋转和缩放。

# 概览
虚幻引擎中最多最常见的内容就是AActor，而虚幻引擎中的游戏也是由一个个的AActor构成的。
Actor可以通过gameplay代码（C++或蓝图）来创建和销毁。
Actor之间还可以互相“嵌套”，拥有相对的“父子”关系。
AActor继承自UObject，但功能比Uobject多了：Replication（网络复制）,Spawn，Tick。

# 关系
**`AActor` 是所有Actor的基类。**
![[Actor实体-1.png]]
Actor中的很多功能都能在Component实现，但是Actor也有独特的功能：
1. 世界级对象身份
    - Actor 可以直接放置在关卡中，而组件只能作为 Actor 的一部分存在
    - Actor 拥有在游戏世界中的独立身份和生命周期：负责自身及其所拥有组件的初始化、销毁和提供 BeginPlay、Tick、EndPlay 等关键生命周期事件
2. 组件的所有权与管理
    - 组件必须依附于 Actor 才能存在，无法独立于 Actor
    - Actor 负责组织和管理多个组件之间的协作关系，通过 `OwnedComponents`、`InstanceComponents` 等集合管理所有组件
3. 网络复制的主体
    - Actor 是网络复制系统的基本单位，组件只能通过所属 Actor 进行复制
    - 所有的网络 RPC 调用都必须基于 Actor 进行
4. 游戏逻辑的主控者
    - Actor 可以操控和协调多个组件共同完成复杂任务
    - 处理整体性的游戏逻辑，而不仅限于单一功能
5. 玩家控制与拥有机制
    - 只有 Actor 可以直接被 Controller 控制或拥有
    - 组件无法直接接收玩家输入，必须通过 Actor 转发
6. 生成和销毁其他 Actor
    - Actor 可以直接生成(Spawn)或销毁其他 Actor
    - 组件只能与自己所属的 Actor 生命周期捆绑
7. Level Streaming 交互
    - 在关卡流送系统中，Actor 可以直接参与交互
    - 组件无法独立应对关卡加载/卸载事件
8. 插件系统集成能力
    - 多数引擎插件系统直接与 Actor 交互，而不是针对单个组件