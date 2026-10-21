**桥接模式（Bridge）是 GoF 结构型模式之一：将抽象部分与它的实现部分分离，使它们都可以独立地变化。做法是把一个类在两个维度上的变化拆成两棵独立的继承层次——“抽象”层次持有一个“实现”对象的引用，用组合代替继承把两边连接起来，这个引用就是“桥”。**

> 参考：GoF《设计模式》Bridge 一章（动机示例是可移植窗口系统：`Window` 层次与 `WindowImp` 层次，后者有 `XWindowImp`、`PMWindowImp` 等平台实现）；refactoring.guru 的 Bridge 页面。代码沿用原笔记的 Java。

## 为什么要有桥接

桥接要解决的问题可以用一个乘法来说明。假设有“形状”这个维度：圆、矩形；还有“绘制方式”这个维度：画到屏幕上的光栅渲染、导出成 SVG 矢量文件。如果用继承把两者绑在一起，就要写 `RasterCircle`、`SvgCircle`、`RasterRectangle`、`SvgRectangle` 四个类；再加一种形状三角形、一种绘制方式 PDF，就是 3×3=9 个类。两个维度各有 m 和 n 种变化，类的数量是 m×n，而且同一种绘制逻辑在多个类里重复。

GoF 的例子是可移植的窗口系统：窗口有普通窗口、图标窗口、对话框等种类（抽象维度），又要跑在 X Window、Presentation Manager 等不同平台上（实现维度）。用继承会让每种窗口 × 每个平台都出一个子类，而且客户端代码创建窗口时就得写死平台，完全谈不上可移植。

桥接的做法是承认这是两个独立的维度，各自建一棵继承层次：形状层次只关心“画什么”（圆由什么基本操作构成），绘制层次只关心“怎么画”（在某个后端上画一条弧、一个矩形）。形状持有一个绘制器的引用，把具体绘制委派给它。于是类的数量变成 m+n，加一种形状不用碰绘制器，加一种绘制器不用碰形状，两边可以由不同的人独立开发。

这里的“抽象”和“实现”不是 Java 里抽象类与实现类的意思。GoF 的“抽象”指客户端直接使用的高层控制层，“实现”指它依赖的底层平台操作。抽象层的操作通常由若干个实现层的基本操作组合而成，所以两者的接口不必一致——这是桥接和适配器、代理在形式上的一个区别。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Abstraction（抽象） | 客户端使用的高层接口，持有一个 Implementor 引用 | `Shape` |
| RefinedAbstraction（扩充抽象） | 抽象的具体变体 | `Circle`、`Rectangle` |
| Implementor（实现者） | 底层基本操作的接口，不必与 Abstraction 的接口一致 | `DrawAPI` |
| ConcreteImplementor（具体实现者） | 在某个平台或后端上实现基本操作 | `RasterDrawAPI`、`SvgDrawAPI` |

原笔记示例的类图：

![[桥接模式-1.png]]

原示例把 `RedCircle`、`GreenCircle` 作为 `DrawAPI` 的实现。颜色更像是一个参数而不是一个“实现平台”，而且实现类名字里带了 Circle，说明实现层反过来依赖了抽象层的具体种类，桥两边没有真正分开。下面把实现维度换成两个渲染后端，实现层只提供与形状种类无关的基本操作。

## Java 实现：形状 × 渲染后端

```java
// BridgeDemo.java
// 实现者：底层基本操作，不知道有哪些形状
interface DrawAPI {
    void drawArc(int cx, int cy, int radius, String color);
    void drawBox(int x, int y, int w, int h, String color);
}

class RasterDrawAPI implements DrawAPI {
    public void drawArc(int cx, int cy, int r, String color) {
        System.out.println("[raster] 光栅化圆弧 center=(" + cx + "," + cy + ") r=" + r + " " + color);
    }
    public void drawBox(int x, int y, int w, int h, String color) {
        System.out.println("[raster] 填充像素矩形 " + w + "x" + h + " at (" + x + "," + y + ") " + color);
    }
}

class SvgDrawAPI implements DrawAPI {
    public void drawArc(int cx, int cy, int r, String color) {
        System.out.println("<circle cx=\"" + cx + "\" cy=\"" + cy + "\" r=\"" + r + "\" fill=\"" + color + "\"/>");
    }
    public void drawBox(int x, int y, int w, int h, String color) {
        System.out.println("<rect x=\"" + x + "\" y=\"" + y + "\" width=\"" + w + "\" height=\"" + h + "\" fill=\"" + color + "\"/>");
    }
}

// 抽象：持有实现者引用，这就是“桥”
abstract class Shape {
    protected final DrawAPI drawAPI;
    protected final String color;

    protected Shape(DrawAPI drawAPI, String color) {
        this.drawAPI = drawAPI;
        this.color = color;
    }

    public abstract void draw();
}

// 扩充抽象：只描述“画什么”，把“怎么画”交给 drawAPI
class Circle extends Shape {
    private final int x, y, radius;

    Circle(int x, int y, int radius, String color, DrawAPI api) {
        super(api, color);
        this.x = x; this.y = y; this.radius = radius;
    }

    public void draw() {
        drawAPI.drawArc(x, y, radius, color);
    }
}

class Rectangle extends Shape {
    private final int x, y, w, h;

    Rectangle(int x, int y, int w, int h, String color, DrawAPI api) {
        super(api, color);
        this.x = x; this.y = y; this.w = w; this.h = h;
    }

    public void draw() {
        drawAPI.drawBox(x, y, w, h, color);
    }
}

public class BridgeDemo {
    public static void main(String[] args) {
        DrawAPI[] backends = { new RasterDrawAPI(), new SvgDrawAPI() };
        // 2 种形状 × 2 种后端，只写了 2 + 2 个类
        for (DrawAPI api : backends) {
            new Circle(100, 100, 10, "red", api).draw();
            new Rectangle(0, 0, 40, 20, "green", api).draw();
        }
    }
}
```

`DrawAPI` 的方法是 `drawArc`、`drawBox` 这种与形状种类无关的基本操作。如果写成 `drawCircle`、`drawRectangle`，每加一种形状就要给所有后端加方法，两个维度又耦合回去了。桥接能不能成立，关键就在于实现层的接口能否提炼得足够“底层”。

## 桥接和相近模式的区别

| 对比 | 桥接 | 另一方 |
| --- | --- | --- |
| vs 适配器 | 设计之初就规划好两个维度的分离 | 事后补救，让已有的不兼容类协作；两者结构相似 |
| vs 策略 | 结构同样是“持有一个接口引用并委派”；桥接关注类层次的拆分，抽象层本身也有多个子类 | 策略关注替换算法，上下文通常只有一个类 |
| vs 状态 | 实现对象一般在构造时确定，运行中较少切换 | 状态对象随状态转移频繁切换，且状态之间互相知道 |
| vs 抽象工厂 | 两者常配合：用抽象工厂创建与平台匹配的实现对象 | GoF 在桥接一章中提到了这种搭配 |

refactoring.guru 专门指出，桥接、状态、策略（以及某种程度上的适配器）结构非常相似，都基于组合把工作委派给其他对象，但它们解决的是不同的问题。模式不只是结构的配方，还传达了设计意图。

## 真实例子

JDBC 常被当作桥接的例子：应用代码面对的是 `java.sql` 里的 `Connection`、`Statement` 等接口，各数据库厂商的驱动提供具体实现，`DriverManager` 根据连接 URL 找到对应的驱动。应用层的使用方式和数据库实现可以各自演化。前面外观一文提到的 SLF4J 同样兼有桥接的性质。

图形引擎的渲染硬件接口层是桥接思想的典型场景：上层渲染代码（画网格、做后处理）是“抽象”，D3D12、Vulkan、Metal 的后端是“实现”，上层只调用一组与平台无关的基本命令。虚幻引擎的 RHI（Render Hardware Interface）就承担这个角色，由 `FDynamicRHI` 的各平台子类实现。它的实际类结构比教科书复杂得多，这里只作思路上的对照。

## 容易踩的坑

**实现层接口按抽象层的种类设计。** 见代码后的说明，这是桥接最常见的失败方式。

**两个维度其实并不独立。** 如果某些形状只能用某些后端画，或者每加一个形状都得让后端配合改，强行拆成桥接只会增加复杂度。先确认真的是两个正交的变化方向。

**只有一个维度在变。** 实现只有一种、也看不到第二种的时候，桥接就是过度设计。原笔记提到桥接“增加了系统的理解与设计难度”，在只有一个维度变化时这个代价没有回报。

**实现对象在运行中被替换却没考虑状态。** 如果允许 `setDrawAPI` 切换后端，要保证已经在旧后端上积累的状态（缓存、句柄）被正确迁移或释放。

## 相关

[[ACAEC-适配器模式（Adapter Pattern）]] [[ACAFB-策略模式（Strategy Pattern）]] [[ACAFK-状态模式（State Pattern）]] [[ACADB-抽象工厂模式（Abstract Factory Pattern）]] [[ACAED-外观模式（Facade Pattern）]]
