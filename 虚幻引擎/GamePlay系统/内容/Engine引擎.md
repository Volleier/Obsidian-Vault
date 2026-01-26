
# 概览
虚幻引擎最根本的东西：Engine（引擎）
![[引擎-1.png]]此处UEngine分化出了两个子类：UGameEngine和UEditorEngine。UE的编辑器也是UE用自己的引擎渲染出来的，所以本质上来说，UE的编辑器其实也是个游戏。我们是在编辑器这个游戏里面创造我们自己的另一个游戏。话虽如此，但比较编辑器和游戏还是有一定差别的，所以UE会在不同模式下根据编译环境而采用不同的具体Engine类，而在基类UEngine里通过一个WorldList保存了所有的World。
- Standalone Game：会使用UGameEngine来创建出唯一的一个GameWorld，因为也只有一个，所以为了方便起见，就直接保存了GameInstance指针。
- 而对于编辑器来说，EditorWorld其实只是用来预览，所以并不拥有OwningGameInstance，而PlayWorld里的OwningGameInstance才是间接保存了GameInstance.
目前来说，因为UE还不支持同时运行多个World（当前只能一个，但可以切换），所以GameInstance其实也是唯一的。提前说些题外话，虽然目前网络部分还没涉及到，但是当我们在Editor里进行MultiplePlayer的测试时，每一个Player Window里都是一个World。如果是DedicateServer模式，那DedicateServer也会是一个World。  
最后实例化出来的UEngine实例用一个全局的GEngine变量来保存。至此，我们已经到了引擎的最根本
```cpp
//UnrealEngine\Engine\Source\Runtime\Engine\Private\UnrealEngine.cpp
ENGINE_API UEngine*	GEngine = NULL;
```