**代理模式（Proxy）是 GoF 结构型模式之一：为另一个对象提供一个替身或占位符，以控制对它的访问。代理与真实对象实现同一个接口，客户端分不出面对的是哪一个；代理持有（或负责找到、创建）真实对象，在转发请求前后加入控制逻辑。**

> 参考：GoF《设计模式》Proxy 一章（动机示例是文档编辑器里延迟加载大图片的 `ImageProxy`）；refactoring.guru 的 Proxy 页面。代码沿用原笔记的 Java。

## 为什么要有代理

有些对象不适合让客户端直接碰。GoF 的例子是文档编辑器：一篇文档里嵌着很多大图片，打开文档时把所有图片从磁盘读进来会很慢，而大部分图片根本不在屏幕上。编辑器的排版代码又需要把图片当成普通图形对象对待（问它的尺寸、让它绘制）。解决办法是先放一个轻量的 `ImageProxy` 进文档：它知道文件名和尺寸，能回答排版的问题；只有真正被要求 `draw()` 时，才去加载真实的图片对象，然后把请求转交给它。

排版代码不需要为此做任何修改，因为代理和真实图片实现同一个接口。这就是代理的核心：**接口不变，访问方式变了**。至于“访问方式”怎么变，GoF 列了四种典型的代理：

| 代理类型 | 控制的是什么 | 例子 |
| --- | --- | --- |
| 虚拟代理（Virtual Proxy） | 创建时机：昂贵的对象延迟到真正使用时才创建 | 延迟加载的大图片、ORM 的懒加载关联对象 |
| 远程代理（Remote Proxy） | 位置：让客户端像调用本地对象一样调用另一地址空间的对象 | Java RMI 的 stub、gRPC 生成的客户端桩 |
| 保护代理（Protection Proxy） | 权限：检查调用者是否有权访问 | 只读视图、按角色拦截写操作 |
| 智能引用（Smart Reference） | 访问时附加的簿记工作 | 引用计数、首次访问时加锁、记录访问日志 |

refactoring.guru 还补充了缓存代理和日志代理，本质上都属于“在转发前后附加逻辑”。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Subject（抽象主题） | 真实对象和代理共同的接口 | `Shape` |
| RealSubject（真实主题） | 真正干活的对象 | `RealShape` |
| Proxy（代理） | 实现同一接口，持有真实对象的引用，控制对它的访问 | `ProxyShape` |
| Client（客户端） | 只通过 Subject 接口使用对象 | `ProxyDemo` |

一个重要细节：代理通常**自己管理真实对象的生命周期**，由它决定何时创建、是否创建。这是它和装饰器在实现上的主要区别，后面还会展开。

原笔记示例的类图，`ProxyShape` 延迟创建 `RealShape` 以减少加载时的内存占用：

![[代理模式-1.png]]

## Java 实现：虚拟代理与动态代理

第一部分沿用原笔记的形状示例，`ProxyShape` 是虚拟代理，第一次 `show()` 时才从“磁盘”加载真实对象。第二部分用 JDK 的 `java.lang.reflect.Proxy` 在运行时生成一个日志代理，不用为每个接口手写代理类。

```java
// ProxyDemo.java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

interface Shape {
    void show();
}

class RealShape implements Shape {
    private final String fileName;

    RealShape(String fileName) {
        this.fileName = fileName;
        loadFromDisk();                 // 构造就要付出加载代价
    }

    private void loadFromDisk() {
        System.out.println("Loading " + fileName);
    }

    public void show() {
        System.out.println("Showing " + fileName);
    }
}

// 虚拟代理：同一接口，延迟创建真实对象
class ProxyShape implements Shape {
    private final String fileName;
    private RealShape realShape;        // 由代理自己管理

    ProxyShape(String fileName) {
        this.fileName = fileName;
    }

    public void show() {
        if (realShape == null) {
            realShape = new RealShape(fileName);
        }
        realShape.show();
    }
}

public class ProxyDemo {
    // 动态代理：运行时为任意接口生成“先打日志再转发”的代理
    @SuppressWarnings("unchecked")
    static <T> T withLogging(T target, Class<T> iface) {
        InvocationHandler handler = (proxy, method, args) -> {
            System.out.println("[log] 调用 " + method.getName());
            return method.invoke(target, args);
        };
        return (T) Proxy.newProxyInstance(iface.getClassLoader(), new Class<?>[] { iface }, handler);
    }

    public static void main(String[] args) {
        Shape shape = new ProxyShape("Rectangle.shape");
        System.out.println("代理已创建，尚未加载");
        shape.show();   // 第一次：加载 + 显示
        shape.show();   // 第二次：直接显示

        Shape logged = withLogging(new ProxyShape("Circle.shape"), Shape.class);
        logged.show();
    }
}
```

`ProxyShape` 不是线程安全的：两个线程同时第一次调用 `show()`，可能加载两次。真实场景里的虚拟代理要么加锁，要么用 Holder 惯用法之类的方式保证只初始化一次（参见 [[ACADA-单例模式（Singleton Pattern）]] 的双重检查锁）。动态代理只能代理接口；要代理没有接口的类，Spring 等框架会改用 CGLIB、ByteBuddy 这类字节码生成库创建子类。

## 代理、装饰器、适配器、外观的区别

这四个模式在代码形态上很像——都是一个对象包着另一个对象、把调用转过去——区别在于意图和接口：

| 模式 | 对外接口 | 意图 | 被包装对象由谁创建 |
| --- | --- | --- | --- |
| 代理 | 与被代理对象**相同** | 控制访问（何时、能否、在哪里） | 通常由代理自己创建或查找 |
| 装饰器 | 与被装饰对象**相同**（可扩展） | 动态叠加新职责，可层层嵌套 | 由客户端创建后传进来 |
| 适配器 | 与被适配对象**不同** | 转换接口，让不兼容的类能协作 | 客户端或适配器 |
| 外观 | 一个**新的、更简单**的接口 | 简化一整个子系统的使用 | 外观自己持有多个子系统对象 |

原笔记的两句总结“适配器改变接口，代理不改变接口”“装饰器用于增强功能，代理用于控制访问”是准确的。refactoring.guru 对代理和装饰器补了一句很有用的判断标准：装饰器的组合由客户端控制，而代理通常自己管理服务对象的生命周期。实践中两者的边界确实模糊，例如日志代理和日志装饰器写出来几乎一样，这时按“是否让客户端自由叠加”来区分即可。

## 真实例子

Java 生态里代理无处不在：`java.lang.reflect.Proxy` 生成的动态代理是 Spring AOP（接口场景）的基础，`@Transactional` 方法就是被一个在调用前后开启/提交事务的代理包住的；Hibernate/JPA 的懒加载实体是虚拟代理，访问未初始化的关联时才发 SQL；Java RMI 的客户端 stub 是远程代理。

虚幻引擎里有不少名字带 Proxy 的类型，比如渲染用的 `FPrimitiveSceneProxy`（`UPrimitiveComponent::CreateSceneProxy()` 创建，是组件在渲染线程上的镜像）。它和 GoF 代理不是一回事：它不实现和组件相同的接口，也不替组件转发调用，而是为了跨线程把渲染所需数据复制一份。读引擎源码时不要看到 Proxy 就套用这个模式。

## 容易踩的坑

**代理泄漏了真实对象。** 如果代理提供了 `getReal()` 之类的方法，或者真实对象的方法返回 `this`，客户端就能绕过代理，保护代理的权限检查形同虚设。

**自调用绕过代理。** 在 Spring 里，一个 Bean 的方法 A 内部调用自己的方法 B，调用的是 `this.B()` 而不是代理的 `B()`，B 上的事务、缓存注解都不会生效。这是基于代理的 AOP 最常见的坑。

**懒加载代理在上下文关闭后被访问。** ORM 的懒加载代理在会话关闭后才第一次访问，会抛异常（Hibernate 的 `LazyInitializationException`）。

**虚拟代理没考虑并发。** 见上面代码后的说明。

**过多的间接层。** 每一层代理都增加一次调用和一层调试堆栈，远程代理更会把网络延迟藏在一个看起来很普通的方法调用后面，调用方容易在循环里无意识地发起大量远程调用。

## 相关

[[ACAEF-装饰器模式（Decorator Pattern）]] [[ACAEC-适配器模式（Adapter Pattern）]] [[ACAED-外观模式（Facade Pattern）]] [[ACADA-单例模式（Singleton Pattern）]] [[ACAEE-享元模式（Flyweight Pattern）]]
