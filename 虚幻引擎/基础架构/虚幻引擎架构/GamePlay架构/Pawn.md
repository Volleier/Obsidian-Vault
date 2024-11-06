Actor是虚幻引擎中最基本的对象类型，几乎所有的游戏对象都是由Actor派生的，但不是所有的Actor本身具备被玩家或AI控制的能力，而具有能被`PlayerController`或者`AIController`控制的Acotr就称为Pawn。
Pawn是可那些由玩家或 AI 控制的所有 Actor 的基类。Pawn 是玩家或 AI 实体在游戏场景中的具化体现。
同其他AInfo一样，UE也是从Actor中再派生出了APawn，并定义了3块基本的模板方法接口：  
1. 可被Controller控制  
2. PhysicsCollision表示  
3. MovementInput的基本响应接口
![[Pawn-1.png]]
Actor是我们用来表示3D游戏中对象的，所以Pawn继承于Actor，概念的重点是在于更清楚的去表示，而不是重点在于Pawn被当作逻辑的载体，就像棋子本身只能简单的表达出出个棋子，但是该如何走还是得再靠外部的Controller机制。你也可以想象成提线木偶，那个木偶就是Pawn，而提线的是Controller。Pawn表达的最关键点是可被玩家或AI操纵的能力。 
![[Pawn-2.png]]