抽象工厂模式（Abstract Factory Pattern）是围绕一个超级工厂创建其他工厂。该超级工厂又称为其他工厂的工厂。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
在抽象工厂模式中，接口是负责创建一个相关对象的工厂，不需要显式指定它们的类。每个生成的工厂都能按照工厂模式提供对象。
抽象工厂模式提供了一种创建一系列相关或相互依赖对象的接口，而无需指定具体实现类。通过使用抽象工厂模式，可以将客户端与具体产品的创建过程解耦，使得客户端可以通过工厂接口来创建一族产品。
# 介绍
## 概览
抽象工厂提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们的具体类。解决接口选择的问题。适用于当系统需要创建多个相关或依赖的对象，而不需要指定具体类时。
在一个产品族中定义多个产品，由具体工厂实现创建这些产品的方法。在一个工厂中聚合多个同类产品的创建方法。

## 优缺点
* 确保同一产品族的对象一起工作。
* 客户端不需要知道每个对象的具体类，简化了代码。
* 扩展产品族非常困难。增加一个新的产品族需要修改抽象工厂和所有具体工厂的代码。
## 注意事项
增加新的产品族相对容易，而增加新的产品等级结构比较困难。

## 结构
抽象工厂模式包含以下几个主要角色：
- 抽象工厂（Abstract Factory）：声明了一组用于创建产品对象的方法，每个方法对应一种产品类型。抽象工厂可以是接口或抽象类。
- 具体工厂（Concrete Factory）：实现了抽象工厂接口，负责创建具体产品对象的实例。
- 抽象产品（Abstract Interface）：定义了一组产品对象的共同接口或抽象类，描述了产品对象的公共方法。
- 具体产品（Concrete Interface）：实现了抽象产品接口，定义了具体产品的特定行为和属性。
抽象工厂模式通常涉及一族相关的产品，每个具体工厂类负责创建该族中的具体产品。客户端通过使用抽象工厂接口来创建产品对象，而不需要直接使用具体产品的实现类。

# 实现
创建 `Shape` 和 `Color` 接口和实现这些接口的实体类。下一步是创建抽象工厂类 `AbstractFactory`。接着定义工厂类 `ShapeFactory` 和 `ColorFactory`，这两个工厂类都是扩展了 `AbstractFactory`。然后创建一个工厂创造器/生成器类 `FactoryProducer`。

`AbstractFactoryPatternDemo` 类使用 `FactoryProducer` 来获取 `AbstractFactory` 对象。它将向 `AbstractFactory` 传递形状信息 `Shape`（`CIRCLE / RECTANGLE / SQUARE`），以便获取它所需对象的类型。同时它还向 `AbstractFactory` 传递颜色信息 `Color`（`RED / GREEN / BLUE`），以便获取它所需对象的类型。
![[抽象工厂模式-1.png]]
## 创建两个接口
创建形状何颜色的接口
Shape.java
```java
public interface Shape {
   void draw();
}
```
Color.java
```java
public interface Color {
   void fill();
}
```


## 创建实现接口的实体类
为Shape创建实体类
Rectangle.java
```java
public class Rectangle implements Shape {
 
   @Override
   public void function1() {
      System.out.println("Inside Rectangle::draw() method.");
   }
}
```
Square.java
```java
public class Square implements Shape {
 
   @Override
   public void function1() {
      System.out.println("Inside Square::draw() method.");
   }
}
```
Circle.java
```java
public class Circle implements Shape {
 
   @Override
   public void function1() {
      System.out.println("Inside Circle::draw() method.");
   }
}
```
以及为Color创建实体类
Red.java
```java
public class Red implements Color {
 
   @Override
   public void fill() {
      System.out.println("Inside Red::fill() method.");
   }
}
```
Blue.java
```java
public class Blue implements Color {
 
   @Override
   public void fill() {
      System.out.println("Inside Blue::fill() method.");
   }
}
```
Yellow.java
```java
public class Yellow implements Color {
 
   @Override
   public void fill() {
      System.out.println("Inside Yellow::fill() method.");
   }
}
```

## 为 Color 和 Shape 对象创建抽象类来获取工厂
AbstractFactory.java
```java
public abstract class AbstractFactory {
   public abstract Color getColor(String color);
   public abstract Shape getShape(String shape);
}
```

## 创建扩展 AbstractFactory 的工厂类
基于给定的信息生成实体类的对象
ShapeFactory.java
```java
public class ShapeFactory extends AbstractFactory {
    
   @Override
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
   
   @Override
   public Color getColor(String color) {
      return null;
   }
}
```
ColorFactory.java
```java
public class ColorFactory extends AbstractFactory {
    
   @Override
   public Shape getShape(String shapeType){
      return null;
   }
   
   @Override
   public Color getColor(String color) {
      if(color == null){
         return null;
      }        
      if(color.equalsIgnoreCase("RED")){
         return new Red();
      } else if(color.equalsIgnoreCase("GREEN")){
         return new Green();
      } else if(color.equalsIgnoreCase("BLUE")){
         return new Blue();
      }
      return null;
   }
}
```

## 创建一个工厂创造器/生成器类
创建一个工厂创造器/生成器类，通过传递形状或颜色信息来获取工厂
FactoryProducer.java
```java
public class FactoryProducer {
   public static AbstractFactory getFactory(String choice){
      if(choice.equalsIgnoreCase("SHAPE")){
         return new ShapeFactory();
      } else if(choice.equalsIgnoreCase("COLOR")){
         return new ColorFactory();
      }
      return null;
   }
}
```

## 使用 FactoryProducer 来获取 AbstractFactory
使用 FactoryProducer 来获取 AbstractFactory，通过传递类型信息来获取实体类的对象。
AbstractFactoryPatternDemo.java
```java
public class AbstractFactoryPatternDemo {
   public static void main(String[] args) {
 
      //获取形状工厂
      AbstractFactory shapeFactory = FactoryProducer.getFactory("SHAPE");
 
      //获取形状为 Circle 的对象
      Shape shape1 = shapeFactory.getShape("CIRCLE");
 
      //调用 Circle 的 draw 方法
      shape1.draw();
 
      //获取形状为 Rectangle 的对象
      Shape shape2 = shapeFactory.getShape("RECTANGLE");
 
      //调用 Rectangle 的 draw 方法
      shape2.draw();
      
      //获取形状为 Square 的对象
      Shape shape3 = shapeFactory.getShape("SQUARE");
 
      //调用 Square 的 draw 方法
      shape3.draw();
 
      //获取颜色工厂
      AbstractFactory colorFactory = FactoryProducer.getFactory("COLOR");
 
      //获取颜色为 Red 的对象
      Color color1 = colorFactory.getColor("RED");
 
      //调用 Red 的 fill 方法
      color1.fill();
 
      //获取颜色为 Green 的对象
      Color color2 = colorFactory.getColor("GREEN");
 
      //调用 Green 的 fill 方法
      color2.fill();
 
      //获取颜色为 Blue 的对象
      Color color3 = colorFactory.getColor("BLUE");
 
      //调用 Blue 的 fill 方法
      color3.fill();
   }
}
```

## 输出Belike
```text
Inside Circle::draw() method.
Inside Rectangle::draw() method.
Inside Square::draw() method.
Inside Red::fill() method.
Inside Green::fill() method.
Inside Blue::fill() method.
```