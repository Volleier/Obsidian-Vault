适配器模式（Adapter Pattern）充当两个不兼容接口之间的桥梁，属于结构型设计模式。它通过一个中间件（适配器）将一个类的接口转换成客户期望的另一个接口，使原本不能一起工作的类能够协同工作。
这种模式涉及到一个单一的类，该类负责加入独立的或不兼容的接口功能。例如：读卡器。

# 介绍
## 概览
适配器模式是一种软件设计模式，旨在解决不同接口之间的兼容性问题，目的是将一个类的接口转换为另一个接口，使得原本不兼容的类可以协同工作。在软件系统中，需要将现有的对象放入新环境，而新环境要求的接口与现有对象不匹配。适配器模式可以使用继承或依赖，推荐使用依赖关系，而不是继承，以保持灵活性。
## 关键代码
适配器通过继承或依赖现有对象，并实现所需的目标接口。
## 优缺点
- 促进了类之间的协同工作，即使它们没有直接的关联。
- 提高了类的复用性。
- 增加了类的透明度。
- 提供了良好的灵活性。
- 过度使用适配器可能导致系统结构混乱，难以理解和维护。
- 在Java中，由于只能继承一个类，因此只能适配一个类，且目标类必须是抽象的。

## 注意事项
- 适配器模式应谨慎使用，特别是在详细设计阶段，它更多地用于解决现有系统的问题。
- 在考虑修改一个正常运行的系统接口时，适配器模式是一个合适的选择。
通过这种方式，适配器模式可以清晰地表达其核心概念和应用，同时避免了不必要的复杂性。

## 结构
- 目标接口（Target）：定义客户需要的接口。
- 适配者类（Adaptee）：定义一个已经存在的接口，这个接口需要适配。
- 适配器类（Adapter）：实现目标接口，并通过组合或继承的方式调用适配者类中的方法，从而实现目标接口。

# 实现
我们有一个 `SquareGraphic` 接口和一个实现了 `SquareGraphic` 接口的实体类 `ShowSquare`，该类可以展示正方形。
我们还有另一个接口 `RectangleGraphic` 和实现了 `RectangleGraphic` 接口的实体类`RectangleShowcase`和`ColorShowcase`，该类可以展示长方形和颜色
如果想要让 `RectangleGraphic` 展示正方形，需要创建一个实现了 `SquareGraphic` 接口的适配器类 `GraphicAdapter`，并使用 `RectangleGraphics` 对象来展示对应图形。
`ShowSquare` 使用适配器类 `GraphicAdapter` 传递所需的图形，不需要知道实际类。`AdapterPatternDemo` 类使用 `ShowSquare` 类来播放各种格式。
![[适配器模式-1.png]]
## 为正方形和长方形创建接口
SquareGraphic.java
```java
public interface SquareGraphic {
    public void showSquare(String fileName);
}
```
RectangleGraphic.java
```java
public interface RectangleGraphic {
    public void showRectangle(String fileName);
    public void showColor(String fileName);
}
```
## 创建实现了 RectangleGraphics 接口的实体类
RectangleShowcase.java
```java
public class RectangleShowcase implements RectangleGraphic {
    @Override
    public void showRectangle(String fileName) {
        System.out.println("Showing rectangle: " + fileName);
    }
    
    @Override
    public void showColor(String fileName) {
        // 空实现
    }
}
```
ColorShowcase.java
```java
public class ColorShowcase implements RectangleGraphic {
    @Override
    public void showRectangle(String fileName) {
        // 空实现
    }
    
    @Override
    public void showColor(String fileName) {
        System.out.println("Showing color: " + fileName);
    }
}
```

## 创建实现了 SquareGraphic 接口的适配器类
GraphicAdapter.java
```java
public class GraphicAdapter implements SquareGraphic {
    private RectangleGraphic rectangleGraphic;
    
    public GraphicAdapter(String graphicType) {
        if(graphicType.equalsIgnoreCase("rectangle")) {
            rectangleGraphic = new RectangleShowcase();
        } else if(graphicType.equalsIgnoreCase("color")) {
            rectangleGraphic = new ColorShowcase();
        }
    }
    
    @Override
    public void showSquare(String fileName) {
        rectangleGraphic.showRectangle(fileName);
    }
}
```

## 创建实现了 SquareGraphic 接口的实体类
SquareShowcase.java
```java
public class SquareShowcase implements SquareGraphic {
   GraphicAdapter GraphicAdapter; 
 
   @Override
   public void play(String GraphicsType, String fileName) {    
 
      if(GraphicsType.equalsIgnoreCase("square")){
         System.out.println("Playing square file. Name: "+ fileName);         
      } 
      //GraphicAdapter 提供了展示长方形的支持
      else if(GraphicsType.equalsIgnoreCase("rectangle") 
         || GraphicsType.equalsIgnoreCase("color")){
         GraphicAdapter = new GraphicAdapter(GraphicsType);
         GraphicAdapter.play(GraphicsType, fileName);
      }
      else{
         System.out.println("Invalid graphy. "+
            GraphicsType + " format not supported");
      }
   }   
}
```

## 使用 AudioPlayer 来播放不同类型的音频格式
AdapterPatternDemo.java
```java
public class AdapterPatternDemo {
    public static void main(String[] args) {
        SquareGraphic square = new ShowSquare();
        square.showSquare("red_square");
        
        SquareGraphic adaptedRectangle = new GraphicAdapter("rectangle");
        adaptedRectangle.showSquare("blue_rectangle");
        
        SquareGraphic adaptedColor = new GraphicAdapter("color");
        adaptedColor.showSquare("green_color");
    }
}
```