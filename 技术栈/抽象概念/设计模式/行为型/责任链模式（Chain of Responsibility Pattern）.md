责任链模式（Chain of Responsibility Pattern）为请求创建了一个接收者对象的链。这种模式给予请求的类型，对请求的发送者和接收者进行解耦。
责任链模式通过将多个处理器（处理对象）以链式结构连接起来，使得请求沿着这条链传递，直到有一个处理器处理该请求为止。
责任链模式允许多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递请求。

# 介绍
## 概览
责任链模式允许将请求沿着处理者链传递，直到请求被处理为止。责任链模式解耦请求发送者和接收者，使多个对象都有可能接收请求，而发送者不需要知道哪个对象会处理它。责任链模式适用于当有多个对象可以处理请求，且具体由哪个对象处理由运行时决定时；以及当需要向多个对象中的一个提交请求，而不想明确指定接收者时。

## 实现方式
- 定义处理者接口：所有处理者必须实现同一个接口。
- 创建具体处理者：实现接口的具体类，包含请求处理逻辑和指向链中下一个处理者的引用。

## 关键代码
- Handler接口：定义一个方法用于处理请求。
- ConcreteHandler类：实现Handler接口，包含请求处理逻辑和对下一个处理者的引用。

## 优缺点
- 降低耦合度：发送者和接收者之间解耦。
- 简化对象：对象不需要知道链的结构。
- 灵活性：通过改变链的成员或顺序，动态地新增或删除责任。
- 易于扩展：增加新的请求处理类很方便。
- 请求未被处理：不能保证请求一定会被链中的某个处理者接收。
- 性能影响：可能影响系统性能，且调试困难，可能导致循环调用。
- 难以观察：运行时特征不明显，可能妨碍除错。

## 注意事项

- 在处理请求时，如果有多个潜在的处理者，考虑使用责任链模式。
- 确保链中的每个处理者都明确知道如何传递请求到链的下一个环节。
- 在Java Web开发中，责任链模式有广泛应用，如过滤器链、拦截器等。

## 结构
- 抽象处理者（Handler）：定义一个处理请求的接口，通常包含一个处理请求的方法（如 `handleRequest`）和一个指向下一个处理者的引用（后继者）。
- 具体处理者（ConcreteHandler）：实现了抽象处理者接口，负责处理请求。如果能够处理该请求，则直接处理；否则，将请求传递给下一个处理者。
- 客户端（Client）：创建处理者对象，并将它们连接成一条责任链。通常，客户端只需要将请求发送给责任链的第一个处理者，无需关心请求的具体处理过程。

# 实现

创建抽象类 `AbstractLogger`，带有详细的日志记录级别。然后我们创建三种类型的记录器，都扩展了 `AbstractLogger`。每个记录器消息的级别是否属于自己的级别，如果是则相应地打印出来，否则将不打印并把消息传给下一个记录器。
![[责任链模式-1.png]]

## 创建抽象的记录器类
AbstractLogger.java
```java
public abstract class AbstractLogger {
   public static int INFO = 1;
   public static int DEBUG = 2;
   public static int ERROR = 3;
 
   protected int level;
 
   //责任链中的下一个元素
   protected AbstractLogger nextLogger;
 
   public void setNextLogger(AbstractLogger nextLogger){
      this.nextLogger = nextLogger;
   }
 
   public void logMessage(int level, String message){
      if(this.level <= level){
         write(message);
      }
      if(nextLogger !=null){
         nextLogger.logMessage(level, message);
      }
   }
 
   abstract protected void write(String message);
}
```

## 创建扩展了该记录器类的实体类
ConsoleLogger.java
```java
public class ConsoleLogger extends AbstractLogger {
 
   public ConsoleLogger(int level){
      this.level = level;
   }
 
   @Override
   protected void write(String message) {    
      System.out.println("Standard Console::Logger: " + message);
   }
}
```
ErrorLogger.java
```java
public class ErrorLogger extends AbstractLogger {
 
   public ErrorLogger(int level){
      this.level = level;
   }
 
   @Override
   protected void write(String message) {    
      System.out.println("Error Console::Logger: " + message);
   }
}
```
FileLogger.java
```java
public class FileLogger extends AbstractLogger {
 
   public FileLogger(int level){
      this.level = level;
   }
 
   @Override
   protected void write(String message) {    
      System.out.println("File::Logger: " + message);
   }
}
```

## 创建不同类型的记录器
创建不同类型的记录器。赋予它们不同的错误级别，并在每个记录器中设置下一个记录器。每个记录器中的下一个记录器代表的是链的一部分。
```java
public class ChainPatternDemo {
   
   private static AbstractLogger getChainOfLoggers(){
 
      AbstractLogger errorLogger = new ErrorLogger(AbstractLogger.ERROR);
      AbstractLogger fileLogger = new FileLogger(AbstractLogger.DEBUG);
      AbstractLogger consoleLogger = new ConsoleLogger(AbstractLogger.INFO);
 
      errorLogger.setNextLogger(fileLogger);
      fileLogger.setNextLogger(consoleLogger);
 
      return errorLogger;  
   }
 
   public static void main(String[] args) {
      AbstractLogger loggerChain = getChainOfLoggers();
 
      loggerChain.logMessage(AbstractLogger.INFO, "This is an information.");
 
      loggerChain.logMessage(AbstractLogger.DEBUG, 
         "This is a debug information.");
 
      loggerChain.logMessage(AbstractLogger.ERROR, 
         "This is an error information.");
   }
}
```

## 输出Belike
```text
Standard Console::Logger: This is an information.
File::Logger: This is a debug information.
Standard Console::Logger: This is a debug information.
Error Console::Logger: This is an error information.
File::Logger: This is an error information.
Standard Console::Logger: This is an error information.
```