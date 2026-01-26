Player玩家是外界输入的接收者，接受外界的输入并返回对应的动作。
Actor是必须在World中才能存在的，而Player却是比World更高一级的对象。玩游戏的过程中，LevelWorld在不停的切换，但是玩家的模式却是脱离不变的。另外，Player也不需要被摆放在Level中，也不需要各种Component组装，所以从AActor继承并不合适。因此直接从UObject继承是一个更好的选择
![[Player玩家-1.png]]
如上图，Player和一个PlayerController关联起来，因此虚幻引擎就可以把输入和PlayerController关联起来，这也符合了[[PlayerController玩家角色控制器|PlayerController]]接受玩家输入的描述。不管是本地玩家还是远程玩家，都需要PlayerController控制一个玩家Pawn的，所以自然也就需要为每个玩家分配一个PlayerController，所以把PlayerController放在UPlayer基类里是合理的。

## ULocalPlayer
对本地环境中，一个本地玩家关联着输入，也一般需要关联着输出。玩家对象的上层就是引擎了，所以会在GameInstance里保存有LocalPlayer列表。
![[Player玩家-2.png]]
ULocalPlayer比UPlayer多了Viewport相关的配置，也终于用SpawnPlayerActor实现了创建出PlayerController的功能。GameInstance里有LocalPlayers的信息之后，就可以方便的遍历访问，来实现跟本地玩家相关操作。
UE在初始化GameInstance的时候，会先默认创建出一个GameViewportClient，然后在内部再转发到GameInstance的CreateLocalPlayer：
```cpp
ULocalPlayer* UGameInstance::CreateLocalPlayer(int32 ControllerId, FString& OutError, bool bSpawnActor)
{
	ULocalPlayer* NewPlayer = NULL;
	int32 InsertIndex = INDEX_NONE;
	const int32 MaxSplitscreenPlayers = (GetGameViewportClient() != NULL) ? GetGameViewportClient()->MaxSplitscreenPlayers : 1;
    //已略去错误验证代码，MaxSplitscreenPlayers默认为4
	NewPlayer = NewObject<ULocalPlayer>(GetEngine(), GetEngine()->LocalPlayerClass);
	InsertIndex = AddLocalPlayer(NewPlayer, ControllerId);
	if (bSpawnActor && InsertIndex != INDEX_NONE && GetWorld() != NULL)
	{
		if (GetWorld()->GetNetMode() != NM_Client)
		{
			// server; spawn a new PlayerController immediately
			if (!NewPlayer->SpawnPlayActor("", OutError, GetWorld()))
			{
				RemoveLocalPlayer(NewPlayer);
				NewPlayer = NULL;
			}
		}
		else
		{
			// client; ask the server to let the new player join
			NewPlayer->SendSplitJoin();
		}
	}
	return NewPlayer;
}
```
可以看到，如果是在Server模式，会直接创建出ULocalPlayer，然后创建出相应的PlayerController。而如果是Client（比如Play的时候选择NumberPlayer=2,则有一个为Client），则会先发送JoinSplit消息到服务器，在载入服务器上的Map之后，再为LocalPlayer创建出PlayerController。  
而在每个PlayerController创建的过程中，在其内部会调用InitPlayerState：
```cpp
void AController::InitPlayerState()
{
	if ( GetNetMode() != NM_Client )
	{
		UWorld* const World = GetWorld();
		const AGameModeBase* GameMode = World ? World->GetAuthGameMode() : NULL;
		//已省略其他验证和无关部分
		if (GameMode != NULL)
		{
			FActorSpawnParameters SpawnInfo;
			SpawnInfo.Owner = this;
			SpawnInfo.Instigator = Instigator;
			SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
			SpawnInfo.ObjectFlags |= RF_Transient;	// We never want player states to save into a map
			PlayerState = World->SpawnActor<APlayerState>(GameMode->PlayerStateClass, SpawnInfo );
	
			// force a default player name if necessary
			if (PlayerState && PlayerState->PlayerName.IsEmpty())
			{
				// don't call SetPlayerName() as that will broadcast entry messages but the GameMode hasn't had a chance
				// to potentially apply a player/bot name yet
				PlayerState->PlayerName = GameMode->DefaultPlayerName.ToString();
			}
		}
	}
}
```
这样LocalPlayer最终就和PlayerState对应了起来。而网络联机时其他玩家的PlayerState是通过Replicated过来的。  
虚幻引擎将玩家视作输入体现在在每个PlayerController接受Player的时候：
```cpp
void APlayerController::SetPlayer( UPlayer* InPlayer )
{
    //[...]
	// Set the viewport.
	Player = InPlayer;
	InPlayer->PlayerController = this;
	// initializations only for local players
	ULocalPlayer *LP = Cast<ULocalPlayer>(InPlayer);
	if (LP != NULL)
	{
		// Clients need this marked as local (server already knew at construction time)
		SetAsLocalPlayerController();
		LP->InitOnlineSession();
		InitInputSystem();
	}
	else
	{
		NetConnection = Cast<UNetConnection>(InPlayer);
		if (NetConnection)
		{
			NetConnection->OwningActor = this;
		}
	}
	UpdateStateInputComponents();
	// notify script that we've been assigned a valid player
	ReceivedPlayer();
}
```
于ULocalPlayer，APlayerController内部会开始InitInputSystem()，接着会创建相应的UPlayerInput，BuildInputStack等初始化出和Input相关的组件对象。现在先明白到LocalPlayer才是PlayerController产生的源头，也因此才有了Input就够了。
## UNetConnection
在UE里，一个网络连接也是个Player：
![[Player玩家-3.png]]包含Socket的IpConnection也是玩家，甚至对于一些平台的特定实现如OculusNet的连接也可以当作玩家，因为对于玩家，只要能提供输入信号，就可以当作一个玩家。  
追根溯源，UNetConnection的列表保存在UNetDriver，再到FWorldContext，最后也依然是UGameInstance，所以和LocalPlayer的列表一样，是在World上层的对象。  