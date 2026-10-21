**命令模式（Command，又称动作 Action 或事务 Transaction）是 GoF 行为型模式之一：将一个请求封装为一个对象，从而可以用不同的请求对客户进行参数化，对请求排队或记录请求日志，以及支持可撤销的操作。参与者是命令接口、具体命令、接收者（真正干活的对象）和调用者（触发命令的对象）。**

> 参考：GoF《设计模式》Command 一章（动机示例是图形界面工具包里的菜单项 `MenuItem`，点击时执行一个 `Command` 对象，如 `PasteCommand`、`OpenCommand`）；refactoring.guru 的 Command 页面。代码沿用原笔记的 Java。

## 为什么要有命令

GoF 的例子是界面工具包：工具包提供菜单、按钮这些控件，但它不可能知道点击某个菜单项之后应用程序要做什么——那是应用的业务。控件和业务之间需要一个约定。如果让菜单项直接持有文档对象并调用 `document.paste()`，工具包就依赖了具体应用；给每个菜单项写一个子类覆写 `onClick`，子类数量又会随功能数爆炸。

命令模式的做法是把“一个请求”本身变成对象：`Command` 接口只有一个 `execute()`，`PasteCommand` 持有文档，`execute()` 里调用 `document.paste()`。菜单项只持有一个 `Command`，被点击时调用它的 `execute()`，完全不知道背后是谁、做了什么。

一旦请求成了对象，它就能像普通数据一样被处理，这带来了很多直接调用做不到的事情：

| 能力 | 做法 |
| --- | --- |
| 参数化 | 同一个按钮、快捷键、菜单项可以配置不同的命令；同一个命令也可以绑到多个入口 |
| 排队与延迟执行 | 把命令放进队列，由另一个线程或下一帧执行 |
| 撤销/重做 | 命令记录执行前的必要信息，提供 `undo()`；调用者维护一个历史栈 |
| 日志与重放 | 把执行过的命令记下来，崩溃后按顺序重放即可恢复（GoF 提到的“记录请求日志”） |
| 宏命令 | 一个命令内部包含一组命令，依次执行 |

原笔记说命令模式主要解决“请求者和执行者之间的紧耦合”，这是它的出发点；撤销、排队、日志是解耦之后顺带获得的能力，也是实际工程中使用命令模式最常见的理由。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Command（命令） | 声明执行操作的接口 | `Command`（`execute`、`undo`） |
| ConcreteCommand（具体命令） | 绑定接收者和一个动作，调用接收者完成请求；保存撤销所需的状态 | `MoveCommand`、`RotateCommand`、`MacroCommand` |
| Receiver（接收者） | 知道如何实施与请求相关的操作 | `Shape` |
| Invoker（调用者） | 要求命令执行请求，不关心具体内容；可维护历史 | `Editor` |
| Client（客户端） | 创建具体命令并设置接收者 | `CommandDemo` |

原笔记示例的类图：

![[命令模式-1.png]]

原示例改编自常见的“股票买卖”教程，把 `buy/sell` 换成了 `move/rotate`，但 `Stock` 类里的输出仍是 “bought/sold”，构造函数也有 `MovStock` 的拼写错误，无法编译。下面沿用“移动、旋转”这两个动作，接收者换成一个形状，并补上撤销。

## Java 实现：可撤销的形状编辑命令

```java
// CommandDemo.java
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.List;

// 接收者：真正干活的对象，不知道命令的存在
class Shape {
    private final String name;
    private int x, y, angle;

    Shape(String name) { this.name = name; }

    void move(int dx, int dy) { x += dx; y += dy; }
    void rotate(int deg)      { angle = Math.floorMod(angle + deg, 360); }

    @Override
    public String toString() {
        return name + " at (" + x + "," + y + ") angle=" + angle;
    }
}

interface Command {
    void execute();
    void undo();
}

// 具体命令：绑定接收者 + 动作 + 参数
class MoveCommand implements Command {
    private final Shape shape;
    private final int dx, dy;

    MoveCommand(Shape shape, int dx, int dy) { this.shape = shape; this.dx = dx; this.dy = dy; }

    public void execute() { shape.move(dx, dy); }
    public void undo()    { shape.move(-dx, -dy); }   // 逆操作
}

class RotateCommand implements Command {
    private final Shape shape;
    private final int deg;

    RotateCommand(Shape shape, int deg) { this.shape = shape; this.deg = deg; }

    public void execute() { shape.rotate(deg); }
    public void undo()    { shape.rotate(-deg); }
}

// 宏命令：一组命令当作一个命令
class MacroCommand implements Command {
    private final List<Command> commands;

    MacroCommand(List<Command> commands) { this.commands = commands; }

    public void execute() {
        for (Command c : commands) c.execute();
    }

    public void undo() {                                // 逆序撤销
        for (int i = commands.size() - 1; i >= 0; i--) commands.get(i).undo();
    }
}

// 调用者：只和 Command 接口打交道，维护撤销/重做栈
class Editor {
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    void run(Command c) {
        c.execute();
        undoStack.push(c);
        redoStack.clear();          // 新操作使重做历史失效
    }

    void undo() {
        if (undoStack.isEmpty()) return;
        Command c = undoStack.pop();
        c.undo();
        redoStack.push(c);
    }

    void redo() {
        if (redoStack.isEmpty()) return;
        Command c = redoStack.pop();
        c.execute();
        undoStack.push(c);
    }
}

public class CommandDemo {
    public static void main(String[] args) {
        Shape rect = new Shape("Rectangle");
        Editor editor = new Editor();

        editor.run(new MoveCommand(rect, 10, 0));
        editor.run(new RotateCommand(rect, 90));
        editor.run(new MacroCommand(List.of(
                new MoveCommand(rect, 0, 5),
                new RotateCommand(rect, 45))));
        System.out.println("执行后: " + rect);

        editor.undo();
        System.out.println("撤销宏: " + rect);
        editor.undo();
        System.out.println("再撤销: " + rect);
        editor.redo();
        System.out.println("重做:   " + rect);
    }
}
```

`Editor.run` 里清空重做栈这一步容易漏：撤销两步后又做了一个新操作，原先被撤销的那两步已经不在当前的历史分支上，再重做它们会得到错误的状态。宏命令的撤销必须逆序进行，因为后面的子命令可能依赖前面的子命令执行后的状态（例如先移动再以当前位置为中心旋转）。

## 命令和相近模式的区别

| 对比 | 命令 | 另一方 |
| --- | --- | --- |
| vs 策略 | 封装“一个请求”，关心何时、由谁触发，以及排队、撤销 | 封装“一件事的某种做法”，关心算法可替换；两者都把行为变成对象，refactoring.guru 专门指出它们意图不同 |
| vs 备忘录 | 撤销靠执行逆操作 | 撤销靠恢复快照；无法写出逆操作时，命令可以在执行前保存一份备忘录 |
| vs 责任链 | 请求对象可以沿着责任链传递，由某个处理者执行 | 两者可以组合：链上传递的就是命令对象 |
| vs 原型 | 需要把命令副本存进历史时，可以用原型复制命令 | — |
| vs 组合 | 宏命令就是组合模式应用在命令上 | — |

## 真实例子

Java 里 `Runnable` 和 `Callable` 是最朴素的命令接口：`ExecutorService.submit(task)` 把请求封装成对象交给线程池，排队、延迟、在其他线程执行都由此而来。Swing 的 `javax.swing.Action` 接口让同一个动作同时绑到菜单项、工具栏按钮和快捷键上，并统一控制启用状态，与 GoF 的菜单示例几乎一样。

虚幻引擎编辑器的 UI 命令系统是命令模式的直接应用：用 `TCommands<>` 子类注册一组 `FUICommandInfo`（命令的名字、说明、默认快捷键），再通过 `FUICommandList::MapAction` 把某个命令映射到执行委托和“是否可执行”的检查委托上；菜单、工具栏、快捷键都只引用命令信息，由命令列表负责分派。编辑器的撤销则主要依靠事务系统保存对象状态（见 [[ACAFA-备忘录模式（Memento Pattern）]]），不是靠每个命令写逆操作。

游戏玩法里，把玩家输入转换成命令对象也很常见：输入层只生成“跳跃”“射击”命令，角色或 AI 都能执行同样的命令；把每帧的命令记录下来，就能做回放。

## 容易踩的坑

**逆操作写不对。** 很多操作不是简单的可逆运算：删除要能恢复原来的位置，裁剪到边界的移动反向移动回不到原处。写不出可靠逆操作的，就在执行前保存状态。

**撤销后没有清空重做栈。** 见代码后的说明。

**命令持有已失效的接收者。** 命令在历史里存放很久，接收者可能早已被删除或替换。撤销时要能检测这种情况，或让“删除对象”本身也是一个可撤销的命令。

**命令对象太多太碎。** 每个小动作都一个类，类的数量会膨胀。只需要解耦、不需要撤销和排队时，直接用 lambda 或方法引用作为命令即可。

**异步执行时的顺序和并发。** 命令放进队列在其他线程执行，多个命令操作同一个接收者时要考虑执行顺序和线程安全。

## 相关

[[ACAFA-备忘录模式（Memento Pattern）]] [[ACAFB-策略模式（Strategy Pattern）]] [[ACAFI-责任链模式（Chain of Responsibility Pattern）]] [[ACAEG-组合模式（Composite Pattern）]] [[ACADE-原型模式（Prototype Pattern）]]
