# 文件架构
创建一个虚幻引擎项目，可以在Visual Studio工程文件结构中看到
- Binaries:存放编译生成的结果二进制文件。该目录可以gitignore，反正每次都会生成。
- Config:配置文件。
- Content:平常最常用到，所有的资源和蓝图等都放在该目录里。
- DerivedDataCache：“DDC”，存储着引擎针对平台特化后的资源版本。比如同一个图片，针对不同的平台有不同的适合格式，这个时候就可以在不动原始的uasset的基础上，比较轻易的再生成不同格式资源版本。gitignore。
- Intermediate：中间文件（gitignore），存放着一些临时生成的文件。有：  
    - Build的中间文件，.obj和预编译头等
    - UHT预处理生成的.generated.h/.cpp文件
    - VS.vcxproj项目文件，可通过.uproject文件生成编译生成的Shader文件。
    - AssetRegistryCache：Asset Registry系统的缓存文件，Asset Registry可以简单理解为一个索引了所有uasset资源头信息的注册表。CachedAssetRegistry.bin文件也是如此。
- Saved：存储自动保存文件，其他配置文件，日志文件，引擎崩溃日志，硬件信息，烘培信息数据等。gitignore
- Source：代码文件。

# 编译类型
虚幻引擎的编译不仅仅只包含DevelopmentEditor，实际上编译选项非常重要
>UE5本身包含网络模式和编辑器，这意味着工程在部署的时候将包含Server和Client，而在开发的时候，也将有Editor和Stand-alone之分；同时也可以单独选择是否为Engine和Game生成调试信息，接着还可以选择是否在游戏里内嵌控制台等。

每种编译配置包含两种关键字。第一种表明了引擎以及游戏项目的*状态*。第二个关键字表明正在编译的*目标*。
状态：

| 状态              | Engine  | Game    | 其他                         |     |
| --------------- | ------- | ------- | -------------------------- | --- |
| Debug（调试）       | Debug   | Debug   | 必须在编辑器上加-debug参数才能反射查看代码更改 |     |
| DebugGame（调试游戏） | Release | Debug   | 适合只调试游戏代码                  |     |
| Development（开发） | Release | Release | 允许编辑器反射查看代码更改              |     |
| Shipping（发行）    | Release | Release | 无控制台命令、统计数据、性能分析           |     |
| Test（测试）        | Release | Release | 启动了一些控制台命令、统计数据和性能分析       |     |
目标：

| 目标           | 描述                                         |
| ------------ | ------------------------------------------ |
| \[empty]（空白） | 不带编辑器的一个独立可执行版本，需要提前打包烘培游戏内容资源             |
| Editor（编辑器）  | 直接在编辑器里打开游戏项目                              |
| Client（客户端）  | 多人联机项目，生成客户端版本，需要提供\<Game>Client.Target.cs |
| Server（服务端）  | 多人联机项目，生成服务端版本，需要提供\<Game>Server.Target.cs |
组合的各种情况：

| Debug        | DebugGame | Development | Shipping | Test |
| ------------ | :-------: | :---------: | :------: | :--: |
| \[empty]（空白） |     -     |      -      |    -     |  -   |
| Editor（编辑器）  |     -     |      -      |          |      |
| Client（客户端）  |     -     |      -      |    -     |  -   |
| Server（服务端）  |     -     |      -      |    -     |  -   |

# 命名约定
官网有详细的命名约定[Epic C++ Coding Standard for Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine?application_version=5.4)
基本来说，就几种情况：
- 模版类以T作为前缀，比如TArray,TMap,TSet UObject派生类都以U前缀  
- AActor派生类都以A前缀  
- SWidget派生类都以S前缀  
- 抽象接口以I前缀  
- 枚举以E开头  
- bool变量以b前缀，如bPendingDestruction  
- 其他的大部分以F开头，如FString,FName  
- typedef的以原型名前缀为准，如typedef TArray FArrayOfMyTypes;  
- 在编辑器里和C#里，类型名是去掉前缀过的  
- UHT在工作的时候需要提供正确的前缀

# 基础概念
在UE5中，几乎所有的对象都继承于UObject，UObject为它们提供了基础的垃圾回收，反射，元数据，序列化等，相应的，就有各种"UClass"的派生们定义了属性和行为的数据。  

跟Unity（GameObject-Component）有些像的是，UE5也采用了组件式的架构，但细分起来却又有些不一样。在UE中，3D世界是由Actors构建起来的，而Actor又拥有各种Component，之后又有各种Controller可以控制Actor（Pawn）的行为。
Unity中的Prefab，在UE5中变成了BlueprintClass，其实Class的概念确实更加贴近C++的底层一些。  Unity中，你可以为一个GameObject添加一个ScriptComponent，然后继承MonoBehaviour来编写游戏逻辑。在UE5中，你也可以为一个Actor添加一个蓝图或者C++ Component,然后实现它来直接组织逻辑。 

# 编译系统
UE5支持众多平台，包括Windows,IOS，Android等，因此UE5为了方便你配置各个平台的参数和编译选项，简化编译流程,UE5实现了自己的一套编译系统，否则我们就得接受各个平台再单独配置一套项目之苦了。  
这套工具的编译流程结果，在VS里的运行，背后会运行UE5的一些命令行工具来完成编译，其他最重要的两个组件：  
- UnrealBuildTool（UBT，C#）：UE5的自定义工具，来编译UE5的逐个模块并处理依赖等。我们编写的Target.cs，Build.cs都是为这个工具服务的。
- UnrealHeaderTool （UHT，C++）：UE5的C++代码解析生成工具，我们在代码里写的那些宏UCLASS等和 \#include "\*.generated.h" 都为UHT提供了信息来生成相应的C++反射代码。  
一般来说，UBT会先调用UHT会先负责解析一遍C++代码，生成相应其他代码。然后开始调用平台特定的编译工具(VisualStudio,LLVM)来编译各个模块。最后启动Editor或者是Game.