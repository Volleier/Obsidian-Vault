UE有一组用于自动执行编译虚幻引擎过程的工具，包括 **虚幻编译工具（UnrealBuildTool）** 和 **虚幻头工具（UnrealHeaderTool）**（以及其他工具）。**实现这一套工具的目的是为了简化多平台编译，方便你配置各个平台的参数和编译选项**

# UnrealBuildTool
UnrealBuildTool（UBT，C#）是一个自定义工具，负责管理通过各种编译配置来编译虚幻引擎4（UE4）源代码的过程。

-   该工具处理所有复杂的项目编译工作，编译UE4的逐个模块并处理依赖等。我们编写的Target.cs，Build.cs都是为这个工具服务的。 并将项目与引擎关联起来。该过程以透明方式进行，这样，您只需通过标准的Visual Studio 构建工作流程构建项目即可。

# UnrealHeaderTool 
UnrealHeaderTool （UHT，C++）：自定义编译方法，负责处理引擎反射系统编译所必需的信息，是支持UObject系统的自定义解析和代码生成工具。代码编译分两个阶段进行：

-   调用UHT。它将解析C++头文件中引擎相关类元数据，并生成自定义代码，以实现诸多UObject相关的功能。
-   调用普通C++编译器，以便对结果进行编译。

> [什么是UBT？](https://zhuanlan.zhihu.com/p/89220125)  
> 我们要知道，我们写的UE4代码不是标准的C++代码，是基于UE4源代码层层改装了很多层的（如反射），所以，UHT将UE4代码转化成标准的C++代码，而UBT负责调用UHT来实现这个转化工作的，转化完以后，UBT调用标准C++代码的编译器来将UHT转化后的标准C++代码完全编译成二进制文件，整体上看，UHT是UBT的编译流程的一部分。  

**完整编译流程**：UBT搜集目录中的.cs文件，然后UBT调用UHT分析需要分析的.h .cpp文件（典型根据文件是否含有#include"FileName.generated.h"，是否有UCLASS()、UPROPERTY等宏）生成generated.h和gen.cpp文件，生成的文件路径为 Intermediate->Build-Win64-UE4-Inc，最后UBT调用MSBuild，将.h.cpp和generated.h gen.cpp结合到一起然后编译。

UBT通过扫描头文件，记录所有包含反射类型的modules（模块），当其中有头文件改变时，就会用UHT更新反射数据。UHT解析头文件，扫描标记，生成用于支持反射的C++代码。编译时，两个工具皆有可能出现错误，必须仔细检查。

