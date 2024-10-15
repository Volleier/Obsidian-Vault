## UE_LOG
在官网中有对UE_LOG的详细介绍 [虚幻引擎中的日志记录](https://docs.unrealengine.com/5.3/zh-CN/logging-in-unreal-engine/)
`UE_LOG`是虚幻引擎中用于记录日志的宏。日志是开发过程中非常重要的一部分，它可以帮助开发者了解程序的运行情况，找出和修复错误。
```cpp
    UE_LOG(LogTemp, Error, TEXT("Error"));
    UE_LOG(LogTemp, Warning, TEXT("Warning"));
	UE_LOG(LogTemp, Display, TEXT("Display"));
```
UE_LOG的三个输入分别是以下作用：
- **LogTemp**：这是日志类别。虚幻引擎中有许多预定义的日志类别，如`LogTemp`、`LogActor`、`LogAnimation`等。你也可以自定义你自己的日志类别。日志类别可以帮助你过滤和查找日志。
- **Error**：这是日志级别。虚幻引擎中有几个预定义的日志级别，如`Error`、`Warning`、`Display`、`Log`等。
	- **Error**：这是最高的日志级别，表示一个严重的错误，它可能会导致程序崩溃。当你在代码中遇到一个无法恢复的错误时，你应该记录一个`Error`级别的日志。
	- **Warning**：这个级别表示一个可能会导致问题的情况，但不会立即导致程序崩溃。E.g：如果一个函数接收到一个无效的参数，你可能会记录一个`Warning`级别的日志。
	- **Display**：这个级别用于记录一般的信息，例如程序的运行状态、进度等。这些信息对于理解程序的运行情况很有用，但它们通常不表示一个错误或问题。
- **TEXT("Error")**：这是日志消息。`TEXT`是一个宏，它用于创建一个宽字符字符串。`()`中是要打印出来的消息。

 UE_LOG输出日志可以在虚幻编辑器 > 窗口（Window）>  输出日志（Output Log）中查看

## AddOnScreenDebugMessage

与UE_LOG不同的是，AddOnScreenDebugMessage可以在游戏屏幕上显示调试信息
```cpp
GEngine->AddOnScreenDebugMessage(-1, 5.f, FColor::Red, TEXT("Hello World!"));
```
在虚幻引擎中，`GEngine`是一个全局变量，它是`UEngine`类的一个实例。`UEngine`类是虚幻引擎的主要类之一，它包含了许多用于管理和控制游戏引擎的函数。

当你使用`GEngine->`时，你实际上是在访问`UEngine`类的一个实例，并调用它的成员函数或访问它的成员变量。例如，`GEngine->AddOnScreenDebugMessage`是在调用`UEngine`类的`AddOnScreenDebugMessage`函数。

这个函数有四个输入：
- **`-1`**：这是消息的键。如果你想要更新或删除一个已经显示的消息，你可以使用这个键。在虚幻引擎的`AddOnScreenDebugMessage`函数中，消息的键用于标识和管理屏幕上的调试消息。
	如果你想要更新或删除一个已经显示的消息，你可以使用这个键。例如，如果你想要更新屏幕上的一个消息，你可以再次调用`AddOnScreenDebugMessage`函数，使用相同的键和新的消息文本。虚幻引擎会找到使用这个键的消息，并用新的消息文本替换它。
	如果你想要删除一个消息，你可以调用`ClearOnScreenDebugMessage`函数，使用这个消息的键。
	在`AddOnScreenDebugMessage`函数中，如果你使用-1作为键，这表示这个消息是临时的，它不会替换任何已经显示的消息。这在你只想要显示一个临时消息，而不需要管理它时非常有用。
- **`5.f`**：这是消息的持续时间，以秒为单位。
- **`FColor::Red`**：这是消息的颜色。在虚幻引擎中，你可以使用`FColor`类来创建和管理颜色。`FColor`类有四个成员变量：`R`、`G`、`B`和`A`，它们分别表示颜色的红色、绿色、蓝色和透明度分量。
	如果你想要创建一个自定义的颜色，你可以创建一个`FColor`的实例，并设置它的`R`、`G`、`B`和`A`成员变量。例如，如果你想要创建一个红色和蓝色的混合色，你可以这样做：
```cpp
FColor MyColor(255, 0, 255, 255);
```
在这个例子中，`MyColor`的红色和蓝色分量都是255，绿色分量是0，透明度分量是255（表示完全不透明）。这会创建一个紫色。

你也可以使用`FColor`类的`FromRGB`和`FromRGBA`函数来创建颜色。例如：

FColor MyColor = FColor::FromRGB(255, 0, 255);

这行代码和上面的例子有相同的效果，它也会创建一个紫色。
- **`TEXT("Hello World!")`**：这是消息的文本。在这个例子中，消息的文本是"Hello World!"。