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
创建一个 _Person_ 对象、_Criteria_ 接口和实现了该接口的实体类，来过滤 _Person_ 对象的列表。_CriteriaPatternDemo_ 类使用 _Criteria_ 对象，基于各种标准和它们的结合来过滤 _Person_ 对象的列表。

## 创建一个类，在该类上应用标准。
Person.java

public class Person { private String name; private String gender; private String maritalStatus; public Person(String name,String gender,String maritalStatus){ this.name = name; this.gender = gender; this.maritalStatus = maritalStatus; } public String getName() { return name; } public String getGender() { return gender; } public String getMaritalStatus() { return maritalStatus; } }

## 为标准（Criteria）创建一个接口。

Criteria.java

import java.util.List; public interface Criteria { public List<Person> meetCriteria(List<Person> persons); }

## 创建实现了 _Criteria_ 接口的实体类。

## CriteriaMale.java

import java.util.ArrayList; import java.util.List; public class CriteriaMale implements Criteria { @Override public List<Person> meetCriteria(List<Person> persons) { List<Person> malePersons = new ArrayList<Person>(); for (Person person : persons) { if(person.getGender().equalsIgnoreCase("MALE")){ malePersons.add(person); } } return malePersons; } }

## CriteriaFemale.java

import java.util.ArrayList; import java.util.List; public class CriteriaFemale implements Criteria { @Override public List<Person> meetCriteria(List<Person> persons) { List<Person> femalePersons = new ArrayList<Person>(); for (Person person : persons) { if(person.getGender().equalsIgnoreCase("FEMALE")){ femalePersons.add(person); } } return femalePersons; } }

## CriteriaSingle.java

import java.util.ArrayList; import java.util.List; public class CriteriaSingle implements Criteria { @Override public List<Person> meetCriteria(List<Person> persons) { List<Person> singlePersons = new ArrayList<Person>(); for (Person person : persons) { if(person.getMaritalStatus().equalsIgnoreCase("SINGLE")){ singlePersons.add(person); } } return singlePersons; } }

## AndCriteria.java

import java.util.List; public class AndCriteria implements Criteria { private Criteria criteria; private Criteria otherCriteria; public AndCriteria(Criteria criteria, Criteria otherCriteria) { this.criteria = criteria; this.otherCriteria = otherCriteria; } @Override public List<Person> meetCriteria(List<Person> persons) { List<Person> firstCriteriaPersons = criteria.meetCriteria(persons); return otherCriteria.meetCriteria(firstCriteriaPersons); } }

## OrCriteria.java

import java.util.List; public class OrCriteria implements Criteria { private Criteria criteria; private Criteria otherCriteria; public OrCriteria(Criteria criteria, Criteria otherCriteria) { this.criteria = criteria; this.otherCriteria = otherCriteria; } @Override public List<Person> meetCriteria(List<Person> persons) { List<Person> firstCriteriaItems = criteria.meetCriteria(persons); List<Person> otherCriteriaItems = otherCriteria.meetCriteria(persons); for (Person person : otherCriteriaItems) { if(!firstCriteriaItems.contains(person)){ firstCriteriaItems.add(person); } } return firstCriteriaItems; } }

### 步骤4

使用不同的标准（Criteria）和它们的结合来过滤 _Person_ 对象的列表。

## CriteriaPatternDemo.java

import java.util.ArrayList; import java.util.List; public class CriteriaPatternDemo { public static void main(String[] args) { List<Person> persons = new ArrayList<Person>(); persons.add(new Person("Robert","Male", "Single")); persons.add(new Person("John","Male", "Married")); persons.add(new Person("Laura","Female", "Married")); persons.add(new Person("Diana","Female", "Single")); persons.add(new Person("Mike","Male", "Single")); persons.add(new Person("Bobby","Male", "Single")); Criteria male = new CriteriaMale(); Criteria female = new CriteriaFemale(); Criteria single = new CriteriaSingle(); Criteria singleMale = new AndCriteria(single, male); Criteria singleOrFemale = new OrCriteria(single, female); System.out.println("Males: "); printPersons(male.meetCriteria(persons)); System.out.println("\nFemales: "); printPersons(female.meetCriteria(persons)); System.out.println("\nSingle Males: "); printPersons(singleMale.meetCriteria(persons)); System.out.println("\nSingle Or Females: "); printPersons(singleOrFemale.meetCriteria(persons)); } public static void printPersons(List<Person> persons){ for (Person person : persons) { System.out.println("Person : [ Name : " + person.getName() +", Gender : " + person.getGender() +", Marital Status : " + person.getMaritalStatus() +" ]"); } } }