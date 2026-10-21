**原型模式（Prototype）是 GoF 创建型模式之一：用一个已有对象（原型）指定要创建的对象种类，通过复制这个原型来得到新对象，而不是通过 `new` 具体类。核心是原型接口上的 `clone()` 方法，常配合一个按名字登记原型的注册表使用。**

> 参考：GoF《设计模式》Prototype 一章（动机示例是乐谱编辑器里的 `GraphicTool` 用原型创建音符等图形）；refactoring.guru 的 Prototype 页面；Joshua Bloch《Effective Java》第 3 版 Item 13（谨慎地覆盖 clone）。代码沿用原笔记的 Java。

## 为什么要有原型

用 `new` 创建对象要满足两个前提：写代码时知道具体类，并且知道如何把它初始化到想要的状态。原型模式针对的是这两个前提不成立或代价很高的情况。

**不知道具体类。** GoF 的例子是一个通用图形编辑框架：工具栏上的“画音符”工具属于框架代码，框架不认识乐谱应用里的 `WholeNote`、`HalfNote` 类。与其为每种音符写一个 `GraphicTool` 子类（那会形成一套和产品类层次平行的创建者层次），不如给工具一个原型对象，点击时就复制它。框架只需要知道“这东西能 `clone()`”。

**初始化很贵或很繁琐。** 对象的初始状态来自一次数据库查询、一次文件解析，或是用户在编辑器里一项项调出来的配置。已经有了一个配好的对象，复制它比重新走一遍初始化便宜得多。原笔记举的“在代价高昂的数据库操作之后缓存对象，下次请求返回它的克隆”就是这种情况。

**状态组合只有有限几种。** 一个类的实例只会出现在少数几种配置下时，预先做好这几个原型放进注册表，比每次按参数构造更直接。

代价是每个具体类都要正确实现复制，而“正确复制”在对象含有引用、循环引用或外部资源时并不简单，这是原型模式真正的难点。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Prototype（原型） | 声明克隆自身的接口 | `Shape`（`clone()`） |
| ConcretePrototype（具体原型） | 实现克隆，决定哪些字段深拷贝、哪些共享 | `Circle`、`Rectangle` |
| Client（客户端） | 让原型克隆自身来得到新对象 | `PrototypeDemo` |
| Prototype Registry（原型注册表，可选） | 按键保存常用原型，取出时返回克隆 | `ShapeCache` |

原笔记示例的类图，`ShapeCache` 缓存形状，`PrototypePatternDemo` 从中取克隆：

![[原型模式-1.png]]

## 浅拷贝与深拷贝

复制一个对象时，基本类型和不可变对象（`String`、`Integer`）直接复制值就行；麻烦在可变的引用字段。

| 方式 | 引用字段的处理 | 结果 |
| --- | --- | --- |
| 浅拷贝 | 只复制引用，新旧对象指向同一个子对象 | 改克隆体的子对象，原型也跟着变 |
| 深拷贝 | 递归复制子对象 | 完全独立，但成本高，还要处理循环引用 |

Java 的 `Object.clone()` 默认做浅拷贝：它按字段逐个复制，引用字段只复制引用。因此只要类里有 `List`、数组或可变的自定义对象，覆写 `clone()` 时就得手动把这些字段再复制一遍。原笔记示例里的 `Shape` 只有 `String` 字段，浅拷贝恰好够用，所以看不出问题；下面的示例给形状加了一个可变的标签列表，演示必须深拷贝的地方。

## Java 实现：带注册表的形状原型

```java
// PrototypeDemo.java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

abstract class Shape implements Cloneable {
    private String id;
    protected String type;
    private List<String> tags = new ArrayList<>();   // 可变引用字段

    abstract void draw();

    public String getType() { return type; }
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public List<String> getTags() { return tags; }

    // 协变返回类型：调用方不用强转
    @Override
    public Shape clone() {
        try {
            Shape copy = (Shape) super.clone();       // 浅拷贝全部字段
            copy.tags = new ArrayList<>(this.tags);   // 可变字段手动深拷贝
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);              // 实现了 Cloneable，不会发生
        }
    }
}

class Circle extends Shape {
    Circle() { type = "Circle"; }
    void draw() { System.out.println("Circle::draw() tags=" + getTags()); }
}

class Rectangle extends Shape {
    Rectangle() { type = "Rectangle"; }
    void draw() { System.out.println("Rectangle::draw() tags=" + getTags()); }
}

// 原型注册表：存原型，取克隆
class ShapeCache {
    private static final Map<String, Shape> PROTOTYPES = new HashMap<>();

    static void loadCache() {
        // 假设这些原型的初始化代价很高（查库、解析文件）
        Circle circle = new Circle();
        circle.setId("1");
        circle.getTags().add("round");
        PROTOTYPES.put(circle.getId(), circle);

        Rectangle rect = new Rectangle();
        rect.setId("2");
        rect.getTags().add("box");
        PROTOTYPES.put(rect.getId(), rect);
    }

    static Shape getShape(String id) {
        Shape proto = PROTOTYPES.get(id);
        if (proto == null) throw new IllegalArgumentException("没有原型: " + id);
        return proto.clone();
    }
}

public class PrototypeDemo {
    public static void main(String[] args) {
        ShapeCache.loadCache();

        Shape c1 = ShapeCache.getShape("1");
        c1.getTags().add("selected");     // 只改克隆体
        c1.draw();

        Shape c2 = ShapeCache.getShape("1");
        c2.draw();                        // 原型未被污染
        System.out.println("两次克隆是不同对象: " + (c1 != c2));
    }
}
```

把 `copy.tags = new ArrayList<>(this.tags)` 这一行删掉再运行，第二次取出的圆也会带上 `selected`：两个克隆体和原型共用同一个列表。这就是浅拷贝最常见的事故现场。

## 原型和相近模式的区别

| 对比 | 原型 | 另一方 |
| --- | --- | --- |
| vs 工厂方法 | 不需要为每个产品写一个创建者子类，靠复制对象 | 靠子类覆写创建方法；refactoring.guru 指出原型不依赖继承，但需要复杂的克隆初始化 |
| vs 抽象工厂 | 用一组原型对象代替一族工厂类，可在运行时注册新原型 | 产品族在编译期由工厂类确定 |
| vs 备忘录 | 克隆出来的是一个可以独立使用的新对象 | 备忘录只是状态快照，用于恢复原发器；对象简单时可以用原型代替备忘录 |
| vs 享元 | 每次得到一个独立副本 | 所有人共享同一个对象 |

## 真实例子

Java 里除了 `Cloneable`，更推荐的复制手段是**复制构造器**或**复制工厂**（如 `new ArrayList<>(other)`）。《Effective Java》Item 13 的结论是：`Cloneable` 的设计有缺陷——它是一个没有方法的标记接口，`clone()` 在 `Object` 上且是 `protected`，还会绕过构造函数，`final` 字段也没法在 `clone()` 里重新赋值；新代码最好用复制构造器，只有数组适合用 `clone()`。

虚幻引擎的对象系统到处是原型思想。每个 `UClass` 都有一个类默认对象（CDO），新建实例时属性默认值就是从 CDO 复制来的；`NewObject` 的 `Template` 参数可以指定另一个对象作为模板，源码注释写明：指定时会把该对象的属性值复制到新对象上，并把新对象的 `ObjectArchetype` 设为它，传 `nullptr` 则使用 CDO。编辑器里放进关卡的 Actor、蓝图里的组件模板，都是以这种“原型（Archetype）→ 实例”的关系组织的。需要完整复制一个对象及其子对象时，可以用 `DuplicateObject`。

## 容易踩的坑

**浅拷贝共享可变状态。** 见上面的示例；集合、数组、可变的日期对象都要特别处理。

**深拷贝遇到循环引用。** 父节点引用子节点、子节点又引用父节点时，天真的递归复制会无限循环。需要一个“原对象 → 已复制对象”的映射表来去重。

**克隆体带上了不该复制的身份。** 数据库主键、唯一 ID、监听器列表、线程或文件句柄，复制过去往往是错的，克隆后要重置。

**忘了 `Cloneable` 就抛异常。** 没实现 `Cloneable` 却调用 `super.clone()`，会抛 `CloneNotSupportedException`；原笔记示例把它 `printStackTrace()` 后返回 `null`，调用方随后空指针，更难排查。

## 相关

[[ACADC-工厂模式（Factory Pattern）]] [[ACADB-抽象工厂模式（Abstract Factory Pattern）]] [[ACADD-建造者模式（Builder Pattern）]] [[ACAFA-备忘录模式（Memento Pattern）]] [[ACAEE-享元模式（Flyweight Pattern）]] [[ACAEG-组合模式（Composite Pattern）]]
