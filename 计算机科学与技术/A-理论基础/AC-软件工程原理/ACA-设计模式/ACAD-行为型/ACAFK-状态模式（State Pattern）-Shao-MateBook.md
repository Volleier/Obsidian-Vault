**状态模式（State，又称状态对象 Objects for States）是 GoF 行为型模式之一：允许一个对象在其内部状态改变时改变它的行为，对象看起来似乎修改了它的类。做法是把每个状态下的行为封装进一个状态类，上下文（Context）持有当前状态对象并把请求委派给它；状态转移通过替换这个对象完成。**

> 参考：GoF《设计模式》State 一章（动机示例是网络连接 `TCPConnection`，在 `TCPEstablished`、`TCPListen`、`TCPClosed` 等状态下对 `Open`、`Close`、`Acknowledge` 请求作出不同响应）；refactoring.guru 的 State 页面。代码沿用原笔记的 Java。

## 为什么要有状态模式

很多对象的行为取决于它当前处在什么状态。GoF 的例子是 TCP 连接：同样收到“打开”请求，已关闭的连接会开始建立连接，已建立的连接则忽略或报错；同样收到“关闭”请求，不同状态下要做的事也各不相同。最直接的写法是用一个枚举字段表示状态，然后在每个方法里 `switch`：

```text
open():  switch (state) { case CLOSED: ...; case LISTEN: ...; case ESTABLISHED: ... }
close(): switch (state) { case CLOSED: ...; case LISTEN: ...; case ESTABLISHED: ... }
```

状态少、操作少时这完全没问题。但随着状态和操作增多，问题就出来了：同一个状态的行为分散在所有方法的 `case` 里，想看“已建立状态下会发生什么”要翻遍整个类；加一个新状态，要去每个 `switch` 里补一个分支，漏一个编译器也不会提醒；状态转移的规则（从哪里能到哪里）埋在各个分支里，没有一处能看清全貌。

状态模式把 `switch` 旋转了九十度：原来按“操作”组织代码，每个操作里列出所有状态；现在按“状态”组织代码，每个状态类里列出所有操作。`TCPClosed` 里集中了关闭状态下对所有请求的处理，`TCPEstablished` 里集中了已建立状态下的处理。上下文 `TCPConnection` 只把请求转交给当前状态对象。于是：

- 与某个状态相关的行为集中在一个类里；
- 加新状态就是加一个类，不用改动已有状态里的代码（前提是其他状态不需要转移到它）；
- 状态转移变成显式的“替换状态对象”，而不是悄悄改一个整数字段。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| Context（上下文） | 定义客户端使用的接口，持有当前状态对象，把请求委派给它 | `Player` |
| State（状态） | 声明与各状态相关的行为接口 | `State` |
| ConcreteState（具体状态） | 实现某个状态下的行为，并决定是否转移、转移到哪个状态 | `Stopped`、`Playing`、`Paused` |

转移由谁决定是一个设计选择，GoF 讨论过两种方式：由上下文集中决定（转移规则固定且简单时），或由各个状态类自己决定下一个状态（更灵活，但状态类之间彼此知道）。后一种更常见，下面的示例也采用这种方式。

原笔记示例的类图：

![[状态模式-1.png]]

原示例里状态对象的 `doAction(context)` 只是把自己设为上下文的当前状态并打印一句话，行为和转移都由客户端从外部驱动（客户端自己 new 一个 `StartState` 然后调用它），上下文并没有“根据状态表现出不同行为”，更接近于给上下文贴一个标签。状态模式的关键在于：**客户端调用的是上下文的同一个方法，行为却随内部状态变化**。下面沿用原示例“播放器开始/停止”的题材重写。

## Java 实现：播放器的三个状态

```java
// StateDemo.java
interface State {
    void play(Player p);
    void pause(Player p);
    void stop(Player p);
    String name();
}

// 状态对象无内部字段，可以共享为单例
final class Stopped implements State {
    static final Stopped INSTANCE = new Stopped();
    private Stopped() {}

    public void play(Player p)  { System.out.println("从头开始播放"); p.setState(Playing.INSTANCE); }
    public void pause(Player p) { System.out.println("已停止，无法暂停"); }
    public void stop(Player p)  { System.out.println("已经是停止状态"); }
    public String name() { return "Stopped"; }
}

final class Playing implements State {
    static final Playing INSTANCE = new Playing();
    private Playing() {}

    public void play(Player p)  { System.out.println("正在播放，忽略"); }
    public void pause(Player p) { System.out.println("暂停，记住进度"); p.setState(Paused.INSTANCE); }
    public void stop(Player p)  { System.out.println("停止并回到开头"); p.setState(Stopped.INSTANCE); }
    public String name() { return "Playing"; }
}

final class Paused implements State {
    static final Paused INSTANCE = new Paused();
    private Paused() {}

    public void play(Player p)  { System.out.println("从暂停处继续"); p.setState(Playing.INSTANCE); }
    public void pause(Player p) { System.out.println("已经暂停"); }
    public void stop(Player p)  { System.out.println("停止并回到开头"); p.setState(Stopped.INSTANCE); }
    public String name() { return "Paused"; }
}

// 上下文：对外接口不变，行为随内部状态变化
class Player {
    private State state = Stopped.INSTANCE;

    void setState(State next) {          // 包级可见，只给状态类用
        System.out.println("  [" + state.name() + " -> " + next.name() + "]");
        state = next;
    }

    void play()  { state.play(this); }
    void pause() { state.pause(this); }
    void stop()  { state.stop(this); }
}

public class StateDemo {
    public static void main(String[] args) {
        Player player = new Player();
        player.pause();   // Stopped 下暂停：无效
        player.play();
        player.pause();
        player.play();
        player.stop();
        player.stop();
    }
}
```

客户端只调用 `play()`、`pause()`、`stop()`，同一个调用在不同状态下结果不同，这正是“对象看起来修改了它的类”。三个状态类都没有字段，所以用单例共享即可，GoF 也提到了没有内部状态的状态对象可以被多个上下文共享。若状态需要保存数据（例如播放进度），应把数据放在上下文里，而不是放进共享的状态对象。

## 状态和策略的区别

两者的类图几乎一模一样——上下文持有一个接口引用，把工作委派出去。区别在意图和使用方式：

| 方面 | 状态 | 策略 |
| --- | --- | --- |
| 谁来切换 | 通常由状态对象自己在处理请求时触发转移 | 由客户端选择并注入 |
| 彼此是否知道 | 具体状态之间相互知道（要知道转移到哪个） | 具体策略之间互不知道 |
| 切换频率 | 随每次请求可能变化，是对象生命周期的一部分 | 通常配置一次，偶尔替换 |
| 表达的是 | 对象“处在什么情况下” | 一件事“用哪种做法” |
| 客户端是否感知 | 通常不感知当前是哪个状态对象 | 知道自己选了哪个策略 |

refactoring.guru 的说法是：状态可以看作策略的扩展，两者都基于组合，但策略中的对象彼此完全独立，状态模式则不限制具体状态之间的依赖，允许它们主动改变上下文的状态。

## 真实例子

游戏开发里状态模式和有限状态机无处不在：角色的待机、奔跑、跳跃、受击，AI 的巡逻、追击、攻击。虚幻引擎的动画蓝图状态机把每个状态对应的动画和状态之间的转移规则做成可视化的图（见 [[EBC-动画状态机]]），它是数据驱动的状态机，而不是一个状态一个 C++ 类，但“按状态组织行为、显式声明转移”的思想是相同的。复杂的角色可以使用分层状态机（见 [[EBCE-分层动画状态机]]），把一组相关状态嵌套进一个父状态里，减少转移数量。UE 5 的 StateTree 则用于 AI 和通用逻辑，是分层状态机与行为树思路的结合。

## 容易踩的坑

**状态少也拆成类。** 两三个状态、两三个操作时，一个枚举加 `switch` 更直观，原笔记“适用于状态数量有限”的提醒也是这个意思——状态太少时不必用，太多时要考虑分层或数据驱动的状态机。

**状态对象里存了上下文相关的数据却被共享。** 共享的状态对象一旦有字段，多个上下文会互相干扰。

**转移规则分散，看不清全貌。** 状态类各自决定下一个状态，整体的状态图只存在于代码之间。状态多时，最好配一张状态转移表或图作为文档，或者改用表驱动的状态机。

**进入/退出动作遗漏。** 很多状态需要在进入时初始化、退出时清理（例如离开“播放”状态时释放音频设备）。可以在 `State` 接口中加 `onEnter`/`onExit`，由上下文的 `setState` 统一调用，避免每个转移点各写一遍。

**在状态方法内部转移后继续使用旧状态的假设。** 调用 `p.setState(...)` 之后，当前方法仍在旧状态对象里执行，后面的代码如果继续按旧状态的逻辑操作上下文，就会出现混乱。转移通常应该是状态方法的最后一步。

## 相关

[[ACAFB-策略模式（Strategy Pattern）]] [[ACAEH-桥接模式（Bridge Pattern）]] [[ACADA-单例模式（Singleton Pattern）]] [[ACAEE-享元模式（Flyweight Pattern）]] [[ACAFA-备忘录模式（Memento Pattern）]] [[EBC-动画状态机]] [[EBCE-分层动画状态机]]
