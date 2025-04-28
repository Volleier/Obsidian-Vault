>**组件** 是一种特殊类型的 **对象**，**Actor** 可以将组件作为子对象附加到自身。组件适用于共享相同的行为，例如显示视觉表现、播放声音。它们还可以表示项目特有的概念，例如载具解译输入和改变其速度与方向的方式。举例而言，某个项目拥有用户可控制车辆、飞机和船只。可以通过更改载具Actor所使用的组件来实现载具控制和移动的差异。

# 概览
如果游戏世界中只有Actor，随着构建游戏世界的进行，各种Actor需要被附加上各种各样的功能，这会使得Actor会变得臃肿不堪。因此虚幻引擎将许多功能抽象成Component并提供组装的接口，使Actor能够自由组装（模块化）

# 关系
**`UActorComponent` 是所有组件的基类。** 而`UActorComponent`是基础于UObject的一个子类，Component拥有UObject的通用功能的。


`TSet<UActorComponent \*> OwnedComponents` 保存着这个Actor所拥有的所有Component,一般其中会有一个`SceneComponent`作为`RootComponent`。

`TArray<UActorComponent \*> InstanceComponents` 保存着实例化的Components。实例化的Components意思就是在蓝图里Details定义的Component，当这个Actor被实例化的时候，这些附属的Component也会被实例化。但`OwnedComponents`总是最全的，`ReplicatedComponents`、`InstanceComponents`可以看作一个预先的分类。

一个Actor必须实例化`USceneComponent* RootComponent`才可以被放进Level里。`OwnedComponents`其实也是可以包容多个不同`SceneComponent`的，然后可以动态获取不同的`SceneComponent`来当作`RootComponent`，只不过这种用法确实不太自然，而且也得非常小心维护不同状态，不推荐如此用。
![[Component组件-1.png]]
一个封装过后的Actor应该是一个整体，它能被放进Level中，拥有变换，这一整个整体的概念更加符合自然意识。
`ActorComponent`下面最重要的一个Component是`SceneComponent`。`SceneComponent`提供了两大能力：一是Transform，二是`SceneComponent`的互相嵌套。
![[Component组件-2.png]]
##  ChildComponent
同作为最常用到的Component之一，ChildActorComponent担负着Actor之间互相组合的胶水。
ChildComponent在蓝图里静态存在的时候其实并不真正的创建Actor，而是在之后Component实例化的时候才真正创建。
```cpp
/** The class of Actor to spawn */
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=ChildActorComponent, meta=(OnlyPlaceable, AllowPrivateAccess="true", ForceRebuildProperty="ChildActorTemplate"))
TSubclassOf<AActor>	ChildActorClass;

/** The actor that we spawned and own */
UPROPERTY(Replicated, BlueprintReadOnly, Category=ChildActorComponent, TextExportTransient, NonPIEDuplicateTransient, meta=(AllowPrivateAccess="true"))
TObjectPtr<AActor>	ChildActor;

/** Property to point to the template child actor for details panel purposes */
UPROPERTY(VisibleDefaultsOnly, DuplicateTransient, Category=ChildActorComponent, meta=(ShowInnerProperties))
TObjectPtr<AActor> ChildActorTemplate;
```
引擎代码提供了ChildActorTemplate这个对象，来为ChildActorClass生成一个Actor模板，方便我们编辑属性，之后在生成ChildActor的时候，可以把ChildActorTemplate身上的属性拷贝给ChildActor，这样运作看起来就像生成了想要配置的那个Actor

# 附加
