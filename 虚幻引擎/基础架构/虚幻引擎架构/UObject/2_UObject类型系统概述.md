众所周知，一般游戏引擎最底层面对的都是操作系统API，硬件SDK，所能借助到的工具也往往只有C++本身。所以考虑从原生的C++基础上，搭建对象系统，往往得从头开始造轮子，最底层也是最核心的机制当然必须得掌控在自己的手中，以后升级改善增加功能也才能不受限制。

那么，从头开始的话，Object系统有那么多功能：GC，反射，序列化，编辑器支持……应该从哪一个开始？哪一个是必需的？GC不是，因为大不了我可以直接new/delete或者智能指针引用技术，毕竟别的很多引擎也都是这么干的。序列化也不是，大不了每个子类里手写数据排布，麻烦是麻烦，但是功能上也是可以实现的。编辑器支持，默认类对象，统计等都是之后额外附加的功能了。那你说反射为何是必需的？大多数游戏引擎用的C++没有反射，不也都用得好好的？确实也如此，不利用反射的那些功能，不动态根据类型创建对象，不遍历属性成员，不根据名字调用函数，大不了手写绕一下，没有过不去的坎。但是既然上文已经论述了一个统一Object模型的好处，那么如果在Object身上不加上反射，无疑就像是砍掉了Object的一双翅膀，让它只能在地上行走，而不能在更宽阔空间内发挥威力。还有另一个方面的考虑是，反射作为底层的系统，如果实现完善了，也可以大大裨益其他系统的实现，比如有了反射，实现序列化起来就很方便了；有没有反射，也关系到GC实现时的方案选择，完全是两种套路。简单举个例，反射里对每个object有个class对象保存信息，所以理论上class身上就可以保存所有该类型的object指针引用，这个信息GC就可以利用起来实现一些功能；而没有这个class对象的话，GC的实现就得走别的方案路子了。所以说是先实现反射，有了一个更加扎实的对象系统基础后，再在此之上实现GC才更加的明智。
# 类型系统
虽然之上一直用反射的术语来描述我们熟知的那一套运行时得到类型信息的系统，动态创建类对象等，但是其实“反射”只是在“类型系统”之后实现的附加功能，人们往往都太过注重最后表露的强大功能，而把朴实的本质支撑给忘记了。想想看，如果我实现了class类提供Object的类型信息，但是不提供动态创建，动态调用函数等功能，请问还有没有意义？其实还仍然是非常有意义的，光光是提供了一个类型信息，就提供了一个Object之外的静态信息载体，也能构建起来object之间的派生从属关系，想想UE里如果去掉了根据名字创建类对象的能力，是会损失一些便利功能，但确实也还没有到元气大伤的程度，GC依然能跑得起来。

所以以后更多用“类型系统”这个更精确的术语来表述object之外的类型信息构建，而用“反射”这个术语来描述运行时得到类型的功能，通过类型信息反过来创建对象，读取修改属性，调用方法的功能行为。反射更多是一种行为能力，更偏向动词。类型系统指的是程序运行空间内构建出来的类型信息树组织

# C# Type

因C++本身运行时类型系统的疲弱，所以我们首先拿一个已经实现完善的语言，来看看其最后成果是什么样子。这里选择了C#而不是java，是因为我认为C#比java更强大优雅（不辩），Unity用C#作为脚本语言，UE本身也是用C#作为编译UBT的实现语言。  
在C#里，你可以通过以下一行代码方便的得到类型信息：
```cpp
Type type = obj.GetType();  //or typeof(MyClass)
```

1. Assembly是程序集的意思，通常指的是一个dll。
2. Module是程序集内部的子模块划分。
3. Type就是我们最关心的Class对象了，完整描述了一个对象的类型信息。并且Type之间也可以通过BaseType，DeclaringType之类的属性来互相形成Type关系图。
4. ConstructorInfo描述了Type中的构造函数，可以通过调用它来调用特定的构造函数。
5. EventInfo描述了Type中定义的event事件（UE中的delegate大概）
6. FiedInfo描述了Type中的字段，就是C++的成员变量，得到之后可以动态读取修改值
7. PropertyInfo描述了Type中的属性，类比C++中的get/set方法组合，得到后可以获取设置属性值。
8. MethodInfo描述了Type中的方法。获得方法后就可以动态调用了。
9. ParameterInfo描述了方法中的一个个参数。
10. Attributes指的是Type之上附加的特性，这个C++里并没有，可以简单理解为类上的定义的元数据信息。

可以看到C#里的Type几乎提供了一切信息数据，简直就像是把编译器编译后的数据都给暴露出来了给你。实际上C#的反射还可以提供其他更高级的功能，比如运行时动态创建出新的类，动态Emit编译代码，不过这些都是后话了（在以后讲解蓝图时应该还会提到）。当前来说，我希望读者们能有一个大概的印象就是，用代码声明定义出来的类型，当然可以通过一种数据结构完整描述出来，并在运行时再得到。

# C++ RTTI

而谈到C++中的运行时类型系统，我们一般会说RTTI（Run-Time Type Identification），只提供了两个最基本的操作符：

  

## typeid

这个关键字的主要作用就是用于让用户知道是什么类型，并提供一些基本对比和name方法，作用也顶多只是让用户判断从属于不同的类型，所以其实说起来type_info的应用并不广泛，一般来说也只是把它当作编译器提供的一个唯一类型Id。

  

```cpp
const std::type_info& info = typeid(MyClass);

class type_info
{
public:
    type_info(type_info const&) = delete;
    type_info& operator=(type_info const&) = delete;
    size_t hash_code() const throw();
    bool operator==(type_info const& _Other) const throw();
    bool operator!=(type_info const& _Other) const throw();
    bool before(type_info const& _Other) const throw();
    char const* name() const throw();
};
```

## dynamic_cast

该转换符用于将一个指向派生类的基类指针或引用转换为派生类的指针或引用，使用条件是只能用于含有虚函数的类。转换引用失败会抛出bad_cast异常，转换指针失败会返回null。

  

```cpp
Base* base=new Derived();
Derived* p=dynamic_cast<Derived>(base);
if(p){...}else{...}
```

dynamic_cast内部机制其实也是利用虚函数表里的类型信息来判断一个基类指针是否指向一个派生类对象。其目的更多是用于在运行时判断对象指针是否为特定一个子类的对象。

其他的比如运用模板，宏标记就都是编译期的手段了。C++在RTTI方面也确实是非常的薄弱，传说中的标准反射提案也遥遥无期，所以大家就都得八仙过海各显神通，采用各种方式模拟实现了。C++都能用于去实现别的语言底层，不就是多一个轮子的事嘛。

  

# C++当前实现反射的方案

既然C++本身没提供足够的类型信息，那我们就采用各种其他各种额外方式来搜集，并保存构建起来之后供程序使用。根据搜集信息的方式不同，C++的反射方案也有以下流派：

  

## 宏

基本思想是采用手动标记。在程序中用手动的方式注册各个类，方法，数据。大概就像这样：

  

```cpp
struct Test
{
    Declare_Struct(Test);
    Define_Field(1, int, a)
    Define_Field(2, int, b)
    Define_Field(3, int, c)
    Define_Metadata(3)
};
```

用宏偷梁换柱的把正常的声明换成自己的结构。简单可见这种方式还比较的原始，写起来也非常的繁琐。因此往往用的不多。更重要的是往往需要打破常规的书写方式，因此常常被摒弃掉。

  

## 模板

C++中的模板是应该也可以算是最大区别于别的语言的一个大杀器了，引导其强大的编译器类型识别能力构建出相应的数据结构，理论上也是可以实现出一定的类型系统。举一个Github实现比较优雅的C++RTTI反射库做例子：[rttr](https://link.zhihu.com/?target=https%3A//github.com/rttrorg/rttr)

  

```cpp
#include <rttr/registration>
using namespace rttr;
struct MyStruct { MyStruct() {}; void func(double) {}; int data; };
RTTR_REGISTRATION
{
    registration::class_<MyStruct>("MyStruct")
         .constructor<>()
         .property("data", &MyStruct::data)
         .method("func", &MyStruct::func);
}
```

说实话，这写得已经非常简洁优雅了。算得上是达到了C++模板应用的巅峰。但是可以看到，仍然需要一个个的手动去定义类并获取方法属性注册。优点是轻量程序内就能直接内嵌，缺点是不适合懒人。

  

## 编译器数据分析

还有些人就想到既然C++编译器编译完整个代码，那肯定是有完整类型信息数据的。那能否把它们转换保存起来供程序使用呢？事实上这也是可行的，比如@vczh的GacUI里就分析了VC编译生成后pdb文件，然后抽取出类型定义的信息实现反射。VC确实也提供了IDiaDataSource COM组件用来读取pdb文件的内容。用法可以参考：[GacUI Demo：PDB Viewer（分析pdb文件并获取C++类声明的详细内容）](https://link.zhihu.com/?target=http%3A//www.cppblog.com/vczh/archive/2011/12/30/163200.html)。  
理论上来说，只要你能获取到跟编译器同级别的类型信息，你基本上就像是全知了。但是缺点是分析编译器的生成数据，太过依赖平台（比如只能VC编译，换了Clang就是另一套方案），分析提取的过程往往也比较麻烦艰深，在正常的编译前需要多加一个编译流程。但优点也是得到的数据最是全面。  
这种方案也因为太过麻烦，所以业内用的人不多。

  

## 工具生成代码

自然的有些人就又想到，既然宏和模板的方法，太过麻烦。那我能不能写一个工具来自动完成呢？只要分析好C++代码文件，或者分析编译器数据也行，然后用预定义好的规则生成相应的C++代码来跟源文件对应上。  
一个好例子就是Qt里面的反射：

  

```cpp
#include <QObject>
class MyClass : public QObject
{
    Q_OBJECT
　　Q_PROPERTY(int Member1 READ Member1 WRITE setMember1 )
　　Q_PROPERTY(int Member2 READ Member2 WRITE setMember2 )
　　Q_PROPERTY(QString MEMBER3 READ Member3 WRITE setMember3 )
　　public:
　　    explicit MyClass(QObject *parent = 0);
　　signals:
　　public slots:
　　public:
　　　 Q_INVOKABLE int Member1();
　　　 Q_INVOKABLE int Member2();
　　　 Q_INVOKABLE QString Member3();
　　　 Q_INVOKABLE void setMember1( int mem1 );
　　　 Q_INVOKABLE void setMember2( int mem2 );
　　　 Q_INVOKABLE void setMember3( const QString& mem3 );
　　　 Q_INVOKABLE int func( QString flag );
　　private:
　　　 int m_member1;
　　　 int m_member2;
　　　 QString m_member3;
　};
```

大概过程是Qt利用基于moc(meta object compiler)实现，用一个元对象编译器在程序编译前，分析C++源文件，识别一些特殊的宏Q_OBJECT、Q_PROPERTY、Q_INVOKABLE……然后生成相应的moc文件，之后再一起全部编译链接。

  

# UE里UHT的方案

不用多说，你们也能想到UE当前的方案也是如此，实现在C++源文件中空的宏做标记，然后用UHT分析生成generated.h/.cpp文件，之后再一起编译。

  

```cpp
UCLASS()
class HELLO_API UMyClass : public UObject
{
	GENERATED_BODY()
public:
	UPROPERTY(BlueprintReadWrite, Category = "Test")
	float Score;

	UFUNCTION(BlueprintCallable, Category = "Test")
	void CallableFuncTest();

	UFUNCTION(BlueprintNativeEvent, Category = "Test")
	void NativeFuncTest();

	UFUNCTION(BlueprintImplementableEvent, Category = "Test")
	void ImplementableFuncTest();
};
```

这种方式的优点是能够比较小的对C++代码做修改，所要做的只是在代码里加一些空标记，并没有破坏原来的类声明结构，而且能够以比较友好的方式把元数据和代码关联在一起，生成的代码再复杂也可以对用户隐藏起来。一方面分析源码得力的话能够得到和编译器差不多的信息，还能通过自己的一些自定义标记来提供更多生成代码的指导。缺点是实现起来其实也是挺累人的，完整的C++的语法分析往往是超级复杂的，所以限制是自己写的分析器只能分析一些简单的C++语法规则和宏标记，如果用户使用比较复杂的语法时候，比如用#if /#endif包裹一些声明，就会让自己的分析器出错了，还好这种情况不多。关于多一次编译的问题，也可以通过自定义编译器的编译脚本UBT来规避。

如果是熟悉C#的朋友，一眼就能看出来这和C#的Attribute的语法简直差不多一模一样，所以UE也是吸收了C#语法反射的一些优雅写法，并利用上了C++的宏魔法，当然生成的代码里模板肯定也是少不了的。采取众长最后确定了这种类型信息搜集方案。

  

# 总结

本篇主要是解释了为何要以类型系统作为搭建Object系统的第一步，并描绘了C#语言里完善的类型系统看起来是什么样子，接着讨论了C++当前的RTTI工具，然后环顾一下当前C++业内的各种反射方案。知道别人家好的是什么样子，知道自己现在手里有啥，知道当前业内别人家是怎么尝试解决这个问题的，才能心中有数知道为何UE选择了目前的方案，知道UE的这套方案在业内算是什么水平。