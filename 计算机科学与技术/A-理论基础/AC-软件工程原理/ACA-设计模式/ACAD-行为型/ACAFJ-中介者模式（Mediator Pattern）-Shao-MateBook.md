**中介者模式（Mediator）是 GoF 行为型模式之一：用一个中介对象来封装一系列对象之间的交互，使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。参与交互的对象称为同事（Colleague），它们只认识中介者；多对多的网状依赖因此变成以中介者为中心的星状依赖。**

> 参考：GoF《设计模式》Mediator 一章（动机示例是字体对话框：列表框、输入框、按钮之间的联动由 `FontDialogDirector` 统一协调）；refactoring.guru 的 Mediator 页面。代码沿用原笔记的 Java。

## 为什么要有中介者

GoF 的例子是一个字体选择对话框。里面有字体列表框、字号输入框、粗体复选框、预览区、确定按钮。它们之间的联动很多：在列表里选了一个字体，输入框要显示字体名、预览区要刷新；输入框被清空，确定按钮要变灰；勾上粗体，预览区又要刷新……如果让每个控件直接引用并调用其他控件，n 个控件之间最多会有 n(n-1) 条依赖，形成一张网。结果是：

- 每个控件都和对话框里的其他控件绑死，拿到别的对话框里就用不了；
- 联动规则分散在各个控件的代码里，想弄清“选中字体后会发生什么”得翻遍所有控件；
- 改一条联动规则可能要改好几个控件。

中介者的做法是新增一个 `FontDialogDirector`，所有控件都只认识它。控件状态变化时，只通知中介者“我变了”；由中介者决定这件事要影响哪些其他控件、怎么影响。联动规则全部集中在中介者里，控件本身变成了可复用的普通组件。

原笔记“将多个对象间的一对多关系转换为一对一关系”说的就是这件事：每个同事只和中介者一对一通信。代价也很明显：原本分散在各处的复杂度并没有消失，而是全部搬进了中介者，它很容易变成一个难以维护的庞然大物。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Mediator（中介者） | 定义同事对象与之通信的接口 | `ChatMediator` |
| ConcreteMediator（具体中介者） | 了解并维护各个同事，实现协调逻辑 | `ChatRoom` |
| Colleague（同事） | 只持有中介者的引用；需要与其他同事通信时都通过中介者 | `User` |

原笔记示例的类图：

![[中介者模式-1.png]]

原示例的 `ChatRoom` 只有一个静态方法 `showMessage`，打印一行带时间的消息，它并不认识任何用户，也不负责把消息投递给别人——用户之间实际上没有交互，中介者也就没有东西可协调。演示代码创建的用户名是 `Rectangle` 和 `Square`，列出的输出里却是 `[Robert]`、`[John]`，与代码对不上（输出是从原教程照搬的）。下面保留聊天室和这两个用户名，把中介者做成真正负责投递和协调的对象。

## Java 实现：负责投递与禁言的聊天室

```java
// MediatorDemo.java
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

// 中介者接口：同事只依赖它
interface ChatMediator {
    void join(User user);
    void send(User from, String message);
    void sendTo(User from, String toName, String message);
}

// 具体中介者：集中所有交互规则
class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();
    private final Set<String> muted = new HashSet<>();

    public void join(User user) {
        for (User u : users) {
            u.receive("系统", user.getName() + " 加入了聊天室");
        }
        users.add(user);
    }

    void mute(String name) {
        muted.add(name);
    }

    public void send(User from, String message) {
        if (muted.contains(from.getName())) {
            from.receive("系统", "你已被禁言");
            return;
        }
        for (User u : users) {
            if (u != from) u.receive(from.getName(), message);   // 不发给自己
        }
    }

    public void sendTo(User from, String toName, String message) {
        for (User u : users) {
            if (u.getName().equals(toName)) {
                u.receive(from.getName() + "(私聊)", message);
                return;
            }
        }
        from.receive("系统", "找不到用户 " + toName);
    }
}

// 同事：不持有任何其他 User 的引用
class User {
    private final String name;
    private final ChatMediator mediator;

    User(String name, ChatMediator mediator) {
        this.name = name;
        this.mediator = mediator;
        mediator.join(this);
    }

    String getName() { return name; }

    void say(String message)                  { mediator.send(this, message); }
    void whisper(String to, String message)   { mediator.sendTo(this, to, message); }

    void receive(String from, String message) {
        System.out.println("  " + name + " 收到 [" + from + "]: " + message);
    }
}

public class MediatorDemo {
    public static void main(String[] args) {
        ChatRoom room = new ChatRoom();
        User rectangle = new User("Rectangle", room);
        User square = new User("Square", room);
        User circle = new User("Circle", room);

        System.out.println("Rectangle 群发:");
        rectangle.say("Hi! Square!");
        System.out.println("Square 私聊 Rectangle:");
        square.whisper("Rectangle", "Hello! Rectangle!");

        room.mute("Circle");
        System.out.println("Circle 被禁言后发言:");
        circle.say("有人吗？");
    }
}
```

`User` 里没有任何其他 `User` 的引用，“不发给自己”“禁言”“私聊找不到人”这些规则全都在 `ChatRoom` 里。要加一条新规则（比如敏感词过滤、离线消息），只改中介者，用户类不用动。反过来也能看到风险：规则越加越多，`ChatRoom` 就越来越胖。

## 中介者和相近模式的区别

| 对比 | 中介者 | 另一方 |
| --- | --- | --- |
| vs 外观 | 双向：同事认识中介者，中介者也认识同事；目的是协调同事之间的交互 | 单向：外观调用子系统，子系统不知道外观；目的是给外部提供简单入口 |
| vs 观察者 | 中介者知道每个同事，按具体规则决定通知谁、做什么 | 发布者不关心订阅者是谁，只负责广播；refactoring.guru 指出中介者常用观察者实现同事到中介者的通知 |
| vs 责任链 | 所有请求都经过一个中心对象 | 请求沿链逐个传递，没有中心 |
| vs 命令 | — | 同事发给中介者的请求可以封装成命令对象 |

中介者和观察者的边界最容易模糊。一个常见的判断方法是：如果中心对象里写着“当 A 变化时去更新 B 和 C”这样具体的协调逻辑，它是中介者；如果中心对象只是按主题转发消息，不知道也不关心谁在收，它更接近事件总线或发布-订阅。

## 真实例子

refactoring.guru 提到，Java 中中介者最常见的用途是协调图形界面组件之间的通信，MVC 中的控制器（Controller）就扮演了中介者的角色：视图组件不直接互相调用，而是通过控制器协调。

虚幻引擎 Lyra 示例项目带的 GameplayMessageRouter 插件提供了 `UGameplayMessageSubsystem`：发送方用 `BroadcastMessage` 往某个 GameplayTag 频道发一个结构体消息，接收方用 `RegisterListener` 订阅频道，双方都不需要持有对方的引用。它让原本互不相识的玩法对象通过一个中心对象通信，起到了解耦的作用；但它本身不包含“谁该响应什么”的协调逻辑，只按频道转发，按上面的判断方法更接近发布-订阅的消息总线，而不是 GoF 意义上的具体中介者。真正的协调规则（比如击杀消息触发计分、播报、成就）分散在各个订阅方里。

## 容易踩的坑

**中介者变成上帝对象。** 所有交互逻辑集中到一处，中介者膨胀到几千行，改任何联动都要动它。可以按功能拆成多个中介者，或者只把真正跨组件的协调放进去。

**同事之间偷偷建立直接引用。** 为了图方便，某个同事直接调用了另一个同事，星状结构就又退回网状，而且这种依赖不在中介者里，更难发现。

**中介者与同事的生命周期不同步。** 同事被销毁时没有从中介者注销，中介者继续给它发消息，在 C++ 或 UE 里就是野指针；Java 里则是内存泄漏。前面提到 Lyra 消息系统时，社区示例都强调要保存 `RegisterListener` 返回的句柄，并在对象销毁时注销。

**通知循环。** 同事 A 变化 → 中介者更新 B → B 的变化又通知中介者 → 中介者更新 A……需要在中介者里防止重入，或者区分“用户操作引起的变化”和“程序设置引起的变化”。

## 相关

[[ACAED-外观模式（Facade Pattern）]] [[ACAFI-责任链模式（Chain of Responsibility Pattern）]] [[ACAFG-命令模式（Command Pattern）]] [[ACAFK-状态模式（State Pattern）]]
