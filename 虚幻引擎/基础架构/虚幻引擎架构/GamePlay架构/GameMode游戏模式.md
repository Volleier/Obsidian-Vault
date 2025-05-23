>即使最开放的游戏也拥有基础规则，而这些规则构成了 GameMode。在最基础的层面上，这些规则包括：
- 出现的玩家和观众数量，以及允许的玩家和观众最大数量。
- 玩家进入游戏的方式，可包含选择生成地点和其他生成/重生成行为的规则。
- 游戏是否可以暂停，以及如何处理游戏暂停。
- 关卡之间的过渡，包括游戏是否以动画模式开场。

# 概览
一个World就是一个Game，而游戏玩法自然就称为Mode。
GameMode身为一场游戏的唯一逻辑操纵者身兼重任，在功能实现上有许多的接口，但主要可以分为以下几大块：
1. **Class登记**：GameMode里登记了游戏里基本需要的类型信息，在需要的时候通过UClass的反射可以自动Spawn出相应的对象来添加进关卡中。
2. **游戏内实体的Spawn**：GameMode既然作为一场游戏的主要负责人，那么游戏的加载释放过程中涉及到的实体的产生，包括玩家Pawn和PlayerController，AIController也都是由GameMode负责。GameMode也控制着本场游戏支持的玩家、旁观者和AI实体的数目。
3. **游戏的进度**：一个游戏支不支持暂停，怎么重启等这些涉及到游戏内状态的操作也都是GameMode的工作之一，SetPause、ResartPlayer等函数可以控制相应逻辑。
4. **Level的切换**：GameMode也决定了刚进入一场游戏的时候是否应该开始播放开场动画，也决定了当要切换到下一个关卡时是否要bUseSeamlessTravel。一旦开启后，可以选择重载GameMode和PlayerController的GetSeamlessTravelActorList方法和GetSeamlessTravelActorList来指定哪些Actors不被释放而进入下一个World的Level。
5. **多人游戏的步调同步**：在多人游戏的时候，需要等所有加入的玩家连接服务器完成之后，载入地图完毕后才能一起开始逻辑。因此UE提供了一个MatchState来指定一场游戏运行的状态，使用一个状态机来标记开始和结束的状态，并触发各种回调。

除了配置全局的GameModeClass之外，我们还能为每个Level单独的配置不同的GameModeClass。但是当一个World由多个Level组成的时候，这样就相当于配置了多个GameModeClass，那么应用的是哪一个？首先第一个原则需要记住的就是，一个World里只会有一个GameMode实例，否则肯定乱套了。因此当有多个Level的时候，一定是PersistentLevel和多个StreamingLevel，这时就算它们配置了不同的GameModeClass，UE也只会为第一次创建World时加载PersistentLevel的时候创建GameMode，在后续的LoadStreamingLevel时候，并不会再动态创建出别的GameMode，所以GameMode从始至终只有一个，PersistentLevel的那个。
  
Level迁移的时候GameMode重新生成。无论travelling采用哪种方式，当前的World都会被释放掉，然后加载创建新的World。但这个过程中，有点区别的是根据bUseSeamlessTravel的不同，UE可以选择哪些Actor迁移到下一个World中去（实现方式是先创建个中间过渡World进行二段迁移（为了避免同时加载进两个大地图撑爆内存）。
分两种情况：  不开启bUseSeamlessTravel，那么在travelling的时候（ServerTravel或ClientTravel），当前的World会被释放，所以当前的GameMode就被释放掉。新的World加载，就会根据新的GameModeClass创建新的GameMode。所以这时是不同的。  
开启bUseSeamlessTravel，travelling时，当前World的GameMode会调用GetSeamlessTravelActorList：
```cpp
void AGameMode::GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)
{
	UWorld* World = GetWorld();

	// Get allocations for the elements we're going to add handled in one go
	const int32 ActorsToAddCount = World->GameState->PlayerArray.Num() + (bToTransition ?  3 : 0);
	ActorList.Reserve(ActorsToAddCount);

	// always keep PlayerStates, so that after we restart we can keep players on the same team, etc
	ActorList.Append(World->GameState->PlayerArray);

	if (bToTransition)
	{
		// keep ourselves until we transition to the final destination
		ActorList.Add(this);
		// keep general game state until we transition to the final destination
		ActorList.Add(World->GameState);
		// keep the game session state until we transition to the final destination
		ActorList.Add(GameSession);

		// If adding in this section best to increase the literal above for the ActorsToAddCount
	}
}
```
在第一步从CurrentWorld到TransitionWorld的迁移时候，bToTransition\==true，这个时候GameMode也会迁移进TransitionWorld（TransitionMap可以在ProjectSettings里配置），也包括GameState和GameSession，然后CurrentWorld释放掉。第二步从TransitionWorld到NewWorld的迁移，GameMode（已经在TransitionWorld中了）会再次调用GetSeamlessTravelActorList，这个时候bToTransition\==false，所以第二次的时候如代码所见当前的GameMode、GameState和GameSession就被排除在外了。这样NewWorld再继续InitWorld的时候，一发现当前没有GameMode，就会根据配置的GameModeClass重新生成一个出来。所以这个时候GameMode也是不同的。  
结论是，UE的流程travelling，GameMode在新的World里是会新生成一个的，即使Class类型一致，即使bUseSeamlessTravel，因此在travelling的时候要小心GameMode里保存的状态丢失。不过Pawn和Controller默认是一致的。

# 关系
## AGameModeBase
GameMode均继承自`GameModeBase`，`GameModeBase` 由AInfo派生，负责游戏逻辑。
![[GameMode游戏模式-1.png]]
`AGameModeBase` 包含大量可覆盖的基础功能。部分常见函数包括：

| 函数/事件                         | 目的                                                                                                                                        |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `InitGame`                    | `InitGame` 事件在其他脚本之前调用，由 `AGameModeBase` 使用，初始化参数并生成其助手类。<br><br>它在任意 Actor 运行 `PreInitializeComponents` 前调用。                             |
| `PreLogin`                    | 接受或拒绝尝试加入服务器的玩家。如它将 `ErrorMessage` 设为一个非空字符串，会导致 `Login` 函数失败。`PreLogin` 在 `Login` 前调用，Login 调用前可能需要大量时间，加入的玩家需要下载游戏内容时尤其如此。              |
| `PostLogin`                   | 成功登录后调用。这是首个在 `PlayerController` 上安全调用复制函数之处。`OnPostLogin` 可在蓝图中实现，以添加额外的逻辑。                                                              |
| `HandleStartingNewPlayer`     | 在 `PostLogin` 后或无缝游历后调用，可在蓝图中覆盖，修改新玩家身上发生的事件。它将默认创建一个玩家 pawn。                                                                             |
| `RestartPlayer`               | 调用开始生成一个玩家 pawn。如需要指定 Pawn 生成的地点，还可使用 `RestartPlayerAtPlayerStart` 和 `RestartPlayerAtTransform` 函数。`OnRestartPlayer` 可在蓝图中实现，在此函数完成后添加逻辑。 |
| `SpawnDefaultPawnAtTransform` | 这实际生成玩家 Pawn，可在蓝图中覆盖。                                                                                                                     |
| `Logout`                      | 玩家离开游戏或被摧毁时调用。可实现 `OnLogout` 执行蓝图逻辑。                                                                                                      |


## AGameMode
`AGameMode` 是 `AGameModeBase` 的子类，拥有一些额外的功能支持多人游戏和旧行为。所有新建项目默认使用 `AGameModeBase`。如果需要此额外行为，可切换到从 `AGameMode` 进行继承。如从 `AGameMode` 进行继承，也可从 `AGameState` 继承游戏状态（其支持比赛状态机）。