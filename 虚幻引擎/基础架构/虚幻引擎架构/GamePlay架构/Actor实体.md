Actor无疑是UE中最重要的角色之一，组织庞大，最常见的有StaticMeshActor, CameraActor和 PlayerStartActor等。
Actor之间还可以互相“嵌套”，拥有相对的“父子”关系。
AActor继承自UObject，但功能比Uobject多了：Replication（网络复制）,Spawn，Tick。
![[Actor实体-1.png]]