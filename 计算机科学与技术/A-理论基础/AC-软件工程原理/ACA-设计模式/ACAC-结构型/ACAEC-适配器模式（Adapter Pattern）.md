**适配器模式（Adapter，又称包装器 Wrapper）是 GoF 结构型模式之一：把一个类的接口转换成客户期望的另一个接口，使原本因接口不兼容而不能一起工作的类可以一起工作。有两种实现形式：通过组合实现的对象适配器，和通过继承实现的类适配器。**

> 参考：GoF《设计模式》Adapter 一章（动机示例是绘图编辑器用 `TextShape` 把现成的 `TextView` 适配成 `Shape`）；refactoring.guru 的 Adapter 页面。代码沿用原笔记的 Java。

## 为什么要有适配器

适配器几乎总是出现在“两边都不能改”的时候。一边是已经写好的客户端代码，它按某个接口调用对象；另一边是一个现成的类——第三方库、遗留代码、另一个团队的模块——功能正好是需要的，但方法名、参数、数据格式和客户端期望的对不上。改客户端会牵连大量调用点，改现成的类要么没有源码，要么会影响它的其他使用者。

GoF 的例子是一个绘图编辑器：所有图形都实现 `Shape` 接口（提供包围盒、可以被拖动操作），编辑器要加一个文本图形，而工具包里已经有一个功能完善的 `TextView`，能显示和编辑文本，但它不是 `Shape`，方法也叫别的名字。与其重写一个文本控件，不如写一个 `TextShape`：对外实现 `Shape`，内部把 `boundingBox()` 这样的请求翻译成对 `TextView` 相应方法的调用。

生活里的类比是电源转接头和读卡器：插头和插座都没改，中间多了一个转换形状的东西。原笔记举的读卡器例子就是这个意思。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Target（目标接口） | 客户端期望的接口 | `Shape` |
| Adaptee（被适配者） | 已存在、接口不兼容但功能可用的类 | `LegacyRectangle` |
| Adapter（适配器） | 实现 Target，把调用翻译给 Adaptee | `RectangleAdapter`、`RectangleClassAdapter` |
| Client（客户端） | 只按 Target 接口工作 | `AdapterDemo` |

两种形式的区别：

| 形式 | 做法 | 能适配 | 能否覆写被适配者行为 |
| --- | --- | --- | --- |
| 对象适配器 | 适配器**持有**一个 Adaptee 实例（组合） | Adaptee 及其所有子类 | 不能直接覆写，只能包装 |
| 类适配器 | 适配器**继承** Adaptee 并实现 Target | 只有这一个具体类 | 能覆写 Adaptee 的方法 |

类适配器要求语言支持多重继承，或者至少 Target 是接口。Java 只能单继承，所以类适配器只能写成“继承 Adaptee、实现 Target 接口”，并且一个适配器只能适配一个类——这就是原笔记里“只能适配一个类，且目标必须是抽象的”的来历，准确说法是 Target 必须是接口。原笔记“推荐使用依赖关系而不是继承”指的就是优先用对象适配器。

原笔记示例的类图：

![[适配器模式-1.png]]

原示例的代码有不少对不上的地方（类名 `ShapeAdapter` 与构造函数 `GraphicAdapter` 不一致、`SquareShowcase` 实现的方法并不在接口里、演示代码里的 `ShowSquare` 没有定义），输出也是空的，无法编译运行。下面保留“让长方形能力接入只认正方形/通用形状的客户端”这个题材，重写成可运行的版本。

## Java 实现：对象适配器与类适配器

客户端的 `Shape` 接口用“左上角 + 宽高”描述位置；遗留的 `LegacyRectangle` 却用“两个对角点”，方法名也不一样。

```java
// AdapterDemo.java
import java.util.List;

// 目标接口：客户端期望的形式
interface Shape {
    void draw(int x, int y, int width, int height);
}

// 已有的客户端实现，直接符合接口
class Square implements Shape {
    public void draw(int x, int y, int width, int height) {
        int side = Math.min(width, height);
        System.out.println("Square at (" + x + "," + y + ") side=" + side);
    }
}

// 被适配者：遗留类，接口不兼容且不能修改
class LegacyRectangle {
    void render(int x1, int y1, int x2, int y2) {
        System.out.println("LegacyRectangle from (" + x1 + "," + y1 + ") to (" + x2 + "," + y2 + ")");
    }
}

// 对象适配器：组合一个 LegacyRectangle
class RectangleAdapter implements Shape {
    private final LegacyRectangle adaptee;

    RectangleAdapter(LegacyRectangle adaptee) {
        this.adaptee = adaptee;
    }

    public void draw(int x, int y, int width, int height) {
        // 接口转换：宽高 -> 对角点
        adaptee.render(x, y, x + width, y + height);
    }
}

// 类适配器：继承被适配者，同时实现目标接口
class RectangleClassAdapter extends LegacyRectangle implements Shape {
    public void draw(int x, int y, int width, int height) {
        render(x, y, x + width, y + height);
    }
}

public class AdapterDemo {
    // 客户端只认识 Shape
    static void drawAll(List<Shape> shapes) {
        for (Shape s : shapes) {
            s.draw(10, 20, 100, 50);
        }
    }

    public static void main(String[] args) {
        drawAll(List.of(
                new Square(),
                new RectangleAdapter(new LegacyRectangle()),
                new RectangleClassAdapter()));
    }
}
```

适配器里真正有价值的代码只有 `render(x, y, x + width, y + height)` 这一行坐标换算。实际项目里的适配器往往也就是这样薄薄一层：参数换算、单位转换、异常类型转换、同步接口转异步接口。一旦适配器里开始写业务逻辑，它就不再只是适配器了。

## 适配器和相近模式的区别

| 模式 | 接口变化 | 意图 | 包装对象数量 |
| --- | --- | --- | --- |
| 适配器 | 旧接口 → **另一个已有的**目标接口 | 让不兼容的东西能用 | 通常一个 |
| 装饰器 | 接口不变（或扩展） | 增加职责，可递归嵌套 | 一个，但可层层叠加 |
| 代理 | 接口不变 | 控制访问 | 一个 |
| 外观 | 定义一个**新的、更简单的**接口 | 简化子系统 | 多个 |
| 桥接 | 事先设计的抽象/实现分离 | 让两个维度独立变化 | 一个实现对象 |

适配器和桥接的结构图几乎一样，都是“抽象持有另一个对象”，但 refactoring.guru 点出了它们的时间差：桥接通常在设计之初就规划好，让各部分能独立开发；适配器通常用在已有的应用上，让原本不兼容的类能协作。外观和适配器的区别也类似：适配器让现有对象去符合一个**已有的**接口，外观为一堆对象**定义新的**接口。

## 真实例子

Java 标准库里最典型的是 `java.io.InputStreamReader`：它把面向字节的 `InputStream` 适配成面向字符的 `Reader`，中间做了字符集解码。`java.util.Arrays.asList(array)` 把数组适配成 `List` 接口，返回的列表直接以原数组为存储，所以不能增删元素，修改元素会反映到原数组上。`Collections.enumeration(collection)` 则把新式集合适配成旧的 `Enumeration` 接口，供老 API 使用。

游戏和引擎集成第三方库时也离不开适配层：物理、音频、平台 SDK 各有自己的数据类型和坐标约定（比如 Y 轴朝上还是 Z 轴朝上、厘米还是米），引擎通常会写一层转换代码，把第三方类型包装成引擎自己的接口。

## 容易踩的坑

**适配器里做了隐式的有损转换。** 单位换算、精度截断、时区转换这类问题写在适配器里最隐蔽，调用方以为拿到的是原数据。转换规则要写清楚，最好有测试覆盖边界值。

**双向适配引起的循环。** A 适配成 B、B 又适配成 A，嵌套几层后调用栈和语义都会乱。

**`Arrays.asList` 的“不能增删”。** 调用 `add` 会抛 `UnsupportedOperationException`；需要可变列表时写 `new ArrayList<>(Arrays.asList(...))`。

**把适配器当作逃避重构的长期方案。** 原笔记说得对，适配器更多用于解决已有系统的问题，是一种补救；设计新系统时如果一开始就需要大量适配器，往往说明接口没有设计好。

**被适配者的异常和失败语义没转换。** 遗留接口用返回码表示失败，目标接口期望抛异常，适配器如果只转了参数、没转失败语义，错误就被吞掉了。

## 相关

[[ACAEA-代理模式（Proxy Pattern）]] [[ACAEF-装饰器模式（Decorator Pattern）]] [[ACAED-外观模式（Facade Pattern）]] [[ACAEH-桥接模式（Bridge Pattern）]]
