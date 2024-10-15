# BeginPlay、Tick、EndPlay

BeginPlay、Tick、EndPlay这三个函数是虚幻引擎的Actor生命周期的重要部分：

- **BeginPlay**：这个函数在Actor被创建并添加到游戏世界后调用，也就是说，当一个Actor实例在游戏中被首次“播放”时，这个函数就会被调用。这是初始化Actor的理想位置，例如，你可能需要在这里设置变量的初始值，启动定时器，或者调用其他需要在Actor开始运行时执行的函数。
- **Tick**：这个函数在每一帧都会被调用，它的主要用途是处理那些需要在每帧更新的逻辑，例如角色的移动、旋转、AI决策等。Tick函数有一个参数DeltaTime，这个参数表示自上一帧以来的时间，这对于处理速度和移动非常有用，因为你可以使用它来确保你的代码在不同的帧率下表现一致。
- **EndPlay**：这个函数在Actor从游戏世界中被移除前调用，无论是因为游戏结束，还是因为Level被卸载或是切换Level，或者Actor被明确销毁。这个函数通常用于执行清理操作，例如停止正在运行的定时器，断开网络连接，或者保存需要在下次Actor被创建时恢复的状态。
# 重写
```cpp
UCLASS()
class UELEARNC_API AMyGameMode : public AGameMode
{
    GENERATED_BODY()
    AMyGameMode();
  
public:
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaSeconds) override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
};
```
`public:`是一个访问标签，它表示后面的成员都是公有的。这意味着这些成员可以在类的内部和外部访问。

在重写虚函数时，通常会将重写的函数声明为`public`，这样其他的类就可以通过这个类的对象来调用这个函数。例如，虚幻引擎的`BeginPlay`函数就是一个公有的虚函数，它在游戏开始时被虚幻引擎调用，用于初始化Actor。

在重写过程中，`virtual`关键字表示这是一个虚函数。在C++中，虚函数是可以在派生类中被重写的函数。`void`是这个函数的返回类型，表示这个函数不返回任何值。`BeginPlay`、`Tick`、`EndPlay`是这个函数的名称，这些是虚幻引擎中Actor类的虚函数。`()`表示这个函数的参数。`override`关键字表示这个函数是重写基类中的虚函数。在C++中，如果一个函数声明为`override`，编译器会检查基类中是否存在一个同名的虚函数，如果不存在，编译器会报错。

这三个函数的传入函数分别是以下含义：
- **BeginPlay**：这个函数没有输入参数。它在Actor被创建并添加到游戏世界后调用，通常用于初始化Actor。
- **Tick**：这个函数有一个输入参数`DeltaSeconds`，它是一个浮点数，表示自上一帧以来的时间（以秒为单位）。这个函数在每一帧都会被调用，通常用于更新Actor的状态。
- **EndPlay**：这个函数有一个输入参数`EndPlayReason`，它是一个枚举类型`EEndPlayReason::Type`，表示Actor结束播放的原因。这个函数在Actor从游戏世界中被移除前调用，通常用于清理Actor。