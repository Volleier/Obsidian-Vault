外观模式（Facade Pattern）隐藏系统的复杂性，并向客户端提供了一个客户端可以访问系统的接口。这种类型的设计模式属于结构型模式，它向现有的系统添加一个接口，来隐藏系统的复杂性。
这种模式涉及到一个单一的类，该类提供了客户端请求的简化方法和对现有系统类方法的委托调用。
![[外观模式-1.png]]
# 介绍
## 概览
为一个复杂的子系统提供一个一致的高层接口。这样，客户端代码就可以通过这个简化的接口与子系统交互，而不需要了解子系统内部的复杂性。主要解决降低客户端与复杂子系统之间的耦合度，和简化客户端对复杂系统的操作，隐藏内部实现细节。使用场景：当客户端不需要了解系统内部的复杂逻辑和组件交互时，和当需要为整个系统定义一个清晰的入口点时。

## 实现方式
- 创建外观类：定义一个类（外观），作为客户端与子系统之间的中介。
- 封装子系统操作：外观类将复杂的子系统操作封装成简单的方法。

## 关键代码
- Facade类：提供高层接口，简化客户端与子系统的交互。
- 子系统类：实现具体的业务逻辑，被Facade类调用。

### 优缺点

* 减少依赖：客户端与子系统之间的依赖减少。
* 提高灵活性：子系统的内部变化不会影响客户端。
* 增强安全性：隐藏了子系统的内部实现，只暴露必要的操作。
- 违反开闭原则：对子系统的修改可能需要对外观类进行相应的修改。

### 注意事项
- 在需要简化复杂系统访问时使用外观模式。
- 确保外观类提供的方法足够简单，以便于客户端使用。
- 外观模式适用于层次化结构，可以为每一层提供一个清晰的入口。
- 避免过度使用外观模式，以免隐藏过多的细节，导致维护困难。
### 结构
- 外观（Facade）：提供一个简化的接口，封装了系统的复杂性。外观模式的客户端通过与外观对象交互，而无需直接与系统的各个组件打交道。
- 子系统（Subsystem）：由多个相互关联的类组成，负责系统的具体功能。外观对象通过调用这些子系统来完成客户端的请求。
- 客户端（Client）：使用外观对象来与系统交互，而不需要了解系统内部的具体实现。

# 实现
我们将创建一个 `Shape` 接口和实现了 `Shape` 接口的实体类。下一步是定义一个外观类 `ShapeMaker`。
`ShapeMaker` 类使用实体类来代表用户对这些类的调用。`FacadePatternDemo` 类使用 `ShapeMaker` 类来显示结果。

## 创建一个接口
Shape.java
```java
public interface Shape {
   void draw();
}
```

## 创建实现接口的实体类
Rectangle.java
```java
public class Rectangle implements Shape {
 
   @Override
   public void draw() {
      System.out.println("Rectangle::draw()");
   }
}
```
Square.java
```java
public class Square implements Shape {
 
   @Override
   public void draw() {
      System.out.println("Square::draw()");
   }
}
```
Circle.java
```java
public class Circle implements Shape {
 
   @Override
   public void draw() {
      System.out.println("Circle::draw()");
   }
}
```

## 创建一个外观类
ShapeMaker.java
```java
public class ShapeMaker {
   private Shape circle;
   private Shape rectangle;
   private Shape square;
 
   public ShapeMaker() {
      circle = new Circle();
      rectangle = new Rectangle();
      square = new Square();
   }
 
   public void drawCircle(){
      circle.draw();
   }
   public void drawRectangle(){
      rectangle.draw();
   }
   public void drawSquare(){
      square.draw();
   }
}
```

## 使用该外观类画出各种类型的形状
FacadePatternDemo.java
```java
public class FacadePatternDemo {
   public static void main(String[] args) {
      ShapeMaker shapeMaker = new ShapeMaker();
 
      shapeMaker.drawCircle();
      shapeMaker.drawRectangle();
      shapeMaker.drawSquare();      
   }
}
```

## 输出Belike
```text
Circle::draw()
Rectangle::draw()
Square::draw()
```