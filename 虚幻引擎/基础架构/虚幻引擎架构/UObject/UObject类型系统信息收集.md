## C++ Static 自动注册模式

另一种常用的C++常用的[设计模式](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F&zhida_source=entity)：Static Auto Register。典型的，当你想要在程序启动后往一个容器里注册一些对象，或者簿记一些信息的时候，一种直接的方式是在程序启动后手动的一个个调用[注册函数](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E6%B3%A8%E5%86%8C%E5%87%BD%E6%95%B0&zhida_source=entity)：
```cpp
#include "ClassA.h"
#include "ClassB.h"
int main()
{
    ClassFactory::Get().Register<ClassA>();
    ClassFactory::Get().Register<ClassB>();
    [...]
}
```

这种方式的缺点是你必须手动的一个include之后再手动的一个个注册，当要继续添加注册的项时，只能再手动的依次序在该文件里加上一条条目，可维护性较差。  
所以根据C++ static对象会在main函数之前初始化的特性，可以设计出一种static自动注册模式，新增加注册条目的时候，只要Include进相应的类.h.cpp文件，就可以自动在程序启动main函数前自动执行一些操作。简化的代码大概如下：

  

```cpp
//StaticAutoRegister.h
template<typename TClass>
struct StaticAutoRegister
{
	StaticAutoRegister()
	{
		Register(TClass::StaticClass());
	}
};
//MyClass.h
class MyClass
{
    //[...]
};
//MyClass.cpp
#include "StaticAutoRegister.h"
const static StaticAutoRegister<MyClass> AutoRegister;
```

这样，在程序启动的时候就会执行Register(MyClass)，把因为新添加类而产生的改变行为限制在了新文件本身，对于一些顺序无关的注册行为这种模式尤为合适。利用这个static初始化特性，也有很多个变种，比如你可以把StaticAutoRegister声明进MyClass的一个[静态成员变量](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E9%9D%99%E6%80%81%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F&zhida_source=entity)也可以。不过注意的是，这种模式只能在独立的[地址空间](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4&zhida_source=entity)才能有效，如果该文件被[静态链接](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E9%9D%99%E6%80%81%E9%93%BE%E6%8E%A5&zhida_source=entity)且没有被引用到的话则很可能会绕过static的初始化。不过UE因为都是dll[动态链接](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5&zhida_source=entity)，且没有出现静态lib再引用Lib，然后又不引用文件的情况出现，所以避免了该问题。或者你也可以找个地方强制的去include一下来触发static初始化。

  

## UE Static 自动注册模式

而UE里同样是采用这种模式：

  

```cpp
template <typename TClass>
struct TClassCompiledInDefer : public FFieldCompiledInInfo
{
	TClassCompiledInDefer(const TCHAR* InName, SIZE_T InClassSize, uint32 InCrc)
	: FFieldCompiledInInfo(InClassSize, InCrc)
	{
		UClassCompiledInDefer(this, InName, InClassSize, InCrc);
	}
	virtual UClass* Register() const override
	{
		return TClass::StaticClass();
	}
};

static TClassCompiledInDefer<TClass> AutoInitialize##TClass(TEXT(#TClass), sizeof(TClass), TClassCrc); 
```

或者

  

```cpp
struct FCompiledInDefer
{
	FCompiledInDefer(class UClass *(*InRegister)(), class UClass *(*InStaticClass)(), const TCHAR* Name, bool bDynamic, const TCHAR* DynamicPackageName = nullptr, const TCHAR* DynamicPathName = nullptr, void (*InInitSearchableValues)(TMap<FName, FName>&) = nullptr)
	{
		if (bDynamic)
		{
			GetConvertedDynamicPackageNameToTypeName().Add(FName(DynamicPackageName), FName(Name));
		}
		UObjectCompiledInDefer(InRegister, InStaticClass, Name, bDynamic, DynamicPathName, InInitSearchableValues);
	}
};
static FCompiledInDefer Z_CompiledInDefer_UClass_UMyClass(Z_Construct_UClass_UMyClass, &UMyClass::StaticClass, TEXT("UMyClass"), false, nullptr, nullptr, nullptr);
```

都是对该模式的应用，把static变量声明再用宏包装一层，就可以实现一个简单的自动注册流程了。

  

## 收集

在上文里，我们详细介绍了Class、Struct、Enum、Interface的代码生成的信息，显然的，生成的就是为了拿过来用的。但是在用之前，我们就还得辛苦一番，把散乱分布在各个.h.cpp文件里的元数据都收集到我们想要的[数据结构](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84&zhida_source=entity)里保存，以便下一个阶段的使用。

这里回顾一下，为了让新创建的类不修改既有的代码，所以我们选择了[去中心化](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96&zhida_source=entity)的为每个新的类生成它自己的cpp生成文件——上文里已经分别介绍每个cpp文件的内容。但是这样我们就接着迎来了一个新问题：这些cpp文件里的元数据散乱在各个模块dll里，我们需要用一种方法重新归拢这些数据，这就是我们在一开头就提到的C++ Static自动注册模式了。通过这种模式，每个[cpp文件](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=5&q=cpp%E6%96%87%E4%BB%B6&zhida_source=entity)里的static对象在程序一开始的时候就会全部有机会去做一些事情，包括信息的收集工作。

UE4里也是如此，在程序启动的时候，UE利用了Static自动注册模式把所有类的信息都一一登记一遍。而紧接着另一个就是顺序问题了，这么多类，谁先谁后，互相若是有依赖该怎么解决。众所周知，UE是以Module来组织引擎结构的（关于Module的细节会在以后章节叙述），一个个Module可以通过脚本配置来选择性的编译加载。在[游戏引擎](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E6%B8%B8%E6%88%8F%E5%BC%95%E6%93%8E&zhida_source=entity)众多的模块中，玩家自己的Game模块是处于比较高级的层次的，都是依赖于引擎其他更基础底层的模块，而这些模块中，最最底层的就是Core模块（C++的基础库），接着就是CoreUObject，正是实现Object类型系统的模块！因此在类型系统注册的过程中，不止要注册玩家的Game模块，同时也要注册CoreUObject本身的一些支持类。

很多人可能会担心这么多模块的[静态初始化](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E9%9D%99%E6%80%81%E5%88%9D%E5%A7%8B%E5%8C%96&zhida_source=entity)的顺序正确性如何保证，在c++标准里，不同编译单元的全局[静态变量](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E9%9D%99%E6%80%81%E5%8F%98%E9%87%8F&zhida_source=entity)的初始化顺序并没有明确规定，因此实现上完全由编译器自己决定。该问题最好的解决方法是尽可能的避免这种情况，在设计上就让各个变量不互相引用依赖，同时也采用一些[二次检测](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E4%BA%8C%E6%AC%A1%E6%A3%80%E6%B5%8B&zhida_source=entity)的方式避免重复注册，或者触发一个强制引用来保证前置对象已经被初始化完成。目前在MSVC平台上是先注册玩家的Game模块，接着是CoreUObject，接着再其他，不过这其实无所谓的，只要保证不依赖顺序而结果正确，顺序就并不重要了。

  

## Static的收集

在讲完了收集的必要性和顺序问题的解决之后，我们再来分别的看各个类别的结构的信息的收集。依然是按照上文生成的顺序，从Class（Interface同理）开始，然后是Enum，接着Struct。接着请读者朋友们对照着上文的生成代码来理解。

  

## Class的收集

对照着上文里的Hello.generated.cpp展开，我们注意到里面有：

  

```cpp
static TClassCompiledInDefer<UMyClass> AutoInitializeUMyClass(TEXT("UMyClass"), sizeof(UMyClass), 899540749);
//……
static FCompiledInDefer Z_CompiledInDefer_UClass_UMyClass(Z_Construct_UClass_UMyClass, &UMyClass::StaticClass, TEXT("UMyClass"), false, nullptr, nullptr, nullptr);
```

再一次找到其定义：

  

```text
//Specialized version of the deferred class registration structure.
template <typename TClass>
struct TClassCompiledInDefer : public FFieldCompiledInInfo
{
	TClassCompiledInDefer(const TCHAR* InName, SIZE_T InClassSize, uint32 InCrc)
	: FFieldCompiledInInfo(InClassSize, InCrc)
	{
		UClassCompiledInDefer(this, InName, InClassSize, InCrc);    //收集信息
	}
	virtual UClass* Register() const override
	{
		return TClass::StaticClass();
	}
};

//Stashes the singleton function that builds a compiled in class. Later, this is executed.
struct FCompiledInDefer
{
	FCompiledInDefer(class UClass *(*InRegister)(), class UClass *(*InStaticClass)(), const TCHAR* Name, bool bDynamic, const TCHAR* DynamicPackageName = nullptr, const TCHAR* DynamicPathName = nullptr, void (*InInitSearchableValues)(TMap<FName, FName>&) = nullptr)
	{
		if (bDynamic)
		{
			GetConvertedDynamicPackageNameToTypeName().Add(FName(DynamicPackageName), FName(Name));
		}
		UObjectCompiledInDefer(InRegister, InStaticClass, Name, bDynamic, DynamicPathName, InInitSearchableValues);//收集信息
	}
};
```

可以见到前者调用了UClassCompiledInDefer来收集类名字，类大小，CRC信息，并把自己的指针保存进来以便后续调用Register方法。而UObjectCompiledInDefer（现在暂时不考虑动态类）最重要的收集的信息就是第一个用于构造UClass*对象的[函数指针](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E5%87%BD%E6%95%B0%E6%8C%87%E9%92%88&zhida_source=entity)回调。

再往下我们会发现这二者其实都只是在一个静态Array里添加信息记录：

  

```cpp
void UClassCompiledInDefer(FFieldCompiledInInfo* ClassInfo, const TCHAR* Name, SIZE_T ClassSize, uint32 Crc)
{
    //...
    // We will either create a new class or update the static class pointer of the existing one
	GetDeferredClassRegistration().Add(ClassInfo);  //static TArray<FFieldCompiledInInfo*> DeferredClassRegistration;
}
void UObjectCompiledInDefer(UClass *(*InRegister)(), UClass *(*InStaticClass)(), const TCHAR* Name, bool bDynamic, const TCHAR* DynamicPathName, void (*InInitSearchableValues)(TMap<FName, FName>&))
{
    //...
	GetDeferredCompiledInRegistration().Add(InRegister);    //static TArray<class UClass *(*)()> DeferredCompiledInRegistration;
}
```

而在整个引擎里会触发此Class的[信息收集](https://zhida.zhihu.com/search?content_id=2594561&content_type=Article&match_order=1&q=%E4%BF%A1%E6%81%AF%E6%94%B6%E9%9B%86&zhida_source=entity)的有UCLASS、UINTERFACE、IMPLEMENT_INTRINSIC_CLASS、IMPLEMENT_CORE_INTRINSIC_CLASS，其中UCLASS和UINTERFACE我们上文已经见识过了，而IMPLEMENT_INTRINSIC_CLASS是用于在代码中包装UModel，IMPLEMENT_CORE_INTRINSIC_CLASS是用于包装UField、UClass等引擎内建的类，后两者内部也都调用了IMPLEMENT_CLASS来实现功能。