假设UHT已经分析得到了足够的类型元数据信息，下一步就是利用这个信息在程序内存中构建起前文的类型系统结构，这个过程我们称之为注册。同一般程序的构建流程需要经过预处理、编译、汇编、链接一样，UE为了在内存中模拟构建的过程，在概念上也需要以下几个阶段：生成，收集，注册，链接。
本篇章介绍生成环节
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

**第5行：**#include "UObject/NoExportTypes.h" 通过查看文件内容，发现这个文件在编译的时候就是Include了其他一些更基础的头文件，比如#include "Math/Vector.h"，因此你才能在MyClass里不用include就引用这些类。当然，还有一些内容是专门供UHT使用来生成蓝图类型的，现在暂时不需要管。

**第6行：**#include "MyClass.generated.h"，就是为了引用生成的头文件。这里请注意的是，该文件include位置在类声明的前面，之后谈到宏处理的时候会用到该信息。

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