GameInstance是比World更高的层次，独立于Level的逻辑和数据也一般存放在GameInstance中。换句话说，GameInstance里保存着当前的WorldConext和其他整个游戏的信息。
![[GameInstance游戏实例-1.png]]UE的GameInstance因为继承于UObject，所以就拥有了动态创建的能力，所以我们可以通过指定GameInstanceClass来让UE创建使用我们自定义的GameInstance子类。所以不论是C++还是BP，我们通常会继承于GameInstance，然后在里面编写应用于整个游戏范围的逻辑。
UGameInstance里的接口大概有4类：
- 引擎的初始化加载，Init和ShutDown等（在引擎流程章节会详细叙述）
- Player的创建，如CreateLocalPlayer，GetLocalPlayers之类的。
- GameMode的重载修改，这是从4.14新增加进来改进，本来你只能为特定的某个Map配置好GameModeClass，但是现在GameInstance允许你重载它的PreloadContentForURL、CreateGameModeForURL和OverrideGameModeClass方法来hook改变这一流程。
- OnlineSession的管理，这部分逻辑跟网络的机制有关（到时候再详细介绍），目前可以简单理解为有一个网络会话的管理辅助控制类。

而GameInstance是在GameEngine里创建的，先从配置中取出GameInstanceClass，然后动态创建
```cpp
void UGameEngine::Init(IEngineLoop* InEngineLoop)
{
    //[...]
	// Create game instance.  For GameEngine, this should be the only GameInstance that ever gets created.
	{
		FStringClassReference GameInstanceClassName = GetDefault<UGameMapsSettings>()->GameInstanceClass;
		UClass* GameInstanceClass = (GameInstanceClassName.IsValid() ? LoadObject<UClass>(NULL, *GameInstanceClassName.ToString()) : UGameInstance::StaticClass());
		if (GameInstanceClass == nullptr)
		{
			UE_LOG(LogEngine, Error, TEXT("Unable to load GameInstance Class '%s'. Falling back to generic UGameInstance."), *GameInstanceClassName.ToString());
			GameInstanceClass = UGameInstance::StaticClass();
		}
		GameInstance = NewObject<UGameInstance>(this, GameInstanceClass);
		GameInstance->InitializeStandalone();
	}
	//[...]
 }
//在BaseEngine.ini或DefaultEngine.init里你可以配置GameInstanceClass
[/Script/EngineSettings.GameMapsSettings]
GameInstanceClass=/Script/Engine.GameInstance
```

GameInstance是作为游戏中全局唯一的从开始运行到结束的实例，拥有全局的控制权。
在逻辑层面，GameInstance往下看是：  
1. Worlds，Level的切换实际发生地是Engine，而GameInstance可以说是UE之神其下的唯一代言人，所以GameInstance也可以代之管理World的切换等。我们可以在GameInstance里实现各种逻辑最后调用Engine的OpenLevel等接口。  
2. Players，虽然一般来说我们直接控制Players的机会不多，都是配置好了就行。但要是到了需要的时候，GameInstance也实现了许多的接口可以让你动态的添加删除Players。  
3. UI，UE的UI是另一套World之外的系统，虽然同属于Viewport的显示之下，但是控制结构跟Actor们并不一样。所以常常会需要控制UI各种切换的业务逻辑，虽然在Widget的Graph里也可以写些简单的切换，但是要想复用某些切换逻辑的时候，在特定的Wdiget里就不合适了，而GameMode一方面局限于Level，另一方面又只存在于Server；PlayerController也是会切换掉的，同时又只存在于World中，所以最后比较合适的就只有GameInstance了。  
4. 全局的配置，也常常需要根据平台改变一些游戏的配置，Execute一些ConsoleCommand，GameInstance也是这些命令的存放地。  
5. 游戏的额外第三方逻辑，如果游戏需要其他一些控制，比如自己写的网络通信、自定义的配置文件或者自己的一些程序算法，如果简单的话，GameInstance也可以一放，等复杂起来了，也可以把GameInstance当作一个模块容器，在里面再扩展出来其他的子逻辑模块。当然如果是插件的话，还是在自己的插件Module里面自行管理逻辑，然后把协调工作交给GameInstance来做。
数据层面上：
虚幻引擎层层上来，已经有了针对一个Player的Contoller的PlayerState，也有了针对World的GameMode的GameState，到了更全局之上，自然的GameInstance就应该存储一些全局的状态数据。所以你可以在GameInstance的成员变量中添加一些全局的状态，或者是那些想要在Level之外持续存在的对象。需要注意的是，GameInstance成员变量中最好只保存那些“临时”的数据，而对于那些想要持久序列化保存的数据，我们就需要接下来的SaveGame了。把持久的数据直接放在SaveGame，用的时候直接读取出来，之后再直接在其上更新，好处是只用维护一份，省得要保存的时候，还去想到底要选GameInstance的哪些成员变量中来保存，一开始就设计选好，以后就方便了。