**责任链模式（Chain of Responsibility）是 GoF 行为型模式之一：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系；将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。每个处理者只知道自己的下一个处理者（后继），发送者只知道链头。**

> 参考：GoF《设计模式》Chain of Responsibility 一章（动机示例是图形界面的上下文帮助：点击某个按钮请求帮助时，请求从按钮沿着 `HelpHandler` 链传到对话框、再到应用程序，直到有对象能提供帮助信息）；refactoring.guru 的 Chain of Responsibility 页面；Epic 官方 `FReply` API 文档。代码沿用原笔记的 Java。

## 为什么要有责任链

GoF 的例子很直观：用户在界面上点了某个按钮的“帮助”。最具体的帮助信息应该由这个按钮提供；如果按钮没有专门的帮助，就显示它所在对话框的帮助；对话框也没有，就显示整个应用的通用帮助。发出请求的地方（按钮）并不知道最终是谁来回应，而“谁能回应”取决于运行时的界面结构和每个对象的配置。

如果让发送者自己判断该交给谁，它就得知道所有可能的处理者以及它们的判断条件，写成一长串 `if/else if`；处理者一变，发送者就得改。责任链把这串判断拆散到各个处理者自身：每个处理者只回答“我能不能处理”，能就处理，不能就交给后继。于是：

- 发送者只和链头打交道，完全不知道链上有谁；
- 处理者只知道自己的后继，不知道整条链的结构；
- 增加、删除、调换处理者只需要改链的组装方式，不需要改发送者或其他处理者。

代价是请求**不保证被处理**：链走到底也没人接，请求就悄悄消失了。另外，请求在链上走了哪条路径不直观，调试时要一环环跟下去。

## 纯与不纯的责任链

实践中“责任链”这个名字覆盖了两种行为不同的结构：

| 形式 | 行为 | 例子 |
| --- | --- | --- |
| 纯责任链（GoF 原意） | 某个处理者处理了请求，传递就**停止**；否则继续往后传 | 界面帮助、UI 事件冒泡、审批链 |
| 不纯的责任链 / 处理管道 | 每个处理者都可以做一部分处理，然后**继续**往后传（也可以选择中断） | Servlet 过滤器链、Web 框架的中间件、日志处理器链 |

原笔记的日志记录器示例属于后者：每个记录器只要消息级别够高就写一次，**并且总是继续往下传**，所以一条 ERROR 消息会被三个记录器都写一遍。这不是错误，只是和 GoF“找到一个处理者就停”的定义不同。refactoring.guru 对两种都有描述：处理者可以决定不再往下传，从而终止后续处理。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Handler（处理者） | 定义处理请求的接口，（可选）实现到后继的链接 | `Handler` |
| ConcreteHandler（具体处理者） | 处理自己负责的请求；处理不了就转发给后继 | `ShapeHandler`、`GroupHandler`、`CanvasHandler` |
| Client（客户端） | 组装链，并把请求提交给链头 | `ChainDemo` |

原笔记示例的类图：

![[责任链模式-1.png]]

## Java 实现一：点击事件的冒泡（纯责任链）

模拟界面上点击一个形状：形状自己能处理的就处理，处理不了交给它所在的组，再交给画布。每个处理者的判断条件各不相同，发送者完全不需要知道：

```java
// ChainDemo.java
class ClickEvent {
    final String target;
    final boolean doubleClick;

    ClickEvent(String target, boolean doubleClick) {
        this.target = target;
        this.doubleClick = doubleClick;
    }
}

abstract class Handler {
    private Handler next;

    // 返回后继，便于链式组装
    Handler setNext(Handler next) {
        this.next = next;
        return next;
    }

    // 模板：自己处理不了就交给后继；返回是否已被处理
    final boolean handle(ClickEvent e) {
        if (tryHandle(e)) return true;
        if (next != null) return next.handle(e);
        return false;           // 链尾也没人处理
    }

    protected abstract boolean tryHandle(ClickEvent e);
}

class ShapeHandler extends Handler {
    protected boolean tryHandle(ClickEvent e) {
        if (e.target.equals("circle") && e.doubleClick) {
            System.out.println("ShapeHandler: 双击圆，进入编辑");
            return true;
        }
        return false;
    }
}

class GroupHandler extends Handler {
    protected boolean tryHandle(ClickEvent e) {
        if (!e.doubleClick) {
            System.out.println("GroupHandler: 单击，选中整个组 (" + e.target + ")");
            return true;
        }
        return false;
    }
}

class CanvasHandler extends Handler {
    protected boolean tryHandle(ClickEvent e) {
        if (e.target.equals("background")) {
            System.out.println("CanvasHandler: 点击空白处，取消选择");
            return true;
        }
        return false;
    }
}

public class ChainDemo {
    public static void main(String[] args) {
        Handler chain = new ShapeHandler();
        chain.setNext(new GroupHandler()).setNext(new CanvasHandler());

        ClickEvent[] events = {
            new ClickEvent("circle", true),
            new ClickEvent("square", false),
            new ClickEvent("background", true),
            new ClickEvent("square", true),     // 没人处理
        };
        for (ClickEvent e : events) {
            boolean handled = chain.handle(e);
            if (!handled) System.out.println("未处理: " + e.target + (e.doubleClick ? " 双击" : " 单击"));
        }
    }
}
```

`handle` 返回布尔值，让发送者能知道请求最终有没有被处理。这是对“请求可能无人处理”的直接应对：可以在链尾放一个兜底处理者，或者像这里一样把结果告诉发送者，由它决定是报错还是忽略。`handle` 本身是 `final` 的模板方法（见 [[ACAFH-模板模式（Template Pattern）]]），子类只实现 `tryHandle`，于是“传给后继”的逻辑不会被某个子类忘写。

## Java 实现二：原笔记的日志链（不纯的责任链）

```java
// LoggerChainDemo.java
public class LoggerChainDemo {
    static final int INFO = 1, DEBUG = 2, ERROR = 3;

    abstract static class AbstractLogger {
        protected final int level;
        private AbstractLogger next;

        AbstractLogger(int level) { this.level = level; }

        AbstractLogger setNext(AbstractLogger next) { this.next = next; return next; }

        void logMessage(int msgLevel, String message) {
            if (msgLevel >= level) write(message);          // 级别够就处理
            if (next != null) next.logMessage(msgLevel, message);  // 无论如何都继续传
        }

        protected abstract void write(String message);
    }

    static class ErrorLogger extends AbstractLogger {
        ErrorLogger() { super(ERROR); }
        protected void write(String m) { System.out.println("Error Console::Logger: " + m); }
    }

    static class FileLogger extends AbstractLogger {
        FileLogger() { super(DEBUG); }
        protected void write(String m) { System.out.println("File::Logger: " + m); }
    }

    static class ConsoleLogger extends AbstractLogger {
        ConsoleLogger() { super(INFO); }
        protected void write(String m) { System.out.println("Standard Console::Logger: " + m); }
    }

    public static void main(String[] args) {
        AbstractLogger chain = new ErrorLogger();
        chain.setNext(new FileLogger()).setNext(new ConsoleLogger());

        chain.logMessage(INFO, "This is an information.");
        chain.logMessage(DEBUG, "This is a debug information.");
        chain.logMessage(ERROR, "This is an error information.");
    }
}
```

输出与原笔记给出的一致：INFO 只被控制台写，DEBUG 被文件和控制台写，ERROR 被三个记录器都写。原示例把级别写成可修改的 `public static int`，这里改成了常量。

## 责任链和相近模式的区别

| 对比 | 责任链 | 另一方 |
| --- | --- | --- |
| vs 装饰器 | 处理者可以中断传递，各处理者互相独立 | 每层装饰都会执行并继续往里转发，不能打断流程；两者结构相似 |
| vs 命令 | 链上传递的请求常被封装成命令对象 | 命令关注把请求对象化，不关心谁处理 |
| vs 中介者 | 请求沿链依次传递，发送者不知道接收者 | 所有通信经过一个中心对象，由它决定转给谁 |
| vs 观察者 | 通常一个处理者处理后就停止 | 所有订阅者都会收到通知 |
| vs 组合 | 组合树中子节点把请求交给父节点，父链就构成一条责任链（GoF 的帮助示例正是如此） | — |

## 真实例子

Java Servlet 的 `javax.servlet.Filter`（Jakarta EE 中为 `jakarta.servlet.Filter`）通过 `FilterChain.doFilter(request, response)` 把请求交给下一个过滤器，不调用它就中断了处理，属于可中断的处理管道。`java.util.logging` 中，一个 `Logger` 处理完日志记录后，默认还会把它交给父级 Logger 的处理器（由 `useParentHandlers` 控制），日志沿命名层次向上传递。

虚幻引擎的 Slate 界面事件是纯责任链的典型：`OnMouseButtonDown` 等事件处理函数必须返回一个 `FReply`，返回 `FReply::Handled()` 表示事件已处理，按官方蓝图文档对 Handled 的描述，这会阻止事件继续在控件层次中冒泡；返回 `FReply::Unhandled()` 则事件继续交给父控件处理。UMG 蓝图里对应的是 Event Reply 的 Handled / Unhandled 节点。

## 容易踩的坑

**请求无人处理却没有任何反馈。** 纯责任链的固有问题。在链尾放一个兜底处理者，或像上面的示例那样返回处理结果。

**链中出现环。** 组装错误时 A 的后继是 B、B 的后继又是 A，请求会无限传递直到栈溢出。

**处理者忘了转发。** 不纯的链里，某个处理者忘记调用下一个，后面的处理全部被跳过（在 Servlet 里就是忘了调 `chain.doFilter`）。用模板方法把转发逻辑固定在基类中可以避免。

**UE 里该 Unhandled 的事件返回了 Handled。** 子控件把事件吞掉，父控件或外层界面就收不到点击、滚轮，表现为“某块区域点不动”。

**链太长。** 每个请求都要从头走一遍，热点路径上可能成为性能问题；也让调试变得困难，原笔记提到的“运行时特征不明显，妨碍除错”指的就是这个。

## 相关

[[ACAFG-命令模式（Command Pattern）]] [[ACAEF-装饰器模式（Decorator Pattern）]] [[ACAEG-组合模式（Composite Pattern）]] [[ACAFJ-中介者模式（Mediator Pattern）]] [[ACAFH-模板模式（Template Pattern）]] [[ACAEB-过滤器模式（Filter Pattern）]]
