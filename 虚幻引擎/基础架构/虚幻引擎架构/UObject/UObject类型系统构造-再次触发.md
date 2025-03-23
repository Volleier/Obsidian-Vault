## **引言**

上文讲解到了UObject系统的初始化和各UClass\*对象的初级构造，但是这还没有完成，UClass\*对象里还有很多东西没有塞进去。本文将继续`[FEngineLoop](https://zhida.zhihu.com/search?content_id=101211422&content_type=Article&match_order=1&q=FEngineLoop&zhida_source=entity)::PreInit()`里AppInit的下一个非常重要的函数ProcessNewlyLoadedUObjects。同时非常需要注意的是ProcessNewlyLoadedUObjects会在模块加载后再次触发调用，所以脑袋里一定要时刻意识到两点，一是它是重复调用多次的，二是它的内部流程是一个完整的流程。

## **ProcessNewlyLoadedUObjects**

```cpp
void ProcessNewlyLoadedUObjects()
{
    UClassRegisterAllCompiledInClasses();   //为代码里定义的那些类生成UClass*
    //提取收集到的注册项信息
    const TArray<UClass* (*)()>& DeferredCompiledInRegistration=GetDeferredCompiledInRegistration();
    const TArray<FPendingStructRegistrant>& DeferredCompiledInStructRegistration=GetDeferredCompiledInStructRegistration();
    const TArray<FPendingEnumRegistrant>& DeferredCompiledInEnumRegistration=GetDeferredCompiledInEnumRegistration();
    //有待注册项就继续循环注册
    bool bNewUObjects = false;
    while (GFirstPendingRegistrant || 
    DeferredCompiledInRegistration.Num() || 
    DeferredCompiledInStructRegistration.Num() || 
    DeferredCompiledInEnumRegistration.Num())
    {
        bNewUObjects = true;
        UObjectProcessRegistrants();    //注册UClass*
        UObjectLoadAllCompiledInStructs();  //为代码里的枚举和结构构造类型对象
        UObjectLoadAllCompiledInDefaultProperties();    //为代码里的类继续构造UClass对象
    }

    if (bNewUObjects && !GIsInitialLoad)
    {
        UClass::AssembleReferenceTokenStreams();    //构造引用记号流，为后续GC用
    }
}
```

代码里的结构也比较简单，还有一些我们之前介绍过的函数，忘了的朋友请回顾前文，特别是信息收集，因为此刻就开始真正使用这些数据了。

1. `UClassRegisterAllCompiledInClasses`在前文介绍过，里面主要是为每一个编译进来的class调用TClass::[StaticClass](https://zhida.zhihu.com/search?content_id=101211422&content_type=Article&match_order=1&q=StaticClass&zhida_source=entity)()来构造出UClass*对象。
2. `GetDeferredCompiledInRegistration()`是之前信息收集的时候static FCompiledInDefer变量初始化时收集到的全局数组，和定义的**class**一对一。
3. `GetDeferredCompiledInStructRegistration()`是之前信息收集的时候static FCompiledInDeferStruct变量初始化时收集到的全局数组，和定义的**struct**一对一。
4. `GetDeferredCompiledInEnumRegistration()`是之前信息收集的时候static FCompiledInDeferEnum变量初始化时收集到的全局数组，和定义的**enum**一对一。
5. `UObjectProcessRegistrants()`前文刚讲过，为之前生成的UClass*注册，生成其Package。这里调用的目的是在后续的操作之前确保内存里已经把相关的类型UClass*对象都已经注册完毕。
6. `UObjectLoadAllCompiledInStructs()`里为enum和struct分别生成UEnum和UScriptStruct对象。后文详细讲解。
7. `UObjectLoadAllCompiledInDefaultProperties()`里为UClass*们继续构造和创建类默认对象([CDO](https://zhida.zhihu.com/search?content_id=101211422&content_type=Article&match_order=1&q=CDO&zhida_source=entity))。后文详细讲解。
8. 最后一步判断如果有新UClass*对象生成了，并且现在不在**初始化载入阶段**（GIsInitialLoad初始=true，只有在后续开启GC后才=false表示初始化载入过程结束了），用AssembleReferenceTokenStreams为UClass创建**引用记号流**（一种辅助GC分析对象引用的数据结构，挖坑留待以后讲GC的时候讲解。）。所以第一次的`FEngineLoop::PreInit()`里的ProcessNewlyLoadedUObjects并不会触发AssembleReferenceTokenStreams的调用但也会在后续的[GUObjectArray](https://zhida.zhihu.com/search?content_id=101211422&content_type=Article&match_order=1&q=GUObjectArray&zhida_source=entity).CloseDisregardForGC()里面调用AssembleReferenceTokenStreams。只有后续模块动态加载后触发的ProcessNewlyLoadedUObjects才会AssembleReferenceTokenStreams。通过这个判断保证了在两种情况下，AssembleReferenceTokenStreams只会被调用一次。

**思考：为何ProcessNewlyLoadedUObjects函数里前面的步骤总有一种既视感？**

抛开后面的两个Load函数，前面的`UClassRegisterAllCompiledInClasses()`和`UObjectProcessRegistrants()`都是我们前面两篇文章里讲过的：

- Static初始化阶段，会调用`UClassRegisterAllCompiledInClasses()`来生成UClass*。
- CoreUObject模块加载阶段，先初始化一下对象分配系统。GUObjectAllocator和GUObjectArray。
- CoreUObject模块加载阶段，接着调用`UObjectProcessRegistrants()`来注册UClass*。

因为此刻当然对象分配系统已经初始化过了，所以这二者的调用顺序和ProcessNewlyLoadedUObjects里的是一样的！原因很简单，每一个模块dll的加载，我们其实都应该为他安排一次一条龙的类型树的构建服务，所以这个构造UClass*的过程都要依照此流程走一遍。当然这个过程也是可以批量合并的，在引擎启动的时候，已经加载了很多模块dll，因此反正也是先执行了前面这两步。

## **UObjectLoadAllCompiledInStructs**

让我们继续深挖，类型UClass*都有了，这一步开始真正的构造UEnum和UScriptStruct。

```cpp
static void UObjectLoadAllCompiledInStructs()
{
    TArray<FPendingEnumRegistrant> PendingEnumRegistrants = MoveTemp(GetDeferredCompiledInEnumRegistration());
    for (const FPendingEnumRegistrant& EnumRegistrant : PendingEnumRegistrants)
    {
        CreatePackage(nullptr, EnumRegistrant.PackageName); //创建其所属于的Package
    }

    TArray<FPendingStructRegistrant> PendingStructRegistrants = MoveTemp(GetDeferredCompiledInStructRegistration());
    for (const FPendingStructRegistrant& StructRegistrant : PendingStructRegistrants)
    {
        CreatePackage(nullptr, StructRegistrant.PackageName);   //创建其所属于的Package
    }

    for (const FPendingEnumRegistrant& EnumRegistrant : PendingEnumRegistrants)
    {
        EnumRegistrant.RegisterFn();    //调用生成代码里Z_Construct_UEnum_Hello_EMyEnum
    }
    for (const FPendingStructRegistrant& StructRegistrant : PendingStructRegistrants)
    {
        StructRegistrant.RegisterFn(); //调用生成代码里Z_Construct_UScriptStruct_FMyStruct
    }
}
```

开头的两个数组就不赘述了。这个代码写得也很有意思：

1. 先创建EnumRegistrant.PackageName再创建StructRegistrant.PackageName。这两个名字值都是在生成代码里定义的，同UClass一样，表示了其所在的Package。注意的是，CreatePackage的里面总是会先查找该名字的Package是否已经存在，不会重复创建。
2. MoveTemp会触发TArray的右移引用赋值，把源数组里的数据迁移到目标数组里去。所以外层的while判断值才会改变。
3. 先enum再struct的调用其注册函数RegisterFn()。RegisterFn是个函数指针，指向生成代码里Z_Construct开头的函数，用来真正构造出UEnum和UScriptStruct对象。
4. 有意思的是，顺序总是先enum再struct。其原因其实是因为更基础的类型总是先构造。代码里enum不能嵌套struct，但struct里却可以包含enum。所以在struct里包含一个enum变量的时候，构造UScriptStruct时就需要用enum名字查找到其相应的UEnum*对象，因此当然希望UEnum在前面都先构造好了。有位朋友问了，那如果struct里包含一个UMyClass*变量怎么办？不怎么办，因为各Class所属于的UClass*对象都已经在前面的UObjectProcessRegistrants()里注册好了，所有的对象引用类型反正只是通过Class名字来查找到UClass*，因此UClass*对象就算还没有真正构造完毕也没有关系，反正只要能用名字查找到就好了！

## **UObjectLoadAllCompiledInDefaultProperties**

针对UClass*对象的构造的重头戏来了：

```cpp
static void UObjectLoadAllCompiledInDefaultProperties()
{
    static FName LongEnginePackageName(TEXT("/Script/Engine")); //引擎包的名字
    if(GetDeferredCompiledInRegistration().Num() <= 0) return;
    TArray<UClass*> NewClassesInCoreUObject;
    TArray<UClass*> NewClassesInEngine;
    TArray<UClass*> NewClasses;
    TArray<UClass* (*)()> PendingRegistrants = MoveTemp(GetDeferredCompiledInRegistration());
    for (UClass* (*Registrant)() : PendingRegistrants) 
    {
        UClass* Class = Registrant();//调用生成代码里的Z_Construct_UClass_UMyClass创建UClass*
        //按照所属于的Package分到3个数组里
        if (Class->GetOutermost()->GetFName() == GLongCoreUObjectPackageName)
        {
            NewClassesInCoreUObject.Add(Class);
        }
        else if (Class->GetOutermost()->GetFName() == LongEnginePackageName)
        {
            NewClassesInEngine.Add(Class);
        }
        else
        {
            NewClasses.Add(Class);
        }
    }
    //分别构造CDO对象
    for (UClass* Class : NewClassesInCoreUObject)   { Class->GetDefaultObject(); }
    for (UClass* Class : NewClassesInEngine)        { Class->GetDefaultObject(); }
    for (UClass* Class : NewClasses)                { Class->GetDefaultObject(); }
}
```

步骤共有：

1. 从`GetDeferredCompiledInRegistration()`的源数组里MoveTemp出来遍历。
2. 依次调用`Registrant()`来继续构造UClass*，这个函数指向了生成代码里形如Z_Construct_UClass_UMyClass的函数。
3. 对生成的UClass*对象，依照属于的Package划分到3个数组里。
4. 对3个数组分别依次手动构造CDO对象。这三个数组的顺序是：CoreUObject、Engine和其他。按照此顺序构造的原因是根据依赖关系。构造CDO的过程，有可能触发uassset的加载和UObject构造函数的调用，所以就可能在内部触发其他Package里对象的加载构造。CoreUObject最底层（它不会引用其他的Package里的对象）、Engine次之（它有可能引用底层的对象）、其他（就不确定会引用啥了）。所以依照此顺序能避免依赖倒置，从而减少重复调用查找。
5. 我们知道，代码里Class里可以包含结构和枚举，因此UObjectLoadAllCompiledInDefaultProperties被安排到UObjectLoadAllCompiledInStructs之后，可以让此时构造的UClass*对象能够通过enum和struct的类型名字查找到相应的UEnum*和UScriptStruct*对象。顺序还是很讲究的。

## **CloseDisregardForGC**

UClass*构造之后的还有个尾巴操作，此时可以开启GC了。因为之前一直都是在初始化载入阶段，这个阶段构造的类型UClass*对象和CDO对象，及其属于的UPackage对象，都是属于引擎底层的**必要**对象，它们是只有在游戏推出的时候才会销毁，因此它们就不属GC管了——GC一开始也是关着的：OpenForDisregardForGC=true。在类型系统都构建完了之后，就可以放心的打开GC了，因为后续就有可能NewObject来生成对象了。

```cpp
void FUObjectArray::CloseDisregardForGC()
{
    if (!GIsRequestingExit)
    {
        ProcessNewlyLoadedUObjects();//之前仍然有可能加载了别的模块dll
    }

    UClass::AssembleReferenceTokenStreams(); //此时才是真正的第一次为所有的UClass\*构建引用记号流
    //...
    OpenForDisregardForGC = false;
    GIsInitialLoad = false;//初始化载入阶段结束
}
```

代码省略了其他无关操作，关于GC的内容挖坑后续讲解。此时提前先了解一下，以对全局流程有个总览。

## **总结**

现在到此为止，我们能了解到内存中已经构造出来了各类型表达对象，之前描述过的各个信息收集阶段收集到各种信息也都得到了消费应用，让我们来梳理一下：

  

![](https://pic3.zhimg.com/v2-4469fb4075483a76a516494c17a2f3cc_1440w.jpg)

  

1. 大部分是不言自明的，从左到右是类型信息的收集和消费过程。从上到下是依据代码的执行顺序。 
2. 红色箭头代表数据的产生添加，蓝色箭头代表数据的消费使用。这二者一起表达了类型信息的数据流向。 
3. 浅蓝色箭头和矩形，代表内存中UClass\*以及类型对象的创建和构造。 4. 信息收集里黄色的3个矩形，代表它们的数据会一直在内存中，用来做查找用，不会被清空。