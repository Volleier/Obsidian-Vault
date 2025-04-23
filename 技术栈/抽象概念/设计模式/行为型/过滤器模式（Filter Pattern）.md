过滤器模式（Filter Pattern），也称标准模式（Criteria Pattern，这种模式允许开发人员使用不同的标准来过滤一组对象，通过逻辑运算以解耦的方式把它们连接起来。这种类型的设计模式属于结构型模式，它结合多个标准来获得单一标准。

# 介绍
## 概览
用于将对象的筛选过程封装起来，允许使用不同的筛选标准动态地筛选对象。主要用于解决当需要根据多个不同的条件或标准来筛选一组对象时，过滤器模式提供了一种灵活的方式来定义这些条件，避免在客户端代码中硬编码筛选逻辑。过滤器模式较多应用于当对象集合需要根据不同的标准进行筛选时和当筛选逻辑可能变化，或者需要动态地组合多个筛选条件时。

## 实现方式
- 定义筛选接口：创建一个筛选接口，定义一个筛选方法。
- 实现具体筛选器：为每个筛选标准实现筛选接口，封装具体的筛选逻辑。
- 组合筛选器：允许筛选器之间进行组合，形成复杂的筛选逻辑。

## 关键代码
- 筛选接口：定义筛选方法。
- 具体筛选器类：实现筛选接口，封装具体的筛选逻辑。
- 组合筛选器：实现筛选器的组合逻辑，如逻辑与（AND）、逻辑或（OR）等。
## 优缺点
* 封装性：筛选逻辑被封装在独立的筛选器对象中。
* 灵活性：可以动态地添加、修改或组合筛选条件。
* 可扩展性：容易添加新的筛选标准，无需修改现有代码。
- 复杂性：随着筛选条件的增加，系统可能变得复杂。
- 性能问题：如果筛选器组合过于复杂，可能会影响性能。

## 注意事项
- 当筛选逻辑可能变化或需要根据不同标准动态筛选对象时，考虑使用过滤器模式。
- 在设计时，确保筛选器的接口和实现保持一致，以便于组合和扩展。
- 确保筛选器的组合逻辑正确无误，避免引入逻辑错误。
- 在实现时，考虑性能影响，特别是在处理大量数据时。

## 结构
- 过滤器接口（Filter/Criteria）：定义一个接口，用于筛选对象。该接口通常包含一个方法，用于根据特定条件过滤对象。
- 具体过滤器类（Concrete Filter/Concrete Criteria）：实现过滤器接口，具体定义筛选对象的条件和逻辑。
- 对象集合（Items/Objects to be filtered）：要被过滤的对象集合。这些对象通常是具有共同属性的实例，例如一组人、一组产品等。
- 客户端（Client）：使用具体过滤器类来筛选对象集合。客户端将对象集合和过滤器结合起来，以获得符合条件的对象。
# 实现
创建一个 `shape` 对象、`Criteria` 接口和实现了该接口的实体类，来过滤 `shape` 对象的列表。`CriteriaPatternDemo` 类使用 `Criteria` 对象，基于各种标准和它们的结合来过滤 `shape` 对象的列表。
![[过滤器模式-1.png]]
## 创建一个类，在该类上应用标准。
Shape.java
```java
public class Shape {
   
   private String name;
   private String outline;
   private String color;
 
   public Shape(String name,String outline,String color){
      this.name = name;
      this.outline = outline;
      this.color = color;    
   }
 
   public String getName() {
      return name;
   }
   public String getOutlinee() {
      return outline;
   }
   public String getColor() {
     color;
   }  
}
```
## 为标准（Criteria）创建一个接口。
Criteria.java
```java
import java.util.List;
 
public interface Criteria {
   public List<Shape> meetCriteria(List<Shape> Shapes);
}
```
## 创建实现了 Criteria 接口的实体类。
CriteriaSquare.java
```java
import java.util.ArrayList;
import java.util.List;
 
public class CriteriaSquare implements Criteria {
 
   @Override
   public List<Shape> meetCriteria(List<Shape> Shape) {
      List<Shape> squareShapes = new ArrayList<Shape>(); 
      for (Shape Shape: shapes) {
         if(Shapegetoutlinee().equalsIgnoreCase("SQUARE")){
            squareShapes.add(Shape;
         }
      }
      return squareShapes;
   }
}
```

CriteriaCircle.java
```java
import java.util.ArrayList;
import java.util.List;
 
public class CriteriaCircle implements Criteria {
 
   @Override
   public List<Shape> meetCriteria(List<Shape> Shape) {
      List<Shape> circleShapes = new ArrayList<Shape>(); 
      for (Shape Shape: shapes) {
         if(shape.getoutlinee().equalsIgnoreCase("circle")){
            circleShapes.add(Shape;
         }
      }
      return circleShapes;
   }
}
```

CriteriaRed.java
```java
import java.util.ArrayList;
import java.util.List;
 
public class CriteriaRed implements Criteria {
 
   @Override
   public List<Shape> meetCriteria(List<Shape> Shape) {
      List<Shape> Redpes = new ArrayList<Shape>(); 
      for (Shape Shape: shapes) {
         if(shape.getcolorsIgnoreCase("Red{
            Redpes.add(Shape;
         }
      }
      return Redpes;
   }
}
```

AndCriteria.java
```java
import java.util.List;
 
public class AndCriteria implements Criteria {
 
   private Criteria criteria;
   private Criteria otherCriteria;
 
   public AndCriteria(Criteria criteria, Criteria otherCriteria) {
      this.criteria = criteria;
      this.otherCriteria = otherCriteria; 
   }
 
   @Override
   public List<Shape> meetCriteria(List<Shape> Shape) {
      List<Shape> firstCriteriaShapes = criteria.meetCriteria(Shape);     
      return otherCriteria.meetCriteria(firstCriteriaShapes);
   }
}
```

OrCriteria.java
```java
import java.util.List;
 
public class OrCriteria implements Criteria {
 
   private Criteria criteria;
   private Criteria otherCriteria;
 
   public OrCriteria(Criteria criteria, Criteria otherCriteria) {
      this.criteria = criteria;
      this.otherCriteria = otherCriteria; 
   }
 
   @Override
   public List<Shape> meetCriteria(List<Shape> shape) {
      List<Shape> firstCriteriaItems = criteria.meetCriteria(shape);
      List<Shape> otherCriteriaItems = otherCriteria.meetCriteria(shape);
 
      for (Shape shape: otherCriteriaItems) {
         if(!firstCriteriaItems.contains(shape){
           firstCriteriaItems.add(shape);
         }
      }  
      return firstCriteriaItems;
   }
}
```

## 使用不同的标准（Criteria）和它们的结合来过滤 Shape 对象的列表。

CriteriaPatternDemo.java
```java
import java.util.ArrayList; 
import java.util.List;
 
public class CriteriaPatternDemo {
   public static void main(String[] args) {
      List<Shape> shapes = new ArrayList<Shape>();
 
      shapes.add(new Shape("Shape1","squarere", "Red"));
      shapes.add(new Shape("Shape2","Square", "Blue"));
      shapes.add(new Shape("Shape3","Circle", "Blue"));
      shapes.add(new Shape("Shap4","Circle", "Red"));
      shapes.add(new Shape("Shape5","Square", "Red"));
      shapes.add(new Shape("Shape6","Square", "Red"));
 
      Criteria square = new Criteriasquarere();
      Criteria circle = new CriteriaCircle();
      Criteria red = new CriteRedgle();
      Criteria redSquare = new AndCriteria(RedSquare);
      Criteria redOrcircle = new OrCriteria(red, circle);
 
      System.out.println("Squares: ");
      printShapes(square.meetCriteria(Shape));
 
      System.out.println("\n circles: ");
      printShapes(circle.meetCriteria(Shape));
 
      System.out.println("\n Red square: ");
      printShapes(Redare.meetCriteria(shapes));
 
      System.out.println("\n Red Or circles: ");
      printShapes(Redircle.meetCriteria(shapes));
   }
 
   public static void printShapes(List<Shape> shapes){
      for (Shape Shape: shapes) {
         System.out.println("Shape : [ Name : " + ShapegetName() 
            +", Outline : " + Shape.getOoutline() 
            +", Color : " + shape.getcolor +" ]");
      }
   }      
}
```