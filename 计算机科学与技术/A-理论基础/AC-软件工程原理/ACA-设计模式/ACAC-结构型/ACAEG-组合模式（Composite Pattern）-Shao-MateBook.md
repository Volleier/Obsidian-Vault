**组合模式（Composite，又叫部分-整体模式）是 GoF 结构型模式之一：将对象组合成树形结构以表示“部分-整体”的层次结构，使客户端对单个对象（叶子）和组合对象（容器）的使用具有一致性。核心是叶子和容器实现同一个组件接口，容器的操作通过递归委派给子组件完成。**

> 参考：GoF《设计模式》Composite 一章（动机示例是绘图编辑器里的 `Graphic`，基本图形 `Line`、`Rectangle` 与可包含其他图形的 `Picture`）；refactoring.guru 的 Composite 页面。代码沿用原笔记的 Java。

## 为什么要有组合模式

很多数据天生是树：文件系统里目录套目录，界面里面板套按钮，场景里物体挂在物体下面，公司里部门下有小组、小组下有员工。处理这类数据时，代码很容易写成“先判断是叶子还是容器，是容器就遍历子节点，子节点再判断……”。每个需要处理这棵树的地方都要重复这段判断，新增一种节点类型还得回去改所有判断。

GoF 的例子是绘图编辑器：用户可以把几条线、几个矩形组合成一个图组，图组还可以再和别的图形组合。对编辑器来说，拖动、绘制、删除一个图组，和操作一个单独的矩形应该没有区别。组合模式的做法是定义一个抽象的 `Graphic`：`Line`、`Rectangle` 是叶子，直接实现 `draw()`；`Picture` 是容器，也实现 `Graphic`，它的 `draw()` 就是依次调用每个子图形的 `draw()`。因为子图形也是 `Graphic`，子图形可以又是一个 `Picture`，递归自然就发生了。客户端只管对根节点调用 `draw()`，整棵树就画完了。

关键点在于**容器本身也是组件**。这让“一个”和“一组”在类型上统一，客户端不再需要分辨它们。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Component（组件） | 叶子和容器的公共接口；可选地声明管理子节点的方法 | `Graphic` |
| Leaf（叶子） | 没有子节点，实现具体操作 | `Circle`、`Square` |
| Composite（容器/组合） | 保存子组件，把操作递归委派给子组件，可在前后加入自己的处理 | `Group` |
| Client（客户端） | 通过 Component 接口操作整棵树 | `CompositeDemo` |

原笔记示例的类图：

![[组合模式-1.png]]

原示例用一个 `Shape` 类同时充当叶子和容器，每个形状都带一个 `relations` 列表。这能表示树，但没有叶子和容器的区分，也没有递归操作：演示代码只打印根节点和它的直接子节点，列出的输出却包含了孙子节点（如 `Square Black`），代码和输出其实对不上。要打印整棵树，必须递归，这正是组合模式要把操作放在组件接口里的原因。

## 透明性与安全性

管理子节点的 `add`、`remove`、`getChildren` 该放在哪里，是组合模式里一个真正的设计取舍，GoF 专门讨论过：

| 方案 | 做法 | 好处 | 代价 |
| --- | --- | --- | --- |
| 透明式 | 在 Component 里声明 `add`/`remove` | 客户端完全不用区分叶子和容器 | 对叶子调用 `add` 没有意义，只能抛异常或静默忽略，错误到运行时才暴露 |
| 安全式 | 只在 Composite 里声明 `add`/`remove` | 编译期就不可能给叶子加子节点 | 构建树的代码需要知道具体是容器，失去部分一致性 |

GoF 书中偏向透明性；实践中安全式更常见，因为“构建树”和“使用树”通常发生在不同的代码里：构建时本来就知道谁是容器，使用时只需要 `draw()` 这类通用操作。下面的实现采用安全式。

## Java 实现：可嵌套的图组

```java
// CompositeDemo.java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

// 组件：只声明通用操作（安全式）
interface Graphic {
    void draw(String indent);
    int area();
}

// 叶子
class Circle implements Graphic {
    private final String color;
    private final int radius;

    Circle(String color, int radius) { this.color = color; this.radius = radius; }

    public void draw(String indent) { System.out.println(indent + "Circle(" + color + ", r=" + radius + ")"); }
    public int area() { return (int) Math.round(Math.PI * radius * radius); }
}

class Square implements Graphic {
    private final String color;
    private final int side;

    Square(String color, int side) { this.color = color; this.side = side; }

    public void draw(String indent) { System.out.println(indent + "Square(" + color + ", side=" + side + ")"); }
    public int area() { return side * side; }
}

// 容器：本身也是 Graphic，操作递归委派给子节点
class Group implements Graphic {
    private final String name;
    private final List<Graphic> children = new ArrayList<>();

    Group(String name) { this.name = name; }

    Group add(Graphic g) {
        if (g == this) throw new IllegalArgumentException("不能把自己加为子节点");
        children.add(g);
        return this;
    }

    void remove(Graphic g) { children.remove(g); }

    List<Graphic> getChildren() { return Collections.unmodifiableList(children); }

    public void draw(String indent) {
        System.out.println(indent + "Group " + name + " {");
        for (Graphic child : children) {
            child.draw(indent + "  ");   // 递归
        }
        System.out.println(indent + "}");
    }

    public int area() {
        int sum = 0;
        for (Graphic child : children) sum += child.area();
        return sum;
    }
}

public class CompositeDemo {
    public static void main(String[] args) {
        Group squares = new Group("squares")
                .add(new Square("Black", 5))
                .add(new Square("White", 6));
        Group circles = new Group("circles")
                .add(new Circle("Black", 3))
                .add(new Circle("White", 4));
        Group root = new Group("root")
                .add(new Square("Blue", 10))
                .add(squares)
                .add(circles);

        root.draw("");                           // 客户端只对根调用一次
        System.out.println("total area = " + root.area());
    }
}
```

`add` 里只防了“把自己加进自己”，更隐蔽的是把祖先节点加为后代（A 包含 B，再把 A 加进 B），这会让递归无限进行直到栈溢出。需要防范时，可以给组件加父指针，`add` 前沿父链向上检查。

## 组合模式和相近模式的区别

| 对比 | 组合 | 另一方 |
| --- | --- | --- |
| vs 装饰器 | 容器可以有多个子节点，目的是聚合 | 只有一个被装饰对象，目的是加职责；两者常一起用 |
| vs 迭代器 | 定义树的结构 | 常用来遍历组合树，把遍历顺序（深度/广度优先）从树里分离出去 |
| vs 访问者 | 操作写在组件类里 | 对一棵稳定的组合树执行多种操作时，可把操作移到访问者里 |
| vs 享元 | 组合树的叶子很多且相似时 | 可把叶子做成享元共享（GoF 的字符例子） |
| vs 解释器 | — | 解释器的抽象语法树本身就是一棵组合树 |
| vs 责任链 | — | 组合树里子节点把请求沿父链上传，就构成一条责任链 |

## 真实例子

Java AWT 的 `java.awt.Container` 继承自 `java.awt.Component`，又可以 `add(Component)`，于是面板里放面板、面板里放按钮的界面树就是一棵组合树，布局和绘制都沿树递归进行。Swing 的 `JComponent` 继承自 `Container`，延续了同样的结构。

虚幻引擎里有好几棵组合树。场景组件 `USceneComponent` 可以通过 `AttachToComponent` 挂到另一个场景组件下面，子组件的变换相对父组件计算，移动父组件整棵子树跟着动，`GetChildrenComponents` 可以取到子组件。UMG 中，`UPanelWidget` 继承自 `UWidget`，又能容纳多个子 `UWidget`（`AddChild`），`UCanvasPanel`、`UVerticalBox` 等都是它的子类，而 `UTextBlock`、`UImage` 这类不能有子控件的是叶子。

## 容易踩的坑

**树里出现环。** 见代码后的说明。

**对超深的树递归导致栈溢出。** 平衡的树没问题，但退化成链表的“树”（比如几万层的嵌套）会耗尽调用栈，需要改成显式栈的迭代遍历。

**叶子被迫实现无意义的方法。** 透明式设计下，叶子的 `add` 要么抛异常，要么什么也不做。静默忽略最危险，调用方以为加进去了。

**子节点顺序和所有权不清。** 同一个子节点能否同时属于两个容器？删除容器时子节点是否一起删除？这些语义组件接口不会告诉你，要在设计时定下来。GoF 也讨论过共享子组件的问题：共享会让“父节点”变得不唯一。

**原笔记提到的“违反依赖倒置原则”。** 这是针对原示例而言的：它的 `Shape` 是具体类而不是接口。把组件定义为接口、叶子和容器分别实现，就不存在这个问题。

## 相关

[[ACAEF-装饰器模式（Decorator Pattern）]] [[ACAFC-迭代器模式（Iterator Pattern）]] [[ACAFD-访问者模式（Visitor Pattern）]] [[ACAFF-解释器模式（Interpreter Pattern）]] [[ACAEE-享元模式（Flyweight Pattern）]] [[ACAFI-责任链模式（Chain of Responsibility Pattern）]] [[ACADD-建造者模式（Builder Pattern）]]
