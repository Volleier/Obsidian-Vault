>**Actor** 是可以放置在关卡中的任意对象，如摄像机、静态网格体或玩家出生点。Actor支持3D变换，如平移、旋转和缩放。

# 概览
虚幻引擎中最多最常见的内容就是AActor，而虚幻引擎中的游戏也是由一个个的AActor构成的。
Actor可以通过gameplay代码（C++或蓝图）来创建和销毁。
Actor之间还可以互相“嵌套”，拥有相对的“父子”关系。
AActor继承自UObject，但功能比Uobject多了：Replication（网络复制）,Spawn，Tick。

# 关系
`AActor` 是所有Actor的基类。
![[Actor实体-1.png]]