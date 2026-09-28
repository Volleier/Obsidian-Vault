**抽象工厂（Abstract Factory）是 GoF 创建型模式之一：提供一个接口，用来创建一系列相关或相互依赖的对象（一个“产品族”），而无需指定它们的具体类。客户端只持有抽象工厂接口，换一个具体工厂对象，就整套换掉所有产品。**

> 参考：GoF《设计模式》Abstract Factory 一章（动机示例是支持多种界面风格的 `WidgetFactory`）；refactoring.guru 的 Abstract Factory 页面。代码沿用原笔记的 Java。

## 为什么要有抽象工厂

GoF 书里的例子最能说明问题：一个图形界面工具包要同时支持 Motif 和 Presentation Manager 两种外观。滚动条、窗口、按钮都有两套实现，而且**必须成套使用**——Motif 风格的窗口里放一个 PM 风格的滚动条，界面就乱了。如果客户端代码到处写 `new MotifScrollBar()`，切换外观就要改遍全部代码；如果每种控件各自用一个简单工厂，又没法保证它们来自同一套风格。

抽象工厂把“一整套风格”变成一个对象：`WidgetFactory` 接口声明 `createScrollBar()`、`createWindow()` 等方法，`MotifWidgetFactory` 和 `PMWidgetFactory` 各自实现一套。程序启动时选定一个工厂，之后所有控件都从它那里拿，天然保证同族。

这里有两个方向的维度，理解抽象工厂的关键是分清它们：

| 维度 | 含义 | 示例 |
| --- | --- | --- |
| 产品等级结构（产品种类） | 同一种产品的不同实现，对应一个抽象产品接口 | `Shape`、`Color` |
| 产品族（风格/平台） | 不同种类但需要配套使用的一组产品，对应一个具体工厂 | 手绘风格的形状 + 颜色；霓虹风格的形状 + 颜色 |

新增一个产品族很容易：写一个新的具体工厂和一组具体产品，旧代码不动。新增一个产品种类很难：抽象工厂接口要加一个方法，所有具体工厂都得跟着改。原笔记“增加新的产品族相对容易，而增加新的产品等级结构比较困难”说的就是这件事；而优缺点里写的“扩展产品族非常困难”与之矛盾，应当是“扩展产品种类困难”。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| AbstractFactory | 声明创建各种抽象产品的方法 | `ThemeFactory` |
| ConcreteFactory | 实现这些方法，创建同一族的具体产品 | `SketchThemeFactory`、`NeonThemeFactory` |
| AbstractProduct | 一种产品的接口 | `Shape`、`Color` |
| ConcreteProduct | 属于某一族的具体产品 | `SketchCircle`、`NeonRed` 等 |
| Client | 只通过上面的抽象接口工作 | `Canvas` |

原笔记示例的类图：

![[抽象工厂模式-1.png]]

这张图对应的原示例（来自常见的 Java 教程）其实不是一个合格的抽象工厂：`ShapeFactory` 只会造形状、`getColor()` 返回 `null`，`ColorFactory` 只会造颜色、`getShape()` 返回 `null`，它们只是两个简单工厂被硬塞进同一个抽象类。抽象工厂的每个具体工厂都应当能造出族里的**每一种**产品，区别在于风格不同。下面的示例按这个意思重写：形状和颜色仍是两种产品，产品族换成“手绘”和“霓虹”两种渲染风格。

## Java 实现：成套切换的渲染风格

```java
// AbstractFactoryDemo.java
// 抽象产品一：形状
interface Shape {
    String outline();
}

// 抽象产品二：颜色
interface Color {
    String fill();
}

// —— 手绘风格族 ——
class SketchCircle implements Shape {
    public String outline() { return "铅笔勾勒的圆"; }
}
class SketchRed implements Color {
    public String fill() { return "水彩红"; }
}

// —— 霓虹风格族 ——
class NeonCircle implements Shape {
    public String outline() { return "发光描边的圆"; }
}
class NeonRed implements Color {
    public String fill() { return "荧光红"; }
}

// 抽象工厂：一个方法对应一种产品
interface ThemeFactory {
    Shape createCircle();
    Color createRed();
}

class SketchThemeFactory implements ThemeFactory {
    public Shape createCircle() { return new SketchCircle(); }
    public Color createRed()    { return new SketchRed(); }
}

class NeonThemeFactory implements ThemeFactory {
    public Shape createCircle() { return new NeonCircle(); }
    public Color createRed()    { return new NeonRed(); }
}

// 客户端：只认识抽象工厂和抽象产品
class Canvas {
    private final ThemeFactory factory;

    Canvas(ThemeFactory factory) {
        this.factory = factory;
    }

    void drawRedCircle() {
        Shape shape = factory.createCircle();
        Color color = factory.createRed();
        System.out.println(shape.outline() + "，填充" + color.fill());
    }
}

public class AbstractFactoryDemo {
    // 工厂通常在程序入口按配置选定一次
    static ThemeFactory chooseFactory(String theme) {
        switch (theme) {
            case "sketch": return new SketchThemeFactory();
            case "neon":   return new NeonThemeFactory();
            default: throw new IllegalArgumentException("未知风格: " + theme);
        }
    }

    public static void main(String[] args) {
        for (String theme : new String[] { "sketch", "neon" }) {
            new Canvas(chooseFactory(theme)).drawRedCircle();
        }
    }
}
```

`Canvas` 里没有任何具体类名，也不可能拼出“铅笔圆 + 荧光红”这种混搭。`chooseFactory` 是整个程序里唯一出现具体工厂类名的地方，它本身就是一个简单工厂——原示例里的 `FactoryProducer` 扮演的正是这个角色。具体工厂通常整个程序只需要一个，所以 GoF 也提到具体工厂常实现为单例。

## 抽象工厂和相近模式的区别

| 对比 | 抽象工厂 | 另一方 |
| --- | --- | --- |
| vs 工厂方法 | 一个**对象**提供多个创建方法，靠组合（持有工厂对象）替换 | 一个**方法**由子类覆写，靠继承替换；只造一种产品 |
| vs 建造者 | 立刻返回产品，关注“成套、同族” | 分步骤组装一个复杂产品，最后才取结果 |
| vs 原型 | 每族一个工厂类 | 用一组原型对象代替工厂类，克隆得到产品，可在运行时增减“族” |
| vs 外观 | 负责创建对象 | 负责简化对子系统的调用；refactoring.guru 提到抽象工厂可以代替外观，只用来隐藏子系统对象的创建方式 |

抽象工厂内部的每个 `createX()` 常常就是一个工厂方法，所以说“抽象工厂由一组工厂方法组成”并不错；两者真正的区别在于客户端是持有一个工厂对象，还是继承一个创建者类。

## 真实例子

Java 的 `javax.xml.parsers.DocumentBuilderFactory` 是抽象类，`newInstance()` 按系统属性和服务配置找到具体实现（JDK 自带的实现，或 Xerces 等第三方实现），客户端只通过抽象接口拿 `DocumentBuilder`，从而与具体解析器实现解耦。跨平台 GUI 的“外观/主题”切换是 GoF 原书的例子，也是这个模式最直观的场景。

游戏里“按平台选一整套实现”的需求很常见，比如渲染后端：同一套上层代码要在 D3D12、Vulkan、Metal 上运行，缓冲区、纹理、着色器对象必须来自同一个后端。虚幻引擎的 RHI 层在概念上就是这种分离：上层通过 `RHICreate…` 系列接口创建资源，具体由当前平台的 RHI 模块实现。RHI 的完整类结构比教科书抽象工厂复杂得多，这里只作类比，不代表它严格按此模式实现。

## 容易踩的坑

**工厂接口里塞进不相干的产品。** 原示例的 `ShapeFactory` 必须实现一个永远返回 `null` 的 `getColor()`，说明这两个产品根本不属于同一族，不该放进同一个抽象工厂。

**产品种类频繁增加。** 每加一个 `createX()`，所有具体工厂都要改。如果产品种类本身不稳定，抽象工厂会很痛苦，可以考虑用参数化的创建方法（传入产品类型），代价是失去编译期类型检查。

**在业务代码深处到处选工厂。** 选工厂的判断应只在组合根（程序入口、依赖注入配置）出现一次；否则就退化成了四处散落的 `if`。

**类的数量爆炸。** 产品种类 × 产品族个具体产品类，外加每族一个工厂。两三族、三四种产品还好，规模再大就需要评估是否真的需要“成套保证”。

## 相关

[[ACADC-工厂模式（Factory Pattern）]] [[ACADD-建造者模式（Builder Pattern）]] [[ACADE-原型模式（Prototype Pattern）]] [[ACADA-单例模式（Singleton Pattern）]] [[ACAED-外观模式（Facade Pattern）]] [[ACAEH-桥接模式（Bridge Pattern）]]
