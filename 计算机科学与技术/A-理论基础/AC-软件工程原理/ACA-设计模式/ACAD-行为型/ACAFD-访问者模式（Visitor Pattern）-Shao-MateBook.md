**访问者模式（Visitor）是 GoF 行为型模式之一：表示一个作用于某对象结构中各元素的操作，使你可以在不改变各元素类的前提下定义作用于这些元素的新操作。实现上依靠“双分派”：元素的 `accept(visitor)` 回调 `visitor.visit(this)`，从而同时根据元素类型和访问者类型选中要执行的代码。**

> 参考：GoF《设计模式》Visitor 一章（动机示例是编译器对抽象语法树做类型检查、代码生成、美化打印等多种操作）；refactoring.guru 的 Visitor 页面；JDK `java.nio.file.FileVisitor` 的 API 文档。代码沿用原笔记的 Java。

## 为什么要有访问者

GoF 的例子是编译器。源代码被解析成一棵抽象语法树，节点类型有赋值节点、变量引用节点、表达式节点等。编译器要对这棵树做很多件事：类型检查、代码优化、代码生成、格式化打印、统计度量……最直接的做法是给每个节点类加上 `typeCheck()`、`generateCode()`、`prettyPrint()` 方法。结果是：

- 每加一种操作，就要修改**所有**节点类；
- 每个节点类里混杂了类型检查、代码生成等毫不相关的逻辑，一种操作的代码分散在十几个类里；
- 节点类是编译器的核心数据结构，频繁修改它们风险很大。

访问者的思路是把“操作”从元素类中搬出去：一种操作对应一个访问者类，里面为每种元素类型写一个 `visit` 方法。`TypeCheckingVisitor` 里集中了所有节点的类型检查逻辑，`CodeGeneratingVisitor` 里集中了所有节点的代码生成逻辑。元素类只需要提供一个 `accept(visitor)` 方法，以后加新操作不再碰元素类。

这个模式的前提是**元素类型稳定、操作经常增加**。反过来，如果经常增加新的元素类型，每个访问者都要加一个 `visit` 方法，访问者就成了负担。这一对取舍和面向对象、函数式两种风格的“表达式问题”（expression problem）是同一件事的两面。

## 双分派

为什么需要 `accept` 绕一圈，而不是直接写 `visitor.visit(element)`？因为 Java 的方法重载是在**编译期**按静态类型选择的。如果 `element` 的静态类型是接口 `ShapePart`，`visitor.visit(element)` 在编译时找不到对应 `visit(ShapePart)` 的重载（或者只能选中一个通用的版本），根本分辨不出它实际是 `Circle` 还是 `Square`。

`accept` 解决了这个问题。调用 `element.accept(visitor)` 时，先通过虚函数分派到 `Circle.accept`（第一次分派，按元素的运行时类型）；在 `Circle.accept` 里写 `visitor.visit(this)`，此时 `this` 的静态类型就是 `Circle`，编译器选中 `visit(Circle)` 这个重载，再通过虚函数分派到具体访问者的实现（第二次分派，按访问者的运行时类型）。两次分派合起来，执行的代码同时取决于元素类型和访问者类型，这就是双分派。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Visitor（访问者） | 为每个具体元素类声明一个 `visit` 操作 | `ShapePartVisitor` |
| ConcreteVisitor（具体访问者） | 实现一种操作在各元素上的行为，可累积状态 | `ShowVisitor`、`AreaVisitor` |
| Element（元素） | 声明 `accept(visitor)` | `ShapePart` |
| ConcreteElement（具体元素） | 实现 `accept`，回调 `visitor.visit(this)` | `Rectangle`、`Circle`、`Square` |
| ObjectStructure（对象结构） | 能枚举元素，可以是集合或组合树 | `Shape`（包含多个部件） |

原笔记示例的类图：

![[访问者模式-1.png]]

## Java 实现：两种操作，同一组元素

保留原笔记的形状部件结构，并在“显示”之外再加一个“求面积”的访问者，展示新增操作时元素类一行都不用改。原示例里接口名有 `ShapePartVisit`、`ShpaePartVisitor` 等拼写不一致，这里统一为 `ShapePartVisitor`：

```java
// VisitorDemo.java
interface ShapePartVisitor {
    void visit(Rectangle rectangle);
    void visit(Circle circle);
    void visit(Square square);
    void visit(Shape shape);
}

interface ShapePart {
    void accept(ShapePartVisitor visitor);
}

class Rectangle implements ShapePart {
    final int w = 4, h = 3;
    public void accept(ShapePartVisitor v) { v.visit(this); }   // this 的静态类型是 Rectangle
}

class Circle implements ShapePart {
    final int r = 2;
    public void accept(ShapePartVisitor v) { v.visit(this); }
}

class Square implements ShapePart {
    final int side = 5;
    public void accept(ShapePartVisitor v) { v.visit(this); }
}

// 对象结构：由多个部件组成，负责把访问者传给每个部件
class Shape implements ShapePart {
    private final ShapePart[] parts = { new Rectangle(), new Circle(), new Square() };

    public void accept(ShapePartVisitor v) {
        for (ShapePart p : parts) {
            p.accept(v);
        }
        v.visit(this);
    }
}

// 操作一：显示
class ShowVisitor implements ShapePartVisitor {
    public void visit(Rectangle r) { System.out.println("Showing Rectangle " + r.w + "x" + r.h); }
    public void visit(Circle c)    { System.out.println("Showing Circle r=" + c.r); }
    public void visit(Square s)    { System.out.println("Showing Square side=" + s.side); }
    public void visit(Shape s)     { System.out.println("Showing Shape."); }
}

// 操作二：求总面积，访问者可以在遍历中累积状态
class AreaVisitor implements ShapePartVisitor {
    private double total = 0;

    public void visit(Rectangle r) { total += r.w * r.h; }
    public void visit(Circle c)    { total += Math.PI * c.r * c.r; }
    public void visit(Square s)    { total += s.side * s.side; }
    public void visit(Shape s)     { /* 组合本身没有额外面积 */ }

    double getTotal() { return total; }
}

public class VisitorDemo {
    public static void main(String[] args) {
        ShapePart shape = new Shape();

        shape.accept(new ShowVisitor());

        AreaVisitor area = new AreaVisitor();
        shape.accept(area);
        System.out.printf("Total area = %.2f%n", area.getTotal());
    }
}
```

`AreaVisitor` 展示了访问者的另一个用处：它可以在遍历过程中累积结果，而不需要把累加器作为参数在元素之间传来传去。为了让访问者能计算面积，元素把 `w`、`h`、`r` 这些字段暴露给了同包的访问者——这正是访问者模式的代价之一，原笔记也提到了“元素需要向访问者公开其内部信息”。

## 访问者和相近模式的区别

| 对比 | 访问者 | 另一方 |
| --- | --- | --- |
| vs 迭代器 | 决定“对每种元素做什么” | 决定“按什么顺序走到每个元素”；两者常一起用 |
| vs 组合模式 | 对一棵组合树执行多种操作时，把操作移出组合类 | 组合模式本身把操作放在组件类里 |
| vs 策略 | 按元素类型分派到不同方法，处理的是一组异构对象 | 对同一类输入替换整个算法 |
| vs 解释器 | 解释器把求值写在每个表达式节点里；表达式种类稳定但操作多时，可改用访问者 | GoF 在解释器一章提到了这种替换 |
| vs 命令 | refactoring.guru 把访问者看作命令的加强版：它的对象可以对多种不同类的对象执行操作 | 命令通常只作用于一个接收者 |

## 真实例子

`java.nio.file.Files.walkFileTree(start, visitor)` 接受一个 `FileVisitor`，在遍历目录树时依次回调 `preVisitDirectory`、`visitFile`、`visitFileFailed`、`postVisitDirectory`；`SimpleFileVisitor` 提供了默认实现，只需覆写关心的方法。Java 注解处理 API 中的 `javax.lang.model.element.ElementVisitor` 用于遍历程序元素（类、方法、字段），是编译器场景下的经典访问者。

Java 21 正式引入的 `switch` 模式匹配（JEP 441）配合 `sealed` 接口，可以写 `switch (part) { case Circle c -> ...; case Square s -> ...; }`，编译器能检查是否覆盖了所有子类型。在元素类型封闭、只是想按类型分派的场景里，它可以替代经典访问者，省去 `accept` 这套样板代码。

## 容易踩的坑

**元素类型经常变化。** 每加一种元素，所有访问者接口和实现都要改，这时访问者是错误的选择。

**破坏元素的封装。** 访问者需要读取元素的内部数据，元素只好把字段或 getter 公开出来。

**忘了在具体元素里写 `accept`，或在基类里写一次就想复用。** `accept` 里的 `visitor.visit(this)` 必须在每个具体元素类中各写一遍，写在父类里时 `this` 的静态类型是父类，重载就选错了。

**访问者遍历时修改了结构。** 访问者在遍历过程中增删元素，会和对象结构的遍历逻辑冲突，和迭代器的“遍历中修改集合”是同一类问题。

**原笔记所说的“依赖具体类”。** 访问者接口为每个具体元素类写了一个 `visit` 方法，访问者因此依赖所有具体元素类，这是模式本身决定的，无法消除，只能靠元素类型稳定来抵消。

## 相关

[[ACAEG-组合模式（Composite Pattern）]] [[ACAFC-迭代器模式（Iterator Pattern）]] [[ACAFF-解释器模式（Interpreter Pattern）]] [[ACAFG-命令模式（Command Pattern）]] [[ACAFB-策略模式（Strategy Pattern）]]
