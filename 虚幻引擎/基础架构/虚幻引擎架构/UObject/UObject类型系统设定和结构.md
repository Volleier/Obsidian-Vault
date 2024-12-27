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
- 聚合类型（UStruct）：  
    - UFunction，只可包含属性作为函数的输入输出参数
    - UScriptStruct，只可包含属性，可以理解为C++中的POD struct，在UE里，你可以看作是一种“轻量”UObject，拥有和UObject一样的反射支持，序列化，复制等。但是和普通UObject不同的是，其不受GC控制，你需要自己控制[内存分配](https://zhida.zhihu.com/search?content_id=2102677&content_type=Article&match_order=1&q=%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D&zhida_source=entity)和释放。
    - UClass，可包含属性和函数，是我们平常接触到最多的类型
- 原子类型：  
    - UEnum，支持普通的枚举和enum class。
    - int，FString等基础类型没必要特别声明，因为可以简单的枚举出来，可以通过不同的UProperty子类来支持。
![[Pasted image 20241227215149.png]]