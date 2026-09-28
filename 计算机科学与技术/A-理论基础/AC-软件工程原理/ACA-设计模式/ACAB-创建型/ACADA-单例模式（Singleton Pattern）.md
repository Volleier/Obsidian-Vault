**单例模式（Singleton）是 GoF 23 种设计模式中的一种创建型模式：保证一个类在进程里只有一个实例，并由这个类自己提供访问该实例的全局入口。典型写法是私有构造函数 + 静态的 `getInstance()`。**

> 参考：GoF《设计模式：可复用面向对象软件的基础》（1994）Singleton 一章；refactoring.guru 的 Singleton 页面；Joshua Bloch《Effective Java》第 3 版 Item 3（用私有构造器或枚举类型强化单例属性）。原笔记的代码是 Java，本文沿用 Java。

## 为什么要有单例

有些东西在一个程序里天然只该有一份：进程的运行时环境、全局配置、日志系统、线程池、某个硬件设备的驱动句柄。如果任由调用方各自 `new`，就会出现两份配置互相覆盖、两个对象同时写同一个文件、同一块资源被重复初始化这类问题。直接用全局变量也能“只有一份”，但全局变量既管不住别人再造一个，也没法把创建推迟到第一次使用时。

单例把两件事绑在一起解决：一是**控制实例数量**，构造函数设成私有，外部根本写不出 `new Singleton()`；二是**提供访问入口**，所有人通过同一个静态方法拿到同一个对象。refactoring.guru 特意指出，单例同时解决了这两个问题，所以它违反单一职责原则——一个类既要做自己的业务，又要管理自己的生命周期。这是单例被批评最多的地方，也是后面“容易踩的坑”的根源。

## 结构

单例是 GoF 里结构最简单的模式，参与者只有一个类，但这个类身上有几个固定部件：

| 部件 | 作用 |
| --- | --- |
| 私有构造函数 | 阻止外部直接 `new`，这是“只有一个”的前提 |
| 私有静态字段 `instance` | 保存唯一实例 |
| 公有静态方法 `getInstance()` | 全局访问点；负责在需要时创建实例 |
| 线程安全机制 | 多线程下保证只创建一次（类加载、`synchronized`、`volatile` 等） |

原笔记示例中，`SingleObject` 提供静态方法供外界获取它的唯一实例，`SingletonPatternDemo` 通过它拿对象并调用 `showMessage()`：

![[单例模式-1.png]]

## Java 里的几种写法

单例难写的部分不在结构，而在“什么时候创建”和“多线程下会不会创建两次”。Java 社区里流传的六种写法可以按这两点排开：

| 写法 | 延迟初始化 | 线程安全 | 说明 |
| --- | --- | --- | --- |
| 懒汉式（无锁） | 是 | 否 | `if (instance == null) instance = new ...`，两个线程可能同时通过判断，严格说不算单例 |
| 懒汉式（`synchronized` 方法） | 是 | 是 | 每次调用都要拿锁，而真正需要同步的只有第一次 |
| 饿汉式 | 否 | 是 | 静态字段直接初始化，靠类加载机制保证只执行一次；类被任何原因加载时都会创建 |
| 双重检查锁（DCL） | 是 | 是（JDK 5 起） | 字段必须声明为 `volatile`，否则可能拿到未构造完的对象 |
| 静态内部类（Holder） | 是 | 是 | 只有调用 `getInstance()` 时才加载内部类，兼顾延迟与无锁 |
| 枚举 | 否 | 是 | 《Effective Java》推荐；天然防反射、防反序列化产生第二个实例 |

饿汉式和静态内部类都依赖同一件事：JVM 保证一个类的静态初始化只执行一次，并且对其他线程可见。区别在于饿汉式的实例挂在外部类上，外部类只要因为别的静态方法被加载，实例就被创建了；Holder 写法把实例挂在一个只有 `getInstance()` 会碰到的内部类上，于是延迟到了第一次调用。

DCL 需要 `volatile` 的原因是 `instance = new Singleton()` 在字节码层面不是原子的：分配内存、执行构造函数、把引用写回字段，后两步可能被重排序。没有 `volatile` 时，另一个线程可能在第一次检查就看到非空引用，拿去用的却是还没构造完的对象。JDK 5 通过 JSR-133 修订内存模型后，`volatile` 才能禁止这种重排序，所以老资料里“DCL 是错的”和新资料里“DCL 可用”说的都对，差别在 JDK 版本。

下面的完整示例把三种推荐写法放在一起，并演示两次获取拿到的是同一个对象：

```java
// SingletonDemo.java
import java.util.concurrent.atomic.AtomicInteger;

// 写法一：静态内部类（延迟 + 无锁）
class ConfigRegistry {
    private static final AtomicInteger CREATED = new AtomicInteger();

    private ConfigRegistry() {
        CREATED.incrementAndGet(); // 统计构造次数，验证只创建一次
    }

    // Holder 只在 getInstance() 第一次被调用时加载
    private static class Holder {
        private static final ConfigRegistry INSTANCE = new ConfigRegistry();
    }

    public static ConfigRegistry getInstance() {
        return Holder.INSTANCE;
    }

    public static int createdCount() {
        return CREATED.get();
    }
}

// 写法二：双重检查锁，instance 必须是 volatile
class Logger {
    private static volatile Logger instance;

    private Logger() {}

    public static Logger getInstance() {
        Logger local = instance;          // 只读一次 volatile 字段
        if (local == null) {
            synchronized (Logger.class) {
                local = instance;
                if (local == null) {
                    local = new Logger();
                    instance = local;
                }
            }
        }
        return local;
    }

    public void log(String msg) {
        System.out.println("[LOG] " + msg);
    }
}

// 写法三：枚举，防反射、防反序列化
enum IdGenerator {
    INSTANCE;

    private final AtomicInteger next = new AtomicInteger(1);

    public int nextId() {
        return next.getAndIncrement();
    }
}

public class SingletonDemo {
    public static void main(String[] args) throws InterruptedException {
        // 多个线程同时取实例
        Thread[] threads = new Thread[8];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(() -> ConfigRegistry.getInstance());
            threads[i].start();
        }
        for (Thread t : threads) {
            t.join();
        }
        System.out.println("ConfigRegistry 构造次数: " + ConfigRegistry.createdCount());

        System.out.println("Logger 是同一个对象: " + (Logger.getInstance() == Logger.getInstance()));
        Logger.getInstance().log("hello");

        System.out.println("id: " + IdGenerator.INSTANCE.nextId() + ", " + IdGenerator.INSTANCE.nextId());
    }
}
```

输出中构造次数恒为 1。DCL 里先把字段读到局部变量 `local` 再判断，是为了在已初始化的常见路径上只读一次 `volatile`；直接写 `if (instance == null) ... return instance;` 也正确，只是多一次 volatile 读。

## 单例和相近做法的区别

| 做法 | 实例数量 | 能否多态替换 | 生命周期由谁管 |
| --- | --- | --- | --- |
| 单例 | 恰好一个 | 调用方通常直接依赖具体类，难替换 | 类自己 |
| 全静态工具类（如 `java.lang.Math`） | 没有实例 | 不能 | 无 |
| 依赖注入容器里的单例作用域（如 Spring 默认 Bean 作用域） | 容器内一个 | 能，调用方依赖接口 | 容器 |
| 享元工厂 | 每种内部状态一个 | 能 | 工厂 |

单例和静态类的根本区别是单例是个对象：它可以实现接口、可以作为参数传递、可以延迟创建。原笔记“没有接口，不能继承”的说法不准确，单例类完全可以实现接口；真正的问题是调用方习惯写 `Logger.getInstance()`，依赖的是具体类而不是接口，测试时换不成假对象。依赖注入容器的“单例作用域”保留了“只有一个”的好处，同时把“怎么拿到它”交给容器，调用方只看到接口，所以现代工程里通常更推荐这种方式。

## 真实例子

Java 标准库里的 `java.lang.Runtime` 是教科书式单例：构造函数私有，通过 `Runtime.getRuntime()` 取得当前进程唯一的运行时对象。

虚幻引擎的 **Subsystem** 是“由引擎管理生命周期的单例”：`UEngineSubsystem`、`UEditorSubsystem`、`UGameInstanceSubsystem`、`UWorldSubsystem`、`ULocalPlayerSubsystem` 的子类不需要手动创建，引擎会在对应宿主（引擎、编辑器、GameInstance、World、LocalPlayer）创建时自动实例化一份，宿主销毁时一起销毁，通过 `GetGameInstance()->GetSubsystem<UMySubsystem>()` 这类接口获取。它解决了手写单例在 UE 里最麻烦的问题：PIE 多次启动、多个 World 并存时，手写静态单例会跨 World 残留状态，而 Subsystem 的“唯一”是相对宿主而言的。

## 容易踩的坑

**把单例当全局变量用。** 任何地方都能 `getInstance()`，依赖关系就从函数签名里消失了，读代码时看不出谁改了它的状态。单元测试之间还会互相污染，因为状态跨测试保留。

**懒汉式忘了同步，或 DCL 忘了 `volatile`。** 单线程测试永远跑得通，问题只在高并发下偶发，很难复现。

**反射与反序列化产生第二个实例。** `Constructor.setAccessible(true)` 能调用私有构造函数；实现了 `Serializable` 的单例反序列化时会新建对象，需要提供 `readResolve()` 返回已有实例。枚举写法天然免疫这两种情况，JVM 不允许反射创建枚举对象。

**多个类加载器下“单例”不唯一。** 同一个类被两个类加载器加载，就是两个不同的类，各自有一个实例。应用服务器、插件系统里会遇到。

**在 UE 里用静态指针保存 UObject 单例。** 静态裸指针不会被 GC 追踪，对象可能被回收而指针还在；PIE 反复启动时旧状态也会残留。优先用 Subsystem。

## 相关

[[ACADB-抽象工厂模式（Abstract Factory Pattern）]] [[ACADC-工厂模式（Factory Pattern）]] [[ACADD-建造者模式（Builder Pattern）]] [[ACADE-原型模式（Prototype Pattern）]] [[ACAEE-享元模式（Flyweight Pattern）]] [[ACAED-外观模式（Facade Pattern）]]
