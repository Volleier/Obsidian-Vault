# Static初始化

在前面的文章里，讲解了在static阶段的信息收集。在[[UObject类型系统信息收集]]中最后UObject的收集有提及`IMPLEMENT_VM_FUNCTION(EX_CallMath, execCallMathFunction)`会触发`UObject::StaticClass()`的调用，因此作为最开始的调用，会生成第一个UClass*
```cpp
#define IMPLEMENT_FUNCTION(func) \
static FNativeFunctionRegistrar UObject#func#Registar(UObject::StaticClass(),#func,&UObject::func);

//...
IMPLEMENT_VM_FUNCTION(EX_CallMath, execCallMathFunction);//ScriptCore.cpp里的定义
```

虽然代码里有很多个`IMPLEMENT_CAST_FUNCTION`和`IMPLEMENT_VM_FUNCTION`的调用，但第一个触发的是`execCallMathFunction`的调用，可以看到`FNativeFunctionRegistrar`对象在构造的时候，第一个参数就会触发`UObject::StaticClass()`的调用。而以前文的内容，`StaticClass()`的调用会被展开为`GetPrivateStaticClass`的调用。
而`GetPrivateStaticClass`是在`IMPLEMENT_CLASS`里定义的，那么UObject的相关`IMPLEMENT_CLASS`是在NoExportTypes.h中定义的

# NoExportTypes.h
这个文件的目的就是为了把CoreUObject模块里的一些基础类型喂给UHT来生成类型的元数据信息。 NoExportTypes.h文件的结构其实理解起来很简单：

```cpp
#if CPP

//包含一些头文件来让NoExportTypes.gen.cpp可以编译通过

#endif

#if !CPP//这里面的部分是不参与编译的，所以不会产生定义冲突，但是却可以让UHT分析，因为UHT只是个文本分析器而已。

//枚举的声明，只是加上了宏标记。
//结构的声明，只是加上了宏标记。

//UObject的声明，C++的内容其实不重要，重要的是让UHT分析得到些什么信息
UCLASS(abstract, noexport)
class UObject
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintImplementableEvent, meta=(BlueprintInternalUseOnly = "true"))
    void ExecuteUbergraph(int32 EntryPoint);
};
#endif
```

而NoExportTypes.gen.cpp就和之前讲过的元数据代码生成一样的内容结构了。文件拉到最后你就会看到`IMPLEMENT_CLASS(UObject, 1563732853);`的定义了。

# **GetPrivateStaticClass**

而`IMPLEMENT_CLASS`根据前文的介绍的展开，里面定义着GetPrivateStaticClass的实现。 虽然最开始的调用是`UObject::StaticClass()`。但是以UMyClass为例会更看清楚里面参数的含义（因为跟我们实际应用时候的值更贴近，而UObject太基础了，很多信息是空的），工程名为Hello。

```cpp
//类的声明值
DECLARE_CLASS(UMyClass, UObject, COMPILED_IN_FLAGS(0), CASTCLASS_None, TEXT("/Script/Hello"), NO_API)
//值的传递
UClass* UMyClass::GetPrivateStaticClass(const TCHAR* Package)
{
    static UClass* PrivateStaticClass = NULL;   //静态变量，下回访问就不用再去查找了
    if (!PrivateStaticClass)
    {
        /* this could be handled with templates, but we want it external to avoid code bloat */
        GetPrivateStaticClassBody(
            Package,    //包名,TEXT("/Script/Hello")，用来把本UClass*构造在该UPackage里
            (TCHAR*)TEXT("UMyClass") + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0),//类名，+1去掉U、A、F前缀，+11去掉Deprecated_前缀
            PrivateStaticClass, //输出引用，所以值会被改变
            StaticRegisterNativesUMyClass,  //注册类Native函数的指针
            sizeof(UMyClass),   //类大小
            UMyClass::StaticClassFlags, //类标记，值为CLASS_Intrinsic，表示在C++代码里定义的类
            UMyClass::StaticClassCastFlags(),   //虽然是调用，但只是简单返回值CASTCLASS_None
            UMyClass::StaticConfigName(),   //配置文件名，用于从config里读取值
            (UClass::ClassConstructorType)InternalConstructor<UMyClass>,//构造函数指针，包了一层
            (UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<UMyClass>,//hotreload的时候使用来构造虚函数表，暂时不管
            &UMyClass::AddReferencedObjects,   //GC使用的添加额外引用对象的静态函数指针，若没有定义，则会调用到UObject::AddReferencedObjects，默认函数体为空。
            &UMyClass::Super::StaticClass,  //获取基类UClass*的函数指针，这里Super是UObject
            &UMyClass::WithinClass::StaticClass //获取对象外部类UClass*的函数指针，默认是UObject
        );
    }
    return PrivateStaticClass;
}
```

虽然值的传递还是很简明的，但有些要点在此还是得提醒一下：

1. Package名字的传入是为了在构建UClass\*之后，把UClass\*对象的OuterPrivate设定为正确的UPackage\*对象。在UE里，UObject必须属于某个UPackage。所以传入名字是为了后续查找或者创建出前置需要的UPackage对象。“/Script/”开头表示这是个代码模块。
2. `StaticRegisterNativesUMyClass`这个函数的名字是用宏拼接的，分别在.generated.h和.gen.cpp里声明和定义。
3. `InternalConstructor<UMyClass>`这个模板函数是为了包一下C++的构造函数，因为你没法直接去获得C++构造函数的函数指针。在.generated.h里会根据情况生成这两个宏的调用（`GENERATED_UCLASS_BODY`接收FObjectInitializer参数，`GENERATED_BODY`不接收参数），从而在以后的UObject\*构造过程中，可以调用到我们自己写的类的构造函数。
4. Super指的是类的基类，WithinClass指的是对象的Outer对象的类型。这里要区分开的是类型系统和对象系统之间的差异，Super表示的是类型上的必须依赖于基类先构建好UClass\*才能构建构建子类的UClass\*；WithinClass表示的是这个UObject\*在构建好之后应该限制放在哪种Outer下面，这个Outer所属于的UClass\*我们必须先提前构建好。

```cpp
#define DEFINE_DEFAULT_CONSTRUCTOR_CALL(TClass) \
static void __DefaultConstructor(const FObjectInitializer& X) {new((EInternal*)X.GetObj())TClass;}

#define DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass) \
static void __DefaultConstructor(const FObjectInitializer& X) {new((EInternal*)X.GetObj())TClass(X);}

template<class T>
void InternalConstructor( const FObjectInitializer& X )
{ 
    T::__DefaultConstructor(X);
}
```

# **GetPrivateStaticClassBody**

接着我们来看看内部的函数实现：

```cpp
void GetPrivateStaticClassBody(
    const TCHAR* PackageName,
    const TCHAR* Name,
    UClass*& ReturnClass,
    void(*RegisterNativeFunc)(),
    uint32 InSize,
    EClassFlags InClassFlags,
    EClassCastFlags InClassCastFlags,
    const TCHAR* InConfigName,
    UClass::ClassConstructorType InClassConstructor,
    UClass::ClassVTableHelperCtorCallerType InClassVTableHelperCtorCaller,
    UClass::ClassAddReferencedObjectsType InClassAddReferencedObjects,
    UClass::StaticClassFunctionType InSuperClassFn,
    UClass::StaticClassFunctionType InWithinClassFn,
    bool bIsDynamic /*= false*/
    )
{
    ReturnClass = (UClass*)GUObjectAllocator.AllocateUObject(sizeof(UClass), alignof(UClass), true);//分配内存
    ReturnClass = ::new (ReturnClass)UClass //用placement new在内存上手动调用构造函数
    (
    EC_StaticConstructor,Name,InSize,InClassFlags,InClassCastFlags,InConfigName,
    EObjectFlags(RF_Public | RF_Standalone | RF_Transient | RF_MarkAsNative | RF_MarkAsRootSet),
    InClassConstructor,InClassVTableHelperCtorCaller,InClassAddReferencedObjects
    );
    InitializePrivateStaticClass(InSuperClassFn(),ReturnClass,InWithinClassFn(),PackageName,Name);//初始化UClass*对象
    RegisterNativeFunc();//注册Native函数到UClass中去
}
```

这个函数内部只做了4件事：

1. 分配内存。GUObjectAllocator是全局的内存分配器，分配了一块内存来存放UClass对象。关于存储的内容后续再说，这里理解为返回一块内存就可。也要注意的是，ReturnClass是引用，这里一赋值，就代表外面static的PrivateStaticClass就有值了。所以就算这个GetPrivateStaticClassBody函数还没返回，但是如果去访问`UMyClass::StaticClass()`也是会立即返回这个值的。
2. 调用UClass的构造函数。这里的EC_StaticConstructor只是个标记用来指定调用特定的UClass构造函数重载版本。该构造函数内只是简单的成员变量赋值，并没有什么特别的。这么二步构造的原因是UObject的内存都是统一管理的，所以应该由GUObjectAllocator来分配，不能像标准C++那样直接new出来一个。
3. `InitializePrivateStaticClass`调用的时候，`InSuperClassFn()`和`InWithinClassFn()`是会先被调用的，所以其会先触发`Super::StaticClass()`和`WithinClass::StaticClass()`，再会堆栈式的加载前置的类型。
4. `RegisterNativeFunc()`就是上文的`StaticRegisterNativesUMyClass`，在此刻调用，用来像UClass里添加Native函数。Native函数指的是在C++有函数体实现的函数，而蓝图中的函数和BlueprintImplementableEvent的函数就不是Native函数。

# **InitializePrivateStaticClass**

```cpp
COREUOBJECT_API void InitializePrivateStaticClass(
    class UClass* TClass_Super_StaticClass,
    class UClass* TClass_PrivateStaticClass,
    class UClass* TClass_WithinClass_StaticClass,
    const TCHAR* PackageName,
    const TCHAR* Name
    )
{
    //...
    if (TClass_Super_StaticClass != TClass_PrivateStaticClass)
    {
        TClass_PrivateStaticClass->SetSuperStruct(TClass_Super_StaticClass);    //设定类之间的SuperStruct
    }
    else
    {
        TClass_PrivateStaticClass->SetSuperStruct(NULL);    //UObject无基类
    }
    TClass_PrivateStaticClass->ClassWithin = TClass_WithinClass_StaticClass;    //设定Outer类类型
    //...
    TClass_PrivateStaticClass->Register(PackageName, Name); //转到UObjectBase::Register()
    //...
}
```

这个函数的名字叫做初始化，但其实没干啥事。

1. 设定类型的SuperStruct。SuperStruct是定义在UStruct里的UStruct* SuperStruct，用来指向本类型的基类。
2. 设定ClassWithin的值。也就是限制Outer的类型。
3. 调用`UObjectBase::Register()`。终于对每个UClass*开始了注册，不枉调用链条上的`UClassRegisterAllCompiledInClasses`的Register之名。

# **UObjectBase::Register**

而该函数比较简单，只是简单的先记录一下信息到一个全局单件Map里和一个全局链表里。

```cpp
struct FPendingRegistrantInfo
{
    const TCHAR*    Name;   //对象名字
    const TCHAR*    PackageName;    //所属包的名字
    static TMap<UObjectBase*, FPendingRegistrantInfo>& GetMap()
    {   //用对象指针做Key，这样才可以通过对象地址获得其名字信息，这个时候UClass对象本身其实还没有名字，要等之后的注册才能设置进去
        static TMap<UObjectBase*, FPendingRegistrantInfo> PendingRegistrantInfo;    
        return PendingRegistrantInfo;
    }
};
//...
struct FPendingRegistrant
{
    UObjectBase*    Object; //对象指针，用该值去PendingRegistrants里查找名字。
    FPendingRegistrant* NextAutoRegister;   //链表下一个节点
};
static FPendingRegistrant* GFirstPendingRegistrant = NULL;  //全局链表头
static FPendingRegistrant* GLastPendingRegistrant = NULL;   //全局链表尾
//...
void UObjectBase::Register(const TCHAR* PackageName,const TCHAR* InName)
{
    //添加到全局单件Map里，用对象指针做Key，Value是对象的名字和所属包的名字。
    TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();
    PendingRegistrants.Add(this, FPendingRegistrantInfo(InName, PackageName));
    //添加到全局链表里，每个链表节点带着一个本对象指针，简单的链表添加操作。
    FPendingRegistrant* PendingRegistration = new FPendingRegistrant(this);
    if(GLastPendingRegistrant)
    {
        GLastPendingRegistrant->NextAutoRegister = PendingRegistration;
    }
    else
    {
        check(!GFirstPendingRegistrant);
        GFirstPendingRegistrant = PendingRegistration;
    }
    GLastPendingRegistrant = PendingRegistration;
}
```

  

**思考：为何Register只是先记录一下信息？**

初看之下肯定会疑惑，为何这里并没有做一些实际的操作。其实是因为UClass的注册分成了多步，在static初始化的时候（连main都没进去呢），甚至到后面CoreUObject模块加载的时候，UObject对象分配索引的机制（GUObjectAllocator和GUObjectArray）还没有初始化完毕，因此这个时候如果走下一步去创建各种UProperty、UFunction或UPackage是不合适，创建出来了也没有合适的地方来保存索引。所以，在最开始的时候，只能先简单的创建出各UClass*对象（简单到对象的名字都还没有设定，更何况填充里面的属性和方法了），先在内存里把这些UClass*对象记录一下，等后续对象的存储结构准备好了，就可以把这些UClass*对象再拉出来继续构造了。先剧透一下，后续的初始化[对象存储](https://zhida.zhihu.com/search?content_id=100797707&content_type=Article&match_order=1&q=%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8&zhida_source=entity)机制的函数调用是`InitUObject()`，继续构造的操作是在`ProcessNewlyLoadedUObjects()`里的。这些信息在后面会被消费用到的，莫急。

**思考：记录信息为何需要一个TMap加一个链表？**

我们可以看到，为了记录信息，明明是用一个数据结构就能保存的（源码里的两个数据结构里的数据数量也是1:1的），为何要麻烦的设置成这样。原因有三：

1. 是快速查找的需要。在后续的别的代码（获取CDO等）里也会经常调用到`UObjectForceRegistration(NewClass)`，因此常常有通过一个对象指针来查找注册信息的需要，这个时候为了性能就必须要用字典类的数据结构才能做到O(1)的查找。
2. 顺序注册的需要。而字典类的数据结构一般来说内部为了hash，数据遍历取出的顺序无法保证和添加的顺序一致，而我们又想要遵循添加的顺序来注册（很合理，早添加进来的是早加载的，是更底层的，处在依赖顺序的前提位置。我们前面的SuperClass和WithinClass的访问也表明了这一点），因此就需要另一个顺序数据结构来辅助。
3. 那为什么是链表而不是数组呢？链表比数组优势的地方也只在于可以快速的中间插入。但是UE源码里也没有这个方面的体现，所以其实二者都可以。我在源码里把注册结构改为如下用数组也依然可以正常工作。要嘛是他们的代码写得也挺啰嗦，要嘛是我没懂其他的深意。不过倒也无伤大雅。

```cpp
struct FPendingRegistrant
{
    UObjectBase*    Object;
    FPendingRegistrant(UObjectBase* InObject)
        : Object(InObject)
    {}
    static TArray<FPendingRegistrant>& GetArray()
    {
        static TArray<FPendingRegistrant> PendingRegistrants;
        return PendingRegistrants;
    }
};
```

# **RegisterNativeFunc**

讲完了注册，接着说GetPrivateStaticClassBody的最后一步：RegisterNativeFunc的调用，同样以MyClass为例：

```cpp
//...MyClass.gen.cpp
void UMyClass::StaticRegisterNativesUMyClass()
{
    UClass* Class = UMyClass::StaticClass();   //这里是可以立即返回值的
    static const FNameNativePtrPair Funcs[] = { 
        //exec开头的都是在.generated.h里定义的蓝图用的，暂时不管它，理解为可以调用就行了。
        { "AddHP", &UMyClass::execAddHP },
        { "CallableFunc", &UMyClass::execCallableFunc },
        { "NativeFunc", &UMyClass::execNativeFunc },
    };
    FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, ARRAY_COUNT(Funcs));
}
//...

void FNativeFunctionRegistrar::RegisterFunctions(class UClass* Class, const FNameNativePtrPair* InArray, int32 NumFunctions)
{
    for (; NumFunctions; ++InArray, --NumFunctions)
    {
        Class->AddNativeFunction(UTF8_TO_TCHAR(InArray->NameUTF8), InArray->Pointer);
    }
}
//...
void UClass::AddNativeFunction(const ANSICHAR* InName, FNativeFuncPtr InPointer)
{
    FName InFName(InName);
    new(NativeFunctionLookupTable) FNativeFunctionLookup(InFName,InPointer);
}
```

而NativeFunctionLookupTable是在UClass里的一个成员变量

```cpp
//蓝图调用的函数指针原型
typedef void (*FNativeFuncPtr)(UObject* Context, FFrame& TheStack, RESULT_DECL);
/** A struct that maps a string name to a native function */
struct FNativeFunctionLookup
{
    FName Name; //函数名字
    FNativeFuncPtr Pointer;//函数指针
};
//...
class COREUOBJECT_API UClass : public UStruct
{
public:
    TArray<FNativeFunctionLookup> NativeFunctionLookupTable;
}
```

可以看到这步操作这是简单的往UClass*里添加Native函数的数据。

**思考：为什么这么猴急的需要一开始就往UClass里添加Native函数？**

以`IMPLEMENT_VM_FUNCTION(EX_CallMath, execCallMathFunction)`为例，execCallMathFunction是定义在代码里的一个函数，它的地址必然需要通过一种方式记录下来。当然你也可以像UE4CodeGen_Private做的那样，先用各种Params对象保存起来，然后在后面合适的时候调用提取来添加。只不过这个时候因为UClass对象都已经创建出来了，所以就索性直接存到NativeFunctionLookupTable里面去了，后续要用的时候再用名字去里面查找。稍微提一下，这里不用TMap而用TArray是因为一般来说我们在一个类里写的函数数量并不会太多，对于元素比较少的情况下，TArray的线性查找也很快，而且还省内存。

**思考：那些非Native的函数怎么办？**

其实就是指的就是BlueprintImplementableEvent的函数，它不需要我们自己定义函数体。而UHT会帮我们生成一个函数体，当我们在C++里调用ImplementableFunc的时候，其实会触发一次函数查找，如果在蓝图中有定义该名字的函数，则会得到调用。

```cpp
//...MyClass.h
UFUNCTION(BlueprintImplementableEvent)
void ImplementableFunc();   //C++不实现，蓝图实现

//...MyClass.gen.cpp
void UMyClass::ImplementableFunc()
{
    ProcessEvent(FindFunctionChecked(TEXT("ImplementableFunc"),NULL);
}
```

需要提前注意的是，不管是Native与否，函数后面都会生成一个UFunction对象，只不过Native函数的UFunction在绑定的时候会去它所属于的UClass里的NativeFunctionLookupTable通过函数名字查找真正的函数指针，而非Native的UFunction会把函数指针指向`UObject::ProcessInternal`，用来处理[蓝图虚拟机](https://zhida.zhihu.com/search?content_id=100797707&content_type=Article&match_order=1&q=%E8%93%9D%E5%9B%BE%E8%99%9A%E6%8B%9F%E6%9C%BA&zhida_source=entity)调用的情况。