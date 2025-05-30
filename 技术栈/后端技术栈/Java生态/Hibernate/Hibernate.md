Hibernate将 Java 对象（POJO）与数据库表自动映射，开发者无需直接编写 SQL 即可实现数据库的增删改查（CRUD）操作。

# 核心特性
| 特性                            | 说明                                                |
| ----------------------------- | ------------------------------------------------- |
| 对象关系映射（ORM）                   | 通过注解或 XML 定义 Java 类与数据库表的映射关系                     |
| HQL（Hibernate Query Language） | 面向对象的查询语言，替代原生 SQL                                |
| 缓存机制                          | 一级缓存（Session 级）和二级缓存（应用级），提升性能                    |
| 事务管理                          | 支持声明式事务和编程式事务                                     |
| 懒加载（Lazy Loading）             | 按需加载关联数据，避免不必要查询                                  |
| 跨数据库支持                        | 通过配置方言（Dialect）适配不同数据库（MySQL、PostgreSQL、Oracle 等） |

# Hibernate vs JDBC

| 对比项          | JDBC                                  | Hibernate                          |
|-----------------|---------------------------------------|------------------------------------|
| 代码量      | 需要大量样板代码（Connection/ResultSet） | 通过 API 简化操作，减少代码量       |
| SQL 依赖    | 必须手动编写 SQL                      | 可自动生成 SQL，支持 HQL           |
| 对象映射    | 手动处理 ResultSet 到对象的转换        | 自动映射对象与数据库表             |
| 事务管理    | 手动控制                              | 声明式事务管理（通过注解）         |
| 性能优化    | 需手动处理缓存                        | 内置缓存机制                       |
