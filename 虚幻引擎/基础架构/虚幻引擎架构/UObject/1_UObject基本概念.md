在介绍[[Actor实体|AActor]]的时候有简单介绍UObject。![[UObject基本概念-1.png]]
大部分的游戏引擎底层都是C++，而C++作为一个下接操作系统硬件底层，上接用户逻辑的编程语言，为了适应各种环境，不为你不需要的东西付代价，C++是并没有提供原生GC的。STL库的那些智能指针更多只是在C++的语言层面上再提供一些小辅助。
因此引入一个Object的根基类设计产生的好处有：
1. 万物可追踪。有了一个统一基类Object，我们就可以根据一个object类型指针追踪到所有的派生对象。
2. 通用的属性和接口。得益于继承机制，object里可以加上我们想应用于所有对象的属性和接口，包括但不限于：Equals、Clone、GetHashCode、ToString、GetName、GetMetaData等等。代码只要写一遍，所有的对象就都可以应用上了。
3. 统一的内存分配释放。用引用计数方案的话，你可以在Object上添加Retain+1/Release-1的接口；用GC的方案，也有了一个统一Object可以引用。所以这也是为何几乎所有支持GC的语言都会设计出来一个Object基类。
4. 统一的序列化模型。利用Object上继承机制统一将序列化代码写入Object。而如果再加上设计良好的反射机制，实现序列化就更加的方便了。
5. 统计功能。在有了可追踪和统一接口的基础上，能方便的实现统计程序使用情况。
6. 调试的便利。比如对于一块泄漏了的内存数据，如果是多类型对象，你可能压根没法知道它是哪个对象。但是如果你知道它是Object基类下的一个子类对象，你可以把地址转换为一个Object指针，然后就可以一目了然的查看对象属性了。
7. 为反射提供便利。如果没有一个统一Object，很难为各种对象实现GetType接口，否则就得在每个子类里都定义实现一遍，用宏也只是稍微缓解治标不治本。
8. UI编辑的便利。和编辑器集成的时候，为了让UI的属性面板控件能编辑各种对象。不光需要反射功能的支持，还需要引用一个统一Object指针。否则想象一下如果用一个void* Object，你还得额外添加一个ObjectType枚举用来转换成正确类型的C++对象，而且只能支持特定类型的C++类型对象。
同时也会带来弊端：
1. 臃肿的Object。如果需要Object所有对象提供额外功能，Object里就需要堆积大量的函数接口和成员属性。久而久之，这个Object身上就挂满了各种代码，可理解性就大大降低。而UE在原生的的C++基础上开始搭建这么一套系统，形成了如今这么臃肿的UObject了，大几十个接口，很少有人能全部掌握。
2. 不必要的内存负担。有时候有些属性并不是所有对象都用的到，但是因为不确定，为了所有对象在需要的时候就可以有，不得不放在Object里面。
3. 多重继承的限制。比如C多重继承于A和B，以前A和B都不是Object的时候还好，虽然大家对C++里的多重继承不太推荐使用，但是基本上也是不会有大的使用问题的。然后现在A和B都继承于Object了，现在让C想多重继承于A和B，就得面临一个尴尬的局面，变成菱形继承了！而甭管用不用得上全部用虚继承显然也是不靠谱的。所以一般有object基类的编程语言，都是直接限制多重继承，改为多重实现接口，避免了数据被继承多份的问题。
4. 类型系统的割裂。除非是像java和C#那样，对用户隐藏整个背后系统，否则用户在面对原生C++类型和Object类型时，就不得不去思考划分对象类型。两套系统在交叉引用、互相加载释放、消息通信、内存分配时采用的机制和规则也是大不一样的。哪些对象应该继承于Object，哪些不用；哪些可以GC，哪些只能用智能指针管理；C++对象里new了Object对象该怎么管理，Object对象里new了C++对象什么时候释放？
