**建造者模式（Builder，也译作生成器）是 GoF 创建型模式之一：把一个复杂对象的构建过程与它的表示分离，使同样的构建过程可以创建不同的表示。参与者是产品、建造者接口、具体建造者和可选的指导者（Director）。今天在 Java 里更常见的是它的变体——《Effective Java》推荐的链式静态内部 Builder。**

> 参考：GoF《设计模式》Builder 一章（动机示例是 RTF 阅读器把文档转换成 ASCII、TeX 等多种格式）；refactoring.guru 的 Builder 页面；Joshua Bloch《Effective Java》第 3 版 Item 2（遇到多个构造器参数时考虑使用构建器）。代码沿用原笔记的 Java。

## 为什么要有建造者

“建造者”这个名字底下其实有两个不同的问题。

第一个是 GoF 原书要解决的：**同一套组装步骤，产出不同形态的结果**。RTF 阅读器逐个读出文档里的段落、字体切换、字符，这套“读”的流程是固定的；但读出来的东西要转换成 ASCII 纯文本、TeX 源码还是可编辑的文本控件，各不相同。GoF 的做法是让阅读器（Director）只负责按顺序调用 `convertCharacter()`、`convertParagraph()` 等步骤，具体生成什么交给 `TextConverter` 的子类（Builder）。换一个建造者，就换一种输出，而阅读器的解析代码一行不改。

第二个是日常代码里更常碰到的：**构造参数太多**。一个对象有两三个必填字段和七八个可选字段，写重叠构造器（telescoping constructor）要写一长串重载，调用处 `new Graphic("circle", null, 0, 2, true, null)` 谁也看不懂哪个参数是什么；用无参构造 + setter 又会让对象在构造过程中处于不完整的中间状态，而且没法做成不可变对象。《Effective Java》的方案是给类配一个静态内部 `Builder`：必填参数放进 Builder 构造函数，可选参数用链式方法设置，最后 `build()` 一次性校验并创建不可变对象。

两种用法共享同一个核心：**把“怎么一步步拼”和“最终的对象”分开**，只是前者强调可替换的表示，后者强调可读性和不可变性。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Product（产品） | 被构建的复杂对象，不同建造者的产品可以毫无共同接口 | `String`（SVG 文本）、`List<String>`（绘图指令） |
| Builder（抽象建造者） | 声明构建各个部件的步骤 | `GraphicBuilder` |
| ConcreteBuilder（具体建造者） | 实现各步骤，内部累积结果，并提供取结果的方法 | `SvgBuilder`、`CommandListBuilder` |
| Director（指导者） | 定义步骤的调用顺序，只依赖 Builder 接口；可省略 | `BadgeDirector` |

GoF 特意指出，取结果的 `getResult()` 通常不放在抽象建造者里，因为不同建造者的产品类型可能完全不同（ASCII 字符串和文本控件没有共同父类），客户端知道自己用的是哪个具体建造者，直接向它要结果。

原笔记示例的类图：

![[建造者模式-1.png]]

原示例用 `GraphicsBuilder` 的两个方法 `showGraphics1()`、`showGraphics2()` 各自拼出一个固定组合的 `Graphics`。这其实更像“把固定配方封装进方法”，建造者和指导者揉在了一起，也没有可替换的表示。下面按 GoF 的分工重写，形状、颜色这些部件保留原示例的题材。

## Java 实现一：Director 驱动、两种表示

同一个“徽章”构建流程（先画底形，再填色，再加文字），交给两个建造者，一个输出 SVG 文本，一个输出绘图指令列表：

```java
// BuilderDemo.java
import java.util.ArrayList;
import java.util.List;

// 抽象建造者：只声明步骤，不声明 getResult()
interface GraphicBuilder {
    void buildShape(String shape, int size);
    void buildColor(String color);
    void buildLabel(String text);
}

// 具体建造者一：生成 SVG 片段
class SvgBuilder implements GraphicBuilder {
    private final StringBuilder svg = new StringBuilder();
    private String fill = "none";

    public void buildShape(String shape, int size) {
        if (shape.equals("circle")) {
            svg.append("<circle r=\"").append(size / 2).append("\" FILL/>");
        } else {
            svg.append("<rect width=\"").append(size).append("\" height=\"").append(size).append("\" FILL/>");
        }
    }

    public void buildColor(String color) {
        fill = color;
    }

    public void buildLabel(String text) {
        svg.append("<text>").append(text).append("</text>");
    }

    public String getSvg() {
        // 颜色可能在形状之后才设置，取结果时再统一替换
        return svg.toString().replace("FILL", "fill=\"" + fill + "\"");
    }
}

// 具体建造者二：生成绘图指令，产品类型与 SvgBuilder 完全不同
class CommandListBuilder implements GraphicBuilder {
    private final List<String> commands = new ArrayList<>();

    public void buildShape(String shape, int size) { commands.add("DRAW " + shape + " " + size); }
    public void buildColor(String color)          { commands.add("FILL " + color); }
    public void buildLabel(String text)           { commands.add("TEXT " + text); }

    public List<String> getCommands() {
        return commands;
    }
}

// 指导者：固定步骤顺序，只依赖接口
class BadgeDirector {
    void constructBadge(GraphicBuilder builder, String label) {
        builder.buildShape("circle", 64);
        builder.buildColor("red");
        builder.buildLabel(label);
    }
}

public class BuilderDemo {
    public static void main(String[] args) {
        BadgeDirector director = new BadgeDirector();

        SvgBuilder svg = new SvgBuilder();
        director.constructBadge(svg, "VIP");
        System.out.println(svg.getSvg());

        CommandListBuilder cmd = new CommandListBuilder();
        director.constructBadge(cmd, "VIP");
        System.out.println(cmd.getCommands());
    }
}
```

`BadgeDirector` 完全不知道结果是字符串还是列表。`SvgBuilder` 里颜色要延后到取结果时才替换，正说明了建造者内部可以自由决定如何累积中间状态——这是它相对于“一次性构造”的灵活之处，也是容易写错的地方：步骤顺序一变，某些建造者的中间状态就可能不对，需要在建造者内部处理好。

## Java 实现二：链式 Builder（Effective Java 风格）

```java
// GraphicSpecDemo.java
public class GraphicSpecDemo {
    // 不可变产品
    static final class GraphicSpec {
        private final String shape;      // 必填
        private final int size;          // 必填
        private final String fillColor;  // 可选
        private final int borderWidth;   // 可选
        private final String label;      // 可选

        private GraphicSpec(Builder b) {
            this.shape = b.shape;
            this.size = b.size;
            this.fillColor = b.fillColor;
            this.borderWidth = b.borderWidth;
            this.label = b.label;
        }

        @Override
        public String toString() {
            return shape + "(" + size + ") fill=" + fillColor + " border=" + borderWidth + " label=" + label;
        }

        static final class Builder {
            private final String shape;
            private final int size;
            private String fillColor = "none";
            private int borderWidth = 1;
            private String label = "";

            Builder(String shape, int size) {   // 必填参数走构造函数
                this.shape = shape;
                this.size = size;
            }

            Builder fillColor(String c) { this.fillColor = c; return this; }
            Builder borderWidth(int w)  { this.borderWidth = w; return this; }
            Builder label(String l)     { this.label = l; return this; }

            GraphicSpec build() {
                if (size <= 0) throw new IllegalStateException("size 必须为正");  // 统一校验
                return new GraphicSpec(this);
            }
        }
    }

    public static void main(String[] args) {
        GraphicSpec spec = new GraphicSpec.Builder("rectangle", 100)
                .fillColor("blue")
                .label("Box")
                .build();
        System.out.println(spec);
    }
}
```

校验放在 `build()` 里而不是各个 setter 里，是因为字段之间可能有约束（比如“有边框时边框宽度不能超过尺寸的一半”），只有全部设置完才能判断。

## 建造者和相近模式的区别

| 对比 | 建造者 | 另一方 |
| --- | --- | --- |
| vs 抽象工厂 | 分多步构建一个复杂对象，最后一步才返回 | 一次调用立即返回产品，重点是一族产品配套 |
| vs 工厂方法 | 客户端（或 Director）控制构建过程 | 客户端不参与构建过程，只拿结果 |
| vs 组合模式 | 常用来一步步构建组合树（refactoring.guru 提到可以递归地构建组合结构） | 组合描述树本身的结构 |
| vs 模板方法 | Director 固定步骤顺序、Builder 可替换，是组合关系 | 父类固定步骤顺序、子类覆写步骤，是继承关系 |

原笔记里“与工厂模式的区别是：建造者模式更加关注于零件装配的顺序”大体没错，更准确地说是：工厂关心“造哪个类”，建造者关心“怎么一步步造”。

## 真实例子

JDK 11 的 `java.net.http.HttpRequest.newBuilder()` 是典型的链式建造者：`.uri(...)`、`.header(...)`、`.timeout(...)` 逐项设置，`.build()` 得到不可变的请求对象。`StringBuilder` 名字里有 Builder，但它只是可变字符序列，没有“构建过程与表示分离”，一般不算这个模式。

虚幻引擎 Slate 的声明式语法也是建造者思路：`SNew(STextBlock).Text(...).Font(...)` 中，每个 Slate 控件用 `SLATE_BEGIN_ARGS` 宏声明一个 `FArguments` 结构，链式调用都是在往 `FArguments` 里填参数，最后由 `SNew` 创建控件并调用 `Construct(const FArguments&)`。

## 容易踩的坑

**为只有两三个字段的类写 Builder。** 原笔记“如果产品的属性较少，建造者模式可能会导致代码冗余”是对的，这时一个构造函数更清楚。

**Builder 复用导致状态泄漏。** 链式 Builder 调用 `build()` 后继续修改再 `build()`，如果产品浅拷贝了 Builder 里的可变集合，前一个产品也会被改。`build()` 里要对集合做防御性拷贝。

**校验分散在 setter 里。** 字段间约束只能在 `build()` 统一校验，否则会出现“先设 A 时 B 还是默认值”导致的误判。

**Director 过度设计。** 构建步骤只有一种顺序、只有一个建造者时，Director 可以省掉，GoF 也把它当作可选角色。

## 相关

[[ACADC-工厂模式（Factory Pattern）]] [[ACADB-抽象工厂模式（Abstract Factory Pattern）]] [[ACADE-原型模式（Prototype Pattern）]] [[ACAEG-组合模式（Composite Pattern）]] [[ACAFH-模板模式（Template Pattern）]]
