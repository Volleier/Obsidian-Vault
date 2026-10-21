## **引言**

前文介绍了[Static初始化](https://zhida.zhihu.com/search?content_id=100932088&content_type=Article&match_order=1&q=Static%E5%88%9D%E5%A7%8B%E5%8C%96&zhida_source=entity)阶段的最后一步进行的操作，创建出了第一个UClass对象。接着遵循程序启动的流程，本文就开始介绍Main函数入口进来后的流程。

还请注意以下几点：

- UE引擎这么大，其初始化从WinMain开始必然要经过一系列繁复的过程，而本章节只关注跟CoreUObject模块里，或者说是UObject系统相关的内容和流程，其他的初始化（比如窗口创建，线程启动，模块加载等）我们暂时忽略，挖个坑留待后续讲解。
- 同时也为了最简明的说明流程，忽略编辑器的相关函数调用流程内容，只关心Runtime下的流程（就是游戏打包后运行起来的流程。调试的过程是采用源码版引擎，创建个项目，先在Editor下CookContentForWindows，然后在VS里转为Debug配置，编译运行。这样就可以一起跟踪调试Game和Engine的内容。）
- 流程图里的箭头连接的代码块，不代表源码里就是这么直接相邻，中间仍然可能有其他的代码，只是与主题无关，所以不表示出来。箭头方向向右表示函数的嵌套调用，越向右嵌套越深；箭头向下表示一个函数内部的顺序执行代码块，向下结束表示这个函数完成执行。

## **引擎整体流程**

先大概看一下当运行项目时候的整个引擎启动流程。其中绿色的部分表示有涉及CoreUObject模块。

![](https://pic1.zhimg.com/v2-b03c1e38987d8ac13dacdc1a4d6c6474_1440w.jpg)

- Static初始化就是指的前文说的收集过程。
- 以Windows平台为例，WinMain是LaunchWindows.cpp里定义的程序入口。

```cpp
int32 WINAPI WinMain( _In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char*, _In_ int32 nCmdShow )
{
    //...
    ErrorLevel = GuardedMain( CmdLine, hInInstance, hPrevInstance, nCmdShow );
    //...
    FEngineLoop::AppExit(); //程序的退出
    //...
    return ErrorLevel;
}
```

- GuardedMain是真正的实现程序循环的地方。其中Engine开头的函数内部其实只是简单的转调一个全局的GEngineLoop的内部函数。

```cpp
FEngineLoop GEngineLoop;
int32 GuardedMain( const TCHAR* CmdLine, HINSTANCE hInInstance, HINSTANCE hPrevInstance, int32 nCmdShow )
{
   // make sure GEngineLoop::Exit() is always called.
    struct EngineLoopCleanupGuard 
    { 
        ~EngineLoopCleanupGuard()
        {
            EngineExit();   //保证在函数退出后能调用    转向 GEngineLoop.Exit();
        }
    } CleanupGuard;
    //...
    int32 ErrorLevel = EnginePreInit( CmdLine );    //预初始化  转向 GEngineLoop.PreInit( CmdLine );
    //...
#if WITH_EDITOR
    if (GIsEditor)
    {
        ErrorLevel = EditorInit(GEngineLoop);   //编辑器有其初始化版本
    }
    else
#endif
    {
        ErrorLevel = EngineInit();   //Runtime下的初始化    转向 GEngineLoop.Init();
    }
    //...
    while( !GIsRequestingExit )
    {
        EngineTick();   //无限循环的Tick    转向 GEngineLoop.Tick();
    }
    #if WITH_EDITOR
    if( GIsEditor )
    {
        EditorExit();   //编辑器的退出
    }
#endif
    return ErrorLevel;
}
```

- FEngineLoop::PreInit是我们关心的涉及UObject启动的最开始的地方。

## **FEngineLoop::PreInit**

我们知道，UE是建立在UObject对象系统上的，所以引擎里别的模块想要启动加载起来，就得先把CoreUObject模块初始化完成。因此引擎循环的预初始化部分就得开始加载CoreUObject了。

```cpp
int32 FEngineLoop::PreInit(const TCHAR* CmdLine)
{
    //...
    LoadCoreModules();  //加载CoreUObject模块
    //...
    //LoadPreInitModules();   //加载一些PreInit的模块，比如Engine，Renderer
    //...
    AppInit();  //程序初始化
    //...
    ProcessNewlyLoadedUObjects();   //处理最近加载的对象
    //...
    //LoadStartupModules();   //自己写的LoadingPhase为PreDefault的模块在这个时候加载
    //...
    GUObjectArray.CloseDisregardForGC();    //对象池启用，最开始是关闭的
    //...
    //NotifyRegistrationComplete();   //注册完成事件通知，完成Package加载
}
```

从这个预初始化的流程可以看出，最先加载的是CoreUObject。 其中的`LoadCoreModules()`内部调用`FModuleManager::Get().LoadModule(TEXT("CoreUObject"))`，会接着去触发`FCoreUObjectModule::StartupModule()`:

```cpp
class FCoreUObjectModule : public FDefaultModuleImpl
{
    virtual void StartupModule() override
    {
        // Register all classes that have been loaded so far. This is required for CVars to work.
        UClassRegisterAllCompiledInClasses();   //注册所有编译进来的类，此刻大概有1728多个

        void InitUObject();
        FCoreDelegates::OnInit.AddStatic(InitUObject);  //先注册个回调，后续会在AppInit里被调用
        //...
    }
}
```

## **UClassRegisterAllCompiledInClasses**

展开后是：

```cpp
void UClassRegisterAllCompiledInClasses()
{
    TArray<FFieldCompiledInInfo*>& DeferredClassRegistration = GetDeferredClassRegistration();
    for (const FFieldCompiledInInfo* Class : DeferredClassRegistration)
    {
        //这里的Class其实是TClassCompiledInDefer<TClass>
        UClass* RegisteredClass = Class->Register();    //return TClass::StaticClass();
    }
    DeferredClassRegistration.Empty();  //前面返回的是引用，因此这里可以清空数据。
}
//...
static TArray<FFieldCompiledInInfo*>& GetDeferredClassRegistration()    //返回可变引用
{
    static TArray<FFieldCompiledInInfo*> DeferredClassRegistration; //单件模式
    return DeferredClassRegistration;
}
```

想看懂这里的逻辑需要回顾提醒的有（忘了的请翻阅前三篇）：

1. `GetDeferredClassRegistration()`里的元素是之前收集文章里讲的静态初始化的时候添加进去的，在XXX.gen.cpp里用static TClassCompiledInDefer这种形式添加。
2. `TClassCompiledInDefer<TClass>::Register()`内部只是简单的转调`TClass::StaticClass()`。
3. `TClass::StaticClass()`是在XXX.generated.h里的`[DECLARE_CLASS](https://zhida.zhihu.com/search?content_id=100932088&content_type=Article&match_order=1&q=DECLARE_CLASS&zhida_source=entity)`宏里定义的，内部只是简单的转到`GetPrivateStaticClass(TPackage)`。
4. `GetPrivateStaticClass(TPackage)`的函数是实现是在`[IMPLEMENT_CLASS](https://zhida.zhihu.com/search?content_id=100932088&content_type=Article&match_order=1&q=IMPLEMENT_CLASS&zhida_source=entity)`宏里。其内部会真正调用到`GetPrivateStaticClassBody`。这个函数的内部会创建出UClass对象并调用Register()，在上篇已经具体讲解过了。
5. 总结这里的逻辑就是对之前收集到的所有的XXX.gen.cpp里定义的类，都触发一次其UClass的构造，其实也只有UObject比较特殊，会在Static初始化的时候就触发构造。因此这个过程其实是类型系统里每一个类的UClass的创建过程。
6. 这个函数会被调用多次，在后续的`ProcessNewlyLoadedUObjects`的里仍然会触发该调用。在`FCoreUObjectModule::StartupModule()`的这次调用是最先的，这个时候加载编译进来的的类都是引擎启动一开始就链接进来的。

**思考：猜猜看最先生成的是哪几个类？**

通过对关键代码的增加Log打印（比如在GetPrivateStaticClassBody的最后打印）， 朋友们可能会发现在Editor模式和Runtime模式下，各类的UClass可能会不太一样。这一方面原因是因为dll链接加载的方式顺序不一样，另一方面也是因为static变量的初始化顺序是不确定的，所以会造成进来的FFieldCompiledInInfo顺序不一样。但这其实也没太多影响，因为UE的代码里，有大量的防护性代码去加载前置所需要的类。另一方面，因为这个阶段生成的UClass，也只有SuperStruct和WithinClass之间的依赖，所以一定的顺序不定也没有关系。Static初始化的“Object”Class是最先的，Editor模式下会先加载CoreUObject模块和其他引擎模块，最后才是Hello模块（原因其实是编辑器的exe启动了然后去加载Hello.dll）。而打包后的游戏Runtime就反了过来，会先加载Hello模块，然后才是CoreUObject模块（原因其实是Hello.exe启动后内部加载其他dll）。所以static变量初始化的顺序其实大体上是越顶层的dll会越先被初始化。

附一下CoreUObject里面的各UClass来混个眼熟，反正也不多：

```cpp
//Static初始化:
Object
//CoreUObject:
GCObjectReferencer，TextBuffer，Field，Struct，ScriptStruct，Class，Package，Function，DelegateFunction，DynamicClass，PackageMap，Enum，EnumProperty，Property，Interface，LinkerPlaceholderClass，LinkerPlaceholderExportObject，LinkerPlaceholderFunction，MetaData，ObjectRedirector，ArrayProperty，ObjectPropertyBase，BoolProperty，ByteProperty，NumericProperty，ClassProperty，ObjectProperty，DelegateProperty，DoubleProperty，FloatProperty，IntProperty，Int16Property，Int64Property，Int8Property，InterfaceProperty，LazyObjectProperty，MapProperty，MulticastDelegateProperty，NameProperty，SetProperty，SoftClassProperty，SoftObjectProperty，StrProperty，StructProperty，UInt16Property，UInt32Property，UInt64Property，WeakObjectProperty，TextProperty
```

**思考：Struct和Enum的注册为何在这一个阶段无体现？**

在此阶段，我们好像没有看见在模块里定义的结构和枚举有参与此阶段的注册。其实是因为结构在注册后生成的元数据信息保存的对象是UScriptStruct，枚举对应的是[UEnum](https://zhida.zhihu.com/search?content_id=100932088&content_type=Article&match_order=1&q=UEnum&zhida_source=entity)，类对应的是UClass。 虽然我们在上篇说构造出来的第一个UClass也是一个UObject，但其实除了在Native编译进来的UClass，其他的UObject的构造都得需要有其对应的UClass的辅助，因为UClass里保存了类的构造函数指针。所以如果想构造出UScriptStruct和UEnum对象，就必须先有描述这两个类元数据信息的UClass。而这两个名为“ScriptStruct”和“Enum”的UClass在上述的CoreUObject模块加载里已经完成了。所以就不需要再做啥了。因此在这个阶段，其实已经是加载了所有基本的类型，因为类型就是用UClass描述。

描述对象类型的只有UClass，UScriptStruct和UEnum是两个保存结构和枚举元数据信息的对象，而构造对象就需要先有其UClass。

![](https://pic4.zhimg.com/v2-1bcb4ae016474197e5b3bedafaad967d_1440w.jpg)

  

讲到这，希望大家好好领悟这一句话：

> **UObject对象的类型是UClass，而UClass是个UObject对象。**

## **总结**

篇幅所限，本篇其实才刚刚讲了PreInit里面的`LoadCoreModules()`，这一步骤的目的主要是为了把CoreUObject里面定义的类的UClass都给先构建出来。但是其实这些UClass对象内部的值还没有完成初始化设置，因此下一个步骤的`AppInit()`和`ProcessNewlyLoadedUObjects()`还会继续这个注册的步程。下篇再来讲解AppInit()里的道道。