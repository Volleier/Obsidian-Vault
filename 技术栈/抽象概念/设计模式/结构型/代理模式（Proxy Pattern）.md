在代理模式（Proxy Pattern）中，一个类代表另一个类的功能，这种类型的设计模式属于结构型模式。
代理模式通过引入一个代理对象来控制对原对象的访问。代理对象在客户端和目标对象之间充当中介，负责将客户端的请求转发给目标对象，同时可以在转发请求前后进行额外的处理。
在代理模式中，我们创建具有现有对象的对象，以便向外界提供功能接口。

# 介绍
## 概览
代理模式为其他对象提供一种代理以控制对这个对象的访问，代理模式解决的是在直接访问某些对象时可能遇到的问题，例如对象创建成本高、需要安全控制或远程访问等。当需要在访问一个对象时进行一些控制或额外处理时推荐代理模式。

## 实现方式
- 增加中间层：创建一个代理类，作为真实对象的中间层。
- 代理与真实对象组合：代理类持有真实对象的引用，并在访问时进行控制。

## 关键代码
- 代理类：实现与真实对象相同的接口，并添加额外的控制逻辑。
- 真实对象：实际执行任务的对象。

## 优缺点
- **职责分离**：代理模式将访问控制与业务逻辑分离。
- **扩展性**：可以灵活地添加额外的功能或控制。
- **智能化**：可以智能地处理访问请求，如延迟加载、缓存等。
- **性能开销**：增加了代理层可能会影响请求的处理速度。
- **实现复杂性**：某些类型的代理模式实现起来可能较为复杂。

## 注意事项
- 根据具体需求选择合适的代理类型，如远程代理、虚拟代理、保护代理等。
- 确保代理类与真实对象接口一致，以便客户端透明地使用代理。
- 与适配器模式的区别：适配器模式改变接口，而代理模式不改变接口。
- 与装饰器模式的区别：装饰器模式用于增强功能，代理模式用于控制访问。

## 结构
- 抽象主题（Subject）：定义了真实主题和代理主题的共同接口，这样在任何使用真实主题的地方都可以使用代理主题。
- 真实主题（Real Subject）：实现了抽象主题接口，是代理对象所代表的真实对象。客户端直接访问真实主题，但在某些情况下，可以通过代理主题来间接访问。
- 代理（Proxy）：实现了抽象主题接口，并持有对真实主题的引用。代理主题通常在真实主题的基础上提供一些额外的功能，例如延迟加载、权限控制、日志记录等。
- 客户端（Client）：使用抽象主题接口来操作真实主题或代理主题，不需要知道具体是哪一个实现类。

# 实现
创建一个 `Shape` 接口和实现了 `Shape` 接口的实体类。`ProxyShape` 是一个代理类，减少 `RealShape` 对象加载的内存占用。
`ProxyPatternDemo` 类使用 `ProxyShape` 来获取要加载的 `Shape` 对象，并按照需求进行显示。

## 创建一个接口
Shape.java
```java
public interface Shape {
   void show();
}
```

## 创建实现接口的实体类
RealShape.java
```java
public class RealShape implements Shape {
 
   private String fileName;
 
   public RealShape(String fileName){
      this.fileName = fileName;
      loadFromDisk(fileName);
   }
 
   @Override
   public void show() {
      System.out.println("Showing " + fileName);
   }
 
   private void loadFromDisk(String fileName){
      System.out.println("Loading " + fileName);
   }
}
```
ProxyShape.java
```java
public class ProxyShape implements Image{
 
   private ReaShape ReaShape;
   private String fileName;
 
   public ProxyShape(String fileName){
      this.fileName = fileName;
   }
 
   @Override
   public void show() {
      if(reaShape == null){
         reaShape = new ReaShape(fileName);
      }
      reaShape.show();
   }
}
```

## 当被请求时，使用 ProxyShape 来获取 RealShape 类的对象
ProxyPatternDemo.java
```java
public class ProxyPatternDemo {
   
   public static void main(String[] args) {
      Shape shape = new ProxyShape("Rectangle.shape");
 
      // 图像将从磁盘加载
      image.show(); 
      System.out.println("");
      // 图像不需要从磁盘加载
      image.show();  
   }
}
```

## 输出Belike
```text
Loading Rectangle.shape
Showing Rectangle.shape

Showing Rectangle.shape
```