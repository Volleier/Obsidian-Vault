组合模式（Composite Pattern），又叫部分整体模式，是用于把一组相似的对象当作一个单一的对象。组合模式依据树形结构来组合对象，用来表示部分以及整体层次。这种类型的设计模式属于结构型模式，它创建了对象组的树形结构。
这种模式创建了一个包含自己对象组的类。该类提供了修改相同对象组的方式。
# 介绍
## 概览
组合模式将对象组合成树形结构以表示"部分-整体"的层次结构。组合模式使得用户对单个对象和组合对象的使用具有一致性。组合模式用于解决两个问题：简化树形结构中对象的处理，无论它们是单个对象还是组合对象；解耦客户端代码与复杂元素的内部结构，使得客户端可以统一处理所有类型的节点。推荐以下场景使用：当需要表示对象的层次结构时，如文件系统或组织结构、当希望客户端代码能够以一致的方式处理树形结构中的所有对象时。
## 实现方式
- 统一接口：定义一个接口，所有对象（树枝和叶子）都实现这个接口。
- 组合结构：树枝对象包含一个接口的引用列表，这些引用可以是叶子或树枝。

## 关键代码
- Component接口：定义了所有对象必须实现的操作。
- Leaf类：实现Component接口，代表树中的叶子节点。
- Composite类：也实现Component接口，并包含其他Component对象的集合。

## 优缺点
* 简化客户端代码：客户端可以统一处理所有类型的节点。
* 易于扩展：可以轻松添加新的叶子类型或树枝类型。
- 违反依赖倒置原则：组件的声明是基于具体类而不是接口，这可能导致代码的灵活性降低。

## 注意事项
- 在设计时，优先使用接口而非具体类，以提高系统的灵活性和可维护性。
- 适用于需要处理复杂树形结构的场景，如文件系统、组织结构等。
- 在实现时，确保所有组件都遵循统一的接口，以保持一致性。
- 考虑使用工厂模式来创建不同类型的组件，以进一步解耦组件的创建逻辑。

## 结构
- 组件（Component）：定义了组合中所有对象的通用接口，可以是抽象类或接口。它声明了用于访问和管理子组件的方法，包括添加、删除、获取子组件等。
- 叶子节点（Leaf）：表示组合中的叶子节点对象，叶子节点没有子节点。它实现了组件接口的方法，但通常不包含子组件。
- 复合节点（Composite）：表示组合中的复合对象，复合节点可以包含子节点，可以是叶子节点，也可以是其他复合节点。它实现了组件接口的方法，包括管理子组件的方法。
- 客户端（Client）：通过组件接口与组合结构进行交互，客户端不需要区分叶子节点和复合节点，可以一致地对待整体和部分。

# 实现
有一个类 `Shape`，该类被当作组合模型类。`CompositePatternDemo` 类使用 `Shape` 类来添加形状层次结构，并打印所有图形。
![[组合模式-1.png]]
## 创建 Shape 类，该类带有 Shape 对象的列表
Shape.java
```java
import java.util.ArrayList;
import java.util.List;
 
public class Shape {
   private String name;
   private String color;
   private int size;
   private List<Shape> relation;
 
   //构造函数
   public Shape(String name,String color, int size) {
      this.name = name;
      this.color = color;
      this.size = size;
      relations = new ArrayList<Shape>();
   }
 
   public void add(Shape s) {
      relations.add(s);
   }
 
   public void remove(Shape s) {
      relations.remove(s);
   }
 
   public List<Shape> getRelations(){
     return relations;
   }
 
   public String toString(){
      return ("Shape :[ Name : "+ name 
      +", color : "+ color + ", size :"
      + size+" ]");
   }   
}
```

## 使用 Shape 类来创建和打印图形的层次结构
CompositePatternDemo.java
```java
public class CompositePatternDemo {
	public static void main(String[] args) {
	    Shape rectangle = new Shape("Rectangle","Red", 100);
 
	    Shape square = new Shape("Square","Blue", 200);
 
	    Shape circle = new Shape("Circle","Yellow", 200);
 
	    Shape circleRed = new Shape("Circle","Black", 300);
	    Shape circleBlue = new Shape("Circle","White", 400);
 
		Shape squareBlack = new Shape("Square","Black", 500);
	    Shape squareWhite = new Shape("Square","White", 600);

		rectangle.add(square);
		rectangle.add(circle);

	    circle.add(circleRed);
	    circle.add(circleBlue);
 
	    square.add(squareBlack);
	    square.add(squareWhite);

	    //打印所有图形
	    System.out.println(rectangle); 
	    for (Shape shape : rectangle.getRelations()) {
	        System.out.println(shape);
	    }        
	}
}
```

## 输出Belike
```text
Shape :[ Name : Rectangle, color : Red, size :100 ]
Shape :[ Name : Square, color : Blue, size :200 ]
Shape :[ Name : Square, color : Black, size :500 ]
Shape :[ Name : Square, color : White, size :600 ]
Shape :[ Name : Circle, color : Yellow, size :200 ]
Shape :[ Name : Circle, color : Black, size :300 ]
Shape :[ Name : Circle, color : White, size :400 ]
```