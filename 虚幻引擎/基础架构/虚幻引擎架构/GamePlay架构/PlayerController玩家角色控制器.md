
PlayerController继承自AController，是玩家直接操控的逻辑类。一个游戏中同一时刻只能也只有一个PlayerController响应，但是不同Level可以拥有不同的PlayerController
![[PlayerController玩家角色控制器-1.png]]
PlayerController作为玩家直接控制的实体，很多的跟玩家直接相关的操作也都得委托它来完成，因此PlayerController集成了几大模块：
- Camera的管理，目的都是为了控制玩家的视角，所以有了PlayerCameraManager这一个关联很紧密的摄像机管理类，用来方便的切换摄像机。PlayerController的ControlRotation、ViewTarget等也都是为了更新Camera的位置。因为跟Camera的关系紧密，而Camera最后输出的是屏幕坐标里的图像，所以为了方便一些拾取的HitResult函数也都是实现在这里面。渲染章节会再详细介绍UE的摄像机管理。
- Input系统，包括构建InputStack用来路由输入事件，也包括了自己对输入事件的处理。所以包含了UPlayerInput来委托处理。
- UPlayer关联，既然顾名思义是PlayerController，那自然要和Player对应起来，这也是PlayerController最核心的部分。一个UPlayer可以是本地的LocalPlayer，也可以是一个网络控制UNetConnection。PlayerController只有在SetPlayer之后，才可以开始正常工作。
- HUD显示，用于在当前控制器的摄像机面前一直显示一些UI，这是从UE3迁移过来的组件，现在用UMG的比较多，等介绍UI模块的时候再详细介绍。
- Level的切换，PlayerController作为网络里通道，在一起进行Level Travelling的时候，也都是先通过PlayerController来进行RPC调用，然后由PlayerController来转发到自己World中来实际进行。
- Voice，也是为了方便网络中语音聊天的一些控制函数。
UE的思想是具象化一个“玩家实体”，并把所有的跟该玩家相关的操作和接口都交给它完成。PlayerController秉持着这种概念，集成了几乎所有玩家相关的业务逻辑代码，虽然代码膨胀得比较快，但是可以等某一块功能的代码量过大后抽出一个单独的类来，如PlayerInput和

PlayerCameraManager一样。
对于整套GamePlay系统，一般将以下逻辑写入PlayerController中：
- 对实现游戏逻辑来说，如果是按照MVC的视角，那么View对应的是Pawn的表现，而PlayerController对应的是Controller的部分，那Model就是游戏业务逻辑的数据了。PlayerController的职责应该是一边控制Pawn，一边负责内部正确的调用PlayerState的Coin接口。且根据单一职责原则，PlayerController里的变量的意义在于更好的实现控制，变量与Pawn和Pawn状态有直接关系。
- PlayerController是可被替换的，不同的关卡里也可能是不一样的，所以就不能写入过多的类，这样一旦被替换掉了之后数据就都丢失了。
- PlayerController也不一定存在，例如联机游戏，对方玩家被同步过来的将只有PlayerState，对方玩家的PlayerController只在服务器上存在。所以为了扩展性来说，还是根据职责分明的原则来正确划分业务逻辑会比较好。
- 在任一刻，Player:PlayerController:PlayerState是1:1:1的关系。但是PlayerController可以有多个备选用来切换，PlayerState也可以相应多个切换。