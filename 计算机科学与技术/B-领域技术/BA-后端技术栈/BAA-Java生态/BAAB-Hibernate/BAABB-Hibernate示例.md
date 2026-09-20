# 示例代码（与 Spring Boot 整合）

 1. 实体类映射（Entity）
```java
@Entity
@Table(name = "users")  // 映射到数据库中的 users 表
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_name", nullable = false, length = 50)
    private String username;
    
    @Column(unique = true)
    private String email;
    
    // 关联关系示例：一对多
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
    
    // Getter 和 Setter（省略）
}
```

 1. 数据访问层（Repository）
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // 自动生成查询方法（无需实现）
    User findByEmail(String email);
    
    // 自定义 HQL 查询
    @Query("SELECT u FROM User u WHERE u.username LIKE %:keyword%")
    List<User> searchByUsername(@Param("keyword") String keyword);
}
```

 1. 服务层调用
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(User user) {
        return userRepository.save(user);  // 自动生成 INSERT 语句
    }
    
    public List<User> findUsersByKeyword(String keyword) {
        return userRepository.searchByUsername(keyword);  // 执行 HQL 查询
    }
}
```

# 生产环境最佳实践
1. 配置连接池：使用 HikariCP 或 C3P0 提升数据库连接性能
   ```properties
   spring.datasource.hikari.maximum-pool-size=10
   ```
2. 启用二级缓存：整合 Ehcache 或 Redis
   ```java
   @Entity
   @Cacheable
   @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
   public class User { ... }
   ```
3. 批量操作优化：避免 N+1 查询问题，使用 `@BatchSize` 和 `JOIN FETCH`
   ```java
   @Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
   User getUserWithOrders(@Param("id") Long id);
   ```
4. 日志调试：在开发阶段开启 Hibernate SQL 日志
   ```properties
   spring.jpa.show-sql=true
   spring.jpa.properties.hibernate.format_sql=true
   ```

---

# 适用场景
- 需要快速开发数据库驱动的应用
- 减少 SQL 维护成本
- 跨数据库兼容性要求高
- 复杂对象关系（如一对多、多对多）

---

通过 Hibernate，开发者可以更专注于业务逻辑而非底层数据库操作。在 Spring Boot 中，通常通过 Spring Data JPA（基于 Hibernate 实现）进一步简化 ORM 操作。