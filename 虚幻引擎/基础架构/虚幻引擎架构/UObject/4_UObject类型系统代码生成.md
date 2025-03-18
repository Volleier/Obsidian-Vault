假设UHT已经分析得到了足够的类型元数据信息，下一步就是利用这个信息在程序内存中构建起前文的类型系统结构，这个过程我们称之为注册。同一般程序的构建流程需要经过预处理、编译、汇编、链接一样，UE为了在内存中模拟构建的过程，在概念上也需要以下几个阶段：生成，收集，注册，链接。
本篇章阐述UHT生成的代码的样式。首先从一个最简单的UMyClass开始，观察整体生成代码的结构，接着推进到UMyEnum、UMyStruct、UMyInterface的代码样式，最后返归到UMyClass在其中添加进属性和方法，观察属性和方法是怎么生成代码和怎么和UClass\*对象关联起来的。其实我们也发现，这个阶段最重要的功能就是尽量的把程序的信息用代码给记录下来，对于Enum记下名字和值；对于Struct记下每个Property的名字和字节偏移；对于Interface记下每个函数或包装函数的的函数指针和名字；对于Class同理都记下Property和Function。
# C++ Static Lazy初始化模式

一种我们常用，也是UE中常用的单件懒惰初始化模式是：
```cpp
Hello* StaticGetHello()
{
    static Hello* obj=nullptr;
    if(!obj)
    {
        obj=...
    }
    return obj;
}
或者
Hello& StaticGetHello()
{
    static Hello obj(...);
    return obj;
}
```

前者非常简单，也没考虑多线程安全，但是在单线程环境下足够用了。用指针的原因是，有一些情况，这些对象的生命周期是由别的地方来管理的，比如UE里的GC，因此这里只static化一个指针。否则的话，还是后者更加简洁和安全。

# UHT代码生成

在C++程序中的预处理是用来对源代码进行宏展开，预编译指令处理，注释删除等操作。同样的，一旦我们采用了宏标记的方法，不管是怎么个标记语法，我们都需要进行简单或复杂的词法分析，提取出有用的信息，然后生成所需要的代码。一般来说，在引擎中创建一个不继承的MyClass类，UHT会为我们生成以下4个文件（位于\$ProjectName\$\Intermediate\Build\Win64\\\$ProjectName\$\Inc\\$MyClassName\$）
- MyClass.generated.h：MyClass的生成头文件
- \$ProjectName\$.generated.dep.h：\$ProjectName\$%.generated.cpp的依赖头文件，在顺序包含上述的MyClass.h
- \$ProjectName\$.generated.cpp：该项目的实现编译单元。
# UCLASS的生成代码剖析

先从一个最简单的UMyClass的开始，总览分析生成的代码结构，接着再继而观察其他UEnum、UStruct、UInterface、UProperty、UFunction的代码生成样式。

  

### MyClass.h

首先是我们自己编写或者引擎帮我们生成的文件样式：

  

```text
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "UObject/NoExportTypes.h"
#include "MyClass.generated.h"

UCLASS()
class HELLO_API UMyClass : public UObject
{
	GENERATED_BODY()
};
```

**第5行：**‘#include "UObject/NoExportTypes.h’" 通过查看文件内容，发现这个文件在编译的时候就是Include了其他一些更基础的头文件，比如#include "Math/Vector.h"，因此你才能在MyClass里不用include就引用这些类。当然，还有一些内容是专门供UHT使用来生成蓝图类型的，现在暂时不需要管。

**第6行：**‘#include "MyClass.generated.h"’，就是为了引用生成的头文件。这里请注意的是，该文件include位置在类声明的前面，之后谈到宏处理的时候会用到该信息。

**第11行：**GENERATED_BODY()，该宏是重中之重，其他的UCLASS宏只是提供信息，不参与编译，而GENERATED_BODY正是把声明和元数据定义关联到一起的枢纽。继续查看宏定义：

  

```text
#define BODY_MACRO_COMBINE_INNER(A,B,C,D) A##B##C##D
#define BODY_MACRO_COMBINE(A,B,C,D) BODY_MACRO_COMBINE_INNER(A,B,C,D)
#define GENERATED_BODY(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_GENERATED_BODY)
```

会发现GENERATED_BODY最终其实只是生成另外一个宏的名称，因为：

```text
CURRENT_FILE_ID的定义是在MyClass.generated.h的89行：\#define CURRENT_FILE_ID Hello_Source_Hello_MyClass_h，这是UHT通过分析文件得到的信息。

__LINE__标准宏指向了该宏使用时候的的行数，这里是11。加了一个__LINE__宏的目的是为了支持在同一个文件内声明多个类，比如在MyClass.h里接着再声明UMyClass2，就可以支持生成不同的宏名称。
```

因此总而生成的宏名称是Hello_Source_Hello_MyClass_h_11_GENERATED_BODY，而这个宏就是定义在MyClass.generated.h的77行。值得一提的是，如果MyClass类需要UMyClass(const FObjectInitializer& ObjectInitializer)的构造函数自定义实现，则需要用GENERATED_UCLASS_BODY宏来让最终生成的宏指向Hello_Source_Hello_MyClass_h_11_GENERATED_BODY_LEGACY（MyClass.generated.h的66行），其最终展开的内容会多一个构造函数的内容实现。

  

### MyClass.generated.h

UHT分析生成的文件内容如下：

  

```cpp
PRAGMA_DISABLE_DEPRECATION_WARNINGS
#ifdef HELLO_MyClass_generated_h
#error "MyClass.generated.h already included, missing '#pragma once' in MyClass.h"
#endif
#define HELLO_MyClass_generated_h

#define Hello_Source_Hello_MyClass_h_11_RPC_WRAPPERS    //先忽略
#define Hello_Source_Hello_MyClass_h_11_RPC_WRAPPERS_NO_PURE_DECLS  //先忽略
#define Hello_Source_Hello_MyClass_h_11_INCLASS_NO_PURE_DECLS \
	private: \
	static void StaticRegisterNativesUMyClass(); \
	friend HELLO_API class UClass* Z_Construct_UClass_UMyClass(); \
	public: \
	DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), 0, TEXT("/Script/Hello"), NO_API) \
	DECLARE_SERIALIZER(UMyClass) \
	/** Indicates whether the class is compiled into the engine */ \
	enum {IsIntrinsic=COMPILED_IN_INTRINSIC};


#define Hello_Source_Hello_MyClass_h_11_INCLASS \
	private: \
	static void StaticRegisterNativesUMyClass(); \
	friend HELLO_API class UClass* Z_Construct_UClass_UMyClass(); \
	public: \
	DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), 0, TEXT("/Script/Hello"), NO_API) \
	DECLARE_SERIALIZER(UMyClass) \
	/** Indicates whether the class is compiled into the engine */ \
	enum {IsIntrinsic=COMPILED_IN_INTRINSIC};


#define Hello_Source_Hello_MyClass_h_11_STANDARD_CONSTRUCTORS \
	/** Standard constructor, called after all reflected properties have been initialized */ \
	NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get()); \
	DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyClass) \
	DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyClass); \
DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyClass); \
private: \
	/** Private move- and copy-constructors, should never be used */ \
	NO_API UMyClass(UMyClass&&); \
	NO_API UMyClass(const UMyClass&); \
public:


#define Hello_Source_Hello_MyClass_h_11_ENHANCED_CONSTRUCTORS \
	/** Standard constructor, called after all reflected properties have been initialized */ \
	NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get()) : Super(ObjectInitializer) { }; \
private: \
	/** Private move- and copy-constructors, should never be used */ \
	NO_API UMyClass(UMyClass&&); \
	NO_API UMyClass(const UMyClass&); \
public: \
	DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyClass); \
DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyClass); \
	DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyClass)


#define Hello_Source_Hello_MyClass_h_11_PRIVATE_PROPERTY_OFFSET     //先忽略
#define Hello_Source_Hello_MyClass_h_8_PROLOG   //先忽略
#define Hello_Source_Hello_MyClass_h_11_GENERATED_BODY_LEGACY \ //两个重要的定义
PRAGMA_DISABLE_DEPRECATION_WARNINGS \
public: \
	Hello_Source_Hello_MyClass_h_11_PRIVATE_PROPERTY_OFFSET \
	Hello_Source_Hello_MyClass_h_11_RPC_WRAPPERS \
	Hello_Source_Hello_MyClass_h_11_INCLASS \
	Hello_Source_Hello_MyClass_h_11_STANDARD_CONSTRUCTORS \
public: \
PRAGMA_ENABLE_DEPRECATION_WARNINGS


#define Hello_Source_Hello_MyClass_h_11_GENERATED_BODY \    //两个重要的定义
PRAGMA_DISABLE_DEPRECATION_WARNINGS \
public: \
	Hello_Source_Hello_MyClass_h_11_PRIVATE_PROPERTY_OFFSET \
	Hello_Source_Hello_MyClass_h_11_RPC_WRAPPERS_NO_PURE_DECLS \
	Hello_Source_Hello_MyClass_h_11_INCLASS_NO_PURE_DECLS \
	Hello_Source_Hello_MyClass_h_11_ENHANCED_CONSTRUCTORS \
private: \
PRAGMA_ENABLE_DEPRECATION_WARNINGS

#undef CURRENT_FILE_ID
#define CURRENT_FILE_ID Hello_Source_Hello_MyClass_h    //前文说过的定义
PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

该文件都是宏定义，因为宏定义是有前后顺序的，因此咱们从尾向前看，请读者此时和上文的代码对照着看。  
首先最底下是CURRENT_FILE_ID的定义

接着是两个上文说过的GENERATED_BODY定义，先从最简单的结构开始，不管那些PRIVATE_PROPERTY_OFFSET和PROLOG，以后会慢慢介绍到。这两个宏接着包含了4个声明在上面的其他宏。目前来说Hello_Source_Hello_MyClass_h_11_INCLASS和Hello_Source_Hello_MyClass_h_11_INCLASS_NO_PURE_DECLS的定义一模一样，而Hello_Source_Hello_MyClass_h_11_STANDARD_CONSTRUCTORS和Hello_Source_Hello_MyClass_h_11_ENHANCED_CONSTRUCTORS的宏，如果读者仔细查看对照的话，会发现二者只差了“: Super(ObjectInitializer) { }; ”构造函数的默认实现。

我们继续往上，以Hello_Source_Hello_MyClass_h_11_ENHANCED_CONSTRUCTORS为例：

  

```cpp
#define Hello_Source_Hello_MyClass_h_11_ENHANCED_CONSTRUCTORS \
	/** Standard constructor, called after all reflected properties have been initialized */ \
	NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get()) : Super(ObjectInitializer) { }; \   //默认的构造函数实现
private: \  //禁止掉C++11的移动和拷贝构造
	/** Private move- and copy-constructors, should never be used */ \
	NO_API UMyClass(UMyClass&&); \
	NO_API UMyClass(const UMyClass&); \
public: \
	DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyClass); \     //因为WITH_HOT_RELOAD_CTORS关闭，展开是空宏
    DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyClass); \   //同理，空宏
	DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyClass)
```

继续查看DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL的定义：

  

```text
#define DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass) \
	static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass(X); }
```

声明定义了一个构造函数包装器。需要这么做的原因是，在根据名字反射创建对象的时候，需要调用该类的构造函数。可是类的构造函数并不能用函数指针指向，因此这里就用一个static函数包装一下，变成一个"平凡"的函数指针，而且所有类的签名一致，就可以在UClass里用一个函数指针里保存起来。见引擎里Class.h的声明：

  

```cpp
class COREUOBJECT_API UClass : public UStruct
...
{
    ...
	typedef void (*ClassConstructorType) (const FObjectInitializer&);
	ClassConstructorType ClassConstructor;
	...
}
```

当然，如果读者需要自己实现一套反射框架的时候也可以采用更简洁的模式，采用模板实现也是异曲同工。

  

```text
template<class TClass>
void MyConstructor( const FObjectInitializer& X )
{ 
	new((EInternal*)X.GetObj())TClass(X);
}
```

再继续往上：

  

```cpp
#define Hello_Source_Hello_MyClass_h_11_INCLASS \
	private: \
	static void StaticRegisterNativesUMyClass(); \  //定义在cpp中，目前都是空实现
	friend HELLO_API class UClass* Z_Construct_UClass_UMyClass(); \ //一个构造该类UClass对象的辅助函数
	public: \
	DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), 0, TEXT("/Script/Hello"), NO_API) \   //声明该类的一些通用基本函数
	DECLARE_SERIALIZER(UMyClass) \  //声明序列化函数
	/** Indicates whether the class is compiled into the engine */ \
	enum {IsIntrinsic=COMPILED_IN_INTRINSIC};   //这个标记指定了该类是C++Native类，不能动态再改变，跟蓝图里构造的动态类进行区分。
```

可以说DECLARE_CLASS是最重要的一个声明，对照着定义：DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), 0, TEXT("/Script/Hello"), NO_API)

- TClass：类名 TSuperClass：基类名字
- TStaticFlags：类的属性标记，这里是0，表示最默认，不带任何其他属性。读者可以查看EClassFlags枚举来查看其他定义。
- TStaticCastFlags：指定了该类可以转换为哪些类，这里为0表示不能转为那些默认的类，读者可以自己查看EClassCastFlags声明来查看具体有哪些默认类转换。
- TPackage：类所处于的包名，所有的对象都必须处于一个包中，而每个包都具有一个名字，可以通过该名字来查找。这里是"/Script/Hello"，指定是Script下的Hello，Script可以理解为用户自己的实现，不管是C++还是蓝图，都可以看作是引擎外的一种脚本，当然用这个名字也肯定有UE3时代UnrealScript的影子。Hello就是项目名字，该项目下定义的对象处于该包中。Package的概念涉及到后续Object的组织方式，目前可以简单理解为一个大的Object包含了其他子Object。
- TRequiredAPI：就是用来Dll导入导出的标记，这里是NO_API，因为是最终exe，不需要导出。

  

```cpp
#define DECLARE_CLASS( TClass, TSuperClass, TStaticFlags, TStaticCastFlags, TPackage, TRequiredAPI  ) \
private: \
    TClass& operator=(TClass&&);   \
    TClass& operator=(const TClass&);   \
	TRequiredAPI static UClass* GetPrivateStaticClass(const TCHAR* Package); \
public: \
	/** Bitwise union of #EClassFlags pertaining to this class.*/ \
	enum {StaticClassFlags=TStaticFlags}; \
	/** Typedef for the base class ({{ typedef-type }}) */ \
	typedef TSuperClass Super;\
	/** Typedef for {{ typedef-type }}. */ \
	typedef TClass ThisClass;\
	/** Returns a UClass object representing this class at runtime */ \
	inline static UClass* StaticClass() \
	{ \
		return GetPrivateStaticClass(TPackage); \
	} \
	/** Returns the StaticClassFlags for this class */ \
	inline static EClassCastFlags StaticClassCastFlags() \
	{ \
		return TStaticCastFlags; \
	} \
	DEPRECATED(4.7, "operator new has been deprecated for UObjects - please use NewObject or NewNamedObject instead") \
	inline void* operator new( const size_t InSize, UObject* InOuter=(UObject*)GetTransientPackage(), FName InName=NAME_None, EObjectFlags InSetFlags=RF_NoFlags ) \
	{ \
		return StaticAllocateObject( StaticClass(), InOuter, InName, InSetFlags ); \
	} \
	/** For internal use only; use StaticConstructObject() to create new objects. */ \
	inline void* operator new(const size_t InSize, EInternal InInternalOnly, UObject* InOuter = (UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags) \
	{ \
		return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags); \
} \
	/** For internal use only; use StaticConstructObject() to create new objects. */ \
	inline void* operator new( const size_t InSize, EInternal* InMem ) \
	{ \
		return (void*)InMem; \
	}
```

大部分都是不言自明的，这里的StaticClass就是我们最经常用到的函数，其内部调用了GetPrivateStaticClass，而其实现正是在Hello.generated.cpp里的。

  

### Hello.generated.cpp

而整个Hello项目会生成一个Hello.generated.cpp

  

```cpp
#include "Hello.h"      //包含该项目的头文件，继而包含Engine.h
#include "GeneratedCppIncludes.h"   //包含UObject模块里一些必要头文件
#include "Hello.generated.dep.h"    //引用依赖文件，继而include了MyClass.h
PRAGMA_DISABLE_DEPRECATION_WARNINGS
void EmptyLinkFunctionForGeneratedCode1Hello() {}   //先忽略
	void UMyClass::StaticRegisterNativesUMyClass()  //说是静态注册，但现在都是为空，先忽略
	{
	}
	IMPLEMENT_CLASS(UMyClass, 899540749);   //重要！！！
#if USE_COMPILED_IN_NATIVES //该宏编译的时候会打开
// Cross Module References
	COREUOBJECT_API class UClass* Z_Construct_UClass_UObject(); //引用CoreUObject里的函数，主要是为了得到UObject本身对应的UClass

	HELLO_API class UClass* Z_Construct_UClass_UMyClass_NoRegister();   //构造UMyClass对应的UClass对象，区别是没有后续的注册过程
	HELLO_API class UClass* Z_Construct_UClass_UMyClass();  //构造UMyClass对应的UClass对象
	HELLO_API class UPackage* Z_Construct_UPackage__Script_Hello(); //构造Hello本身的UPackage对象
	UClass* Z_Construct_UClass_UMyClass_NoRegister()
	{
		return UMyClass::StaticClass(); //直接通过访问来获取UClass对象
	}
	UClass* Z_Construct_UClass_UMyClass()   //构造并注册
	{
		static UClass* OuterClass = NULL;   //static lazy模式
		if (!OuterClass)
		{
			Z_Construct_UClass_UObject();   //确保UObject本身的UClass已经注册生成
			Z_Construct_UPackage__Script_Hello();   //确保当前Hello项目的UPackage已经创建，因为后续在生成UMyClass的UClass*对象时需要保存在这个UPacage中
			OuterClass = UMyClass::StaticClass();   //访问获得UClass*
			if (!(OuterClass->ClassFlags & CLASS_Constructed))  //防止重复注册
			{
				UObjectForceRegistration(OuterClass);   //提取信息注册自身
				OuterClass->ClassFlags |= 0x20100080;   //增加CLASS_Constructed|CLASS_RequiredAPI标记


				OuterClass->StaticLink();   //“静态”链接，后续解释
#if WITH_METADATA   //编辑器模式下开始
				UMetaData* MetaData = OuterClass->GetOutermost()->GetMetaData();    //获取关联到的UPackage其身上的元数据映射，并增加元数据信息
				MetaData->SetValue(OuterClass, TEXT("IncludePath"), TEXT("MyClass.h"));
				MetaData->SetValue(OuterClass, TEXT("ModuleRelativePath"), TEXT("MyClass.h"));
#endif
			}
		}
		check(OuterClass->GetClass());
		return OuterClass;
	}
	static FCompiledInDefer Z_CompiledInDefer_UClass_UMyClass(Z_Construct_UClass_UMyClass, &UMyClass::StaticClass, TEXT("UMyClass"), false, nullptr, nullptr, nullptr);    //延迟注册，注入信息，在启动的时候调用
	DEFINE_VTABLE_PTR_HELPER_CTOR(UMyClass);    //HotReload相关，先忽略
	UPackage* Z_Construct_UPackage__Script_Hello()  //构造Hello的UPackage
	{
		static UPackage* ReturnPackage = NULL;
		if (!ReturnPackage)
		{
			ReturnPackage = CastChecked<UPackage>(StaticFindObjectFast(UPackage::StaticClass(), NULL, FName(TEXT("/Script/Hello")), false, false));//注意的是这里只是做一个查找，真正的CreatePackage是在UObjectBase::DeferredRegister里调用的，后续在流程里会讨论到
			ReturnPackage->SetPackageFlags(PKG_CompiledIn | 0x00000000);//设定标记和Guid
			FGuid Guid;
			Guid.A = 0x79A097CD;
			Guid.B = 0xB58D8B48;
			Guid.C = 0x00000000;
			Guid.D = 0x00000000;
			ReturnPackage->SetGuid(Guid);

		}
		return ReturnPackage;
	}
#endif

PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

大部分简单的都注释说明了，本文件的关键点在于IMPLEMENT_CLASS的分析，和上文.h中的DECLARE_CLASS对应，其声明如下：  
对照着定义IMPLEMENT_CLASS(UMyClass, 899540749);

  

```cpp
#define IMPLEMENT_CLASS(TClass, TClassCrc) \
	static TClassCompiledInDefer<TClass> AutoInitialize##TClass(TEXT(#TClass), sizeof(TClass), TClassCrc); \   //延迟注册
	UClass* TClass::GetPrivateStaticClass(const TCHAR* Package) \   //.h里声明的实现，StaticClas()内部就是调用该函数
	{ \
		static UClass* PrivateStaticClass = NULL; \ //又一次static lazy
		if (!PrivateStaticClass) \
		{ \
			/* this could be handled with templates, but we want it external to avoid code bloat */ \
			GetPrivateStaticClassBody( \    //该函数就是真正创建UClass*,以后
				Package, \  //Package名字
				(TCHAR*)TEXT(#TClass) + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0), \//类名，+1去掉U、A、F前缀，+11去掉_Deprecated前缀
				PrivateStaticClass, \   //输出引用
				StaticRegisterNatives##TClass, \
				sizeof(TClass), \
				TClass::StaticClassFlags, \
				TClass::StaticClassCastFlags(), \
				TClass::StaticConfigName(), \
				(UClass::ClassConstructorType)InternalConstructor<TClass>, \
				(UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<TClass>, \
				&TClass::AddReferencedObjects, \
				&TClass::Super::StaticClass, \
				&TClass::WithinClass::StaticClass \
			); \
		} \
		return PrivateStaticClass; \
	}
```

内容也比较简单，就是把该类的信息传进去给GetPrivateStaticClassBody函数

## 最后展开结果

通过人肉预处理展开一下生成的文件，应该会看得更加清楚一些：  
**MyClass.h展开**

  

```cpp
#pragma once
#include "UObject/NoExportTypes.h"

class HELLO_API UMyClass : public UObject
{
private:
	static void StaticRegisterNativesUMyClass();
	friend HELLO_API class UClass* Z_Construct_UClass_UMyClass();
private:
	UMyClass& operator=(UMyClass&&);
	UMyClass& operator=(const UMyClass&);
	NO_API static UClass* GetPrivateStaticClass(const TCHAR* Package);
public:
	/** Bitwise union of #EClassFlags pertaining to this class.*/
	enum {StaticClassFlags = CLASS_Intrinsic};
	/** Typedef for the base class ({{ typedef-type }}) */
	typedef UObject Super;
	/** Typedef for {{ typedef-type }}. */
	typedef UMyClass ThisClass;
	/** Returns a UClass object representing this class at runtime */
	inline static UClass* StaticClass()
	{
		return GetPrivateStaticClass(TEXT("/Script/Hello"));
	}
	/** Returns the StaticClassFlags for this class */
	inline static EClassCastFlags StaticClassCastFlags()
	{
		return 0;
	}
	DEPRECATED(4.7, "operator new has been deprecated for UObjects - please use NewObject or NewNamedObject instead")
	inline void* operator new(const size_t InSize, UObject* InOuter = (UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags)
	{
		return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags);
	}
	/** For internal use only; use StaticConstructObject() to create new objects. */
	inline void* operator new(const size_t InSize, EInternal InInternalOnly, UObject* InOuter = (UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags)
	{
		return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags);
	}
	/** For internal use only; use StaticConstructObject() to create new objects. */
	inline void* operator new(const size_t InSize, EInternal* InMem)
	{
		return (void*)InMem;
	}

	friend FArchive &operator<<(FArchive& Ar, UMyClass*& Res)
	{
		return Ar << (UObject*&)Res;
	}
	/** Indicates whether the class is compiled into the engine */
	enum { IsIntrinsic = COMPILED_IN_INTRINSIC };

	/** Standard constructor, called after all reflected properties have been initialized */
	NO_API UMyClass(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get()) : Super(ObjectInitializer) { };
private:
	/** Private move- and copy-constructors, should never be used */
	NO_API UMyClass(UMyClass&&);
	NO_API UMyClass(const UMyClass&);
public:
	static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())UMyClass(X); }
};
```

**Hello.generated.cpp展开**

  

```cpp
//#include "Hello.h"
#include "Engine.h"	

//#include "GeneratedCppIncludes.h"
#include "CoreUObject.h"
#include "UObject/Object.h"
#include "UObject/Class.h"
#include "UObject/Package.h"
#include "UObject/MetaData.h"
#include "UObject/UnrealType.h"

//#include "Hello.generated.dep.h"
#include "MyClass.h"

void EmptyLinkFunctionForGeneratedCode1Hello() {}
void UMyClass::StaticRegisterNativesUMyClass()
{
}
static TClassCompiledInDefer<UMyClass> AutoInitializeUMyClass(TEXT("UMyClass"), sizeof(UMyClass), 899540749);
UClass* UMyClass::GetPrivateStaticClass(const TCHAR* Package)
{
	static UClass* PrivateStaticClass = NULL;
	if (!PrivateStaticClass)
	{
		/* this could be handled with templates, but we want it external to avoid code bloat */
		GetPrivateStaticClassBody(
			Package,
			(TCHAR*)TEXT("UMyClass") + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0),
			PrivateStaticClass,
			StaticRegisterNativesUMyClass,
			sizeof(UMyClass),
			UMyClass::StaticClassFlags,
			UMyClass::StaticClassCastFlags(),
			UMyClass::StaticConfigName(),
			(UClass::ClassConstructorType)InternalConstructor<UMyClass>,
			(UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<UMyClass>,
			&UMyClass::AddReferencedObjects,
			&UMyClass::Super::StaticClass,
			&UMyClass::WithinClass::StaticClass
		);
	}
	return PrivateStaticClass;
}

// Cross Module References
COREUOBJECT_API class UClass* Z_Construct_UClass_UObject();

HELLO_API class UClass* Z_Construct_UClass_UMyClass_NoRegister();
HELLO_API class UClass* Z_Construct_UClass_UMyClass();
HELLO_API class UPackage* Z_Construct_UPackage__Script_Hello();
UClass* Z_Construct_UClass_UMyClass_NoRegister()
{
	return UMyClass::StaticClass();
}
UClass* Z_Construct_UClass_UMyClass()
{
	static UClass* OuterClass = NULL;
	if (!OuterClass)
	{
		Z_Construct_UClass_UObject();
		Z_Construct_UPackage__Script_Hello();
		OuterClass = UMyClass::StaticClass();
		if (!(OuterClass->ClassFlags & CLASS_Constructed))
		{
			UObjectForceRegistration(OuterClass);
			OuterClass->ClassFlags |= 0x20100080;


			OuterClass->StaticLink();
#if WITH_METADATA
			UMetaData* MetaData = OuterClass->GetOutermost()->GetMetaData();
			MetaData->SetValue(OuterClass, TEXT("IncludePath"), TEXT("MyClass.h"));
			MetaData->SetValue(OuterClass, TEXT("ModuleRelativePath"), TEXT("MyClass.h"));
#endif
		}
	}
	check(OuterClass->GetClass());
	return OuterClass;
}
static FCompiledInDefer Z_CompiledInDefer_UClass_UMyClass(Z_Construct_UClass_UMyClass, &UMyClass::StaticClass, TEXT("UMyClass"), false, nullptr, nullptr, nullptr);
UPackage* Z_Construct_UPackage__Script_Hello()
{
	static UPackage* ReturnPackage = NULL;
	if (!ReturnPackage)
	{
		ReturnPackage = CastChecked<UPackage>(StaticFindObjectFast(UPackage::StaticClass(), NULL, FName(TEXT("/Script/Hello")), false, false));
		ReturnPackage->SetPackageFlags(PKG_CompiledIn | 0x00000000);
		FGuid Guid;
		Guid.A = 0x79A097CD;
		Guid.B = 0xB58D8B48;
		Guid.C = 0x00000000;
		Guid.D = 0x00000000;
		ReturnPackage->SetGuid(Guid);

	}
	return ReturnPackage;
}
```

这样.h的声明和.cpp的定义就全都有了。不管定义了多少函数，要记得注册的入口就是那两个[static对象](https://zhida.zhihu.com/search?content_id=2225851&content_type=Article&match_order=1&q=static%E5%AF%B9%E8%B1%A1&zhida_source=entity)在程序启动的时候登记信息，才有了之后的注册。

  

## UENUM的生成代码剖析

接着是相对简单的Enum，我们测试的Enum如下：

  

```cpp
#pragma once
#include "UObject/NoExportTypes.h"
#include "MyEnum.generated.h"

UENUM(BlueprintType)
enum class EMyEnum : uint8
{
	MY_Dance 	UMETA(DisplayName = "Dance"),
	MY_Rain 	UMETA(DisplayName = "Rain"),
	MY_Song		UMETA(DisplayName = "Song")
};
```

生成的MyEnum.generated.h为：

  

```cpp
PRAGMA_DISABLE_DEPRECATION_WARNINGS
#ifdef HELLO_MyEnum_generated_h
#error "MyEnum.generated.h already included, missing '#pragma once' in MyEnum.h"
#endif
#define HELLO_MyEnum_generated_h

#undef CURRENT_FILE_ID
#define CURRENT_FILE_ID Hello_Source_Hello_MyEnum_h

#define FOREACH_ENUM_EMYENUM(op) \  //定义一个遍历枚举值的宏，只是为了方便使用
	op(EMyEnum::MY_Dance) \
	op(EMyEnum::MY_Rain) \
	op(EMyEnum::MY_Song) 
PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

因此Enum也非常简单，所以发现生成的其实也没有什么重要的信息。同样的，生成的Hello.genrated.cpp中：

  

```cpp
#include "Hello.h"
#include "GeneratedCppIncludes.h"
#include "Hello.generated.dep.h"
PRAGMA_DISABLE_DEPRECATION_WARNINGS
void EmptyLinkFunctionForGeneratedCode1Hello() {}
static class UEnum* EMyEnum_StaticEnum()    //定义一个获取UEnum便利函数，会在延迟注册的时候被用到
{
	extern HELLO_API class UPackage* Z_Construct_UPackage__Script_Hello();
	static class UEnum* Singleton = NULL;
	if (!Singleton)
	{
		extern HELLO_API class UEnum* Z_Construct_UEnum_Hello_EMyEnum();
		Singleton = GetStaticEnum(Z_Construct_UEnum_Hello_EMyEnum, Z_Construct_UPackage__Script_Hello(), TEXT("EMyEnum"));
	}
	return Singleton;
}
static FCompiledInDeferEnum Z_CompiledInDeferEnum_UEnum_EMyEnum(EMyEnum_StaticEnum, TEXT("/Script/Hello"), TEXT("EMyEnum"), false, nullptr, nullptr);   //延迟注册
#if USE_COMPILED_IN_NATIVES
	HELLO_API class UEnum* Z_Construct_UEnum_Hello_EMyEnum();
	HELLO_API class UPackage* Z_Construct_UPackage__Script_Hello();
	UEnum* Z_Construct_UEnum_Hello_EMyEnum()    //构造EMyEnum关联的UEnum*
	{
		UPackage* Outer=Z_Construct_UPackage__Script_Hello();
		extern uint32 Get_Z_Construct_UEnum_Hello_EMyEnum_CRC();
		static UEnum* ReturnEnum = FindExistingEnumIfHotReloadOrDynamic(Outer, TEXT("EMyEnum"), 0, Get_Z_Construct_UEnum_Hello_EMyEnum_CRC(), false);
		if (!ReturnEnum)
		{
			ReturnEnum = new(EC_InternalUseOnlyConstructor, Outer, TEXT("EMyEnum"), RF_Public|RF_Transient|RF_MarkAsNative) UEnum(FObjectInitializer());//直接创建该UEnum对象
			TArray<TPair<FName, uint8>> EnumNames;//设置枚举里的名字和值
			EnumNames.Add(TPairInitializer<FName, uint8>(FName(TEXT("EMyEnum::MY_Dance")), 0));
			EnumNames.Add(TPairInitializer<FName, uint8>(FName(TEXT("EMyEnum::MY_Rain")), 1));
			EnumNames.Add(TPairInitializer<FName, uint8>(FName(TEXT("EMyEnum::MY_Song")), 2));
			EnumNames.Add(TPairInitializer<FName, uint8>(FName(TEXT("EMyEnum::MY_MAX")), 3));   //添加一个默认的MAX字段
			ReturnEnum->SetEnums(EnumNames, UEnum::ECppForm::EnumClass);
			ReturnEnum->CppType = TEXT("EMyEnum");
#if WITH_METADATA   //设置元数据
			UMetaData* MetaData = ReturnEnum->GetOutermost()->GetMetaData();
			MetaData->SetValue(ReturnEnum, TEXT("BlueprintType"), TEXT("true"));
			MetaData->SetValue(ReturnEnum, TEXT("ModuleRelativePath"), TEXT("MyEnum.h"));
			MetaData->SetValue(ReturnEnum, TEXT("MY_Dance.DisplayName"), TEXT("Dance"));
			MetaData->SetValue(ReturnEnum, TEXT("MY_Rain.DisplayName"), TEXT("Rain"));
			MetaData->SetValue(ReturnEnum, TEXT("MY_Song.DisplayName"), TEXT("Song"));
#endif
		}
		return ReturnEnum;
	}
	uint32 Get_Z_Construct_UEnum_Hello_EMyEnum_CRC() { return 2000113000U; }
	UPackage* Z_Construct_UPackage__Script_Hello()  //设置Hello项目的Package属性
	{
		...略
	}
#endif

PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

观察发现EMyEnum_StaticEnum其实并没有比Z_Construct_UEnum_Hello_EMyEnum实现更多的其他的功能。GetStaticEnum目前的实现内部也只是非常简单的调用Z_Construct_UEnum_Hello_EMyEnum返回结果。所以保留着这个EMyEnum_StaticEnum或许只是为了和UClass的结构保持一致。

  

## USTRUCT的生成代码剖析

因为USTRUCT标记的类内部并不能定义函数，因此测试的Struct如下：

  

```text
#pragma once
#include "UObject/NoExportTypes.h"
#include "MyStruct.generated.h"

USTRUCT(BlueprintType)
struct HELLO_API FMyStruct
{
	GENERATED_USTRUCT_BODY()

	UPROPERTY(BlueprintReadWrite)
	float Score;
};
```

生成的MyStruct.generated.h如下：

  

```cpp
PRAGMA_DISABLE_DEPRECATION_WARNINGS
#ifdef HELLO_MyStruct_generated_h
#error "MyStruct.generated.h already included, missing '#pragma once' in MyStruct.h"
#endif
#define HELLO_MyStruct_generated_h

#define Hello_Source_Hello_MyStruct_h_8_GENERATED_BODY \
	friend HELLO_API class UScriptStruct* Z_Construct_UScriptStruct_FMyStruct(); \  //给予全局方法友元权限
	static class UScriptStruct* StaticStruct(); //静态函数返回UScriptStruct*
#undef CURRENT_FILE_ID
#define CURRENT_FILE_ID Hello_Source_Hello_MyStruct_h
PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

同理，根据GENERATED_USTRUCT_BODY的定义，最终会替换成Hello_Source_Hello_MyStruct_h_8_GENERATED_BODY宏。我们发现其实作用只是在内部定义了一个StaticStruct函数，因为FMyStruct并不继承于UObject，所以结构也非常的简单。  
再接着是Hello.genrated.cpp：

  

```cpp
#include "Hello.h"
#include "GeneratedCppIncludes.h"
#include "Hello.generated.dep.h"
PRAGMA_DISABLE_DEPRECATION_WARNINGS
void EmptyLinkFunctionForGeneratedCode1Hello() {}
class UScriptStruct* FMyStruct::StaticStruct()//实现了静态获取UScriptStruct*
{
	extern HELLO_API class UPackage* Z_Construct_UPackage__Script_Hello();
	static class UScriptStruct* Singleton = NULL;
	if (!Singleton)
	{
		extern HELLO_API class UScriptStruct* Z_Construct_UScriptStruct_FMyStruct();
		extern HELLO_API uint32 Get_Z_Construct_UScriptStruct_FMyStruct_CRC();
		Singleton = GetStaticStruct(Z_Construct_UScriptStruct_FMyStruct, Z_Construct_UPackage__Script_Hello(), TEXT("MyStruct"), sizeof(FMyStruct), Get_Z_Construct_UScriptStruct_FMyStruct_CRC());
	}
	return Singleton;
}
static FCompiledInDeferStruct Z_CompiledInDeferStruct_UScriptStruct_FMyStruct(FMyStruct::StaticStruct, TEXT("/Script/Hello"), TEXT("MyStruct"), false, nullptr, nullptr);  //延迟注册
static struct FScriptStruct_Hello_StaticRegisterNativesFMyStruct
{
	FScriptStruct_Hello_StaticRegisterNativesFMyStruct()
	{
		UScriptStruct::DeferCppStructOps(FName(TEXT("MyStruct")),new UScriptStruct::TCppStructOps<FMyStruct>);
	}
} ScriptStruct_Hello_StaticRegisterNativesFMyStruct;    //static注册
#if USE_COMPILED_IN_NATIVES
	HELLO_API class UScriptStruct* Z_Construct_UScriptStruct_FMyStruct();
	HELLO_API class UPackage* Z_Construct_UPackage__Script_Hello();
	UScriptStruct* Z_Construct_UScriptStruct_FMyStruct()    //构造关联的UScriptStruct*
	{
		UPackage* Outer = Z_Construct_UPackage__Script_Hello();
		extern uint32 Get_Z_Construct_UScriptStruct_FMyStruct_CRC();
		static UScriptStruct* ReturnStruct = FindExistingStructIfHotReloadOrDynamic(Outer, TEXT("MyStruct"), sizeof(FMyStruct), Get_Z_Construct_UScriptStruct_FMyStruct_CRC(), false);
		if (!ReturnStruct)
		{
			ReturnStruct = new(EC_InternalUseOnlyConstructor, Outer, TEXT("MyStruct"), RF_Public|RF_Transient|RF_MarkAsNative) UScriptStruct(FObjectInitializer(), NULL, new UScriptStruct::TCppStructOps<FMyStruct>, EStructFlags(0x00000201));//直接创建UScriptStruct对象
			UProperty* NewProp_Score = new(EC_InternalUseOnlyConstructor, ReturnStruct, TEXT("Score"), RF_Public|RF_Transient|RF_MarkAsNative) UFloatProperty(CPP_PROPERTY_BASE(Score, FMyStruct), 0x0010000000000004);//直接关联相应的Property信息
			ReturnStruct->StaticLink(); //链接
#if WITH_METADATA   //元数据
			UMetaData* MetaData = ReturnStruct->GetOutermost()->GetMetaData();
			MetaData->SetValue(ReturnStruct, TEXT("BlueprintType"), TEXT("true"));
			MetaData->SetValue(ReturnStruct, TEXT("ModuleRelativePath"), TEXT("MyStruct.h"));
			MetaData->SetValue(NewProp_Score, TEXT("Category"), TEXT("MyStruct"));
			MetaData->SetValue(NewProp_Score, TEXT("ModuleRelativePath"), TEXT("MyStruct.h"));
#endif
		}
		return ReturnStruct;
	}
	uint32 Get_Z_Construct_UScriptStruct_FMyStruct_CRC() { return 2914362188U; }
	UPackage* Z_Construct_UPackage__Script_Hello()
	{
		...略
	}
#endif

PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

同理，会发现FMyStruct::StaticStruct内部也不会比Z_Construct_UScriptStruct_FMyStruct更多的事情，GetStaticStruct的实现也只是简单的转发到Z_Construct_UScriptStruct_FMyStruct。值得一提的是定义的ScriptStruct_Hello_StaticRegisterNativesFMyStruct，会在程序一启动就调用UScriptStruct::DeferCppStructOps向程序注册该结构的CPP信息（大小，[内存对齐](https://zhida.zhihu.com/search?content_id=2225851&content_type=Article&match_order=1&q=%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90&zhida_source=entity)等），和TClassCompiledInDefer<TClass>的作用相当。FMyStruct的展开也是一目了然，就不再赘述了。

  

## UINTERFACE的生成代码剖析

UE对Interface也有支持，如果说FStruct就是一个纯数据的POD容器，那么UInterface则是一个只能带方法的纯接口，比C++里的抽象类要根据的纯粹一些。当然这里谈的都只涉及到用UPROPERTY和UFUNCTION宏标记的那些，如果是纯C++的字段和函数，UE并不能管到那么宽。  
测试的MyInterface.h为：

  

```cpp
#pragma once
#include "UObject/NoExportTypes.h"
#include "MyInterface.generated.h"

UINTERFACE(BlueprintType)
class UMyInterface : public UInterface
{
	GENERATED_UINTERFACE_BODY()    
};

class IMyInterface
{
	GENERATED_IINTERFACE_BODY()
public:
	UFUNCTION(BlueprintImplementableEvent)
	void BPFunc() const;
};
```

GENERATED_UINTERFACE_BODY和GENERATED_IINTERFACE_BODY都可以替换为GENERATED_BODY以提供一个默认的UMyInterface(const FObjectInitializer& ObjectInitializer)构造函数实现。不过GENERATED_IINTERFACE_BODY替换过后的效果也一样，因为并不需要那么一个构造函数，所以用两个都可以。  
生成的MyInterface.generated.h如下：

  

```cpp
PRAGMA_DISABLE_DEPRECATION_WARNINGS
#ifdef HELLO_MyInterface_generated_h
#error "MyInterface.generated.h already included, missing '#pragma once' in MyInterface.h"
#endif
#define HELLO_MyInterface_generated_h

#define Hello_Source_Hello_MyInterface_h_8_RPC_WRAPPERS
#define Hello_Source_Hello_MyInterface_h_8_RPC_WRAPPERS_NO_PURE_DECLS
#define Hello_Source_Hello_MyInterface_h_8_EVENT_PARMS
extern HELLO_API  FName HELLO_BPFunc;   //函数的名称，在cpp中定义
#define Hello_Source_Hello_MyInterface_h_8_CALLBACK_WRAPPERS
#define Hello_Source_Hello_MyInterface_h_8_STANDARD_CONSTRUCTORS \
	/** Standard constructor, called after all reflected properties have been initialized */ \
	NO_API UMyInterface(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get()); \
	DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyInterface) \
	DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyInterface); \
DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyInterface); \
private: \
	/** Private move- and copy-constructors, should never be used */ \
	NO_API UMyInterface(UMyInterface&&); \
	NO_API UMyInterface(const UMyInterface&); \
public:


#define Hello_Source_Hello_MyInterface_h_8_ENHANCED_CONSTRUCTORS \
	/** Standard constructor, called after all reflected properties have been initialized */ \
	NO_API UMyInterface(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get()) : Super(ObjectInitializer) { }; \
private: \
	/** Private move- and copy-constructors, should never be used */ \
	NO_API UMyInterface(UMyInterface&&); \
	NO_API UMyInterface(const UMyInterface&); \
public: \
	DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyInterface); \
DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyInterface); \
	DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(UMyInterface)


#undef GENERATED_UINTERFACE_BODY_COMMON
#define GENERATED_UINTERFACE_BODY_COMMON() \
	private: \
	static void StaticRegisterNativesUMyInterface(); \  //注册
	friend HELLO_API class UClass* Z_Construct_UClass_UMyInterface(); \ //构造UClass*的方法
public: \
	DECLARE_CLASS(UMyInterface, UInterface, COMPILED_IN_FLAGS(CLASS_Abstract | CLASS_Interface), 0, TEXT("/Script/Hello"), NO_API) \
	DECLARE_SERIALIZER(UMyInterface) \
	enum {IsIntrinsic=COMPILED_IN_INTRINSIC};


#define Hello_Source_Hello_MyInterface_h_8_GENERATED_BODY_LEGACY \
		PRAGMA_DISABLE_DEPRECATION_WARNINGS \
	GENERATED_UINTERFACE_BODY_COMMON() \
	Hello_Source_Hello_MyInterface_h_8_STANDARD_CONSTRUCTORS \
	PRAGMA_ENABLE_DEPRECATION_WARNINGS


#define Hello_Source_Hello_MyInterface_h_8_GENERATED_BODY \
	PRAGMA_DISABLE_DEPRECATION_WARNINGS \
	GENERATED_UINTERFACE_BODY_COMMON() \
	Hello_Source_Hello_MyInterface_h_8_ENHANCED_CONSTRUCTORS \
private: \
	PRAGMA_ENABLE_DEPRECATION_WARNINGS


#define Hello_Source_Hello_MyInterface_h_8_INCLASS_IINTERFACE_NO_PURE_DECLS \
protected: \
	virtual ~IMyInterface() {} \
public: \
	typedef UMyInterface UClassType; \
	static void Execute_BPFunc(const UObject* O); \
	virtual UObject* _getUObject() const = 0;


#define Hello_Source_Hello_MyInterface_h_8_INCLASS_IINTERFACE \
protected: \
	virtual ~IMyInterface() {} \
public: \
	typedef UMyInterface UClassType; \
	static void Execute_BPFunc(const UObject* O); \
	virtual UObject* _getUObject() const = 0;


#define Hello_Source_Hello_MyInterface_h_5_PROLOG \
	Hello_Source_Hello_MyInterface_h_8_EVENT_PARMS


#define Hello_Source_Hello_MyInterface_h_13_GENERATED_BODY_LEGACY \
PRAGMA_DISABLE_DEPRECATION_WARNINGS \
public: \
	Hello_Source_Hello_MyInterface_h_8_RPC_WRAPPERS \
	Hello_Source_Hello_MyInterface_h_8_CALLBACK_WRAPPERS \
	Hello_Source_Hello_MyInterface_h_8_INCLASS_IINTERFACE \
public: \
PRAGMA_ENABLE_DEPRECATION_WARNINGS


#define Hello_Source_Hello_MyInterface_h_13_GENERATED_BODY \
PRAGMA_DISABLE_DEPRECATION_WARNINGS \
public: \
	Hello_Source_Hello_MyInterface_h_8_RPC_WRAPPERS_NO_PURE_DECLS \
	Hello_Source_Hello_MyInterface_h_8_CALLBACK_WRAPPERS \
	Hello_Source_Hello_MyInterface_h_8_INCLASS_IINTERFACE_NO_PURE_DECLS \
private: \
PRAGMA_ENABLE_DEPRECATION_WARNINGS


#undef CURRENT_FILE_ID
#define CURRENT_FILE_ID Hello_Source_Hello_MyInterface_h


PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

因为接口的定义需要用到两个类，所以生成的信息稍微繁复了一些。不过使用的时候，我们的类只是继承于IMyInterface，UMyInerface只是作为一个接口类型的载体，用以区分和查找不同的接口。观察的时候，也请注意行号的定义。  
从底往上，最后两个是IMyInterface里的宏展开，细看之后，会发现_LEGACY和正常版本并没有差别。展开后是：

  

```cpp
class IMyInterface
{
protected: 
	virtual ~IMyInterface() {}  //禁止用接口指针释放对象
public: 
	typedef UMyInterface UClassType;    //设定UMyInterface为关联的类型
	static void Execute_BPFunc(const UObject* O);   //蓝图调用的辅助函数
	virtual UObject* _getUObject() const = 0;   //
public:
	UFUNCTION(BlueprintImplementableEvent)
	void BPFunc() const;
};
```

再往上是UMyInterface的生成，因为UMyInterface继承于UObject的原因，所以也是从属于Object系统的一份子，所以同样需要遵循构造函数的规则。UInterface本身其实也可以算是UClass的一种，所以生成的代码跟UClass中的生成都差不多，区别是用了COMPILED_IN_FLAGS(CLASS_Abstract | CLASS_Interface)的不同标记。有兴趣的读者可以自己展开看下。

生成的Hello.generated.cpp：

  

```cpp
#include "Hello.h"
#include "GeneratedCppIncludes.h"
#include "Hello.generated.dep.h"
PRAGMA_DISABLE_DEPRECATION_WARNINGS
void EmptyLinkFunctionForGeneratedCode1Hello() {}
FName HELLO_BPFunc = FName(TEXT("BPFunc")); //名字的定义
	void IMyInterface::BPFunc() const   //让编译通过，同时加上错误检测
	{
		check(0 && "Do not directly call Event functions in Interfaces. Call Execute_BPFunc instead.");
	}
	void UMyInterface::StaticRegisterNativesUMyInterface()
	{
	}
	IMPLEMENT_CLASS(UMyInterface, 4286549343);  //注册类
	void IMyInterface::Execute_BPFunc(const UObject* O) //蓝图调用方法的实现
	{
		check(O != NULL);
		check(O->GetClass()->ImplementsInterface(UMyInterface::StaticClass()));//检查是否实现了该接口
		UFunction* const Func = O->FindFunction(HELLO_BPFunc);  //通过名字找到方法
		if (Func)
		{
			const_cast<UObject*>(O)->ProcessEvent(Func, NULL);  //在该对象上调用该方法
		}
	}
#if USE_COMPILED_IN_NATIVES
	HELLO_API class UFunction* Z_Construct_UFunction_UMyInterface_BPFunc();
	HELLO_API class UClass* Z_Construct_UClass_UMyInterface_NoRegister();
	HELLO_API class UClass* Z_Construct_UClass_UMyInterface();
	HELLO_API class UPackage* Z_Construct_UPackage__Script_Hello();
	UFunction* Z_Construct_UFunction_UMyInterface_BPFunc()//构造BPFunc的UFunction
	{
		UObject* Outer=Z_Construct_UClass_UMyInterface();   //得到接口UMyInterface*对象
		static UFunction* ReturnFunction = NULL;
		if (!ReturnFunction)
		{
			ReturnFunction = new(EC_InternalUseOnlyConstructor, Outer, TEXT("BPFunc"), RF_Public|RF_Transient|RF_MarkAsNative) UFunction(FObjectInitializer(), NULL, 0x48020800, 65535); //直接构造函数对象
			ReturnFunction->Bind(); //绑定到函数指针
			ReturnFunction->StaticLink();   //链接
#if WITH_METADATA   //元数据
			UMetaData* MetaData = ReturnFunction->GetOutermost()->GetMetaData();
			MetaData->SetValue(ReturnFunction, TEXT("ModuleRelativePath"), TEXT("MyInterface.h"));
#endif
		}
		return ReturnFunction;
	}
	UClass* Z_Construct_UClass_UMyInterface_NoRegister()
	{
		return UMyInterface::StaticClass();
	}
	UClass* Z_Construct_UClass_UMyInterface()
	{
		static UClass* OuterClass = NULL;
		if (!OuterClass)
		{
			UInterface::StaticClass();  //确保基类UInterface已经元数据构造完成
			Z_Construct_UPackage__Script_Hello();
			OuterClass = UMyInterface::StaticClass();
			if (!(OuterClass->ClassFlags & CLASS_Constructed))
			{
				UObjectForceRegistration(OuterClass);
				OuterClass->ClassFlags |= 0x20004081;//CLASS_Constructed|CLASS_Interface|CLASS_Native|CLASS_Abstract

				OuterClass->LinkChild(Z_Construct_UFunction_UMyInterface_BPFunc());//添加子字段

				OuterClass->AddFunctionToFunctionMapWithOverriddenName(Z_Construct_UFunction_UMyInterface_BPFunc(), "BPFunc"); // 1371259725 ,添加函数名字映射
				OuterClass->StaticLink();   //链接
#if WITH_METADATA   //元数据
				UMetaData* MetaData = OuterClass->GetOutermost()->GetMetaData();
				MetaData->SetValue(OuterClass, TEXT("BlueprintType"), TEXT("true"));
				MetaData->SetValue(OuterClass, TEXT("ModuleRelativePath"), TEXT("MyInterface.h"));
#endif
			}
		}
		check(OuterClass->GetClass());
		return OuterClass;
	}
	static FCompiledInDefer Z_CompiledInDefer_UClass_UMyInterface(Z_Construct_UClass_UMyInterface, &UMyInterface::StaticClass, TEXT("UMyInterface"), false, nullptr, nullptr, nullptr);    //延迟注册
	DEFINE_VTABLE_PTR_HELPER_CTOR(UMyInterface);
	UPackage* Z_Construct_UPackage__Script_Hello()
	{
		...略
	}
#endif

PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

基本和UClass中的结构差不多，只是多了一些函数定义的过程和把函数添加到类中的操作。

  

## UClass中的字段和函数生成代码剖析

在最开始的时候，我们用了一个最简单的UMyClass来阐述整体的结构。[行百里者半九十](https://zhida.zhihu.com/search?content_id=2225851&content_type=Article&match_order=1&q=%E8%A1%8C%E7%99%BE%E9%87%8C%E8%80%85%E5%8D%8A%E4%B9%9D%E5%8D%81&zhida_source=entity)，让我们一鼓作气，看看如果UMyClass里多了Property和Function之后又会起什么变化。  
测试的MyClass.h如下：

  

```cpp
#pragma once
#include "UObject/NoExportTypes.h"
#include "MyClass.generated.h"

UCLASS(BlueprintType)
class HELLO_API UMyClass : public UObject
{
	GENERATED_BODY()
public:
	UPROPERTY(BlueprintReadWrite)
	float Score;
public:
	UFUNCTION(BlueprintCallable, Category = "Hello")
	void CallableFunc();    //C++实现，蓝图调用

	UFUNCTION(BlueprintNativeEvent, Category = "Hello")
	void NativeFunc();  //C++实现默认版本，蓝图可重载实现

	UFUNCTION(BlueprintImplementableEvent, Category = "Hello")
	void ImplementableFunc();   //C++不实现，蓝图实现
};
```

增加了一个属性和三个不同方法来测试。  
其生成的MyClass.generated.h为(只包括改变部分)：

  

```cpp
#define Hello_Source_Hello_MyClass_h_8_RPC_WRAPPERS \
	virtual void NativeFunc_Implementation(); \ //默认实现的函数声明，我们可以自己实现
 \
	DECLARE_FUNCTION(execNativeFunc) \  //声明供蓝图调用的函数
	{ \
		P_FINISH; \
		P_NATIVE_BEGIN; \
		this->NativeFunc_Implementation(); \
		P_NATIVE_END; \
	} \
 \
	DECLARE_FUNCTION(execCallableFunc) \    //声明供蓝图调用的函数
	{ \
		P_FINISH; \
		P_NATIVE_BEGIN; \
		this->CallableFunc(); \
		P_NATIVE_END; \
	}


#define Hello_Source_Hello_MyClass_h_8_RPC_WRAPPERS_NO_PURE_DECLS \ //和上面重复，略

//声明函数名称
extern HELLO_API  FName HELLO_ImplementableFunc;
extern HELLO_API  FName HELLO_NativeFunc;
```

”因为CallableFunc是C++里实现的，所以这里并不需要再定义函数体。而另外两个函数其实是在蓝图里定义的，就需要专门生成exec前缀的函数供蓝图虚拟机调用。“

上句我原话写的有点含糊歧义，也有错误，导致评论区有一些疑问纠正。更正一下，要表达的意思是：

因为ImplementableFunc是C++里实现的，指的是UE会自动为你C++里生成ImplementableFunc的实现函数体，所以这里并不需要再定义ImplementableFunc函数体。而另外两个函数NativeFunc和CallableFunc是在蓝图里定义的，指的是这两个函数的调用需要暴露到蓝图里，因此就需要专门生成exec前缀的函数供蓝图虚拟机调用。

  
我们展开execCallableFunc后为：

  

```cpp
void execCallableFunc( FFrame& Stack, void*const Z_Param__Result )  //蓝图虚拟机的使用的函数接口
{
    Stack.Code += !!Stack.Code; /* increment the code ptr unless it is null */
    { 
        FBlueprintEventTimer::FScopedNativeTimer ScopedNativeCallTimer;     //蓝图的计时统计
    	this->CallableFunc(); //调用我们自己的实现
    }
}
```

目前还是非常简单的，当然根据函数签名的不同会加上不同的参数传递，但是大体结构就是如此。以上的函数都是定义在UMyClass类内部的。  
再来看Hello.generated.cpp里的变化（只包括改变部分）：

  

```cpp
//函数名字定义
FName HELLO_ImplementableFunc = FName(TEXT("ImplementableFunc"));
FName HELLO_NativeFunc = FName(TEXT("NativeFunc"));
	void UMyClass::ImplementableFunc()  //C++端的实现
	{
		ProcessEvent(FindFunctionChecked(HELLO_ImplementableFunc),NULL);
	}
	void UMyClass::NativeFunc() //C++端的实现
	{
		ProcessEvent(FindFunctionChecked(HELLO_NativeFunc),NULL);
	}
	void UMyClass::StaticRegisterNativesUMyClass()  //注册函数名字和函数指针映射
	{
		FNativeFunctionRegistrar::RegisterFunction(UMyClass::StaticClass(), "CallableFunc",(Native)&UMyClass::execCallableFunc);
		FNativeFunctionRegistrar::RegisterFunction(UMyClass::StaticClass(), "NativeFunc",(Native)&UMyClass::execNativeFunc);
	}
//...略去中间相同部分
//构造3个函数的UFunction*对象，结构一样，只是EFunctionFlags不一样
UFunction* Z_Construct_UFunction_UMyClass_CallableFunc()
	{
		UObject* Outer=Z_Construct_UClass_UMyClass();
		static UFunction* ReturnFunction = NULL;
		if (!ReturnFunction)
		{
			ReturnFunction = new(EC_InternalUseOnlyConstructor, Outer, TEXT("CallableFunc"), RF_Public|RF_Transient|RF_MarkAsNative) UFunction(FObjectInitializer(), NULL, 0x04020401, 65535); //FUNC_BlueprintCallable|FUNC_Public|FUNC_Native|FUNC_Final
			ReturnFunction->Bind();
			ReturnFunction->StaticLink();
#if WITH_METADATA
			UMetaData* MetaData = ReturnFunction->GetOutermost()->GetMetaData();
			MetaData->SetValue(ReturnFunction, TEXT("Category"), TEXT("Hello"));
			MetaData->SetValue(ReturnFunction, TEXT("ModuleRelativePath"), TEXT("MyClass.h"));
#endif
		}
		return ReturnFunction;
	}
	UFunction* Z_Construct_UFunction_UMyClass_ImplementableFunc()
	{
		UObject* Outer=Z_Construct_UClass_UMyClass();
		static UFunction* ReturnFunction = NULL;
		if (!ReturnFunction)
		{
			ReturnFunction = new(EC_InternalUseOnlyConstructor, Outer, TEXT("ImplementableFunc"), RF_Public|RF_Transient|RF_MarkAsNative) UFunction(FObjectInitializer(), NULL, 0x08020800, 65535); //FUNC_BlueprintEvent|FUNC_Public|FUNC_Event
			ReturnFunction->Bind();
			ReturnFunction->StaticLink();
#if WITH_METADATA
			UMetaData* MetaData = ReturnFunction->GetOutermost()->GetMetaData();
			MetaData->SetValue(ReturnFunction, TEXT("Category"), TEXT("Hello"));
			MetaData->SetValue(ReturnFunction, TEXT("ModuleRelativePath"), TEXT("MyClass.h"));
#endif
		}
		return ReturnFunction;
	}
	UFunction* Z_Construct_UFunction_UMyClass_NativeFunc()
	{
		UObject* Outer=Z_Construct_UClass_UMyClass();
		static UFunction* ReturnFunction = NULL;
		if (!ReturnFunction)
		{
			ReturnFunction = new(EC_InternalUseOnlyConstructor, Outer, TEXT("NativeFunc"), RF_Public|RF_Transient|RF_MarkAsNative) UFunction(FObjectInitializer(), NULL, 0x08020C00, 65535);//FUNC_BlueprintEvent|FUNC_Public|FUNC_Event|FUNC_Native
			ReturnFunction->Bind();
			ReturnFunction->StaticLink();
#if WITH_METADATA
			UMetaData* MetaData = ReturnFunction->GetOutermost()->GetMetaData();
			MetaData->SetValue(ReturnFunction, TEXT("Category"), TEXT("Hello"));
			MetaData->SetValue(ReturnFunction, TEXT("ModuleRelativePath"), TEXT("MyClass.h"));
#endif
		}
		return ReturnFunction;
	}
//...略去中间相同部分
UClass* Z_Construct_UClass_UMyClass()
	{
		static UClass* OuterClass = NULL;
		if (!OuterClass)
		{
			Z_Construct_UClass_UObject();
			Z_Construct_UPackage__Script_Hello();
			OuterClass = UMyClass::StaticClass();
			if (!(OuterClass->ClassFlags & CLASS_Constructed))
			{
				UObjectForceRegistration(OuterClass);
				OuterClass->ClassFlags |= 0x20100080;
                //添加子字段
				OuterClass->LinkChild(Z_Construct_UFunction_UMyClass_CallableFunc());
				OuterClass->LinkChild(Z_Construct_UFunction_UMyClass_ImplementableFunc());
				OuterClass->LinkChild(Z_Construct_UFunction_UMyClass_NativeFunc());

PRAGMA_DISABLE_DEPRECATION_WARNINGS
				UProperty* NewProp_Score = new(EC_InternalUseOnlyConstructor, OuterClass, TEXT("Score"), RF_Public|RF_Transient|RF_MarkAsNative) UFloatProperty(CPP_PROPERTY_BASE(Score, UMyClass), 0x0010000000000004);//添加属性
PRAGMA_ENABLE_DEPRECATION_WARNINGS
                //添加函数名字映射
				OuterClass->AddFunctionToFunctionMapWithOverriddenName(Z_Construct_UFunction_UMyClass_CallableFunc(), "CallableFunc"); // 774395847
				OuterClass->AddFunctionToFunctionMapWithOverriddenName(Z_Construct_UFunction_UMyClass_ImplementableFunc(), "ImplementableFunc"); // 615168156
				OuterClass->AddFunctionToFunctionMapWithOverriddenName(Z_Construct_UFunction_UMyClass_NativeFunc(), "NativeFunc"); // 3085959641
				OuterClass->StaticLink();
#if WITH_METADATA   //元数据
				UMetaData* MetaData = OuterClass->GetOutermost()->GetMetaData();
				MetaData->SetValue(OuterClass, TEXT("BlueprintType"), TEXT("true"));
				MetaData->SetValue(OuterClass, TEXT("IncludePath"), TEXT("MyClass.h"));
				MetaData->SetValue(OuterClass, TEXT("ModuleRelativePath"), TEXT("MyClass.h"));
				MetaData->SetValue(NewProp_Score, TEXT("Category"), TEXT("MyClass"));
				MetaData->SetValue(NewProp_Score, TEXT("ModuleRelativePath"), TEXT("MyClass.h"));
#endif
			}
		}
		check(OuterClass->GetClass());
		return OuterClass;
	}
```

可以看出，对于CallableFunc这种C++实现，蓝图只是调用的方法，生成的代码只是生成了相应的UFunction*对象。而对于NativeFunc和ImplementableFunc，我们不会在C++里写上它们的实现，因此为了编译通过，也为了可以从C++端直接调用，就需要在生成的代码的时候也同样生成一份默认实现。  
在之前的简单类生成代码中，StaticRegisterNativesUMyClass总是空的，在这里UHT为它加上了把函数注册进程序内存的操作。  
3个函数的UFunction*生成，虽然它们的调用方式大相径庭，但是生成的代码的方式却是结构一致的，区别的只是不同的EFunctionFlags值。因此可以推断出，更多的差异应该是在蓝图虚拟机的部分实现的，该部分知识以后介绍蓝图的时候再讨论。  
最后，把1个属性和3个方法添加进UClass中，齐活收工。