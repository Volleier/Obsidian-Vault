# Redis 与 Spring Boot 整合示例
以下是一个简单的 Spring Boot 整合 Redis 的示例：

1. 添加依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

2. 配置 Redis 连接
```properties
# application.properties
spring.redis.host=localhost
spring.redis.port=6379
```

 3. 使用 RedisTemplate 操作 Redis
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