## 引言

在之前的[《InsideUE4》UObject（四）类型系统代码生成](https://zhuanlan.zhihu.com/p/25098685)和[《InsideUE4》UObject（五）类型系统收集](https://zhuanlan.zhihu.com/p/26019216)章节里，我们介绍了UE4是如何根据我们的代码和元标记生成反射代码，并在Main函数调用之前，利用静态变量的初始化来收集类型的元数据信息。经过了我这么长时间的拖更……也经过了Epic这么长时间的版本更替，把UE从4.15.1进化到了4.18.3，自然的，CoreUObject模块也进行了一些改进。本文就先补上一个关于代码生成的改进：在UE4.17（20170722）的时候进行的UObjectGlobals.h.cpp重构，优化了代码生成的内容和组织形式。

## 旧版本代码生成

首先来看一下之前的版本的代码元数据生成：

## [UEnum](https://zhida.zhihu.com/search?content_id=5813180&content_type=Article&match_order=1&q=UEnum&zhida_source=entity)的生成：

```text
//测试代码
#pragma once
#include "UObject/NoExportTypes.h"
#include "MyEnum.generated.h"
UENUM(BlueprintType)
enum class EMyEnum : uint8
{
    MY_Dance    UMETA(DisplayName = "Dance"),
    MY_Rain     UMETA(DisplayName = "Rain"),
    MY_Song     UMETA(DisplayName = "Song")
};

//生成代码节选(Hello.genrated.cpp)：
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
```

## UStruct的生成：

```text
//测试代码：
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
//生成代码节选(Hello.genrated.cpp)：
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
```

## UClass的生成：

```text
//测试代码：
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
//生成代码节选(Hello.genrated.cpp)：
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
```

可以见到，以往的方式在生成的代码里有很多的“套路化”的SetValue、Add段落，都是用来添加字段属性、函数和元数据的信息。虽然这些代码也是UHT程序化生成的，不用人手工操作，看起来也只能说是略有瑕疵，但要是从精益求精的角度上来说，缺点有：

1. 本着DRY(Don't Repeat Yourself)原则，这些模式化的代码在每一个反射文件里也都会重复N次，增大了代码的体积。
2. 代码文件的膨胀，自然会增加编译的时间消耗。
3. 即使是程序生成的代码，有时也难免要阅读Debug，大量的模式代码噪音显然降低了关键代码的可读性和课调试性。
4. UHT的编写维护，更多的代码量生成，自然会带来UHT工具代码的编写量增长，增大了编写维护的成本；代码越多，Bug越多；UHT要输出更多的代码，自然效率会降低，从而导致总编译时间的消耗增长。

## UE4CodeGen_Private

改善方式是显然易得的，同一件事不要做两遍。既然到处都是这些胶水代码，那就把这些代码封装成函数；既然到处都散布着这些元数据信息数据，那就把这些数据封装成结构作为函数的参数。  
所以，UE在4.17的时候，在UObjectGlobals.h.cpp里增加了一个UE4CodeGen_Private的命名空间，里面添加了一些生成函数：

```text
//UObjectGlobals.h
namespace UE4CodeGen_Private
{
    COREUOBJECT_API void ConstructUFunction(UFunction*& OutFunction, const FFunctionParams& Params);
    COREUOBJECT_API void ConstructUEnum(UEnum*& OutEnum, const FEnumParams& Params);
    COREUOBJECT_API void ConstructUScriptStruct(UScriptStruct*& OutStruct, const FStructParams& Params);
    COREUOBJECT_API void ConstructUPackage(UPackage*& OutPackage, const FPackageParams& Params);
    COREUOBJECT_API void ConstructUClass(UClass*& OutClass, const FClassParams& Params);
}

//UObjectGlobals.cpp
namespace UE4CodeGen_Private
{
    void ConstructUProperty(UObject* Outer, const FPropertyParamsBase* const*& PropertyArray, int32& NumProperties);
    void AddMetaData(UObject* Object, const FMetaDataPairParam* MetaDataArray, int32 NumMetaData);
}
```

函数的名字含义显而易见，都是用来构造一些元数据结构：UEnum、UFunction、UProperty、UScriptStruct、UClass、UPackage和添加一些元数据（这些结构后续会详解）。第一个参数都是指针的引用，所以是用来向外构造一个对象用指针返回的；关键的是在第二个参数：都是一个个XXXParams参数，用来传进去信息的。  
所以我们继续查看这些参数信息：

## UEnum的Params和生成：

```text
//UObjectGlobals.h
namespace UE4CodeGen_Private
{
#if WITH_METADATA   //只有在编辑器模式下，才保留元数据信息
    struct FMetaDataPairParam   //元数据对
    {
        //例：MetaData->SetValue(ReturnEnum, TEXT("MY_Song.DisplayName"), TEXT("Song"));
        const char* NameUTF8;   //元数据的键值对信息
        const char* ValueUTF8;
    };
#endif
    struct FEnumeratorParam     //枚举项
    {
        const char*               NameUTF8; //枚举项的名字
        int64                     Value;    //枚举项的值
#if WITH_METADATA
        const FMetaDataPairParam* MetaDataArray;    //一个枚举项依然可以包含多个元数据键值对
        int32                     NumMetaData;
#endif
    };

    struct FEnumParams  //枚举参数
    {
        UObject*                  (*OuterFunc)();   //获取Outer对象的函数指针回调，用于获取所属于的Package
        EDynamicType                DynamicType;    //是否动态，一般是非动态的
        const char*                 NameUTF8;   //枚举的名字
        EObjectFlags                ObjectFlags;    //UEnum对象的标志
        FText                     (*DisplayNameFunc)(int32);    //获取自定义显示名字的回调，一般是nullptr，就是默认规则生成的名字
        uint8                       CppForm; 
        /*CppForm指定这个枚举是怎么定义的，用来在之后做更细的处理。
        enum class ECppForm
        {
            Regular,    //常规的enum MyEnum{}这样定义
            Namespaced, //MyEnum之外套一层namespace的定义
            EnumClass   //enum class定义的
        };
        */
        const char*                 CppTypeUTF8;    //C++里的类型名字，一般是等同于NameUTF8的，但有时定义名字和反射的名字可以不一样
        const FEnumeratorParam*     EnumeratorParams;   //枚举项数组
        int32                       NumEnumerators;
#if WITH_METADATA
        const FMetaDataPairParam*   MetaDataArray;  //元数据数组
        int32                       NumMetaData;
#endif
    };
}

//MyEnum.gen.cpp生成代码：
static const UE4CodeGen_Private::FEnumeratorParam Enumerators[] = { //所有的枚举项
            { "MyEnum::MY_Dance", (int64)MyEnum::MY_Dance },
            { "MyEnum::MY_Rain", (int64)MyEnum::MY_Rain },
            { "MyEnum::MY_Song", (int64)MyEnum::MY_Song },
        };
#if WITH_METADATA
static const UE4CodeGen_Private::FMetaDataPairParam Enum_MetaDataParams[] = {   //枚举的元数据
    { "BlueprintType", "true" },
    { "IsBlueprintBase", "true" },
    { "ModuleRelativePath", "MyEnum.h" },
    { "MY_Dance.DisplayName", "Dance" },
    { "MY_Rain.DisplayName", "Rain" },
    { "MY_Song.DisplayName", "Song" },
};
#endif
static const UE4CodeGen_Private::FEnumParams EnumParams = { //枚举的元数据参数信息
    (UObject*(*)())Z_Construct_UPackage__Script_Hello,
    UE4CodeGen_Private::EDynamicType::NotDynamic,
    "MyEnum",
    RF_Public|RF_Transient|RF_MarkAsNative,
    nullptr,
    (uint8)UEnum::ECppForm::EnumClass,
    "MyEnum",
    Enumerators, //枚举项数组
    ARRAY_COUNT(Enumerators),  
    METADATA_PARAMS(Enum_MetaDataParams, ARRAY_COUNT(Enum_MetaDataParams))//枚举元数据数组
};
UE4CodeGen_Private::ConstructUEnum(ReturnEnum, EnumParams); //利用枚举参数构造UEnum*对象到ReturnEnum
```

先挑最软的椰子开始捏，枚举的构造比较简单，就只是包含枚举项（字符串-整形)，所以只要依次添加进去就可以。元数据对指的就是那些UMETA等宏标记里面那些的内容，可以在很多地方上使用来添加额外的信息。

## UStruct的Params和生成：

因为UStruct里只能包含属性，所以我们在这里着重关注属性信息是怎么生成的。

```text
//UObjectGlobals.h
//PS:为了阅读方便，与源码有一定的代码位置微调，但不影响功能正确性
namespace UE4CodeGen_Private
{
    enum class EPropertyClass   //属性的类型
    {
        Byte,
        Int8,
        Int16,
        Int,
        Int64,
        UInt16,
        UInt32,
        UInt64,
        UnsizedInt,
        UnsizedUInt,
        Float,
        Double,
        Bool,
        SoftClass,
        WeakObject,
        LazyObject,
        SoftObject,
        Class,
        Object,
        Interface,
        Name,
        Str,
        Array,
        Map,
        Set,
        Struct,
        Delegate,
        MulticastDelegate,
        Text,
        Enum,
    };

    // This is not a base class but is just a common initial sequence of all of the F*PropertyParams types below.
    // We don't want to use actual inheritance because we want to construct aggregated compile-time tables of these things.
    struct FPropertyParamsBase  //属性参数基类
    {
        EPropertyClass Type;    //属性的类型
        const char*    NameUTF8;     //属性的名字
        EObjectFlags   ObjectFlags;  //属性生成的UProperty对象标志，标识这个UProperty对象的特征，RF_XXX那些宏
        uint64         PropertyFlags;    //属性生成的UProperty属性标志，标识这个属性的特征，CPF_XXX那些宏
        int32          ArrayDim;        //属性有可能是个数组，数组的长度，默认是1
        const char*    RepNotifyFuncUTF8;   //属性的网络复制通知函数名字
    };

    struct FPropertyParamsBaseWithOffset // : FPropertyParamsBase
    {
        EPropertyClass Type;
        const char*    NameUTF8;
        EObjectFlags   ObjectFlags;
        uint64         PropertyFlags;
        int32          ArrayDim;
        const char*    RepNotifyFuncUTF8;
        int32          Offset;  //在结构或类中的内存偏移，可以理解为成员变量指针（成员变量指针其实本质上就是从对象内存起始位置的偏移）
    };
    //通用的属性参数
    struct FGenericPropertyParams // : FPropertyParamsBaseWithOffset
    {
        EPropertyClass   Type;
        const char*      NameUTF8;
        EObjectFlags     ObjectFlags;
        uint64           PropertyFlags;
        int32            ArrayDim;
        const char*      RepNotifyFuncUTF8;
        int32            Offset;
#if WITH_METADATA
        const FMetaDataPairParam*           MetaDataArray;
        int32                               NumMetaData;
#endif
    };

    //一些普通常用的数值类型就通过这个类型定义别名了
    // These property types don't add new any construction parameters to their base property
    typedef FGenericPropertyParams FInt8PropertyParams;
    typedef FGenericPropertyParams FInt16PropertyParams;

    //枚举类型属性参数
    struct FBytePropertyParams // : FPropertyParamsBaseWithOffset
    {
        EPropertyClass   Type;
        const char*      NameUTF8;
        EObjectFlags     ObjectFlags;
        uint64           PropertyFlags;
        int32            ArrayDim;
        const char*      RepNotifyFuncUTF8;
        int32            Offset;
        UEnum*         (*EnumFunc)();   //定义的枚举对象回调
#if WITH_METADATA
        const FMetaDataPairParam*           MetaDataArray;
        int32                               NumMetaData;
#endif
    };
    //...省略一些定义，可自行去UObjectGlobals.h查看
    //对象引用类型属性参数
    struct FObjectPropertyParams // : FPropertyParamsBaseWithOffset
    {
        EPropertyClass   Type;
        const char*      NameUTF8;
        EObjectFlags     ObjectFlags;
        uint64           PropertyFlags;
        int32            ArrayDim;
        const char*      RepNotifyFuncUTF8;
        int32            Offset;
        UClass*        (*ClassFunc)();  //用于获取该属性定义类型的函数指针回调
#if WITH_METADATA
        const FMetaDataPairParam*           MetaDataArray;
        int32                               NumMetaData;
#endif
    };

    struct FStructParams    //结构参数
    {
        UObject*                          (*OuterFunc)();   //所属于的Package
        UScriptStruct*                    (*SuperFunc)();   //该结构的基类，没有的话为nullptr
        void*                             (*StructOpsFunc)(); // really returns UScriptStruct::ICppStructOps*，结构的构造分配的辅助操作类
        const char*                         NameUTF8;   //结构名字
        EObjectFlags                        ObjectFlags;    //结构UScriptStruct*的对象特征
        uint32                              StructFlags; // EStructFlags该结构的本来特征
        SIZE_T                              SizeOf;     //结构的大小，就是sizeof(FMyStruct)，用以后续分配内存时候用
        SIZE_T                              AlignOf;//结构的内存对齐，就是alignof(FMyStruct)，用以后续分配内存时候用
        const FPropertyParamsBase* const*   PropertyArray;  //包含的属性数组
        int32                               NumProperties;
    #if WITH_METADATA
        const FMetaDataPairParam*           MetaDataArray;  //元数据数组
        int32                               NumMetaData;
    #endif
    };
}

//MyStruct.gen.cpp生成代码：
#if WITH_METADATA
static const UE4CodeGen_Private::FMetaDataPairParam Struct_MetaDataParams[] = { //结构的元数据
    { "BlueprintType", "true" },
    { "ModuleRelativePath", "MyStruct.h" },
};
#endif
auto NewStructOpsLambda = []() -> void* { return (UScriptStruct::ICppStructOps*)new UScriptStruct::TCppStructOps<FMyStruct>(); };   //一个获取操作类的回调
#if WITH_METADATA
//属性的元数据
static const UE4CodeGen_Private::FMetaDataPairParam NewProp_Score_MetaData[] = {
    { "Category", "MyStruct" },
    { "ModuleRelativePath", "MyStruct.h" },
};
#endif
static const UE4CodeGen_Private::FFloatPropertyParams NewProp_Score = { UE4CodeGen_Private::EPropertyClass::Float, "Score", RF_Public|RF_Transient|RF_MarkAsNative, 0x0010000000000004, 1, nullptr, STRUCT_OFFSET(FMyStruct, Score), METADATA_PARAMS(NewProp_Score_MetaData, ARRAY_COUNT(NewProp_Score_MetaData)) };//Score属性的信息
//属性的数组
static const UE4CodeGen_Private::FPropertyParamsBase* const PropPointers[] = {
    (const UE4CodeGen_Private::FPropertyParamsBase*)&NewProp_Score,
};
//结构的参数信息
static const UE4CodeGen_Private::FStructParams ReturnStructParams = {
    (UObject* (*)())Z_Construct_UPackage__Script_Hello,
    nullptr,
    &UE4CodeGen_Private::TNewCppStructOpsWrapper<decltype(NewStructOpsLambda)>::NewCppStructOps,
    "MyStruct",
    RF_Public|RF_Transient|RF_MarkAsNative,
    EStructFlags(0x00000201),
    sizeof(FMyStruct),
    alignof(FMyStruct),
    PropPointers, ARRAY_COUNT(PropPointers),
    METADATA_PARAMS(Struct_MetaDataParams, ARRAY_COUNT(Struct_MetaDataParams))
};
UE4CodeGen_Private::ConstructUScriptStruct(ReturnStruct, ReturnStructParams);//构造UScriptStruct*到ReturnStruct里去
```

代码比较简单，上下对照和看看注释就能大概明白。就是收集一个个属性的信息整合成数组，然后合并到结构参数里去，最后传给ConstructUScriptStruct来构造。

**思考：FPropertyParamsBaseWithOffset以及后续为何不继承于FPropertyParamsBase？**  
我们在FPropertyParamsBase和FPropertyParamsBaseWithOffset等后续的注释后面以及属性成员的相似性上来看，很容易就看到这些F*PropertyParams其实是用了继承语义的，那为何不直接继承而是费劲的再写一遍呢？  
虽然官方在FPropertyParamsBase上已经写了注释，但是有些朋友可能还是依然比较懵懂。其实这里涉及到一个C++的Aggregate类型的[aggregate initialization](https://link.zhihu.com/?target=http%3A//en.cppreference.com/w/cpp/language/aggregate_initialization)规则。具体的C++语法规则请自行去补充学习。简单来说，**一个Aggregate是一个数组或者一个没有用户声明构造函数，没有私有或保护类型的非静态数据成员，没有父类和虚函数的类型。** Aggregatel类型就可以用形如 _T object = {arg1, arg2, ...}_ 的初始化列表来初始化。我们在上文中见到的：

```text
static const UE4CodeGen_Private::FFloatPropertyParams NewProp_Score = { UE4CodeGen_Private::EPropertyClass::Float, "Score", RF_Public|RF_Transient|RF_MarkAsNative, 0x0010000000000004, 1, nullptr, STRUCT_OFFSET(FMyStruct, Score), METADATA_PARAMS(NewProp_Score_MetaData, ARRAY_COUNT(NewProp_Score_MetaData)) };
```

后面的={}就是初始化列表。这么写当然是为了简洁的目的，否则一个个参数的字段设置过去，那也太麻烦了。  
那如果用继承会怎么样呢？我们可以来做个测试：

```text
struct Point2
{
    float X;
    float Y;
};

struct Point3 :public Point2    //这不是个Aggregate类型，因为有父类
{
    float Z;
};

struct Point3_Aggregate //这是个Aggregate类型
{
    float X;
    float Y;
    float Z;
};
const static Point3 pos{ 1.f,2.f,3.f }; // error C2440:'initializing': cannot convert from 'initializer list' to 'Point3'
const static Point3_Aggregate pos2{ 1.f,2.f,3.f };
```

因此UE选择不继承，宁愿每个重复写一遍字段声明，就是为了可以简单用{}初始化列表来构造对象。但是我们也观察到，在PropPointers数组里，也依然把一个个元素都转为FPropertyParamsBase*。因为根据C/C++的对象内存模型，继承的时候，基类成员排在派生类成员之前的内存地址上。又因为F*PropertyParams是如此的POD，所以只要保证内存地址和属性成员顺序一致，就可以保证转为另一个结构指针后依然可以正确的使用。

虽然看起来这么解释的通，但还是感觉很麻烦，本来应该用继承的语义却偏偏为了初始化列表妥协了。对完美主义者来说还是不能忍，那么有没有一种既可以用继承又可以用初始化列表的解决方案呢？  
其实加上构造函数就可以了。不用Aggregate类型，放宽限制，改用POD类型（**POD类型就是没有非静态类型的non-POD类型 （或者这些类型的数组）和引用类型的数据成员，也没有用户定义的赋值操作符和析构函数的类型。**）。如：

```text
struct Point2
{
    float X;
    float Y;
    Point2(float x, float y) :X(x), Y(y) {} //构造函数
};

struct Point3_POD :public Point2
{
    float Z;
    Point3_POD(float x, float y, float z) :Point2(x, y), Z(z) {}//构造函数
};

struct Point3_Aggregate
{
    float X;
    float Y;
    float Z;
};
const static Point3_POD pos{ 1.f,2.f,3.f };     //works happy ^_^
const static Point3_Aggregate pos2{ 1.f,2.f,3.f };  //works happy ^_^
```

所以只要把F*PropertyParams加上构造函数就可以了。至于为啥UE不这么做？问Epic的人去，摊手~

## UFunction和UClass的Params和生成：

为了测试UClass里的函数输入输出参数，所以增加一个AddHP函数。

```text
//测试文件：
UCLASS()
class HELLO_API UMyClass :public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintReadWrite)
    float Score;
public:
    UFUNCTION(BlueprintCallable, Category = "Hello")
    float AddHP(float HP);

    UFUNCTION(BlueprintCallable, Category = "Hello")
    void CallableFunc();    //C++实现，蓝图调用

    UFUNCTION(BlueprintNativeEvent, Category = "Hello")
    void NativeFunc();  //C++实现默认版本，蓝图可重载实现

    UFUNCTION(BlueprintImplementableEvent, Category = "Hello")
    void ImplementableFunc();   //C++不实现，蓝图实现
};

//Class.h
//类里的函数链接信息，一个函数名字对应一个UFunction对象
struct FClassFunctionLinkInfo 
{
    UFunction* (*CreateFuncPtr)();  //获得UFunction对象的函数指针回调
    const char* FuncNameUTF8;       //函数的名字
};
//类在Cpp里的类型信息，用一个结构是为了将来也许还会添加别的字段
struct FCppClassTypeInfoStatic
{
    bool bIsAbstract;   //是否抽象类
};

//UObjectGlobals.h
namespace UE4CodeGen_Private
{
    //函数参数
    struct FFunctionParams
    {
        UObject*                          (*OuterFunc)();   //所属于的外部对象，一般是外部的UClass*对象
        const char*                         NameUTF8;   //函数的名字
        EObjectFlags                        ObjectFlags;    //UFunction对象的特征
        UFunction*                        (*SuperFunc)();   //UFunction的基类，一般为nullptr
        EFunctionFlags                      FunctionFlags;  //函数本身的特征
        SIZE_T                              StructureSize;  //函数的参数返回值包结构的大小
        const FPropertyParamsBase* const*   PropertyArray;  //函数的参数和返回值字段数组
        int32                               NumProperties;  //函数的参数和返回值字段数组大小
        uint16                              RPCId;          //网络间的RPC Id
        uint16                              RPCResponseId;  //网络间的RPC Response Id
    #if WITH_METADATA
        const FMetaDataPairParam*           MetaDataArray;  //元数据数组
        int32                               NumMetaData;
    #endif
    };

    //实现的接口参数，篇幅所限，接口的内容可以自行分析
    struct FImplementedInterfaceParams
    {
        UClass* (*ClassFunc)();     //外部所属于的UInterface对象
        int32     Offset;           //在UMyClass里的实现的IMyInterface的虚函数表地址偏移
        bool      bImplementedByK2; //是否在蓝图中实现
    };

    //类参数
    struct FClassParams
    {
        UClass*                                   (*ClassNoRegisterFunc)(); //获得UClass*对象的函数指针
        UObject*                           (*const *DependencySingletonFuncArray)();    //获取依赖对象的函数指针数组，一般是需要前提构造的基类，模块UPackage对象
        int32                                       NumDependencySingletons;
        uint32                                      ClassFlags; // EClassFlags，类特征
        const FClassFunctionLinkInfo*               FunctionLinkArray;  //链接的函数数组
        int32                                       NumFunctions;
        const FPropertyParamsBase* const*           PropertyArray;  //类里定义的成员变量数组
        int32                                       NumProperties;
        const char*                                 ClassConfigNameUTF8;    //配置文件名字，有些类可以从配置文件从加载数据
        const FCppClassTypeInfoStatic*              CppClassInfo;   //Cpp里定义的信息
        const FImplementedInterfaceParams*          ImplementedInterfaceArray;  //实现的接口信息数组
        int32                                       NumImplementedInterfaces;
    #if WITH_METADATA           //类的元数据
        const FMetaDataPairParam*                   MetaDataArray;
        int32                                       NumMetaData;
    #endif
    };
}

//MyClass.gen.cpp
//构造AddHp函数的UFunction对象
UFunction* Z_Construct_UFunction_UMyClass_AddHP()
{
    struct MyClass_eventAddHP_Parms     //函数的参数和返回值包
    {
        float HP;
        float ReturnValue;
    };
    static UFunction* ReturnFunction = nullptr;
    if (!ReturnFunction)
    {
        //定义两个属性用来传递信息
        static const UE4CodeGen_Private::FFloatPropertyParams NewProp_ReturnValue = { UE4CodeGen_Private::EPropertyClass::Float, "ReturnValue", RF_Public|RF_Transient|RF_MarkAsNative, 0x0010000000000580, 1, nullptr, STRUCT_OFFSET(MyClass_eventAddHP_Parms, ReturnValue), METADATA_PARAMS(nullptr, 0) };

        static const UE4CodeGen_Private::FFloatPropertyParams NewProp_HP = { UE4CodeGen_Private::EPropertyClass::Float, "HP", RF_Public|RF_Transient|RF_MarkAsNative, 0x0010000000000080, 1, nullptr, STRUCT_OFFSET(MyClass_eventAddHP_Parms, HP), METADATA_PARAMS(nullptr, 0) };

        static const UE4CodeGen_Private::FPropertyParamsBase* const PropPointers[] = {
            (const UE4CodeGen_Private::FPropertyParamsBase*)&NewProp_ReturnValue,
            (const UE4CodeGen_Private::FPropertyParamsBase*)&NewProp_HP,
        };

#if WITH_METADATA
        static const UE4CodeGen_Private::FMetaDataPairParam Function_MetaDataParams[] = {
            { "Category", "Hello" },
            { "ModuleRelativePath", "MyClass.h" },
        };
#endif
        static const UE4CodeGen_Private::FFunctionParams FuncParams = { (UObject*(*)())Z_Construct_UClass_UMyClass, "AddHP", RF_Public|RF_Transient|RF_MarkAsNative, nullptr, (EFunctionFlags)0x04020401, sizeof(MyClass_eventAddHP_Parms), PropPointers, ARRAY_COUNT(PropPointers), 0, 0, METADATA_PARAMS(Function_MetaDataParams, ARRAY_COUNT(Function_MetaDataParams)) };

        UE4CodeGen_Private::ConstructUFunction(ReturnFunction, FuncParams); //构造函数
    }
    return ReturnFunction;
}
//...构造其他的函数
//该类依赖的对象列表，用函数指针来获取。
static UObject* (*const DependentSingletons[])() = {
                (UObject* (*)())Z_Construct_UClass_UObject,
                (UObject* (*)())Z_Construct_UPackage__Script_Hello,
            };
//函数链接信息
static const FClassFunctionLinkInfo FuncInfo[] = {
    { &Z_Construct_UFunction_UMyClass_CallableFunc, "CallableFunc" }, // 1841300010
    { &Z_Construct_UFunction_UMyClass_ImplementableFunc, "ImplementableFunc" }, // 2010696670
    { &Z_Construct_UFunction_UMyClass_NativeFunc, "NativeFunc" }, // 2593520329
};
#if WITH_METADATA
static const UE4CodeGen_Private::FMetaDataPairParam Class_MetaDataParams[] = {
    { "IncludePath", "MyClass.h" },
    { "ModuleRelativePath", "MyClass.h" },
};
#endif
#if WITH_METADATA
static const UE4CodeGen_Private::FMetaDataPairParam NewProp_Score_MetaData[] = {
    { "Category", "MyClass" },
    { "ModuleRelativePath", "MyClass.h" },
};
#endif
static const UE4CodeGen_Private::FFloatPropertyParams NewProp_Score = { UE4CodeGen_Private::EPropertyClass::Float, "Score", RF_Public|RF_Transient|RF_MarkAsNative, 0x0010000000000004, 1, nullptr, STRUCT_OFFSET(UMyClass, Score), METADATA_PARAMS(NewProp_Score_MetaData, ARRAY_COUNT(NewProp_Score_MetaData)) };

static const UE4CodeGen_Private::FPropertyParamsBase* const PropPointers[] = {
    (const UE4CodeGen_Private::FPropertyParamsBase*)&NewProp_Score,
};
static const FCppClassTypeInfoStatic StaticCppClassTypeInfo = {
    TCppClassTypeTraits<UMyClass>::IsAbstract,
};
static const UE4CodeGen_Private::FClassParams ClassParams = {
    &UMyClass::StaticClass,
    DependentSingletons, ARRAY_COUNT(DependentSingletons),
    0x00100080u,
    FuncInfo, ARRAY_COUNT(FuncInfo),
    PropPointers, ARRAY_COUNT(PropPointers),
    nullptr,
    &StaticCppClassTypeInfo,
    nullptr, 0,
    METADATA_PARAMS(Class_MetaDataParams, ARRAY_COUNT(Class_MetaDataParams))
};
UE4CodeGen_Private::ConstructUClass(OuterClass, ClassParams);   //构造UClass对象
```

注意，对于一个函数来说，参数和返回值都可以算是函数内部定义的属性，只不过其有不同的特征和用途。类里包含属性和函数，而函数又包含属性。属性的构造和Struct里的规则一样，就不赘述了。不同的是，因为Class里可以包含Function，所以构造UClass之前必须先构造出所有的UFunction。所以整理下，其实上述的那些构造就是结构套结构，加上一些数组整合出来的信息集合而已。

**思考：为什么生成的代码里大量用了函数指针来返回对象？**  
如UClass* (*ClassNoRegisterFunc)()或UFunction* (*CreateFuncPtr)()都用函数指针来获取定义的UClass*对象和前提依赖的UFunction*对象。为什么不直接用个UClass*或UFunction*指针呢？  
答案很简单，因为构造顺序的不确定。  
在一个类型系统中，类型的依赖管理是项很麻烦但又非常重要的事，你必须保证当前类型的所有前置类型都已经定义完毕，才能开始本类型的构造。针对此问题，当然你可以小心翼翼的理清定义顺序，确保所有的顺序都是由底向上的。可是理想很美好，现实很骨感，这一步骤很难实现，是人都会犯错，更何况面对UE4当前的1572个UClass、1039个UStruct、588个Enum……你真的相信有人能管理好这些？所以在类型系统里想人工整理好类型的依赖定义顺序基本不现实，你几乎很难在构造本类型的时候，恰好的取得前置类型的对象。  
那怎么办？也简单，就如同C++里处理static单件对象的依赖顺序一样，既然处理不了，那就不处理！采用懒惰求值的思想，在需要前置类型的时候，先判断有没有构造出来，如果有就立即返回，如果没有就构造后再返回——一个简易版的单件模式。因为这个套路是如此的普遍，所以这一些判断加上构造的逻辑封装一下就成了一个个函数，为了获得那些对象，就变成了先获得那些函数指针了。生成的代码里都是大概这种套路：

```text
UClass* Z_Construct_UClass_UMyClass()   //用以获取UMyClass所对应的UClass的函数
{
    static UClass* OuterClass = nullptr;    //一个函数局部静态指针
    if (!OuterClass)    //反正都是单线程执行，所以不需要线程安全的保护
    {
        //...一些构造代码
        UE4CodeGen_Private::ConstructUClass(OuterClass, ClassParams);
    }
    return OuterClass;
}
```

利用此法，就在代码中形成了一个可自动上溯的前置对象获取链条。任何时候，想得到某一个类的UClass*对象，我们不需要去操心是否已经构造完成，也不需要担心它的依赖项是否已经全部构造了，因为代码的机制保证了前置项的按需构造。

## UPackage的Params和生成：

对于Hello模块而言，按照UE4的Module规则，我们需要定义一个Hello UPackage来存放该模块里定义的类型。  
之前的4.15的代码形式为：

```text
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

在4.17之后改为：

```text
//UObjectGlobals.h
namespace UE4CodeGen_Private
{
    //包参数
    struct FPackageParams
    {
        const char*                        NameUTF8;    //名字
        uint32                             PackageFlags; // EPackageFlags包的特征
        uint32                             BodyCRC; //内容的CRC,CRC的部分后续介绍
        uint32                             DeclarationsCRC; //声明部分的CRC
        UObject*                  (*const *SingletonFuncArray)();   //依赖的对象列表
        int32                              NumSingletons;
#if WITH_METADATA
        const FMetaDataPairParam*          MetaDataArray;//元数据数组
        int32                              NumMetaData;
#endif
    };
}
//Hello.init.gen.cpp
UPackage* Z_Construct_UPackage__Script_Hello()
{
    static UPackage* ReturnPackage = nullptr;
    if (!ReturnPackage)
    {
        static const UE4CodeGen_Private::FPackageParams PackageParams = {
            "/Script/Hello",    //包名字
            PKG_CompiledIn | 0x00000000,
            0xA1EAFF6A,
            0x41CF0543,
            nullptr, 0,
            METADATA_PARAMS(nullptr, 0)
        };
        UE4CodeGen_Private::ConstructUPackage(ReturnPackage, PackageParams);
    }
    return ReturnPackage;
}
```

本块内容比较简单，能坚持看到此处的朋友，对上文的代码应该是一目了然的。有一点需要知道的是，在Hello模块里的定义的类型数据，都是放在"/Script/Hello"Package里的，所以Hello Package是第一个首先构造出来的，因为它被后续的其他类型都依赖着。

## 总结

对比了前后两版本的代码，我们不难看出重构了之后，生成的代码更加的紧致，语法的噪音减少了很多，代码的信息量密度大大提高了。但要注意，本文关注的类型系统阶段是对之前[《InsideUE4》UObject（四）类型系统代码生成](https://zhuanlan.zhihu.com/p/25098685)的补充，后续依然是接着[《InsideUE4》UObject（五）类型系统收集](https://zhuanlan.zhihu.com/p/26019216)章节的内容进行开始收集，所以前文的那些static静态收集机制并没有改变。  
至于UE4CodeGen_Private::ConstructXXX构造的具体实现，我们在后续章节讲到类型系统的结构组织时候再详细讲解，我保证，那天不会太久远。当前阶段你可以简单的理解为都是通过一个个参数构造出一个个类型对象。

**思考：生成的代码能否做得更加的清晰高效？**  
虽然通过此次重构，代码的可读性上升了许多。但平心而论，现在的代码生成依然远远算不上优雅。那么在程序化代码生成的时候一般有哪些手段可以继续提升呢？  
追其本质，让代码变得简洁的手段其实都是在提升信息密度。把代码比作文件的话，重构就像压缩软件一样，把代码的信息量压缩到无所压就算是到了极致了。但是当然这中间当然也要权衡平台移植性（否则直接存一个二进制文件好了）、可读性、编译效率等问题。提升信息密度的手段就只有一个：同样的信息不要书写两遍。因此带来的方式就是封装！而封装，在代码生成的时候，我们其实可以用到：

1. 宏，UE4里其实已经用了一些宏来缩减代码，比如ARRAY_COUNT、VTABLE_OFFSET、IMPLEMENT_CLASS等。但目前的生成代码里依然有大量的长名字，套路化数组拼接，可以用宏来拼接。过度用宏当然也会降低可读性可调试性，但恰当的地方使用可以如同开挂一般优化掉巨量的代码。宏一直是代码拼接的最强大利器。
2. 函数，把相似的逻辑封装成函数可以优化掉大量的操作，只对外提供最简洁的信息输入接口。本文介绍的UE4优化方式就是用了函数来优化。我个人的倾向是宁愿在核心层多定义一些方便的辅助函数来接收多种输入，而不是在代码生成的时候去一个个拼函数的实体，这样可以大大减小生成代码的体积。函数的实现上巧用不定参数、数组和循环，可以使你的函数吞吐能力惊人。
3. 模板，更深层次的挖掘编译器提供的信息，压榨每个字段提供的信息量，利用它，从而自动推导出你所需要的其他信息。比如属性的类型就可以用模板根据字段的c++类型自动推导出来，而不需要手动分析注入了。UE4的生成代码里模板用的不多，是因为模板也有其很大的缺点：编译慢和难理解。在已经有了UHT分析代码的基础上，再用模板推导一遍，好像意义就不是那么大了，所以编译消耗还是能省一点是一点吧。至于模板的难理解，一款开源面向大众的引擎，在技术的选型实现上不应该过分的炫技，因为从业人员的技术水平，初中级的才是占绝大多数。考虑到受众问题和推广，有时候还是应该用一些朴实无华的实现比较能广为接受，同时也能有更大概率争取到重构维护者。否则，你写的代码，是很厉害，但是只有你自己能改得动改得明白，那叫社区里的人怎么为你贡献维护升级。
4. 扩展性，同时建议尽量把类型系统的构造逻辑放到Runtime里去，而不是在生成代码里（之前UE4就是在生成代码里直接new出一个个UXXX类型对象。放到Runtime，对外提供函数API接口，这样的好处是可测试性大大增长，不需要依赖UHT就可以手动构造出想要的类型进行测试。另外对于一些有在运行时动态Emit创建类型的需求来说，脱离UHT，保持自身功能的完备性也是必需的。