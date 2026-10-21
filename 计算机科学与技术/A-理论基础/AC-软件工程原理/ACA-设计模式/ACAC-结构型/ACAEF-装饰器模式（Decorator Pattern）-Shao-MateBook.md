**装饰器模式（Decorator，又称包装器 Wrapper）是 GoF 结构型模式之一：动态地给一个对象添加额外的职责，就增加功能来说比生成子类更灵活。装饰器与被装饰对象实现同一个接口，内部持有被装饰对象，在转发调用的前后加入自己的行为；多个装饰器可以层层嵌套。**

> 参考：GoF《设计模式》Decorator 一章（动机示例是给文本视图 `TextView` 加边框 `BorderDecorator` 和滚动条 `ScrollDecorator`）；refactoring.guru 的 Decorator 页面；JDK `java.io` 包的 API 文档。代码沿用原笔记的 Java。

## 为什么要有装饰器

给一个类加功能，最直接的办法是继承。问题出在功能需要**自由组合**的时候。GoF 的例子是界面上的文本视图：有时要加边框，有时要加滚动条，有时两个都要。用继承就得写 `BorderedTextView`、`ScrollableTextView`、`BorderedScrollableTextView`；再加一个“阴影”，组合数翻倍。n 个可选功能对应 2ⁿ 个子类，这就是所谓的“子类爆炸”。而且继承是静态的：一个对象创建出来是什么类就是什么类，没法在运行时给它加上或去掉滚动条。

装饰器的做法是把每个可选功能做成一个独立的包装类。`BorderDecorator` 实现和 `TextView` 相同的接口，内部持有一个“被装饰的组件”，它的 `draw()` 先让内部组件画自己，再画一圈边框。因为装饰器本身也实现了组件接口，它可以被另一个装饰器再包一层：`new BorderDecorator(new ScrollDecorator(textView))`。n 个功能只需要 n 个装饰器类，组合方式由客户端在运行时决定。

对客户端来说，被装饰过的对象和原对象没有区别——都是那个接口。这叫透明装饰：客户端既不知道也不需要知道自己手上的对象被包了几层。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Component（抽象组件） | 被装饰对象和装饰器的公共接口 | `Shape` |
| ConcreteComponent（具体组件） | 被装饰的原始对象 | `Circle`、`Rectangle` |
| Decorator（抽象装饰器） | 实现 Component，持有一个 Component 引用，默认把所有调用转发过去 | `ShapeDecorator` |
| ConcreteDecorator（具体装饰器） | 在转发前后加入额外行为 | `RedBorderDecorator`、`ShadowDecorator` |

抽象装饰器这一层的作用常被忽视：接口有很多方法时，它负责把每个方法都原样转发，具体装饰器只覆写自己关心的那一两个方法。没有这一层，每个具体装饰器都得把全部方法手写一遍转发。

原笔记示例的类图：

![[装饰器模式-1.png]]

## Java 实现：可叠加的边框和阴影

在原笔记的红色边框装饰器之外，再加一个阴影装饰器和一个尺寸查询方法，演示多层嵌套和“装饰器可以修改返回值”：

```java
// DecoratorDemo.java
interface Shape {
    void draw();
    int area();   // 用于演示装饰器也能修改返回值
}

class Circle implements Shape {
    public void draw() { System.out.println("Shape: Circle"); }
    public int area()  { return 314; }
}

class Rectangle implements Shape {
    public void draw() { System.out.println("Shape: Rectangle"); }
    public int area()  { return 200; }
}

// 抽象装饰器：默认把所有方法转发给被装饰对象
abstract class ShapeDecorator implements Shape {
    protected final Shape decorated;

    protected ShapeDecorator(Shape decorated) {
        this.decorated = decorated;
    }

    public void draw() { decorated.draw(); }
    public int area()  { return decorated.area(); }
}

// 具体装饰器一：画完后加红色边框
class RedBorderDecorator extends ShapeDecorator {
    RedBorderDecorator(Shape s) { super(s); }

    @Override
    public void draw() {
        decorated.draw();
        System.out.println("  + Border Color: Red");
    }
}

// 具体装饰器二：先画阴影，阴影让占用面积变大
class ShadowDecorator extends ShapeDecorator {
    ShadowDecorator(Shape s) { super(s); }

    @Override
    public void draw() {
        System.out.println("  + Shadow");
        decorated.draw();
    }

    @Override
    public int area() {
        return decorated.area() + 20;
    }
}

public class DecoratorDemo {
    public static void main(String[] args) {
        Shape plain = new Circle();
        Shape red = new RedBorderDecorator(new Circle());
        // 两层装饰，顺序由客户端决定
        Shape redWithShadow = new ShadowDecorator(new RedBorderDecorator(new Rectangle()));

        System.out.println("普通圆:");
        plain.draw();
        System.out.println("红边圆:");
        red.draw();
        System.out.println("带阴影的红边矩形, area=" + redWithShadow.area() + ":");
        redWithShadow.draw();
    }
}
```

`redWithShadow` 的输出是先阴影、再矩形、再边框：外层装饰器的“前置行为”最先执行，“后置行为”最后执行，调用顺序像洋葱一样一层层进去再一层层出来。把两层装饰的顺序对调，输出顺序也会变。原笔记示例里声明 `ShapeDecorator redCircle = ...` 用的是装饰器类型，写成 `Shape redCircle = ...` 更好（原示例注释里也给了这种写法），客户端只应依赖组件接口。

## 装饰器和相近模式的区别

| 对比 | 装饰器 | 另一方 |
| --- | --- | --- |
| vs 继承 | 运行时组合，n 个功能 n 个类 | 编译期固定，功能组合需要 2ⁿ 个子类 |
| vs 代理 | 由客户端创建并组合，目的是增加职责，常多层嵌套 | 通常由代理自己管理真实对象，目的是控制访问 |
| vs 适配器 | 接口不变 | 改变接口 |
| vs 组合模式 | 只有一个子组件，给它加职责 | 可以有多个子组件，目的是聚合；refactoring.guru 指出装饰器可以看成只有一个子组件的组合 |
| vs 策略 | 改变对象的“外皮”，从外面包 | 改变对象的“内核”，替换内部算法；GoF 用这个比喻区分两者 |
| vs 责任链 | 每一层都会执行，并继续往里转发 | 某一环可以处理后终止传递 |

装饰器和责任链的结构几乎一样，都是一串对象依次转发请求。refactoring.guru 给出的区别是：责任链的处理者可以各自独立执行任意操作，也可以随时停止传递；装饰器扩展的是同一个对象的行为，并且不能打断整个流程。

## 真实例子

`java.io` 是装饰器的教科书实现。`InputStream` 是组件接口，`FileInputStream` 是具体组件，`FilterInputStream` 是抽象装饰器（它把所有方法转发给内部的流），`BufferedInputStream`、`DataInputStream`、`GZIPInputStream` 等是具体装饰器。于是可以写 `new DataInputStream(new BufferedInputStream(new FileInputStream("a.bin")))`，给文件流叠加缓冲和按类型读取的能力。字符流那一侧的 `BufferedReader` 包 `Reader` 也是同样的结构。

`java.util.Collections.unmodifiableList(list)`、`synchronizedList(list)` 返回的也是装饰器：接口仍是 `List`，一个把修改操作改成抛异常，一个给每个方法加上同步。

## 容易踩的坑

**对象身份被改变。** 装饰后的对象不再 `==` 原对象，`equals`、`hashCode` 默认也不相等；原来存进 `HashSet` 的对象换成装饰后的版本就找不到了。依赖对象身份的代码要格外小心。

**`instanceof` 和强制转换失效。** 被装饰后，`shape instanceof Circle` 变成 `false`。如果客户端需要判断具体类型，说明它并没有真正依赖接口，装饰器在这里会出问题。

**装饰顺序影响结果。** 先压缩再加密和先加密再压缩结果完全不同；Java IO 里把 `BufferedInputStream` 放在 `GZIPInputStream` 里面还是外面，性能也不一样。顺序敏感时应当用工厂或建造者统一组装，而不是让每个调用方自己拼。

**抽象装饰器漏转发方法。** 接口新增方法后，抽象装饰器如果没有同步加上转发，Java 8 的默认方法会让代码照样编译通过，但装饰器会走默认实现而不是被装饰对象的实现，行为悄悄出错。

**层数过多难以调试。** 多层装饰的调用栈很深，每一层都叫 `draw()`，排查时很难看出是哪一层出了问题。

## 相关

[[ACAEA-代理模式（Proxy Pattern）]] [[ACAEC-适配器模式（Adapter Pattern）]] [[ACAEG-组合模式（Composite Pattern）]] [[ACAFB-策略模式（Strategy Pattern）]] [[ACAFI-责任链模式（Chain of Responsibility Pattern）]]
