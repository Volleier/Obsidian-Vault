原型模式（Prototype Pattern）是用于创建重复的对象，同时又能保证性能。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式之一。

这种模式是实现了一个原型接口，该接口用于创建当前对象的克隆。当直接创建对象的代价比较大时，则采用这种模式。例如，一个对象需要在一个高代价的数据库操作之后被创建。我们可以缓存该对象，在下一个请求时返回它的克隆，在需要的时候更新数据库，以此来减少数据库调用。

# 介绍
 使用原型实例指定要创建对象的种类，并通过拷贝这些原型创建新的对象，主要解决在运行时动态建立和删除原型。
 在以下等场景中使用：
- 系统应独立于产品的创建、构成和表示。
- 需要在运行时指定实例化的类，例如通过动态加载。
- 避免创建与产品类层次平行的工厂类层次。
- 类的实例只能有几种不同状态组合，克隆原型比手工实例化更方便。
原型模式通过已有的一个原型对象，快速生成与原型对象相同的实例。

## 关键代码
- 实现克隆操作：
    - 在 Java 中，实现 `Cloneable` 接口，重写 `clone()` 方法。
    - 在 .NET 中，使用 `Object` 类的 `MemberwiseClone()` 方法实现浅拷贝，或通过序列化实现深拷贝。
- 隔离类对象的使用者和具体类型之间的耦合关系，要求"易变类"拥有稳定的接口。

## 优缺点
- 性能提高
- 避免构造函数的约束
- 配备克隆方法需要全面考虑类的功能，对已有类可能较难实现，特别是处理不支持串行化的间接对象或含有循环结构的引用时。
- 必须实现 `Cloneable` 接口。
## 注意事项
与直接实例化类创建新对象不同，原型模式通过拷贝现有对象生成新对象。

## 结构
原型模式包含以下几个主要角色：
- 原型接口（Prototype Interface）：定义一个用于克隆自身的接口，通常包括一个 `clone()` 方法。
- 具体原型类（Concrete Prototype）：实现原型接口的具体类，负责实际的克隆操作。这个类需要实现 `clone()` 方法，通常使用浅拷贝或深拷贝来复制自身。
- 客户端（Client）：使用原型实例来创建新的对象。客户端调用原型对象的 `clone()` 方法来创建新的对象，而不是直接使用构造函数。

# 实现
实现Shape，用ShaoeCache缓存Shape并由PrototypePatternDemo
![[原型模式-1.png]]
## 创建一个实现了 _Cloneable_ 接口的抽象类
Shape.java
```java
public abstract class Shape implements Cloneable {
   
   private String id;
   protected String type;
   
   abstract void draw();
   
   public String getType(){
      return type;
   }
   
   public String getId() {
      return id;
   }
   
   public void setId(String id) {
      this.id = id;
   }
   
   public Object clone() {
      Object clone = null;
      try {
         clone = super.clone();
      } catch (CloneNotSupportedException e) {
         e.printStackTrace();
      }
      return clone;
   }
}
```

## 创建扩展抽象类的实体类
Rectangle.java
```java
public class Rectangle extends Shape {
 
   public Rectangle(){
     type = "Rectangle";
   }
 
   @Override
   public void draw() {
      System.out.println("Inside Rectangle::draw() method.");
   }
}
```
Square.java
```java
public class Square extends Shape {
 
   public Square(){
     type = "Square";
   }
 
   @Override
   public void draw() {
      System.out.println("Inside Square::draw() method.");
   }
}
```
Circle.java
```java
public class Circle extends Shape {
 
   public Circle(){
     type = "Circle";
   }
 
   @Override
   public void draw() {
      System.out.println("Inside Circle::draw() method.");
   }
}
```

## 创建一个类，从数据库获取实体类，并把它们存储在一个 _Hashtable_ 中
ShapeCache.java
```java
import java.util.Hashtable;
 
public class ShapeCache {
    
   private static Hashtable<String, Shape> shapeMap 
      = new Hashtable<String, Shape>();
 
   public static Shape getShape(String shapeId) {
      Shape cachedShape = shapeMap.get(shapeId);
      return (Shape) cachedShape.clone();
   }
 
   // 对每种形状都运行数据库查询，并创建该形状
   // shapeMap.put(shapeKey, shape);
   // 例如，我们要添加三种形状
   public static void loadCache() {
      Circle circle = new Circle();
      circle.setId("1");
      shapeMap.put(circle.getId(),circle);
 
      Square square = new Square();
      square.setId("2");
      shapeMap.put(square.getId(),square);
 
      Rectangle rectangle = new Rectangle();
      rectangle.setId("3");
      shapeMap.put(rectangle.getId(),rectangle);
   }
}
```

## 使用 ShapeCache 类来获取存储在 Hashtable 中的形状的克隆
PrototypePatternDemo.java
```java
public class PrototypePatternDemo {
   public static void main(String[] args) {
      ShapeCache.loadCache();
 
      Shape clonedShape = (Shape) ShapeCache.getShape("1");
      System.out.println("Shape : " + clonedShape.getType());        
 
      Shape clonedShape2 = (Shape) ShapeCache.getShape("2");
      System.out.println("Shape : " + clonedShape2.getType());        
 
      Shape clonedShape3 = (Shape) ShapeCache.getShape("3");
      System.out.println("Shape : " + clonedShape3.getType());        
   }
}
```

## 输出Belike
```text
Shape : Circle
Shape : Square
Shape : Rectangle
```