在状态模式（State Pattern）中，类的行为是基于它的状态改变的，这种类型的设计模式属于行为型模式。
在状态模式中，我们创建表示各种状态的对象和一个行为随着状态对象改变而改变的 context 对象。
状态模式允许对象在内部状态改变时改变其行为，使得对象在不同的状态下有不同的行为表现。通过将每个状态封装成独立的类，可以避免使用大量的条件语句来实现状态切换。

# 介绍
## 概览
允许一个对象在其内部状态改变时改变其行为，看起来就像是改变了其类一样。主要解决状态模式解决对象行为依赖于其状态的问题，使得对象可以在状态变化时切换行为。
当代码中存在大量条件语句，且这些条件语句依赖于对象的状态时使用状态模式。

## 实现方式
- 定义状态接口：声明一个或多个方法，用于封装具体状态的行为。
- 创建具体状态类：实现状态接口，根据状态的不同实现具体的行为。
- 定义上下文类：包含一个状态对象的引用，并在状态改变时更新其行为。

## 关键代码
- 状态接口：声明行为方法。
- 具体状态类：实现状态接口，封装具体行为。
- 上下文类：维护一个状态对象，并提供方法以改变其状态。

## 优缺点
- 封装状态转换规则：将状态转换逻辑封装在状态对象内部。
- 易于扩展：增加新的状态类不会影响现有代码。
- 集中状态相关行为：将所有与特定状态相关的行为集中到一个类中。
- 简化条件语句：避免使用大量的条件语句来切换行为。
- 状态共享：允许多个上下文对象共享同一个状态对象。
- 增加类和对象数量：每个状态都需要一个具体的状态类。
- 实现复杂：模式结构和实现相对复杂。
- 开闭原则支持不足：增加新状态或修改状态行为可能需要修改现有代码。

## 注意事项
- 当对象的行为随状态改变而变化时，考虑使用状态模式。
- 状态模式适用于替代复杂的条件或分支语句。
- 状态模式适用于状态数量有限（通常不超过5个）的情况。
- 谨慎使用，以避免系统变得过于复杂。

## 结构
- 上下文（Context）：定义了客户感兴趣的接口，并维护一个当前状态对象的引用。上下文可以通过状态对象来委托处理状态相关的行为。
- 状态（State）：定义了一个接口，用于封装与上下文相关的一个状态的行为。
- 具体状态（Concrete State）：实现了状态接口，负责处理与该状态相关的行为。具体状态对象通常会在内部维护一个对上下文对象的引用，以便根据不同的条件切换到不同的状态。
# 实现
创建一个 `State` 接口和实现了 `State` 接口的实体状态类。`Context` 是一个带有某个状态的类。
`StatePatternDemo`，我们的演示类使用 `Context` 和状态对象来演示 Context 在状态改变时的行为变化。
![[状态模式-1.png]]

## 创建一个接口
State.java
```java
public interface State {
   public void doAction(Context context);
}
```

## 创建实现接口的实体类
StartState.java
```java
public class StartState implements State {
 
   public void doAction(Context context) {
      System.out.println("Player is in start state");
      context.setState(this); 
   }
 
   public String toString(){
      return "Start State";
   }
}
```
StopState.java
```java
public class StopState implements State {
 
   public void doAction(Context context) {
      System.out.println("Player is in stop state");
      context.setState(this); 
   }
 
   public String toString(){
      return "Stop State";
   }
}
```

## 创建 Context 类
Context.java
```java
public class Context {
   private State state;
 
   public Context(){
      state = null;
   }
 
   public void setState(State state){
      this.state = state;     
   }
 
   public State getState(){
      return state;
   }
}
```

## 使用 Context 来查看当状态 State 改变时的行为变化
StatePatternDemo.java
```java
public class StatePatternDemo {
   public static void main(String[] args) {
      Context context = new Context();
 
      StartState startState = new StartState();
      startState.doAction(context);
 
      System.out.println(context.getState().toString());
 
      StopState stopState = new StopState();
      stopState.doAction(context);
 
      System.out.println(context.getState().toString());
   }
}
```

## 输出Belike
```text
Player is in start state
Start State
Player is in stop state
Stop State
```