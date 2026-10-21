**模板模式（GoF 中的正式名称是模板方法 Template Method）是 GoF 行为型模式之一：在父类的一个方法中定义算法的骨架，把其中某些步骤延迟到子类实现，使子类可以在不改变算法结构的前提下重新定义算法的某些特定步骤。骨架方法称为模板方法，通常声明为 `final`；被延迟的步骤是抽象方法或可选覆写的钩子方法。**

> 参考：GoF《设计模式》Template Method 一章（动机示例是应用框架中 `Application::OpenDocument()` 固定打开文档的流程，由子类实现 `CanOpenDocument`、`DoCreateDocument` 等步骤）；refactoring.guru 的 Template Method 页面。代码沿用原笔记的 Java。

## 为什么要有模板方法

几个类在做“流程相同、细节不同”的事情时，最容易出现复制粘贴：每个类都写一遍“先检查、再打开、再读取、最后通知”，只有“读取”这一步不一样。流程一旦要调整（比如在读取前加一步权限检查），就得去每个类里改一遍，漏掉一个就是 bug。

GoF 的例子是应用框架打开文档：`OpenDocument()` 的步骤是固定的——检查文档能否打开、创建文档对象、把它加入文档列表、读入内容。框架知道这个顺序，但不知道具体应用的文档怎么创建、怎么读。于是框架在父类里写好 `OpenDocument()`，把“能否打开”“如何创建”等步骤声明为抽象操作，由每个具体应用的子类实现。

这就是模板方法的核心分工：

- **不变的部分**（流程顺序、公共步骤）写在父类里，只写一次；
- **会变的部分**（个别步骤的具体实现）留给子类；
- **流程的控制权在父类**：子类只填空，不决定什么时候被调用。

最后一点常被称为“好莱坞原则”——别调用我们，我们会调用你（Don't call us, we'll call you），GoF 在这一章也引用了这个说法。它是框架与类库的根本区别：用类库时是你的代码调用库；用框架时是框架的模板方法调用你覆写的方法。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| AbstractClass（抽象类） | 定义模板方法（骨架）和各步骤的声明；可提供公共步骤的默认实现 | `Shape` |
| ConcreteClass（具体类） | 实现抽象步骤，按需覆写钩子 | `Rectangle`、`Square` |

模板方法里调用的步骤按“子类是否必须实现”分三类：

| 步骤类型 | 声明方式 | 子类 |
| --- | --- | --- |
| 抽象操作 | `abstract` 方法 | 必须实现 |
| 具体操作 | 父类中的普通方法（可 `private` 或 `final`） | 不能改，是流程里固定的公共步骤 |
| 钩子（hook） | 父类中有默认实现（常常是空的或返回默认值）的 `protected` 方法 | 可选覆写，用来在固定点插入行为或影响流程分支 |

GoF 特别强调要明确区分哪些是必须覆写的抽象操作、哪些是可选的钩子，否则子类作者不知道该覆写什么。

原笔记示例的类图：

![[模板模式-1.png]]

## Java 实现：带钩子的形状演示流程

在原笔记“显示 → 移动 → 旋转”的模板上，加入一个父类实现的公共步骤和一个决定是否旋转的钩子：

```java
// TemplateDemo.java
abstract class Shape {
    // 模板方法：final，子类不能改变流程
    public final void play() {
        prepare();                 // 具体操作：公共步骤
        show();                    // 抽象操作
        move();
        if (shouldRotate()) {      // 钩子影响流程分支
            rotate();
        }
        afterPlay();               // 钩子：默认什么也不做
    }

    private void prepare() {
        System.out.println("[" + getClass().getSimpleName() + "] 准备画布");
    }

    protected abstract void show();
    protected abstract void move();
    protected abstract void rotate();

    protected boolean shouldRotate() { return true; }
    protected void afterPlay() {}
}

class Rectangle extends Shape {
    protected void show()   { System.out.println("Rectangle Show"); }
    protected void move()   { System.out.println("Rectangle Move"); }
    protected void rotate() { System.out.println("Rectangle Rotate"); }
}

class Circle extends Shape {
    protected void show()   { System.out.println("Circle Show"); }
    protected void move()   { System.out.println("Circle Move"); }
    protected void rotate() { throw new IllegalStateException("圆旋转没有意义，不应被调用"); }

    // 覆写钩子：圆不需要旋转
    @Override
    protected boolean shouldRotate() { return false; }

    @Override
    protected void afterPlay() { System.out.println("Circle 播放结束，记录日志"); }
}

public class TemplateDemo {
    public static void main(String[] args) {
        Shape shape = new Rectangle();
        shape.play();
        System.out.println();
        shape = new Circle();
        shape.play();
    }
}
```

`play()` 声明为 `final`，是为了防止子类覆写整个流程而绕过公共步骤；步骤方法声明为 `protected` 而不是 `public`，是因为它们只应由模板方法调用，外部直接调用 `move()` 会跳过准备步骤。原示例里步骤方法是包级可见，同包的任何代码都能单独调用它们。`Circle` 通过钩子跳过了旋转，但仍然被迫实现了 `rotate()`，这暴露了模板方法的一个局限：父类定下的抽象步骤，所有子类都得实现，哪怕对它没有意义。

## 模板方法和相近模式的区别

| 对比 | 模板方法 | 另一方 |
| --- | --- | --- |
| vs 策略 | 基于继承，编译期确定，改变算法的**一部分** | 基于组合，运行时可替换，替换**整个**算法；refactoring.guru 用“静态 vs 动态”概括两者 |
| vs 工厂方法 | 工厂方法常常是模板方法中的一个步骤：骨架里有一步“创建对象”，交给子类决定 | GoF 的 `OpenDocument` 里 `DoCreateDocument` 就是工厂方法 |
| vs 建造者 | 父类定步骤顺序，子类实现步骤 | Director 定步骤顺序，Builder 实现步骤；前者是继承，后者是组合 |

当子类之间的差异越来越大、钩子越来越多时，模板方法往往会被重构成策略：把各个步骤抽成独立的接口，由组合代替继承。

## 真实例子

`java.util.AbstractList` 是模板方法的标准示范：子类只需实现 `get(int)` 和 `size()`，`indexOf`、`contains`、`iterator`、`equals`、`hashCode` 等都已经在父类中基于这两个方法实现好了；需要可修改列表时再覆写 `set`、`add`、`remove`。`java.io.InputStream` 的 `read(byte[], int, int)` 由抽象的单字节 `read()` 实现，子类可以为了性能覆写批量版本。Servlet 的 `HttpServlet.service()` 按 HTTP 方法分派到 `doGet`、`doPost` 等，开发者只覆写需要的那几个。

虚幻引擎的 Actor 生命周期体现了同样的“好莱坞原则”：引擎按固定的顺序在合适的时机调用 `BeginPlay`、`Tick`、`EndPlay` 这些虚函数，游戏代码在子类中覆写它们来插入自己的逻辑，并按惯例调用 `Super::BeginPlay()` 保持父类行为。这些虚函数更接近模板方法里的“钩子”，而驱动它们的流程分散在引擎的世界初始化和 Tick 调度代码里，不是某一个具体的模板方法。

## 容易踩的坑

**子类覆写钩子时忘了调用父类实现。** 当父类的钩子本身有逻辑时，子类覆写后不调用 `super`，父类的行为就丢了。UE 里忘写 `Super::BeginPlay()` 导致组件没初始化是典型例子。

**模板方法没有 `final`。** 子类覆写了整个模板方法，模板就失去意义。

**步骤方法可见性太高。** 外部可以绕过模板方法单独调用步骤，破坏流程顺序。

**抽象步骤太多。** 每个子类都要实现一长串方法，其中不少对它没意义（见 `Circle.rotate`）。可以把一部分改成带默认实现的钩子，或者重新审视这些子类是否真的属于同一个流程。

**继承层次太深。** 模板方法层层套模板方法，读代码时要在多层父类之间来回跳才能看清完整流程。原笔记提到的“类数目增加”也是这个原因。

## 相关

[[ACAFB-策略模式（Strategy Pattern）]] [[ACADC-工厂模式（Factory Pattern）]] [[ACADD-建造者模式（Builder Pattern）]] [[ACAFK-状态模式（State Pattern）]]
