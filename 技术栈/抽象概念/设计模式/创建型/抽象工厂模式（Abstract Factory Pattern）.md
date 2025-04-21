抽象工厂模式（Abstract Factory Pattern）是围绕一个超级工厂创建其他工厂。该超级工厂又称为其他工厂的工厂。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
在抽象工厂模式中，接口是负责创建一个相关对象的工厂，不需要显式指定它们的类。每个生成的工厂都能按照工厂模式提供对象。
抽象工厂模式提供了一种创建一系列相关或相互依赖对象的接口，而无需指定具体实现类。通过使用抽象工厂模式，可以将客户端与具体产品的创建过程解耦，使得客户端可以通过工厂接口来创建一族产品。
# 介绍
## 概览
抽象工厂提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们的具体类。解决接口选择的问题。适用于当系统需要创建多个相关或依赖的对象，而不需要指定具体类时。
在一个产品族中定义多个产品，由具体工厂实现创建这些产品的方法。在一个工厂中聚合多个同类产品的创建方法。

## 优缺点
1. 确保同一产品族的对象一起工作。
2. 客户端不需要知道每个对象的具体类，简化了代码。
3. 扩展产品族非常困难。增加一个新的产品族需要修改抽象工厂和所有具体工厂的代码。
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
创建两个接口：
```java
public interface Interface1 {
   void function1();
}
```
```java
public interface Interface2 {
   void function2();
}
```

创建实现接口的实体类：
```java
public class Interface1_Impl1 implements Interface1 {
 
   @Override
   public void function1() {
      System.out.println("Inside Interface1_Impl1::function2 method.");
   }
}
```
```java
public class Interface1_Impl2 implements Interface1 {
 
   @Override
   public void function2() {
      System.out.println("Inside Interface1_Impl2::function2 method.");
   }
}
```
```java
public class Interface1_Impl3 implements Interface1 {
 
   @Override
   public void function3() {
      System.out.println("Inside Interface1_Impl3::function2 method.");
   }
}
```
以及：
```java
public class Interface2_Impl1 implements Interface2 {
 
   @Override
   public void function2() {
      System.out.println("Inside Interface2_Impl1::function2 method.");
   }
}
```
```java
public class Interface2_Impl2 implements Interface2 {
 
   @Override
   public void function2() {
      System.out.println("Inside Interface2_Impl2::function2 method.");
   }
}
```
```java
public class Interface2_Impl3 implements Interface2 {
 
   @Override
   public void function2() {
      System.out.println("Inside Interface2_Impl3::function2 method.");
   }
}
```

为接口对象创建抽象类来获取工厂：
```java
public abstract class AbstractFactory {
   public abstract Interface1 getFunction(String string);
   public abstract Interface2 getFunction(String string);
}
```

创建扩展了 AbstractFactory 的工厂类，基于给定的信息生成实体类的对象：
```java
public class Interface1_Factory extends AbstractFactory {
    
   @Override
   public Interface1 getInterface1(String string){
      if(string == null){
         return null;
      }        
      if(string.equalsIgnoreCase("INTERFACE1 FUNCTION1")){
         return new Interface1_Impl1();
      } else if(string.equalsIgnoreCase("INTERFACE1 FUNCTION2")){
         return new Interface1_Impl2();
      } else if(string.equalsIgnoreCase("INTERFACE1 FUNCTION3")){
         return new Interface1_Impl3();
      }
      return null;
   }
   
   @Override
   public Interface1 getFunction(String string) {
      return null;
   }
}
```
```java
public class Interface2_Factory extends AbstractFactory {
    
   @Override
   public Interface1 getInterface2(String string){
      if(string == null){
         return null;
      }        
      if(string.equalsIgnoreCase("INTERFACE2 FUNCTION1")){
         return new Interface1_Impl1();
      } else if(string.equalsIgnoreCase("INTERFACE2 FUNCTION2")){
         return new Interface1_Impl2();
      } else if(string.equalsIgnoreCase("INTERFACE2 FUNCTION3")){
         return new Interface1_Impl3();
      }
      return null;
   }
   
   @Override
   public Interface2 getFunction(String string) {
      return null;
   }
}
```

创建一个工厂创造器/生成器类，通过传递信息来获取工厂：
```java
public class FactoryProducer {
   public static AbstractFactory getFactory(String choice){
      if(choice.equalsIgnoreCase("INTERFACE1")){
         return new Interface1_Factory();
      } else if(choice.equalsIgnoreCase("INTERFACE2")){
         return new Interface2_Factory();
      }
      return null;
   }
}
```

使用 FactoryProducer 来获取 AbstractFactory，通过传递类型信息来获取实体类的对象：
```java
public class AbstractFactoryPatternDemo {
    public static void main(String[] args) {
        // 获取工厂1
        AbstractFactory interface1Factory = FactoryProducer.getFactory("INTERFACE1");

        // 获取对象
        Interface1 interface1Impl1 = interface1Factory.getInterface1("FUNCTION1");
        Interface1 interface1Impl2 = interface1Factory.getInterface1("FUNCTION2");
        Interface1 interface1Impl3 = interface1Factory.getInterface1("FUNCTION3");

        // 调用方法
        interface1Impl1.function1();
        interface1Impl2.function1();
        interface1Impl3.function1();

        // 获取工厂2
        AbstractFactory interface2Factory = FactoryProducer.getFactory("INTERFACE2");

        // 获取对象
        Interface2 interface2Impl1 = interface2Factory.getInterface2("FUNCTION1");
        Interface2 interface2Impl2 = interface2Factory.getInterface2("FUNCTION2");
        Interface2 interface2Impl3 = interface2Factory.getInterface2("FUNCTION3");

        // 调用方法
        interface2Impl1.function2();
        interface2Impl2.function2();
        interface2Impl3.function2();
    }
}
```
输出：
```TEXT
Inside Interface1_Impl1::function1 method.
Inside Interface1_Impl2::function2 method.
Inside Interface1_Impl3::function3 method.
Inside Interface2_Impl1::function1 method.
Inside Interface2_Impl2::function2 method.
Inside Interface2_Impl3::function3 method.
```