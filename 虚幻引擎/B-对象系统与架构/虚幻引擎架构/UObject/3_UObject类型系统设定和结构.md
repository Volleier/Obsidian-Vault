# 系统设定
在c++写的class（struct一样，只是默认public而已）的头上加宏标记，在其成员变量和成员函数也同样加上宏标记，大概就是类似C#Attribute的语法。在宏的参数可以按照我们自定的语法写上内容。在UE里我们就可以看到这些宏标记
```cpp
#define UPROPERTY(...)
#define UFUNCTION(...)
#define USTRUCT(...)
#define UMETA(...)
#define UPARAM(...)
#define UENUM(...)
#define UDELEGATE(...)
#define UCLASS(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_PROLOG)
#define UINTERFACE(...) UCLASS()
```
真正编译的时候，大体上都是一些空宏。UCLASS有些特殊，一般情况下最后也都是空宏，另外一些情况下会生成一些特定的事件参数声明等等。这里重点有两点，一是我们可以通过给类、枚举、属性、函数加上特定的宏来标记更多的元数据；二是在有必要的时候这些标记宏甚至也可以安插进生成的代码来合成编译。
C++有声明和定义之分，图中黄色的的都可以看作是声明，而绿色的UProperty可以看作是字段的定义。在声明里，我们也可以把类型分为可聚合其他成员的类型和“原子”类型。

# 结构
在清楚了OBject的系统设定后，接下来就是结构：
![[UObject类型系统设定和结构-1.png]]

- 聚合类型（UStruct）：  
    - UFunction，只可包含属性作为函数的输入输出参数
    - UScriptStruct，只可包含属性，可以理解为C++中的POD struct，在UE里，你可以看作是一种“轻量”UObject，拥有和UObject一样的反射支持，序列化，复制等。但是和普通UObject不同的是，其不受GC控制，你需要自己控制内存分配和释放。
    - UClass，可包含属性和函数，是我们平常接触到最多的类型
- 原子类型：  
    - UEnum，支持普通的枚举和enum class。
    - int，FString等基础类型没必要特别声明，因为可以简单的枚举出来，可以通过不同的UProperty子类来支持。
把聚合类型们统一起来，就形成了UStruct基类，可以把一些通用的添加属性等方法放在里面，同时可以实现继承。UStruct这个名字确实比较容易引起歧义，因为实际上C++中USTRUCT宏生成了类型数据是用UScriptStruct来表示的。

还有个类型比较特殊，那就是接口，可以继承多个接口。跟C++中的虚类一样，不同的是UE中的接口只可以包含函数。一般来说，我们自己定义的普通类要继承于UObject，特殊一点，如果是想把这个类当作一个接口，则需要继承于UInterface。但是记得，生成的类型数据依然用UClass存储。从“#define UINTERFACE(...) UCLASS()”就可以看出来，Interface其实就是一个特殊点的类。UClass里通过保存一个TArray\<FImplementedInterface> Interfaces数组，其子项又包含UClass* Class来支持查询当前类实现了那些接口。

最后是定义，在UE里是UProperty，可以理解为用一个类型定义个字段“type instance;”。UE有Property，其Property有子类，子类之多，一屏列不下。实际深入代码的话，会发现UProperty通过模板实例化出特别多的子类，简单的如UBoolProperty、UStrProperty，复杂的如UMapProperty、UDelegateProperty、UObjectProperty。后续再一一展开。
元数据UMetaData其实就是个TMap<FName, FString>的键值对，用于为编辑器提供分类、友好名字、提示等信息，最终发布的时候不会包含此信息。