**过滤器模式（Filter Pattern），又叫标准模式（Criteria Pattern），把“筛选一组对象的条件”封装成独立对象，并允许用与、或、非把多个条件组合成新的条件。它不在 GoF 的 23 种模式之内，是一些教程补充的模式，思想上和 Eric Evans、Martin Fowler 提出的规格模式（Specification）基本一致。**

> 参考：Eric Evans、Martin Fowler《Specifications》（PLoP 1997，martinfowler.com/apsupp/spec.pdf）；Java 8 `java.util.function.Predicate` 的 API 文档。GoF《设计模式》里没有这一章。代码沿用原笔记的 Java。

## 为什么要有过滤器模式

筛选逻辑最初通常就是一个 `for` 循环里的 `if`：找出红色的形状、找出正方形。需求一多，问题就来了——“红色的正方形”“红色或者圆形”“不是蓝色的正方形”……每种组合写一个方法，方法数随条件组合爆炸；写成一个带一堆布尔参数的大方法，调用处又完全看不懂。更麻烦的是，筛选条件常常要在运行时由用户拼出来（搜索页的多个勾选项），根本没法在代码里提前写好每种组合。

过滤器模式的做法是把每一个基本条件做成一个对象，它们实现同一个接口；再提供 `And`、`Or`、`Not` 这几个“组合条件”，它们也实现同一个接口，内部持有别的条件对象。于是任何复杂条件都可以由基本条件搭积木一样拼出来，而且拼出来的东西本身又是一个条件，可以继续参与组合。新增一个基本条件只需要新增一个类，已有的组合逻辑完全不用动。

Evans 和 Fowler 在《Specifications》里从领域建模的角度描述了同一件事：把“某个对象是否满足某条业务规则”抽成一个规格对象，核心方法是 `isSatisfiedBy(candidate)`，同一个规格可以用来**筛选**（从集合里挑出满足的）、**校验**（检查一个对象是否合格）和**按需构造**（描述要创建什么样的对象）。规格之间也用与、或、非组合，他们称之为组合规格（Composite Specification）。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Criteria（过滤器接口） | 声明筛选方法 | `Criteria` |
| ConcreteCriteria（具体过滤器） | 一条基本条件 | `ColorCriteria`、`OutlineCriteria` |
| 组合过滤器 | 持有其他过滤器，实现逻辑与、或、非 | `AndCriteria`、`OrCriteria`、`NotCriteria` |
| 被过滤对象 | 提供条件判断所需的属性 | `Shape` |
| Client | 拼装条件并应用到集合上 | `CriteriaDemo` |

原笔记示例的类图：

![[过滤器模式-1.png]]

一个设计上的选择值得说清楚：原示例的接口是 `List<Shape> meetCriteria(List<Shape>)`，即“对整个列表做筛选”；规格模式的接口是 `boolean isSatisfiedBy(Shape)`，即“判断单个对象”。后者更好组合：`And` 就是两个布尔值取与，`Or` 取或，结果的顺序和去重都由外层的一次遍历决定。原示例的 `OrCriteria` 要把两个结果列表合并去重，用 `contains` 逐个检查，复杂度是平方级，还会改变元素顺序：先放第一个条件的结果，再补第二个的。按原代码实际运行，“红色或圆形”的结果顺序是 Shape1、Shape4、Shape5、Shape6、Shape3，原笔记列出的输出是按原列表顺序排的，和代码其实对不上。下面的实现改用单对象判断的接口。

## Java 实现：可组合的筛选条件

```java
// CriteriaDemo.java
import java.util.ArrayList;
import java.util.List;
import java.util.function.Predicate;
import java.util.stream.Collectors;

class Shape {
    private final String name;
    private final String outline;
    private final String color;

    Shape(String name, String outline, String color) {
        this.name = name;
        this.outline = outline;
        this.color = color;
    }

    String getName()    { return name; }
    String getOutline() { return outline; }
    String getColor()   { return color; }

    @Override
    public String toString() {
        return name + "[" + outline + ", " + color + "]";
    }
}

// 过滤器接口：判断单个对象，组合方法直接放在接口里
interface Criteria {
    boolean isSatisfiedBy(Shape s);

    default Criteria and(Criteria other) { return new AndCriteria(this, other); }
    default Criteria or(Criteria other)  { return new OrCriteria(this, other); }
    default Criteria not()               { return new NotCriteria(this); }

    // 对集合筛选：一次遍历，保持原顺序
    default List<Shape> meetCriteria(List<Shape> shapes) {
        List<Shape> result = new ArrayList<>();
        for (Shape s : shapes) {
            if (isSatisfiedBy(s)) result.add(s);
        }
        return result;
    }
}

class OutlineCriteria implements Criteria {
    private final String outline;
    OutlineCriteria(String outline) { this.outline = outline; }
    public boolean isSatisfiedBy(Shape s) { return s.getOutline().equalsIgnoreCase(outline); }
}

class ColorCriteria implements Criteria {
    private final String color;
    ColorCriteria(String color) { this.color = color; }
    public boolean isSatisfiedBy(Shape s) { return s.getColor().equalsIgnoreCase(color); }
}

class AndCriteria implements Criteria {
    private final Criteria a, b;
    AndCriteria(Criteria a, Criteria b) { this.a = a; this.b = b; }
    public boolean isSatisfiedBy(Shape s) { return a.isSatisfiedBy(s) && b.isSatisfiedBy(s); }
}

class OrCriteria implements Criteria {
    private final Criteria a, b;
    OrCriteria(Criteria a, Criteria b) { this.a = a; this.b = b; }
    public boolean isSatisfiedBy(Shape s) { return a.isSatisfiedBy(s) || b.isSatisfiedBy(s); }
}

class NotCriteria implements Criteria {
    private final Criteria c;
    NotCriteria(Criteria c) { this.c = c; }
    public boolean isSatisfiedBy(Shape s) { return !c.isSatisfiedBy(s); }
}

public class CriteriaDemo {
    public static void main(String[] args) {
        List<Shape> shapes = List.of(
                new Shape("Shape1", "SQUARE", "Red"),
                new Shape("Shape2", "SQUARE", "Blue"),
                new Shape("Shape3", "CIRCLE", "Blue"),
                new Shape("Shape4", "CIRCLE", "Red"),
                new Shape("Shape5", "SQUARE", "Red"));

        Criteria square = new OutlineCriteria("SQUARE");
        Criteria circle = new OutlineCriteria("CIRCLE");
        Criteria red = new ColorCriteria("RED");

        System.out.println("Red Squares:     " + red.and(square).meetCriteria(shapes));
        System.out.println("Red Or Circles:  " + red.or(circle).meetCriteria(shapes));
        System.out.println("Not Red Squares: " + square.and(red.not()).meetCriteria(shapes));

        // 同样的事用 JDK 的 Predicate 完成
        Predicate<Shape> isRed = s -> s.getColor().equalsIgnoreCase("RED");
        Predicate<Shape> isCircle = s -> s.getOutline().equalsIgnoreCase("CIRCLE");
        List<Shape> viaStream = shapes.stream()
                .filter(isRed.or(isCircle))
                .collect(Collectors.toList());
        System.out.println("Predicate 版本:   " + viaStream);
    }
}
```

`And` 和 `Or` 利用了 `&&`、`||` 的短路求值，前一个条件已经能决定结果时不再计算后一个；条件的计算代价差别很大时（比如一个要查数据库），把便宜的条件放前面。最后几行说明 Java 8 之后这个模式基本被标准库吸收了：`Predicate` 自带 `and`、`or`、`negate`，`Stream.filter` 负责遍历，除非需要给条件起名字、序列化或转换成查询语句，一般不必自己定义 `Criteria` 类层次。

## 过滤器模式和相近模式的区别

| 对比 | 过滤器 / 规格 | 另一方 |
| --- | --- | --- |
| vs 组合模式 | `AndCriteria`、`OrCriteria` 持有子条件，本身就是组合模式的应用 | 组合模式是更一般的“树形对象统一处理” |
| vs 解释器模式 | 条件对象树可以看成一个很小的布尔表达式语言的语法树 | 解释器还包含文法定义和对句子的解析 |
| vs 策略模式 | 每个条件是可替换的判断逻辑，这一点和策略相似 | 策略通常一次只用一个，不强调组合 |
| vs 责任链 / 管道-过滤器 | 一次性判断“满足与否” | 同名但含义不同：Servlet 的 `Filter` 是责任链，每个环节处理并决定是否往下传；Unix 管道式“过滤器”是对数据流逐级变换 |

最后一行值得强调：“过滤器”这个词在不同语境下指完全不同的东西，看到 `javax.servlet.Filter` 时要按 [[ACAFI-责任链模式（Chain of Responsibility Pattern）]] 理解。

## 真实例子

JPA 的 Criteria API（`CriteriaBuilder.and(...)`、`or(...)`）和 Spring Data JPA 的 `Specification<T>` 接口，都是把可组合的筛选条件翻译成 SQL 的 `WHERE` 子句，比在内存里过滤高效得多。这说明规格对象除了“在内存里判断”，还可以被“翻译”成别的形式，这也是 Evans 和 Fowler 提到规格可用于查询的原因。

## 容易踩的坑

**在内存里过滤本该交给数据库的条件。** 把十万行全取回来再用 `Criteria` 过滤，是性能事故。条件要能下推时，用能翻译成查询的规格实现。

**`Or` 合并结果时改变顺序或产生重复。** 原示例的写法就有这个问题；判断单个对象、外层一次遍历可以彻底避免。

**依赖 `equals` 去重却没重写 `equals`。** 原示例的 `contains` 默认比较引用，恰好能用；一旦对象被复制过（比如从缓存反序列化），同一个形状会被当成两个。

**条件对象持有可变状态。** 条件对象应当是不可变的，否则同一个条件被复用在多处时，一处修改会悄悄影响其他地方的筛选结果。

## 相关

[[ACAEG-组合模式（Composite Pattern）]] [[ACAFF-解释器模式（Interpreter Pattern）]] [[ACAFB-策略模式（Strategy Pattern）]] [[ACAFI-责任链模式（Chain of Responsibility Pattern）]] [[ACAFC-迭代器模式（Iterator Pattern）]]
