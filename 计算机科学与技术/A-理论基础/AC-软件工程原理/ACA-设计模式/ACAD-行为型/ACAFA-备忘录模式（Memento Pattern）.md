**备忘录模式（Memento，又称快照 Snapshot 或 Token）是 GoF 行为型模式之一：在不破坏封装性的前提下，捕获一个对象的内部状态并保存在对象之外，以便之后把对象恢复到这个状态。三个角色是原发器（Originator，被保存的对象）、备忘录（Memento，状态快照）和负责人（Caretaker，保管快照但不查看其内容）。**

> 参考：GoF《设计模式》Memento 一章（动机示例是图形编辑器里维持连接关系的约束求解器 `ConstraintSolver`，撤销移动操作时需要恢复它的内部状态）；refactoring.guru 的 Memento 页面。代码沿用原笔记的 Java。

## 为什么要有备忘录

撤销（Undo）是最典型的需求：用户做了一串编辑，想回到之前的某一步。要回去，就得在之前把对象的状态存下来。问题在于“谁来存、存什么”。

最直接的做法是让外部代码把对象的字段一个个读出来存好，需要时再一个个写回去。这要求对象把所有内部状态都通过 getter/setter 公开，封装就被打破了：任何人都能随意改它的内部字段，而且一旦对象增加一个字段，所有“保存/恢复”的外部代码都得跟着改。GoF 的例子更能说明问题：图形编辑器里维护连线关系的约束求解器内部有大量复杂状态，这些状态本来就不该公开，但撤销“移动图形”操作时又必须把求解器恢复到移动前的样子。

备忘录的做法是**让对象自己给自己拍快照**。原发器提供 `save()`，把需要的内部状态装进一个备忘录对象交出去；提供 `restore(memento)`，从备忘录里取回状态。备忘录对外几乎什么都不暴露，负责人（撤销栈）只能保管它、把它交还给原发器，看不到也改不了里面的内容。于是状态的保存和恢复逻辑留在原发器内部，封装没有被破坏，撤销历史的管理又放在了原发器之外。

GoF 把这称为备忘录的**双接口**：对负责人是窄接口（只能传递），对原发器是宽接口（能读取全部状态）。在 Java 里，最自然的实现方式是把备忘录做成原发器的静态嵌套类，字段私有，只有外部类能访问。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Originator（原发器） | 创建记录自身当前状态的备忘录；用备忘录恢复状态 | `ShapeEditor` |
| Memento（备忘录） | 保存原发器的内部状态；除原发器外不让别人访问 | `ShapeEditor.Snapshot` |
| Caretaker（负责人） | 保存备忘录，决定何时保存、何时恢复，但不操作也不检查备忘录内容 | `History` |

原笔记示例的类图：

![[备忘录模式-1.png]]

原示例的 `Memento` 是一个公开类，带公开的构造函数和 `getState()`，任何人都能读它、造它，窄接口没有体现出来；`CareTaker` 按下标 `get(index)` 取快照，也更像“存档列表”而不是撤销栈。下面把备忘录收进原发器内部，负责人改成撤销栈。

## Java 实现：带撤销的形状编辑器

```java
// MementoDemo.java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

// 原发器：状态全部私有
class ShapeEditor {
    private String shape = "Circle";
    private int x, y;
    private final List<String> tags = new ArrayList<>();   // 可变状态，快照时要复制

    void moveTo(int x, int y)  { this.x = x; this.y = y; }
    void setShape(String s)    { this.shape = s; }
    void addTag(String t)      { tags.add(t); }

    @Override
    public String toString() {
        return shape + " at (" + x + "," + y + ") tags=" + tags;
    }

    // 备忘录：嵌套类，字段私有，外部拿到也读不了
    static final class Snapshot {
        private final String shape;
        private final int x, y;
        private final List<String> tags;

        private Snapshot(String shape, int x, int y, List<String> tags) {
            this.shape = shape;
            this.x = x;
            this.y = y;
            this.tags = List.copyOf(tags);   // 不可变副本
        }
    }

    Snapshot save() {
        return new Snapshot(shape, x, y, tags);
    }

    void restore(Snapshot s) {
        this.shape = s.shape;
        this.x = s.x;
        this.y = s.y;
        this.tags.clear();
        this.tags.addAll(s.tags);
    }
}

// 负责人：只保管快照，不关心内容
class History {
    private final Deque<ShapeEditor.Snapshot> undoStack = new ArrayDeque<>();

    void push(ShapeEditor.Snapshot s) { undoStack.push(s); }

    boolean undo(ShapeEditor editor) {
        if (undoStack.isEmpty()) return false;
        editor.restore(undoStack.pop());
        return true;
    }
}

public class MementoDemo {
    public static void main(String[] args) {
        ShapeEditor editor = new ShapeEditor();
        History history = new History();

        history.push(editor.save());          // 每次修改前保存
        editor.moveTo(10, 20);

        history.push(editor.save());
        editor.setShape("Square");
        editor.addTag("selected");

        history.push(editor.save());
        editor.moveTo(99, 99);

        System.out.println("当前:   " + editor);
        history.undo(editor);
        System.out.println("撤销 1: " + editor);
        history.undo(editor);
        System.out.println("撤销 2: " + editor);
        history.undo(editor);
        System.out.println("撤销 3: " + editor);
    }
}
```

`Snapshot` 的构造函数和字段都是私有的，`History` 只能持有和传递它，这就是 Java 里对“窄接口”最接近的表达（严格说，同一个顶层类里的代码仍然能访问私有成员，这是 Java 嵌套类的访问规则）。`tags` 在快照时用 `List.copyOf` 复制了一份：如果快照直接引用原发器的列表，之后对列表的修改会同时改掉历史快照，撤销就失效了。

## 备忘录和相近模式的区别

| 对比 | 备忘录 | 另一方 |
| --- | --- | --- |
| vs 命令 | 保存“状态”，撤销时整体恢复 | 保存“操作”，撤销时执行逆操作；两者常配合：命令执行前取备忘录，撤销时用它恢复 |
| vs 原型 | 快照只能用于恢复原发器，不是独立可用的对象 | 克隆出一个完整的新对象；原发器状态简单时可以直接用克隆代替备忘录 |
| vs 迭代器 | — | GoF 提到可以用备忘录保存迭代状态，让迭代器的内部状态不暴露给客户端 |

用“存状态”还是“存逆操作”实现撤销，是一个实际的取舍。存状态实现简单、不会出错，但状态大时内存开销高；存逆操作省内存，但每种操作都要正确写出逆操作，有些操作（如有损的图像滤镜）根本没有逆操作。很多编辑器两者混用：大多数操作用命令记录逆操作，没有逆操作的再退回到存快照。

## 真实例子

虚幻引擎编辑器的撤销/重做系统就是按“存状态”来做的。编辑器代码在修改对象前先开一个事务（C++ 里常用 `FScopedTransaction`），然后调用对象的 `UObject::Modify()`；`Modify()` 内部调用 `SaveToTransactionBuffer`，官方 API 文档的描述是：如果当前正在录制事务，就把这个对象的一份副本存进事务缓冲区。撤销时引擎用这些存下来的数据把对象恢复。这里对象通过自己的序列化逻辑决定保存哪些内容，事务缓冲区只负责保管，角色分工和备忘录一致。对象需要带 `RF_Transactional` 标志才会被记录，这也是自定义编辑器工具“改了却撤销不了”的常见原因。

## 容易踩的坑

**快照共享了可变状态。** 见代码后的说明。浅拷贝的快照是备忘录里最常见的 bug。

**内存无限增长。** 每次操作都存一份完整状态，长时间编辑后撤销栈会非常大。常见的缓解办法是限制撤销步数、只存增量，或只存发生变化的那部分对象。

**快照时机不对。** 应当在修改**之前**保存。修改后才保存，撤销只会回到当前状态。

**把备忘录做成公开的数据类。** 原示例就是这样，外部代码可以随意读取甚至伪造备忘录，封装形同虚设。

**恢复时忽略了外部关联。** 对象状态恢复了，但它注册的监听器、它引用的其他对象、界面上的显示没有同步更新，看起来像是撤销失败。

## 相关

[[ACAFG-命令模式（Command Pattern）]] [[ACADE-原型模式（Prototype Pattern）]] [[ACAFC-迭代器模式（Iterator Pattern）]] [[ACAFK-状态模式（State Pattern）]]
