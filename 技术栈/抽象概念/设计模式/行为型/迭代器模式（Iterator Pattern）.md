迭代器模式（Iterator Pattern）是 Java 和 .Net 编程环境中非常常用的设计模式。
迭代器模式提供一种方法顺序访问一个聚合对象中的各个元素，而又不暴露其内部的表示。

# 介绍
## 概览
迭代器模式允许顺序访问一个聚合对象中的元素，同时不暴露对象的内部表示。主要解决提供一种统一的方法来遍历不同的聚合对象。当需要遍历一个聚合对象，而又不希望暴露其内部结构时使用。

## 实现方式
- 定义迭代器接口：包含`hasNext()`和`next()`等方法，用于遍历元素。
- 创建具体迭代器：实现迭代器接口，定义如何遍历特定的聚合对象。
- 聚合类：定义一个接口用于返回一个迭代器对象。

## 关键代码
- 迭代器接口：规定了遍历元素的方法。
- 具体迭代器：实现了迭代器接口，包含遍历逻辑。

## 优缺点
- 支持多种遍历方式：不同的迭代器可以定义不同的遍历方式。
- 简化聚合类：聚合类不需要关心遍历逻辑。
- 多遍历支持：可以同时对同一个聚合对象进行多次遍历。
- 扩展性：增加新的聚合类和迭代器类都很方便，无需修改现有代码。
- 系统复杂性：每增加一个聚合类，就需要增加一个对应的迭代器类，增加了类的数量。

## 注意事项
- 当需要访问聚合对象内容而不暴露其内部表示时，使用迭代器模式。
- 当需要为聚合对象提供多种遍历方式时，考虑使用迭代器模式。
- 迭代器模式通过分离集合对象的遍历行为，使得外部代码可以透明地访问集合内部数据，同时不暴露集合的内部结构。

## 结构
- 迭代器接口（Iterator）：定义了访问和遍历聚合对象中各个元素的方法，通常包括获取下一个元素、判断是否还有元素、获取当前位置等方法。
- 具体迭代器（Concrete Iterator）：实现了迭代器接口，负责对聚合对象进行遍历和访问，同时记录遍历的当前位置。
- 聚合对象接口（Aggregate）：定义了创建迭代器对象的接口，通常包括一个工厂方法用于创建迭代器对象。
- 具体聚合对象（Concrete Aggregate）：实现了聚合对象接口，负责创建具体的迭代器对象，并提供需要遍历的数据。

# 实现
创建一个叙述导航方法的 `Iterator` 接口和一个返回迭代器的 `Container` 接口。实现了 `Container` 接口的实体类将负责实现 `Iterator` 接口。
`IteratorPatternDemo`，我们的演示类使用实体类 `NamesRepository` 来打印 `NamesRepository` 中存储为集合的 `Names`。
![[迭代器模式-1.png]]

## 创建接口
Iterator.java
```java
public interface Iterator {
   public boolean hasNext();
   public Object next();
}
```
Container.java
```java
public interface Container {
   public Iterator getIterator();
}
```

## 创建实现了 Container 接口的实体类。
该类有实现了 Iterator 接口的内部类 NameIterator
NameRepository.java
```java
public class NameRepository implements Container {
   public String[] names = {"Rectangle" , "Circle" ,"Square"};
 
   @Override
   public Iterator getIterator() {
      return new NameIterator();
   }
 
   private class NameIterator implements Iterator {
 
      int index;
 
      @Override
      public boolean hasNext() {
         if(index < names.length){
            return true;
         }
         return false;
      }
 
      @Override
      public Object next() {
         if(this.hasNext()){
            return names[index++];
         }
         return null;
      }     
   }
}
```

## 使用 NameRepository 来获取迭代器，并打印名字
IteratorPatternDemo.java
```java
public class IteratorPatternDemo {
   
   public static void main(String[] args) {
      NameRepository namesRepository = new NameRepository();
 
      for(Iterator iter = namesRepository.getIterator(); iter.hasNext();){
         String name = (String)iter.next();
         System.out.println("Name: " + name);
      }  
   }
}
```

## 输出Belike
```text
Name: Rectangle
Name: Circle
Name: Square
```