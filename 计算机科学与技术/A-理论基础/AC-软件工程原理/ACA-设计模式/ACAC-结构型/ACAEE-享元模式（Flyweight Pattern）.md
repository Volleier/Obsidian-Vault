**享元模式（Flyweight）是 GoF 结构型模式之一：运用共享技术有效地支持大量细粒度对象。做法是把对象状态拆成可共享的内部状态（intrinsic）和随场景变化的外部状态（extrinsic），内部状态放进少量共享的享元对象，外部状态由客户端保存并在调用时传入；享元工厂负责按键复用已有的享元。**

> 参考：GoF《设计模式》Flyweight 一章（动机示例是文档编辑器把每个字符当成对象，字符对象按字符编码共享）；refactoring.guru 的 Flyweight 页面；《Java 语言规范》§5.1.7 装箱转换与 `Integer.valueOf` 的 Javadoc。代码沿用原笔记的 Java。

## 为什么要有享元

面向对象设计有时会自然地走向“一切皆对象”：文档编辑器里每个字符是一个对象，游戏里每颗子弹、每棵树、每个粒子是一个对象。这样建模最清晰，但对象一多，内存扛不住。GoF 的例子里，一篇文档有几十万个字符，如果每个字符对象都自己存一份字体、字形数据，内存会被大量重复的数据占满。

仔细看这些对象会发现，它们的状态里有两类东西：

| 状态 | 特点 | 字符例子 | 树木例子 |
| --- | --- | --- | --- |
| 内部状态（intrinsic） | 不随使用场景变化，可被许多对象共享；存放在享元内部 | 字符编码、字形 | 网格、贴图、树种名 |
| 外部状态（extrinsic） | 随场景变化，不能共享；由客户端保存，调用时传给享元 | 在文档中的位置、所在行 | 位置、缩放、旋转 |

字母 `a` 在文档里出现一万次，它的字形数据完全一样，不一样的只有位置。享元模式就是让这一万处都引用同一个 `a` 对象，位置由排版结构另外记录，绘制时再传进去。对象数量从“字符出现次数”降到“字符种类数”，内存占用随之下降。

代价是设计上更复杂：必须把状态切干净，外部状态要由别人保存和传递；而且享元一旦被共享，就绝不能再被某个使用者修改。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Flyweight（抽象享元） | 声明接受外部状态作为参数的操作 | `Shape`（`draw(x, y, radius)`） |
| ConcreteFlyweight（具体享元） | 保存内部状态，必须可共享（通常不可变） | `Circle`（只存颜色） |
| UnsharedConcreteFlyweight（非共享享元，可选） | 实现同一接口但不参与共享 | — |
| FlyweightFactory（享元工厂） | 按键创建并缓存享元，保证同键只有一个实例 | `ShapeFactory` |
| Client（客户端） | 保存外部状态，从工厂获取享元并传入外部状态 | `FlyweightDemo` |

原笔记示例的类图，`ShapeFactory` 用 `HashMap` 以颜色为键缓存 `Circle`：

![[享元模式-1.png]]

原示例有一个很典型的错误：`Circle` 内部既存颜色（内部状态），又用 `setX`、`setY`、`setRadius` 存了坐标和半径（外部状态）。客户端拿到共享的圆之后改它的坐标，等于修改了所有使用这个红色圆的地方。原示例之所以看不出问题，是因为它每次改完立刻 `draw()`，没有两个使用者同时持有同一个圆。一旦把圆存进列表稍后统一绘制，所有红色的圆都会画在最后一次设置的位置上。下面的实现把外部状态改成 `draw` 的参数。

## Java 实现：只共享颜色的圆

```java
// FlyweightDemo.java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

// 抽象享元：外部状态通过参数传入
interface Shape {
    void draw(int x, int y, int radius);
}

// 具体享元：只保存内部状态，且不可变
final class Circle implements Shape {
    private final String color;

    Circle(String color) {
        this.color = color;
        System.out.println("Creating circle of color : " + color);
    }

    public void draw(int x, int y, int radius) {
        System.out.println("Circle[" + color + "] at (" + x + "," + y + ") r=" + radius);
    }
}

// 享元工厂：同一颜色只创建一次
class ShapeFactory {
    private static final Map<String, Shape> CIRCLES = new HashMap<>();

    static Shape getCircle(String color) {
        return CIRCLES.computeIfAbsent(color, Circle::new);
    }

    static int count() {
        return CIRCLES.size();
    }
}

// 客户端保存外部状态
class CircleInstance {
    final Shape flyweight;
    final int x, y, radius;

    CircleInstance(Shape flyweight, int x, int y, int radius) {
        this.flyweight = flyweight;
        this.x = x;
        this.y = y;
        this.radius = radius;
    }

    void draw() {
        flyweight.draw(x, y, radius);
    }
}

public class FlyweightDemo {
    public static void main(String[] args) {
        String[] colors = { "Red", "Green", "Blue" };
        List<CircleInstance> scene = new ArrayList<>();

        for (int i = 0; i < 9; i++) {
            String color = colors[i % colors.length];
            scene.add(new CircleInstance(ShapeFactory.getCircle(color), i * 10, i * 5, 100));
        }

        // 先全部创建、后统一绘制，位置也不会串
        for (CircleInstance c : scene) {
            c.draw();
        }
        System.out.println("场景中 " + scene.size() + " 个圆，享元对象 " + ShapeFactory.count() + " 个");
    }
}
```

`CircleInstance` 自己也是对象，看上去并没有省掉对象数量。省下的是**内部状态的重复**：这里的内部状态只有一个颜色字符串，效果不明显；换成网格、贴图、字形这种动辄几 KB 到几 MB 的数据，每个实例只剩一个引用加几个坐标，差距就是数量级的。如果连 `CircleInstance` 这层对象也嫌多，可以把外部状态存进几个并列的基本类型数组，这就走向了数据导向的设计。

`HashMap` 在多线程下不安全，工厂被并发调用时要换成 `ConcurrentHashMap`，它的 `computeIfAbsent` 能保证同一个键只创建一次。

## 享元和相近模式的区别

| 对比 | 享元 | 另一方 |
| --- | --- | --- |
| vs 单例 | 每种内部状态一个实例，可以有很多个；实例不可变 | 整个类只有一个实例，通常可变 |
| vs 对象池 | 同一个享元被**同时**共享给多个使用者 | 池里的对象被借出时**独占**，用完归还再给别人 |
| vs 原型 | 大家引用同一个对象 | 每人复制一份独立的对象 |
| vs 组合模式 | 常配合使用：组合树的叶子节点做成享元（GoF 的字符例子就是这样） | 组合负责树结构本身 |

享元和对象池最容易混：两者都在“复用对象”，但享元复用的是**不可变的共享数据**，对象池复用的是**创建成本高的可变资源**（数据库连接、线程）。

## 真实例子

Java 的 `Integer.valueOf(int)` 会复用缓存的 `Integer` 对象，它的 Javadoc 写明总会缓存 -128 到 127 之间的值；《Java 语言规范》§5.1.7 也规定这个范围内的整数常量装箱后必须是同一个对象（`true`/`false` 和 `\u0000`～`\u007f` 的字符同理）。所以 `Integer a = 127, b = 127; a == b` 为 `true`，换成 128 就可能是 `false`——这也是“用 `==` 比较包装类型”这个经典 bug 的来源。`String.intern()` 和字符串字面量常量池是同样的思路。

游戏里享元最典型的场景是大量重复的物体。虚幻引擎中，关卡里一千个石头的 `UStaticMeshComponent` 都引用同一个 `UStaticMesh` 资产，网格数据只加载一份，每个组件只保存自己的变换和少量覆盖参数；`UInstancedStaticMeshComponent` 更进一步，一个组件里用一个网格加一组实例变换来表示所有实例，渲染时也能批量提交。Mass 框架的共享 Fragment（`FMassSharedFragment` / `FMassConstSharedFragment`）让一组实体共用同一份配置数据，也是内部状态共享的思路。

## 容易踩的坑

**享元可变。** 见上文对原示例的分析，这是享元最常见也最致命的错误。享元类应当是 `final` 的、字段都是 `final` 的。

**用 `==` 比较被缓存的对象。** 在缓存范围内“碰巧”相等，范围外就不等，测试用小数字永远发现不了。

**外部状态的管理成本超过节省。** 如果内部状态本来就很小，拆出外部状态、到处传参数带来的复杂度得不偿失。享元只在“对象数量巨大且内部状态占比高”时才划算。

**工厂缓存无限增长。** 键的种类不受控（比如用任意用户输入当键）时，缓存会变成内存泄漏，需要限制大小或使用弱引用。

## 相关

[[ACAEG-组合模式（Composite Pattern）]] [[ACADA-单例模式（Singleton Pattern）]] [[ACADE-原型模式（Prototype Pattern）]] [[ACADC-工厂模式（Factory Pattern）]] [[IBA-实体-组件-系统 ECS]]
