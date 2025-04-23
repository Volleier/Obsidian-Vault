装饰器模式（Decorator Pattern）允许向一个现有的对象添加新的功能，同时又不改变其结构。这种类型的设计模式属于结构型模式，它是作为现有的类的一个包装。
装饰器模式通过将对象包装在装饰器类中，以便动态地修改其行为。
这种模式创建了一个装饰类，用来包装原有的类，并在保持类方法签名完整性的前提下，提供了额外的功能。
装饰器模式通过嵌套包装多个装饰器对象，可以实现多层次的功能增强。每个具体装饰器类都可以选择性地增加新的功能，同时保持对象接口的一致性。

# 介绍
## 概览
装饰器模式动态地给一个对象添加额外的职责，同时不改变其结构。装饰器模式提供了一种灵活的替代继承方式来扩展功能。主要解决的问题避免通过继承引入静态特征，特别是在子类数量急剧膨胀的情况下；以及允许在运行时动态地添加或修改对象的功能。
在以下场景推荐使用：当需要在不增加大量子类的情况下扩展类的功能、当需要动态地添加或撤销对象的功能。

## 实现方式
- 定义组件接口：创建一个接口，规定可以动态添加职责的对象的标准。
- 创建具体组件：实现该接口的具体类，提供基本功能。
- 创建抽象装饰者：实现同样的接口，持有一个组件接口的引用，可以在任何时候动态地添加功能。
- 创建具体装饰者：扩展抽象装饰者，添加额外的职责。

## 关键代码
- Component接口：定义了可以被装饰的对象的标准。
- ConcreteComponent类：实现Component接口的具体类。
- Decorator抽象类：实现Component接口，并包含一个Component接口的引用。
- ConcreteDecorator类：扩展Decorator类，添加额外的功能。

## 优缺点
- 低耦合：装饰类和被装饰类可以独立变化，互不影响。
- 灵活性：可以动态地添加或撤销功能。
- 替代继承：提供了一种继承之外的扩展对象功能的方式。
- 复杂性：多层装饰可能导致系统复杂性增加。

## 注意事项
- 在需要动态扩展功能时，考虑使用装饰器模式。
- 保持装饰者和具体组件的接口一致，以确保灵活性。
- 装饰器模式可以替代继承，但应谨慎使用，避免过度装饰导致系统复杂。

## 结构
- 抽象组件（Component）：定义了原始对象和装饰器对象的公共接口或抽象类，可以是具体组件类的父类或接口。
- 具体组件（Concrete Component）：是被装饰的原始对象，它定义了需要添加新功能的对象。
- 抽象装饰器（Decorator）：继承自抽象组件，它包含了一个抽象组件对象，并定义了与抽象组件相同的接口，同时可以通过组合方式持有其他装饰器对象。
- 具体装饰器（Concrete Decorator）：实现了抽象装饰器的接口，负责向抽象组件添加新的功能。具体装饰器通常会在调用原始对象的方法之前或之后执行自己的操作。

# 实现
我们将创建一个 `Shape` 接口和实现了 `Shape` 接口的实体类。然后我们创建一个实现了 `Shape` 接口的抽象装饰类 `ShapeDecorator`，并把 `Shape` 对象作为它的实例变量。
`RedShapeDecorator` 是实现了 `ShapeDecorator` 的实体类。
`DecoratorPatternDemo` 类使用 `RedShapeDecorator` 来装饰 `Shape` 对象。
![[装饰器模式-1.png]]
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
      System.out.println("Shape: Rectangle");
   }
}
```
Circle.java
```java
public class Circle implements Shape {
 
   @Override
   public void draw() {
      System.out.println("Shape: Circle");
   }
}
```

## 创建实现了 Shape 接口的抽象装饰类
ShapeDecorator.java
```java
public abstract class ShapeDecorator implements Shape {
   protected Shape decoratedShape;
 
   public ShapeDecorator(Shape decoratedShape){
      this.decoratedShape = decoratedShape;
   }
 
   public void draw(){
      decoratedShape.draw();
   }  
}
```

## 创建扩展了 ShapeDecorator 类的实体装饰类
RedShapeDecorator.java
```java
public class RedShapeDecorator extends ShapeDecorator {
 
   public RedShapeDecorator(Shape decoratedShape) {
      super(decoratedShape);     
   }
 
   @Override
   public void draw() {
      decoratedShape.draw();         
      setRedBorder(decoratedShape);
   }
 
   private void setRedBorder(Shape decoratedShape){
      System.out.println("Border Color: Red");
   }
}
```

## 使用 RedShapeDecorator 来装饰 Shape 对象
```java
public class DecoratorPatternDemo {
   public static void main(String[] args) {
 
      Shape circle = new Circle();
      ShapeDecorator redCircle = new RedShapeDecorator(new Circle());
      ShapeDecorator redRectangle = new RedShapeDecorator(new Rectangle());
      //Shape redCircle = new RedShapeDecorator(new Circle());
      //Shape redRectangle = new RedShapeDecorator(new Rectangle());
      System.out.println("Circle with normal border");
      circle.draw();
 
      System.out.println("\nCircle of red border");
      redCircle.draw();
 
      System.out.println("\nRectangle of red border");
      redRectangle.draw();
   }
}
```

## 输出Belike
```text
Circle with normal border
Shape: Circle

Circle of red border
Shape: Circle
Border Color: Red

Rectangle of red border
Shape: Rectangle
Border Color: Red
```