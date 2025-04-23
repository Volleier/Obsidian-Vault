享元模式（Flyweight Pattern）主要用于减少创建对象的数量，以减少内存占用和提高性能。这种类型的设计模式属于结构型模式，它提供了减少对象数量从而改善应用所需的对象结构的方式。
享元模式尝试重用现有的同类对象，如果未找到匹配的对象，则创建新对象。

# 介绍

## 概览
通过共享对象来减少创建大量相似对象时的内存消耗。主要解决的问题：避免因创建大量对象而导致的内存溢出问题，和通过共享对象，提高内存使用效率。三种推荐使用场景：当系统中存在大量相似或相同的对象、对象的创建和销毁成本较高、对象的状态可以外部化，即对象的部分状态可以独立于对象本身存在。

## 实现方式
- 定义享元接口：创建一个享元接口，规定可以共享的状态。
- 创建具体享元类：实现该接口的具体类，包含内部状态。
- 使用享元工厂：创建一个工厂类，用于管理享元对象的创建和复用。

## 关键代码
- HashMap：使用哈希表存储已经创建的享元对象，以便快速检索。

## 优缺点

- **减少内存消耗**：通过共享对象，减少了内存中对象的数量。
- **提高效率**：减少了对象创建的时间，提高了系统效率。
- **增加系统复杂度**：需要分离内部状态和外部状态，增加了设计和实现的复杂性。
- **线程安全问题**：如果外部状态处理不当，可能会引起线程安全问题。

## 注意事项

- 在创建大量相似对象时考虑使用享元模式。
- 确保享元对象的内部状态是共享的，而外部状态是独立于对象的。
- **状态分离**：明确区分内部状态和外部状态，避免混淆。
- **享元工厂**：使用享元工厂来控制对象的创建和复用，确保对象的一致性和完整性。

## 结构
- 享元工厂（Flyweight Factory）：负责创建和管理享元对象，通常包含一个池（缓存）用于存储和复用已经创建的享元对象。
- 具体享元（Concrete Flyweight）：实现了抽象享元接口，包含了内部状态和外部状态。内部状态是可以被共享的，而外部状态则由客户端传递。
- 抽象享元（Flyweight）：定义了具体享元和非共享享元的接口，通常包含了设置外部状态的方法。
- 客户端（Client）：使用享元工厂获取享元对象，并通过设置外部状态来操作享元对象。客户端通常不需要关心享元对象的具体实现。

# 实现

创建一个 `Shape` 接口和实现了 `Shape` 接口的实体类 `Circle`。下一步是定义工厂类 `ShapeFactory`。
`ShapeFactory` 有一个 `Circle` 的 `HashMap`，其中键名为 `Circle` 对象的颜色。无论何时接收到请求，都会创建一个特定颜色的圆。`ShapeFactory` 检查它的 `HashMap` 中的 circle 对象，如果找到 `Circle` 对象，则返回该对象，否则将创建一个存储在 hashmap 中以备后续使用的新对象，并把该对象返回到客户端。
`FlyWeightPatternDemo` 类使用 `ShapeFactory` 来获取 `Shape` 对象。它将向 `ShapeFactory` 传递信息（`red / green / blue/ black / white`），以便获取它所需对象的颜色。
![[享元模式-1.png]]

## 创建一个接口
Shape.java
```java
public interface Shape {
   void draw();
}
```

## 创建实现接口的实体类
Circle.java
```java
public class Circle implements Shape {
   private String color;
   private int x;
   private int y;
   private int radius;
 
   public Circle(String color){
      this.color = color;     
   }
 
   public void setX(int x) {
      this.x = x;
   }
 
   public void setY(int y) {
      this.y = y;
   }
 
   public void setRadius(int radius) {
      this.radius = radius;
   }
 
   @Override
   public void draw() {
      System.out.println("Circle: Draw() [Color : " + color 
         +", x : " + x +", y :" + y +", radius :" + radius);
   }
}
```

## 创建一个工厂，生成基于给定信息的实体类的对象
ShapeFactory.java
```java
import java.util.HashMap;
 
public class ShapeFactory {
   private static final HashMap<String, Shape> circleMap = new HashMap<>();
 
   public static Shape getCircle(String color) {
      Circle circle = (Circle)circleMap.get(color);
 
      if(circle == null) {
         circle = new Circle(color);
         circleMap.put(color, circle);
         System.out.println("Creating circle of color : " + color);
      }
      return circle;
   }
}
```

## 使用该工厂，通过传递颜色信息来获取实体类的对象
FlyweightPatternDemo.java
```java
public class FlyweightPatternDemo {
   private static final String colors[] = 
      { "Red", "Green", "Blue", "White", "Black" };
   public static void main(String[] args) {
 
      for(int i=0; i < 20; ++i) {
         Circle circle = 
            (Circle)ShapeFactory.getCircle(getRandomColor());
         circle.setX(getRandomX());
         circle.setY(getRandomY());
         circle.setRadius(100);
         circle.draw();
      }
   }
   private static String getRandomColor() {
      return colors[(int)(Math.random()*colors.length)];
   }
   private static int getRandomX() {
      return (int)(Math.random()*100 );
   }
   private static int getRandomY() {
      return (int)(Math.random()*100);
   }
}
```

## 输出Belike
```text
Creating circle of color : Black
Circle: Draw() [Color : Black, x : 36, y :71, radius :100
Creating circle of color : Green
Circle: Draw() [Color : Green, x : 27, y :27, radius :100
Creating circle of color : White
Circle: Draw() [Color : White, x : 64, y :10, radius :100
Creating circle of color : Red
Circle: Draw() [Color : Red, x : 15, y :44, radius :100
Circle: Draw() [Color : Green, x : 19, y :10, radius :100
Circle: Draw() [Color : Green, x : 94, y :32, radius :100
Circle: Draw() [Color : White, x : 69, y :98, radius :100
Creating circle of color : Blue
Circle: Draw() [Color : Blue, x : 13, y :4, radius :100
Circle: Draw() [Color : Green, x : 21, y :21, radius :100
Circle: Draw() [Color : Blue, x : 55, y :86, radius :100
Circle: Draw() [Color : White, x : 90, y :70, radius :100
Circle: Draw() [Color : Green, x : 78, y :3, radius :100
Circle: Draw() [Color : Green, x : 64, y :89, radius :100
Circle: Draw() [Color : Blue, x : 3, y :91, radius :100
Circle: Draw() [Color : Blue, x : 62, y :82, radius :100
Circle: Draw() [Color : Green, x : 97, y :61, radius :100
Circle: Draw() [Color : Green, x : 86, y :12, radius :100
Circle: Draw() [Color : Green, x : 38, y :93, radius :100
Circle: Draw() [Color : Red, x : 76, y :82, radius :100
Circle: Draw() [Color : Blue, x : 95, y :82, radius :100
```