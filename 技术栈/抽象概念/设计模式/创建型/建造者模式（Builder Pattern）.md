建造者模式是一种创建型设计模式，它允许你创建复杂对象的步骤与表示方式相分离。

建造者模式是一种创建型设计模式，它的主要目的是将一个复杂对象的构建过程与其表示相分离，从而可以创建具有不同表示形式的对象。

# 介绍
## 概要
建造者模式将一个复杂的构建过程与其表示相分离，使得同样的构建过程可以创建不同的表示。在软件系统中，一个复杂对象的创建通常由多个部分组成，这些部分的组合经常变化，但组合的算法相对稳定。适用于当一些基本部件不变，而其组合经常变化时。主要是将变与不变的部分分离开。

## 关键代码
- **建造者**：创建并提供实例。
- **导演**：管理建造出来的实例的依赖关系和控制构建过程。

## 优缺点
- 分离构建过程和表示，使得构建过程更加灵活，可以构建不同的表示。
- 可以更好地控制构建过程，隐藏具体构建细节。
- 代码复用性高，可以在不同的构建过程中重复使用相同的建造者。
- 如果产品的属性较少，建造者模式可能会导致代码冗余。
- 增加了系统的类和对象数量。

## 注意事项
与工厂模式的区别是：建造者模式更加关注于零件装配的顺序。

## 结构
建造者模式包含以下几个主要角色：
- 产品（Product）：要构建的复杂对象。产品类通常包含多个部分或属性。
- 抽象建造者（Builder）：定义了构建产品的抽象接口，包括构建产品的各个部分的方法。
- 具体建造者（Concrete Builder）：实现抽象建造者接口，具体确定如何构建产品的各个部分，并负责返回最终构建的产品。
- 指导者（Director）：负责调用建造者的方法来构建产品，指导者并不了解具体的构建过程，只关心产品的构建顺序和方式。

# 实现
一个组合是形状和颜色组成的。形状有长方形圆形等；形状都在轮廓内。颜色有红色蓝色等等；它们都在表面上。

创建一个表示图形条目的 `Combination` 接口和实现 `Combination` 接口的实体类，以及一个表示轮廓和表面的 `Showcase` 接口和实现 `Showcase` 接口的实体类，形状在轮廓类中，颜色在表面类中。

然后我们创建一个 `Graphics` 类，带有 `Combination` 的 `ArrayList` 和一个通过结合 `Combination` 来创建不同类型的 `Graphics` 对象的 `GraphicsBuilder`。`BuilderPatternDemo` 类使用 `GraphicsBuilder` 来创建一个 `Graphics`。

## 创建一个表示形状条目和形状包装的接口
形状包装：形状->轮廓，颜色->形状
Combination.java
```java
public interface Combination {
   public String name();
   public Showcase showcase();
   public float size();    
}
```
Showcase.java
```java
public interface Showcase {
   public String show();
}
```

## 创建实现 Showcase 接口的实体类
Outline.java
```java
public class Outline implements Showcase {
 
   @Override
   public String Show() {
      return "Outline";
   }
}
```
Surface.java
```java
public class Surface implements Showcase {
 
   @Override
   public String Show() {
      return "Surface";
   }
}
```

## 创建实现 Combination 接口的抽象类
Combination类提供了默认的功能
Shape.java
```java
public abstract class Shape implements Combination {
 
   @Override
   public Showcase showcase() {
      return new Outline();
   }
 
   @Override
   public abstract float size();
}
```
Color.java
```java
public abstract class Color implements Combination {
 
    @Override
    public Showcase showcase() {
       return new Surface();
    }
 
    @Override
    public abstract float size();
}
```

## 创建扩展了 Shape 和 Color 的实体类
Rectangle.java
```java
public class Rectangle extends Shape {
 
   @Override
   public float size() {
      return 100.f;
   }
 
   @Override
   public String name() {
      return "Rectangle";
   }
}
```
Circle.java
```java
public class Circle extends Shape {
 
   @Override
   public float size() {
      return 200.f;
   }
 
   @Override
   public String name() {
      return "Circle";
   }
}
```
Red.java
```java
public class Red extends Color {
 
   @Override
   public float size() {
      return 300.f;
   }
 
   @Override
   public String name() {
      return "Red";
   }
}
```
Blue.java
```java
public class Blue extends Color {
 
   @Override
   public float size() {
      return 400.f;
   }
 
   @Override
   public String name() {
      return "Blue";
   }
}
```
## 创建一个 Graphics 类，带有上面定义的 Combination 对象
Graphics.java
```java
import java.util.ArrayList;
import java.util.List;
 
public class Graphics {
   private List<Combination> combinations = new ArrayList<Combination>();    
 
   public void addCombination(Combination combination){
      combinations.add(combination);
   }
 
   public float getSize(){
      float size = 0.0f;
      for (Combination combination : combinations) {
         size += combination.size();
      }        
      return size;
   }
 
   public void showCombinations(){
      for (Combination combination : combinations) {
         System.out.print("Combination : "+combination.name());
         System.out.print(", Showcase : "+combination.showcase().show());
         System.out.println(", Size : "+combination.size());
      }        
   }    
}
```

## 创建一个 GraphicsBuilder 类
实际的 builder 类负责创建 Graphics 对象
Graphics.java
```java
public class GraphicsBuilder {
 
   public Graphics showGraphics1 (){
      Graphics graphics = new Graphics();
      graphics.addCombination(new Rectangle());
      graphics.addCombination(new Red());
      return graphics;
   }   
 
	public Graphics showGraphics2(){
	   Graphics graphics = new Graphics();
	   graphics.addCombination(new Circle());
	   graphics.addCombination(new Blue());
   return graphics;
	}
}
```

## 使用 GraphicsBuilder 来演示
BuilderPatternDemo.java
```java
public class BuilderPatternDemo {
   public static void main(String[] args) {
      GraphicsBuilder graphicsBuilder = new GraphicsBuilder();
 
      Graphics graphics1 = graphicsBuilder.showGraphics1();
      System.out.println("graphics1");
      graphics1.showCombinations();
      System.out.println("Total Size: " +graphics1.getSize());
 
      Graphics graphics2 = graphicsBuilder.showGraphics2();
      System.out.println("graphics2");
      graphics2.showCombinations();
      System.out.println("Total Size: " +graphics2.getSize());
   }
}
```

## 输出Belike
```text
Graphics1
Combination : Rectangle, Showcase : Outline, Size : 100.0
Combination : Red, Showcase : Surface, Size : 200.0
Total Size: 300.0

Graphics2
Combination : Circle, Showcase : Outline, Size : 300.0
Combination : Blue, Showcase : Suface, Size : 400.0
Total Size: 700.0
```