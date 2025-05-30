在模板模式（Template Pattern）中，一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。

# 介绍

## 概览

模板模式在父类中定义了算法的骨架，并允许子类在不改变算法结构的前提下重定义算法的某些特定步骤，模板模式适用于当存在一些通用的方法，可以在多个子类中共用时。模板模式解决在多个子类中重复实现相同的方法的问题，通过将通用方法抽象到父类中来避免代码重复。

## 实现方式
- 定义抽象父类：包含模板方法和一些抽象方法或具体方法。
- 实现子类：继承抽象父类并实现抽象方法，不改变算法结构。

## 关键代码
- 模板方法：在抽象父类中定义，调用抽象方法和具体方法。
- 抽象方法：由子类实现，代表算法的可变部分。
- 具体方法：在抽象父类中实现，代表算法的不变部分。

## 优缺点

- 封装不变部分：算法的不变部分被封装在父类中。
- 扩展可变部分：子类可以扩展或修改算法的可变部分。
- 提取公共代码：减少代码重复，便于维护。
- 类数目增加：每个不同的实现都需要一个子类，可能导致系统庞大。

## 注意事项
- 当有多个子类共有的方法且逻辑相同时，考虑使用模板方法模式。
- 对于重要或复杂的方法，可以考虑作为模板方法定义在父类中。
- 为了防止恶意修改，模板方法通常使用`final`关键字修饰，避免被子类重写。

## 包含的几个主要角色
- 抽象父类（Abstract Class）：定义了模板方法和一些抽象方法或具体方法。
- 具体子类（Concrete Classes）：继承自抽象父类，并实现抽象方法。
- 钩子方法（Hook Method）（可选）：在抽象父类中定义，可以被子类重写，以影响模板方法的行为。
- 客户端（Client）（可选）：使用抽象父类和具体子类，无需关心模板方法的细节。

# 实现
我们将创建一个定义操作的 `Shape` 抽象类，其中，模板方法设置为 final，这样它就不会被重写。`Rectangle` 和 `Square` 是扩展了 `Shape` 的实体类，它们重写了抽象类的方法。
`TemplatePatternDemo`，我们的演示类使用 `Shape` 来演示模板模式的用法。
![[模板模式-1.png]]

## 创建一个抽象类，它的模板方法被设置为 final
Shape.java
```java
public abstract class Shape {
   abstract void show();
   abstract void move();
   abstract void rotate();
 
   //模板
   public final void play(){
 
      show();
 
      move();
 
      rotate();
   }
}
```

## 创建扩展了上述类的实体类
Rectangle.java
```java
public class Rectangle extends Shape {
 
   @Override
   void show() {
      System.out.println("Rectangle Show");
   }
 
   @Override
   void move() {
      System.out.println("Rectangle Move");
   }
 
   @Override
   void rotate() {
      System.out.println("Rectangle Rotate");
   }
}
```
Square.java
```java
public class Square extends Shape {
 
   @Override
   void show() {
      System.out.println("Square Show");
   }
 
   @Override
   void move() {
      System.out.println("Square Move");
   }
 
   @Override
   void rotate() {
      System.out.println("Square Rotate");
   }
}
```

## 使用 Shape 的模板方法 play() 来演示游戏的定义方式
TemplatePatternDemo.java
```java
public class TemplatePatternDemo {
   public static void main(String[] args) {
      Shape shape = new Rectangle();
      shape.play();
      System.out.println();
      shape = new Square();
      shape.play();      
   }
}
```

## 输出Belike
```text
Rectangle Show
Rectangle Move
Rectangle Rotate

Square Show
Square Move
Square Rotate
```