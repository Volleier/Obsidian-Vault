**迭代器模式（Iterator，又称游标 Cursor）是 GoF 行为型模式之一：提供一种方法顺序访问一个聚合对象中的各个元素，而又不暴露该对象的内部表示。聚合对象负责创建迭代器，迭代器负责记住遍历位置；在 Java 中它已经内建为 `java.util.Iterator` 与 `java.lang.Iterable`，for-each 循环就是它的语法糖。**

> 参考：GoF《设计模式》Iterator 一章（动机示例是 `List` 与 `ListIterator`，以及用工厂方法 `CreateIterator()` 创建迭代器）；refactoring.guru 的 Iterator 页面；JDK `java.util.Iterator` / `java.util.ArrayList` 的 API 文档。代码沿用原笔记的 Java。

## 为什么要有迭代器

遍历一个集合看起来不需要什么模式：数组用下标，链表跟着 `next` 指针走。问题在于这样写的遍历代码**知道了集合的内部结构**。哪天把数组换成链表、把链表换成树，所有遍历它的代码都得重写；想提供“倒序遍历”“只遍历满足条件的元素”，又得把这些逻辑塞进集合类，集合的接口越来越臃肿。更隐蔽的问题是，如果遍历状态（当前下标）存在集合里，同一个集合就没法同时进行两次独立的遍历。

迭代器把“遍历”从集合中分离出来，做成一个单独的对象：

- 迭代器对外只提供“还有没有下一个”“给我下一个”这类操作，调用方完全看不到集合是数组、链表还是树；
- 遍历状态存在迭代器里，同一个集合可以同时有多个迭代器，各走各的；
- 不同的遍历方式对应不同的迭代器类，集合只需提供创建它们的方法，自身接口保持简洁；
- 所有集合都提供同一种迭代器接口，于是可以写出对任意集合都有效的通用算法。

GoF 还指出，“由集合创建迭代器”这一步本身就是一个工厂方法：`List` 和 `SkipList` 各自的 `CreateIterator()` 返回各自的迭代器实现，客户端只面对抽象的 `Iterator`。

## 结构

| 参与者 | 职责 | Java 标准库中的对应 | 示例中的类 |
| --- | --- | --- | --- |
| Iterator（迭代器） | 定义访问和遍历元素的接口 | `java.util.Iterator<E>`（`hasNext`、`next`、可选的 `remove`） | 同左 |
| ConcreteIterator（具体迭代器） | 实现遍历，记录当前位置 | `ArrayList` 内部的 `Itr` 等 | `ForwardIterator`、`ReverseIterator` |
| Aggregate（聚合） | 定义创建迭代器的接口 | `java.lang.Iterable<T>`（`iterator()`） | 同左 |
| ConcreteAggregate（具体聚合） | 创建对应的具体迭代器 | `ArrayList`、`HashSet` 等 | `NameRepository` |

原笔记示例的类图，`NameRepository` 通过 `getIterator()` 返回内部类 `NameIterator`：

![[迭代器模式-1.png]]

原示例自定义了 `Iterator` 和 `Container` 接口，用来演示结构没有问题，但在 Java 里实际写代码时应该直接实现标准库的 `Iterable<T>`：这样集合就能用在 for-each 循环里，也能和 Stream、各种库方法配合。原示例 `next()` 在没有元素时返回 `null`，标准接口的约定是抛 `NoSuchElementException`。

## Java 实现：正序与倒序两种迭代器

```java
// IteratorDemo.java
import java.util.ConcurrentModificationException;
import java.util.Iterator;
import java.util.NoSuchElementException;

// 具体聚合：内部用数组存储，但不对外暴露
class NameRepository implements Iterable<String> {
    private String[] names = new String[4];
    private int size = 0;
    private int modCount = 0;   // 结构修改计数，用于快速失败

    void add(String name) {
        if (size == names.length) {
            String[] bigger = new String[size * 2];
            System.arraycopy(names, 0, bigger, 0, size);
            names = bigger;
        }
        names[size++] = name;
        modCount++;
    }

    // 默认迭代器：for-each 使用它
    @Override
    public Iterator<String> iterator() {
        return new ForwardIterator();
    }

    // 另一种遍历方式，集合接口只多一个工厂方法
    Iterable<String> reversed() {
        return ReverseIterator::new;
    }

    private class ForwardIterator implements Iterator<String> {
        private int cursor = 0;
        private final int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < size;
        }

        public String next() {
            if (modCount != expectedModCount) throw new ConcurrentModificationException();
            if (!hasNext()) throw new NoSuchElementException();
            return names[cursor++];
        }
    }

    private class ReverseIterator implements Iterator<String> {
        private int cursor = size - 1;

        public boolean hasNext() { return cursor >= 0; }

        public String next() {
            if (!hasNext()) throw new NoSuchElementException();
            return names[cursor--];
        }
    }
}

public class IteratorDemo {
    public static void main(String[] args) {
        NameRepository repo = new NameRepository();
        repo.add("Rectangle");
        repo.add("Circle");
        repo.add("Square");

        for (String name : repo) {                 // 语法糖，等价于显式调用 iterator()
            System.out.println("Name: " + name);
        }
        for (String name : repo.reversed()) {
            System.out.println("Reversed: " + name);
        }

        // 两个迭代器互不干扰
        Iterator<String> a = repo.iterator();
        Iterator<String> b = repo.iterator();
        a.next();
        System.out.println("a 的下一个: " + a.next() + ", b 的下一个: " + b.next());

        // 遍历中修改集合：快速失败
        try {
            for (String name : repo) {
                if (name.equals("Rectangle")) repo.add("Triangle");
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("遍历中修改集合 -> ConcurrentModificationException");
        }
    }
}
```

`modCount` 的做法是照着 JDK 的 `ArrayList` 写的：集合每次结构性修改都加一，迭代器创建时记下当时的值，每次 `next()` 都核对，不一致就立即抛异常。JDK 文档把这叫作快速失败（fail-fast），并明确说明这只是尽力而为的检测，不能依赖它来保证并发正确性。`reversed()` 返回一个 lambda，是因为 `Iterable` 只有一个抽象方法，`ReverseIterator::new` 正好符合它的签名。

## 内部迭代器与外部迭代器

GoF 把迭代器分成两类：

| 类型 | 谁控制遍历 | Java 中的形式 | 特点 |
| --- | --- | --- | --- |
| 外部迭代器 | 客户端逐个调用 `next()` | `Iterator`、for-each | 灵活：可以随时停止、同时推进两个迭代器做比较 |
| 内部迭代器 | 迭代器自己遍历，客户端提供要对每个元素执行的操作 | `Iterable.forEach(Consumer)`、`Stream` | 简洁，便于并行化；但不能中途 `break` |

Java 8 引入的 Stream 属于内部迭代：客户端只描述“做什么”（过滤、映射、归约），遍历方式甚至是否并行都由库决定。

## 迭代器和相近模式的关系

| 模式 | 关系 |
| --- | --- |
| 组合模式 | 常用迭代器遍历组合树，把深度优先、广度优先等遍历顺序从树结构中分离出去 |
| 工厂方法 | 聚合的 `iterator()` 就是工厂方法，由各个集合子类返回自己的迭代器 |
| 访问者 | 迭代器负责“走到每个元素”，访问者负责“对每种元素做什么”，两者常一起使用 |
| 备忘录 | 可以用备忘录保存迭代状态，GoF 在备忘录一章提到了这种组合 |

## 真实例子

Java 集合框架整体建立在这个模式上，`Map` 本身不是 `Iterable`，但它的 `keySet()`、`values()`、`entrySet()` 视图都是。`java.util.Scanner` 实现了 `Iterator<String>`，把输入流当成词法单元序列来遍历。

虚幻引擎的容器同样提供迭代器：`TArray`、`TMap`、`TSet` 支持 C++ 的范围 for 循环，也提供 `CreateIterator()` / `CreateConstIterator()` 返回可以在遍历中安全删除当前元素的迭代器（`It.RemoveCurrent()`）。遍历世界里的对象则有 `TActorIterator<T>`（遍历某个 World 中指定类型的 Actor）和 `TObjectIterator<T>`（遍历内存中所有该类型的 UObject），它们隐藏了引擎内部对象表的组织方式。

## 容易踩的坑

**遍历中修改集合。** Java 会抛 `ConcurrentModificationException`（单线程下也会，常见于在 for-each 里 `remove`）；需要边遍历边删除时，用迭代器自己的 `remove()`，或者 `removeIf`。UE 的 `TArray` 在范围 for 中增删元素，非 Shipping 构建下会触发检查，删除请用迭代器的 `RemoveCurrent()` 或倒序下标循环。

**`next()` 不先检查 `hasNext()`。** 标准约定是越界时抛 `NoSuchElementException`；原示例返回 `null` 的做法会把错误推迟到后面的空指针。

**迭代器被当成可重用对象。** 迭代器走完就用完了，不能“重置”，要再遍历就再向集合要一个新的。

**误以为快速失败能保证线程安全。** 它只是尽力检测，多线程下仍需要同步或使用并发集合（如 `CopyOnWriteArrayList` 的迭代器遍历的是创建时的快照）。

## 相关

[[ACAEG-组合模式（Composite Pattern）]] [[ACADC-工厂模式（Factory Pattern）]] [[ACAFD-访问者模式（Visitor Pattern）]] [[ACAFA-备忘录模式（Memento Pattern）]] [[ACAEB-过滤器模式（Filter Pattern）]]
