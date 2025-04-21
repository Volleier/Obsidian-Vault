工厂模式提供了一种创建对象的方式，使得创建对象的过程与使用对象的过程分离。
通过使用工厂模式，可以将对象的创建逻辑封装在一个工厂类中，而不是在客户端代码中直接实例化对象，这样可以提高代码的可维护性和可扩展性。
1. 简单工厂模式（Simple Factory Pattern）：简单工厂模式不是一个正式的设计模式，但它是工厂模式的基础。它使用一个单独的工厂类来创建不同的对象，根据传入的参数决定创建哪种类型的对象。
2. 工厂方法模式（Factory Method Pattern）：工厂方法模式定义了一个创建对象的接口，但由子类决定实例化哪个类。工厂方法将对象的创建延迟到子类。
3. 抽象工厂模式（Abstract Factory Pattern）：抽象工厂模式提供一个创建一系列相关或互相依赖对象的接口，而无需指定它们具体的类。

# 介绍
## 概览
工厂模式定义一个创建对象的接口，让其子类决定实例化哪一个具体的类。工厂模式使对象的创建过程延迟到子类。解决接口选择的问题。
当需要在不同条件下创建不同实例时。通过让子类实现工厂接口，返回一个抽象的产品。对象的创建过程在子类中实现。

## 优缺点
1. 调用者只需要知道对象的名称即可创建对象。
2. 扩展性高，如果需要增加新产品，只需扩展一个工厂类即可。
3. 屏蔽了产品的具体实现，调用者只关心产品的接口。
4. 每次增加一个产品时，都需要增加一个具体类和对应的工厂，使系统中类的数量成倍增加，增加了系统的复杂度和具体类的依赖。
## 注意事项
工厂模式适用于生成复杂对象的场景。如果对象较为简单，通过 new 即可完成创建，则不必使用工厂模式。使用工厂模式会引入一个工厂类，增加系统复杂度。

## 结构
工厂模式包含以下几个主要角色：
- 抽象产品（Abstract Product）：定义了产品的共同接口或抽象类。它可以是具体产品类的父类或接口，规定了产品对象的共同方法。
- 具体产品（Concrete Product）：实现了抽象产品接口，定义了具体产品的特定行为和属性。
- 抽象工厂（Abstract Factory）：声明了创建产品的抽象方法，可以是接口或抽象类。它可以有多个方法用于创建不同类型的产品。
- 具体工厂（Concrete Factory）：实现了抽象工厂接口，负责实际创建具体产品的对象。

# 实现
创建一个`Shape`接口和实现`Shape`接口的实体类。下一步是定义工厂类 `ShapeFactory`。
`FactoryPatternDemo` 类使用 `ShapeFactory` 来获取 `Shape` 对象。它将向 `ShapeFactory` 传递信息（`CIRCLE / RECTANGLE / SQUARE`），以便获取它所需对象的类型。![[工厂模式-1.png]]
## 创建接口
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
      System.out.println("Inside Rectangle::draw() method.");
   }
}
```
Square.java
```java
public class Square implements Shape {
 
   @Override
   public void draw() {
      System.out.println("Inside Square::draw() method.");
   }
}
```
Circle.java
```java
public class Circle implements Shape {
 
   @Override
   public void draw() {
      System.out.println("Inside Circle::draw() method.");
   }
}
```

## 创建一个工厂，生成基于给定信息的实体类的对象
ShapeFactory.java
```java
public class ShapeFactory {
    
   //使用 getShape 方法获取形状类型的对象
   public Shape getShape(String shapeType){
      if(shapeType == null){
         return null;
      }        
      if(shapeType.equalsIgnoreCase("CIRCLE")){
         return new Circle();
      } else if(shapeType.equalsIgnoreCase("RECTANGLE")){
         return new Rectangle();
      } else if(shapeType.equalsIgnoreCase("SQUARE")){
         return new Square();
      }
      return null;
   }
}
```
## 使用该工厂，通过传递类型信息来获取实体类的对象
FactoryPatternDemo.java
```java
public class FactoryPatternDemo {
 
   public static void main(String[] args) {
      ShapeFactory shapeFactory = new ShapeFactory();
 
      //获取 Circle 的对象，并调用它的 draw 方法
      Shape shape1 = shapeFactory.getShape("CIRCLE");
 
      //调用 Circle 的 draw 方法
      shape1.draw();
 
      //获取 Rectangle 的对象，并调用它的 draw 方法
      Shape shape2 = shapeFactory.getShape("RECTANGLE");
 
      //调用 Rectangle 的 draw 方法
      shape2.draw();
 
      //获取 Square 的对象，并调用它的 draw 方法
      Shape shape3 = shapeFactory.getShape("SQUARE");
 
      //调用 Square 的 draw 方法
      shape3.draw();
   }
}
```

## 输出belike：
```text
Inside Circle::draw() method.
Inside Rectangle::draw() method.
Inside Square::draw() method.
```