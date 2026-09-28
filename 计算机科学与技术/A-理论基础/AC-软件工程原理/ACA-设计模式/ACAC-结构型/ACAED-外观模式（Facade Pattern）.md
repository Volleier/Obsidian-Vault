**外观模式（Facade，也译作门面模式）是 GoF 结构型模式之一：为子系统中的一组接口提供一个统一的高层接口，让子系统更容易使用。外观类本身不实现新功能，只把客户端的一个简单请求翻译成对多个子系统对象的一串调用。**

> 参考：GoF《设计模式》Facade 一章（动机示例是编程环境里的 `Compiler` 类，封装词法分析、语法分析、代码生成等子系统）；refactoring.guru 的 Facade 页面。代码沿用原笔记的 Java。

## 为什么要有外观

一个子系统做大以后，内部会分出很多类，每个类各管一小块，这对子系统自己的可维护性是好事，对使用者却是负担。GoF 的例子是编译器子系统：里面有 `Scanner`、`Parser`、`ProgramNode`、`BytecodeStream`、`ProgramNodeBuilder` 等一堆类。少数高级用户（比如要做静态分析的工具）确实需要直接操作语法树，但绝大多数调用者只想要一件事：“把这段源码编译了”。让他们自己按顺序创建扫描器、解析器、生成器并串起来，既繁琐又容易出错，而且一旦子系统内部重构，所有调用方都要跟着改。

外观的做法是提供一个 `Compiler` 类，对外只暴露 `compile()` 这样的高层方法，内部按正确顺序调用各个子系统类。于是：

- 普通客户端只依赖外观，子系统内部怎么拆分、怎么重构都与它无关；
- 需要精细控制的客户端仍然可以绕过外观，直接使用子系统类——外观不是封锁，只是提供一条捷径；
- 子系统的类完全不知道外观的存在，外观对它们来说只是又一个调用者。

最后一点是外观和中介者的根本区别，后面会再对比。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Facade（外观） | 知道哪些子系统类负责处理请求，把客户端请求委派给它们 | `ShapeMaker` |
| Subsystem classes（子系统类） | 实现子系统功能；处理外观指派的任务，但不持有外观的引用 | `Canvas`、`Palette`、`ShapeRenderer`、`Exporter` |
| Client（客户端） | 通过外观使用子系统 | `FacadeDemo` |

原笔记示例的类图：

![[外观模式-1.png]]

原示例的 `ShapeMaker` 持有三个形状对象，`drawCircle()` 只是转调 `circle.draw()`，外观和子系统之间一对一，看不出“简化”在哪里。外观的价值在于**把多步、多对象的协作收拢成一步**，下面的示例让画一个形状需要经过准备画布、选颜色、渲染、导出四个子系统对象。

## Java 实现：绘图子系统的外观

```java
// FacadeDemo.java
// ---- 子系统：每个类各管一件事，互相之间、以及对外观都一无所知 ----
class Canvas {
    void create(int w, int h)  { System.out.println("Canvas: 创建 " + w + "x" + h + " 画布"); }
    void clear()               { System.out.println("Canvas: 清空"); }
}

class Palette {
    String pick(String name) {
        String hex;
        switch (name) {
            case "red":  hex = "#FF0000"; break;
            case "blue": hex = "#0000FF"; break;
            default:     hex = "#000000";
        }
        System.out.println("Palette: " + name + " -> " + hex);
        return hex;
    }
}

class ShapeRenderer {
    void circle(int r, String color)          { System.out.println("Renderer: 圆 r=" + r + " " + color); }
    void rectangle(int w, int h, String color){ System.out.println("Renderer: 矩形 " + w + "x" + h + " " + color); }
}

class Exporter {
    void toPng(String file) { System.out.println("Exporter: 写出 " + file); }
}

// ---- 外观：把常用流程收拢成一步 ----
class ShapeMaker {
    private final Canvas canvas = new Canvas();
    private final Palette palette = new Palette();
    private final ShapeRenderer renderer = new ShapeRenderer();
    private final Exporter exporter = new Exporter();

    void drawCircle(int radius, String color, String file) {
        canvas.create(radius * 2, radius * 2);
        canvas.clear();
        renderer.circle(radius, palette.pick(color));
        exporter.toPng(file);
    }

    void drawRectangle(int w, int h, String color, String file) {
        canvas.create(w, h);
        canvas.clear();
        renderer.rectangle(w, h, palette.pick(color));
        exporter.toPng(file);
    }
}

public class FacadeDemo {
    public static void main(String[] args) {
        ShapeMaker maker = new ShapeMaker();
        maker.drawCircle(50, "red", "circle.png");
        System.out.println();
        maker.drawRectangle(200, 100, "blue", "rect.png");

        // 需要精细控制时，客户端仍可直接使用子系统
        new ShapeRenderer().circle(5, "#00FF00");
    }
}
```

`main` 最后一行是有意为之：外观不应该是访问子系统的唯一通道。如果把子系统类都设成包私有、强迫所有人走外观，外观就会不断被加方法来满足各种特殊需求，最终膨胀成一个什么都管的“上帝对象”。

## 外观和相近模式的区别

| 对比 | 外观 | 另一方 |
| --- | --- | --- |
| vs 适配器 | 为一群对象定义**新的**简化接口 | 让一个对象符合**已有的**接口 |
| vs 中介者 | 单向：外观调用子系统，子系统不知道外观 | 双向：同事对象都知道中介者，通过它互相通信 |
| vs 代理 | 接口比子系统简单 | 接口与真实对象完全相同 |
| vs 抽象工厂 | 简化对子系统的调用 | 只负责创建对象；refactoring.guru 提到只想隐藏子系统对象的创建方式时，可以用抽象工厂代替外观 |
| vs 单例 | 外观通常只需要一个实例，因此常做成单例 | — |

外观和中介者最容易混：两者都是“一个对象站在一堆对象前面”。区别在于中介者的目的是**协调同事对象之间的交互**，同事之间不再直接通信，而是都找中介者；外观的目的是**给外部提供简单入口**，子系统内部的类之间照样可以直接通信，也完全不知道外观存在。

## 真实例子

SLF4J 的全称就是 Simple Logging Facade for Java：应用代码只调用 SLF4J 的 `Logger` 接口，底下实际用 Logback、Log4j 2 还是 `java.util.logging`，由部署时的绑定决定。它同时带有桥接的味道（抽象与实现可以独立替换），但名字里明确强调的是“外观”这一面：给多个日志框架提供一个统一的简单接口。

虚幻引擎的 `UGameplayStatics` 是一个只有静态函数的蓝图函数库，把“播放声音”“生成粒子”“获取玩家控制器”“施加径向伤害”“打开关卡”这类需要和 World、音频设备、GameMode 等多个系统打交道的常用操作包装成一个调用。它不是一个有状态的外观对象，但从“为一堆子系统提供简单入口”这一点上看，起的正是外观的作用。

## 容易踩的坑

**外观变成上帝对象。** 每来一个需求就往外观里加一个方法，最后它依赖整个系统、有几百个方法。可以按使用场景拆成多个小外观，refactoring.guru 也建议外观过大时再抽出新的外观。

**外观里写了业务逻辑。** 外观应该只做编排和委派；一旦开始在外观里做计算、做判断，它就成了一个新的子系统，却没有得到子系统应有的设计。

**强制所有访问都走外观。** 见上面代码后的说明；外观是便利，不是封锁。

**原笔记提到的“违反开闭原则”。** 子系统增加能力时，外观往往也要加方法才能把新能力暴露出去。这是外观固有的代价，只能靠“外观只暴露常用路径、其余直接访问子系统”来缓解。

## 相关

[[ACAEC-适配器模式（Adapter Pattern）]] [[ACAFJ-中介者模式（Mediator Pattern）]] [[ACAEA-代理模式（Proxy Pattern）]] [[ACADB-抽象工厂模式（Abstract Factory Pattern）]] [[ACADA-单例模式（Singleton Pattern）]]
