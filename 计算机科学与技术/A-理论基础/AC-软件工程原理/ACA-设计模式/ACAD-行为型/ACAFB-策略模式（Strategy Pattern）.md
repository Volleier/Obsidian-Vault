**策略模式（Strategy，又称政策 Policy）是 GoF 行为型模式之一：定义一系列算法，把它们一个个封装起来，并且使它们可以互相替换，让算法的变化独立于使用算法的客户。结构上是一个上下文（Context）持有一个策略接口的引用，把某项工作委派给当前的策略对象。**

> 参考：GoF《设计模式》Strategy 一章（动机示例是文本排版中的换行算法：`Composition` 把断行工作委派给 `SimpleCompositor`、`TeXCompositor`、`ArrayCompositor` 等不同的 `Compositor`）；refactoring.guru 的 Strategy 页面。代码沿用原笔记的 Java。

## 为什么要有策略

同一件事常常有多种做法。GoF 的例子是文本编辑器的断行：简单的贪心算法一行一行往下排，速度快；TeX 的算法考虑整段的美观，效果好但慢；还有一种按固定数量排版的算法，用于图标阵列。如果把这些算法都写进 `Composition` 类里，用 `if/switch` 选择，这个类会越来越大，加一种算法就要改它，而且不用的算法也一直跟着它。

策略模式把“做法”从“做事的对象”里拿出去：定义一个 `Compositor` 接口，每种断行算法是一个实现类，`Composition` 只持有一个 `Compositor` 引用，需要断行时调用它。换算法就是换一个对象，可以在构造时注入，也可以在运行时切换。上下文和算法之间只通过接口通信，于是：

- 新增算法只需新增一个类，上下文不用改；
- 算法可以单独测试，也可以在多个上下文之间复用；
- 条件分支从上下文里消失了——它被“选哪个策略对象”这一次决定取代，而这个决定通常发生在组装阶段。

原笔记说策略模式解决的是“多种相似算法存在时，使用 if...else 导致的复杂和难以维护”，这是准确的。需要补充的是，条件判断并没有消失，只是从“每次执行时判断”变成了“创建时选一次”。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Strategy（抽象策略） | 所有支持的算法的公共接口 | `Operation` |
| ConcreteStrategy（具体策略） | 实现某个具体算法 | `Add`、`Subtract`、`Multiply` |
| Context（上下文） | 持有一个策略引用，把工作委派给它；可以向策略提供所需数据 | `Calculator` |
| Client（客户端） | 创建具体策略并交给上下文 | `StrategyDemo` |

上下文怎么把数据给策略有两种办法：把数据作为参数传进去，或者把上下文自己传给策略、让策略按需回调取数据。前者耦合低但参数可能很多，后者灵活但策略就依赖了上下文的接口，GoF 对此有专门讨论。

原笔记示例的类图：

![[策略模式-1.png]]

## Java 实现：可切换的运算策略

原笔记用加减乘三种运算作为策略。下面保留这个例子，补上运行时切换，并说明 Java 8 之后策略为什么经常直接写成 lambda：

```java
// StrategyDemo.java
import java.util.Map;
import java.util.function.IntBinaryOperator;

// 抽象策略：只有一个方法的接口，可以用 lambda 实现
@FunctionalInterface
interface Operation {
    int apply(int a, int b);
}

class Add implements Operation {
    public int apply(int a, int b) { return a + b; }
}

class Subtract implements Operation {
    public int apply(int a, int b) { return a - b; }
}

class Multiply implements Operation {
    public int apply(int a, int b) { return a * b; }
}

// 上下文：持有策略，可在运行时替换
class Calculator {
    private Operation operation;

    Calculator(Operation operation) { this.operation = operation; }

    void setOperation(Operation operation) { this.operation = operation; }

    int execute(int a, int b) { return operation.apply(a, b); }
}

public class StrategyDemo {
    public static void main(String[] args) {
        Calculator calc = new Calculator(new Add());
        System.out.println("10 + 5 = " + calc.execute(10, 5));

        calc.setOperation(new Subtract());          // 运行时切换
        System.out.println("10 - 5 = " + calc.execute(10, 5));

        calc.setOperation(new Multiply());
        System.out.println("10 * 5 = " + calc.execute(10, 5));

        // 策略是无状态的单方法对象时，lambda 就够了
        calc.setOperation((a, b) -> Math.floorMod(a, b));
        System.out.println("10 mod 5 = " + calc.execute(10, 5));

        // 按名字选择策略：条件分支集中到一张表里
        Map<String, IntBinaryOperator> table = Map.of(
                "max", Math::max,
                "min", Math::min);
        System.out.println("max(10, 5) = " + table.get("max").applyAsInt(10, 5));
    }
}
```

最后的映射表展示了“选择策略”这件事通常放在哪里：客户端根据配置或输入查表，拿到策略后交给上下文。原示例每次换运算都 `new` 一个新的 `Context`，那样也能工作，但体现不出“同一个上下文在运行时换策略”这一点。

## 策略和相近模式的区别

| 对比 | 策略 | 另一方 |
| --- | --- | --- |
| vs 状态 | 策略由客户端选定，策略之间互不知道，通常不自行切换 | 状态对象知道其他状态，并在内部触发转移；客户端往往不直接选状态 |
| vs 模板方法 | 基于组合，运行时整体替换算法 | 基于继承，编译期固定骨架、子类改写其中某些步骤 |
| vs 命令 | 描述“同一件事的不同做法” | 把“一个请求”封装成对象，用于排队、记录、撤销 |
| vs 装饰器 | 改变对象的内核（内部算法） | 改变对象的外皮（从外面包一层）；GoF 用这个比喻区分 |
| vs 桥接 | 关注算法替换，上下文通常是一个类 | 关注两个类层次的分离，抽象一侧也有自己的子类 |

策略与状态的对比最常被问到。两者的类图几乎一样，区别在谁来切换、策略之间是否互相知道。refactoring.guru 把状态看作策略的扩展：状态模式中的具体状态可以相互知道并主动切换，策略则几乎从不知道其他策略的存在。

## 真实例子

`java.util.Comparator` 是 Java 里最常见的策略：`Collections.sort(list, comparator)`、`list.sort(Comparator.comparing(Shape::getArea))` 中，排序算法是上下文，“如何比较两个元素”是可替换的策略。`java.util.concurrent.ThreadPoolExecutor` 的拒绝策略 `RejectedExecutionHandler` 也是：`AbortPolicy`（抛异常）、`CallerRunsPolicy`（由提交任务的线程自己执行）、`DiscardPolicy`、`DiscardOldestPolicy` 在任务队列满时提供不同的处理方式，构造线程池时选定。

## 容易踩的坑

**策略太少也拆成类。** 只有两种写法、而且以后也不会变时，一个 `if` 比一个接口加两个类清楚。原笔记提到的“策略类数量增多”在这种情况下就是纯粹的负担。

**客户端必须了解所有策略才能选择。** 策略模式把选择责任交给了客户端，客户端就得知道每个策略的区别。可以用工厂或映射表把选择逻辑集中起来。

**有状态的策略被共享。** 策略对象如果内部保存了中间结果，又被多个上下文或多个线程共用，会互相干扰。策略最好无状态；需要状态时每个上下文各持一份。

**上下文与策略的接口过宽。** 为了照顾某个复杂策略，把一大堆参数塞进策略接口，简单策略也被迫接收它们用不到的参数。

## 相关

[[ACAFK-状态模式（State Pattern）]] [[ACAFH-模板模式（Template Pattern）]] [[ACAFG-命令模式（Command Pattern）]] [[ACAEF-装饰器模式（Decorator Pattern）]] [[ACAEH-桥接模式（Bridge Pattern）]] [[ACAEB-过滤器模式（Filter Pattern）]]
