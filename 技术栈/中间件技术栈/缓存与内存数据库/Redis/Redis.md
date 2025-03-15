Redis 是一个开源的、高性能的 键值存储系统（Key-Value Store），通常被用作缓存、消息队列和实时数据处理等场景。它支持多种数据结构（如字符串、哈希、列表、集合等），并将数据存储在内存中以实现极快的读写速度。

---

# 核心特性

| 特性               | 说明                                                                 |
|--------------------|----------------------------------------------------------------------|
| 内存存储       | 数据存储在内存中，读写性能极高（每秒数十万次操作）                   |
| 持久化支持     | 支持 RDB（快照）和 AOF（追加日志）两种持久化方式                      |
| 多种数据结构   | 支持字符串、哈希、列表、集合、有序集合等                              |
| 高可用性       | 支持主从复制、哨兵模式和集群模式                                      |
| 发布/订阅      | 支持消息的发布与订阅，可用于消息队列                                  |
| 事务支持       | 支持简单的事务操作（MULTI/EXEC）                                      |
| Lua 脚本       | 支持通过 Lua 脚本执行复杂操作                                         |

---

# 常用数据结构及操作示例
以下是一些 Redis 常用数据结构的操作示例：
 1. 字符串（String）
```bash
# 设置键值对
SET name "Redis"

# 获取值
GET name  # 返回 "Redis"

# 自增操作
INCR counter  # 初始值为 0，返回 1
```

 1. 哈希（Hash）
```bash
# 设置哈希字段
HSET user:1 name "Alice" age 30

# 获取哈希字段
HGET user:1 name  # 返回 "Alice"

# 获取所有字段
HGETALL user:1  # 返回 name 和 age
```

 1. 列表（List）
```bash
# 从左侧插入元素
LPUSH mylist "item1"
LPUSH mylist "item2"

# 从右侧插入元素
RPUSH mylist "item3"

# 获取列表范围
LRANGE mylist 0 -1  # 返回 ["item2", "item1", "item3"]
```

 1. 集合（Set）
```bash
# 添加元素
SADD myset "item1"
SADD myset "item2"

# 获取所有元素
SMEMBERS myset  # 返回 ["item1", "item2"]

# 检查元素是否存在
SISMEMBER myset "item1"  # 返回 1（存在）
```

 1. 有序集合（Sorted Set）
```bash
# 添加带分数的元素
ZADD leaderboard 100 "Alice"
ZADD leaderboard 200 "Bob"

# 获取排名
ZRANGE leaderboard 0 -1 WITHSCORES  # 返回 ["Alice", "100", "Bob", "200"]
```

---

# Redis 使用场景

| 场景               | 说明                                                                 |
|--------------------|----------------------------------------------------------------------|
| 缓存           | 缓存热点数据，减轻数据库压力                                         |
| 会话存储       | 存储用户会话信息（如登录状态）                                       |
| 排行榜         | 使用有序集合实现实时排行榜                                           |
| 消息队列       | 使用列表或发布/订阅模式实现消息队列                                  |
| 分布式锁       | 使用 `SETNX` 实现分布式锁                                            |
| 实时数据分析   | 存储和处理实时数据（如点击量、在线用户数）                           |

---
# Redis 与 Spring Boot 整合示例
以下是一个简单的 Spring Boot 整合 Redis 的示例：

 1. 添加依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

 1. 配置 Redis 连接
```properties
# application.properties
spring.redis.host=localhost
spring.redis.port=6379
```

 1. 使用 RedisTemplate 操作 Redis
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

@Service
public class RedisService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 设置值
    public void setValue(String key, String value) {
        redisTemplate.opsForValue().set(key, value);
    }

    // 获取值
    public String getValue(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    // 删除键
    public void deleteKey(String key) {
        redisTemplate.delete(key);
    }
}
```

 1. 测试 Redis 操作
```java
@SpringBootApplication
public class RedisDemoApplication implements CommandLineRunner {

    @Autowired
    private RedisService redisService;

    public static void main(String[] args) {
        SpringApplication.run(RedisDemoApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        redisService.setValue("name", "Redis Demo");
        System.out.println("Value from Redis: " + redisService.getValue("name"));
        redisService.deleteKey("name");
    }
}
```

---

# Redis 生产环境建议
1. 持久化配置：根据需求选择 RDB 或 AOF，或两者结合
2. 高可用性：使用哨兵模式或集群模式避免单点故障
3. 性能优化：
   - 使用连接池（如 Lettuce 或 Jedis）
   - 避免大 Key 和热 Key
4. 安全性：
   - 设置密码（`requirepass`）
   - 限制访问 IP（`bind`）

---

# Redis 与其他缓存对比

| 对比项          | Redis                  | Memcached              |
|-----------------|------------------------|------------------------|
| 数据结构    | 支持多种数据结构       | 仅支持字符串           |
| 持久化      | 支持                   | 不支持                 |
| 性能        | 极高                   | 高                     |
| 功能丰富度  | 支持事务、Lua 脚本等   | 功能较为简单           |

---

Redis 是一个功能强大且灵活的工具，适用于需要高性能和低延迟的场景。通过合理使用 Redis，可以显著提升应用的性能和可扩展性。