**“工厂模式”是一组创建型做法的统称：把 `new 具体类` 从使用对象的代码里拿出去，交给专门负责创建的代码。通常分三种——简单工厂（一个根据参数决定造什么的类，不属于 GoF 23 种）、工厂方法（GoF 创建型模式，由子类决定实例化哪个类）和抽象工厂（GoF 创建型模式，创建一族相关对象，见 [[ACADB-抽象工厂模式（Abstract Factory Pattern）]]）。本条目讲前两种。**

> 参考：GoF《设计模式》Factory Method 一章（其动机示例是 `Application` / `Document` 框架）；refactoring.guru 的 Factory Method 页面及其对“简单工厂”的说明。代码沿用原笔记的 Java。

## 为什么要有工厂

直接写 `new Circle()` 的代码同时知道了两件事：它要一个“形状”，以及这个形状的具体类是 `Circle`。后一件事一旦需要变化——根据配置文件决定画哪种形状、给测试换一个假实现、以后加一种 `Triangle`——所有写了 `new Circle()` 的地方都得改。工厂的思路是让使用方只依赖抽象的 `Shape` 接口，“具体造哪个类”这个决定集中到一处。

三种工厂的差别在于“这一处”放在哪里：

| 形式 | 决定具体类的地方 | 扩展新产品时 | 是否 GoF 模式 |
| --- | --- | --- | --- |
| 简单工厂 | 一个工厂类里的 `if/switch` | 改工厂类的条件分支 | 否，是一种编程习惯 |
| 工厂方法 | 子类覆写的一个创建方法 | 新增一个子类，不改旧代码 | 是 |
| 抽象工厂 | 具体工厂对象（含多个创建方法） | 新增一族产品时新增一个工厂类 | 是 |

原笔记的示例 `ShapeFactory.getShape("CIRCLE")` 是简单工厂：调用方传字符串，工厂内部用 `if` 链返回 `Circle`、`Rectangle` 或 `Square`。它已经实现了“使用方不写 `new`”，但每加一种形状都要回去改 `getShape` 的分支，违反开闭原则。refactoring.guru 专门说明：简单工厂只是“一个带大量条件判断的创建方法”，经常被误当成工厂方法模式。

## 工厂方法的结构

GoF 对工厂方法的定义是：定义一个用于创建对象的接口，让子类决定实例化哪一个类，使一个类的实例化延迟到其子类。关键在于创建方法是**创建者类的一个可覆写方法**，而创建者类本身还有别的业务逻辑会调用这个方法。

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Product（产品） | 工厂方法所创建对象的接口 | `Shape` |
| ConcreteProduct（具体产品） | 实现产品接口 | `Circle`、`Square` |
| Creator（创建者） | 声明工厂方法 `createShape()`，并在自己的业务方法里调用它 | `ShapeTool` |
| ConcreteCreator（具体创建者） | 覆写工厂方法，返回具体产品 | `CircleTool`、`SquareTool` |

原笔记示例的类图（简单工厂版本）：

![[工厂模式-1.png]]

GoF 书里的动机例子很典型：一个应用框架的 `Application` 类负责打开、保存文档，但它不知道具体应用的文档是什么类型，于是把 `CreateDocument()` 声明为工厂方法，由绘图应用、文本应用各自的 `Application` 子类返回自己的 `Document`。框架代码 `OpenDocument()` 里调用 `CreateDocument()`，从头到尾只面对抽象的 `Document`。

## Java 实现：从简单工厂到工厂方法

下面的例子先给出原笔记那样的简单工厂（改用 `switch`，并对未知类型抛异常而不是返回 `null`），再用工厂方法重写：一个绘图“工具”类负责“创建形状并画出来”，具体创建哪种形状由子类决定。

```java
// FactoryDemo.java
interface Shape {
    void draw();
}

class Circle implements Shape {
    public void draw() { System.out.println("Circle::draw()"); }
}

class Rectangle implements Shape {
    public void draw() { System.out.println("Rectangle::draw()"); }
}

class Square implements Shape {
    public void draw() { System.out.println("Square::draw()"); }
}

// 简单工厂：集中的条件分支，新增形状要改这里
class ShapeFactory {
    public static Shape getShape(String type) {
        switch (type.toUpperCase()) {
            case "CIRCLE":    return new Circle();
            case "RECTANGLE": return new Rectangle();
            case "SQUARE":    return new Square();
            default: throw new IllegalArgumentException("未知形状: " + type);
        }
    }
}

// 工厂方法：Creator 自己有业务逻辑，创建步骤留给子类
abstract class ShapeTool {
    // 工厂方法
    protected abstract Shape createShape();

    // 业务方法只依赖 Shape 抽象
    public final Shape place(int x, int y) {
        Shape shape = createShape();
        System.out.print("在 (" + x + ", " + y + ") 放置: ");
        shape.draw();
        return shape;
    }
}

class CircleTool extends ShapeTool {
    protected Shape createShape() { return new Circle(); }
}

class SquareTool extends ShapeTool {
    protected Shape createShape() { return new Square(); }
}

public class FactoryDemo {
    public static void main(String[] args) {
        // 简单工厂
        ShapeFactory.getShape("circle").draw();
        ShapeFactory.getShape("RECTANGLE").draw();

        // 工厂方法：新增形状只需新增一个 ShapeTool 子类
        ShapeTool[] tools = { new CircleTool(), new SquareTool() };
        for (ShapeTool tool : tools) {
            tool.place(10, 20);
        }
    }
}
```

对比两种写法：简单工厂里“造什么”由参数决定，是运行时的条件判断；工厂方法里“造什么”由选用哪个子类决定，是类型层面的扩展。后者的代价是每种产品多一个创建者子类，产品种类多而创建逻辑又很简单时，这些子类会显得啰嗦，这时简单工厂或“类型到构造函数的映射表”（如 `Map<String, Supplier<Shape>>`）更实用。

## 工厂方法、抽象工厂与建造者的区别

| 模式 | 一次造出 | 变化点 | 客户端怎么用 |
| --- | --- | --- | --- |
| 工厂方法 | 一个产品 | 子类覆写一个方法 | 调用创建者的业务方法，内部触发工厂方法 |
| 抽象工厂 | 一族相关产品（多个方法） | 换一个工厂对象 | 持有工厂接口，调用 `createX()`、`createY()` |
| 建造者 | 一个复杂产品，分多步组装 | 换一个建造者，或改变步骤 | 逐步调用 `buildPart()`，最后 `getResult()` |

refactoring.guru 把三者描述为一条演化路径：很多设计从工厂方法起步，需要“成套”时演化成抽象工厂，需要“分步、可配置”时演化成建造者。抽象工厂的每个创建方法往往就是一个工厂方法。

## 真实例子

Java 集合框架里，`Collection.iterator()` 是工厂方法：`Collection` 声明它，`ArrayList`、`HashSet` 各自返回自己的迭代器实现，遍历代码只面对 `Iterator` 接口（GoF 在迭代器一章也提到了用工厂方法创建迭代器）。注意 `Calendar.getInstance()`、`List.of()` 这类“静态工厂方法”是《Effective Java》Item 1 讨论的另一回事：它们是替代构造函数的静态方法，不依赖子类覆写，和 GoF 的工厂方法模式不是同一个概念。

虚幻引擎编辑器里的资产工厂 `UFactory`（`Editor/UnrealEd`，头文件 `Factories/Factory.h`）是“一个类型一个工厂”的体系：每个子类用 `SupportedClass` 声明自己能造的资产类，覆写 `FactoryCreateNew()`（新建）或 `FactoryCreateFile()`（从文件导入），比如引擎里的 `UTexture2DFactoryNew`、`UMaterialInstanceConstantFactoryNew`。内容浏览器的“新建资产”菜单只和 `UFactory` 基类打交道，新增一种资产类型就新增一个工厂子类。

## 容易踩的坑

**把简单工厂叫成工厂方法。** 面试和文档里经常混用，但两者的扩展方式完全不同：一个改分支，一个加子类。

**工厂返回 `null`。** 原笔记示例里未知类型返回 `null`，调用方一 `draw()` 就是空指针，而且报错位置离真正的错误很远。未知类型应当直接抛异常。

**为了“解耦”给每个简单对象都配工厂。** 对象只有一种实现、构造也很简单时，工厂只是多一层间接。原笔记里的提醒是对的：能直接 `new` 就不必用工厂。

**字符串类型标识散落各处。** `"CIRCLE"` 这类魔法字符串拼错了只能在运行时发现，可以换成枚举或 `Class<? extends Shape>`。

## 相关

[[ACADB-抽象工厂模式（Abstract Factory Pattern）]] [[ACADD-建造者模式（Builder Pattern）]] [[ACADE-原型模式（Prototype Pattern）]] [[ACADA-单例模式（Singleton Pattern）]] [[ACAFH-模板模式（Template Pattern）]] [[ACAFC-迭代器模式（Iterator Pattern）]]
