单例模式及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。
单例模式是一种创建型设计模式，它确保一个类只有一个实例，并提供了一个全局访问点来访问该实例。
注意：
- 单例类只能有一个实例。
- 单例类必须自己创建自己的唯一实例。
- 单例类必须给所有其他对象提供这一实例。

# 介绍
## 概览
单例模式确保一个类只有一个实例，并提供一个全局访问点来访问该实例。单例模式可以解决频繁创建和销毁全局使用的类实例的问题。用于当需要控制实例数目，节省系统资源时。
单例模式需要检查检查系统是否已经存在该单例，如果存在则返回该实例；如果不存在则创建一个新实例。
## 优缺点
- 内存中只有一个实例，减少内存开销，尤其是频繁创建和销毁实例时（如管理学院首页页面缓存）。
- 避免资源的多重占用（如写文件操作）。
- 没有接口，不能继承。
- 与单一职责原则冲突，一个类应该只关心内部逻辑，而不关心实例化方式。

## 注意事项
- 线程安全：`getInstance()` 方法中需要使用同步锁 `synchronized (Singleton.class)` 防止多线程同时进入造成实例被多次创建。
- 延迟初始化：实例在第一次调用 `getInstance()` 方法时创建。
- 序列化和反序列化：重写 `readResolve` 方法以确保反序列化时不会创建新的实例。
- 反射攻击：在构造函数中添加防护代码，防止通过反射创建新实例。
- 类加载器问题：注意复杂类加载环境可能导致的多个实例问题。

## 结构
单例模式包含以下几个主要角色：
- 单例类：包含单例实例的类，通常将构造函数声明为私有。
- 静态成员变量：用于存储单例实例的静态成员变量。
- 获取实例方法：静态方法，用于获取单例实例。
- 私有构造函数：防止外部直接实例化单例类。
- 线程安全处理：确保在多线程环境下单例实例的创建是安全的。

# 实现
_SingleObject_ 类提供了一个静态方法，供外界获取它的静态实例。_SingletonPatternDemo_ 类使用 _SingleObject_ 类来获取 _SingleObject_ 对象。


创建一个 Singleton 类。
```java
public class SingleObject {
 
   //创建 SingleObject 的一个对象
   private static SingleObject instance = new SingleObject();
 
   //让构造函数为 private，这样该类就不会被实例化
   private SingleObject(){}
 
   //获取唯一可用的对象
   public static SingleObject getInstance(){
      return instance;
   }
 
   public void showMessage(){
      System.out.println("Hello World!");
   }
}
```
从 singleton 类获取唯一的对象。
```java
public class SingletonPatternDemo {
   public static void main(String[] args) {
 
      //不合法的构造函数
      //编译时错误：构造函数 SingleObject() 是不可见的
      //SingleObject object = new SingleObject();
 
      //获取唯一可用的对象
      SingleObject object = SingleObject.getInstance();
 
      //显示消息
      object.showMessage();
   }
}
```