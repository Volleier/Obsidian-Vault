#技术栈/优化/线程优化
使用 Netty 或 Spring WebFlux 实现异步非阻塞 I/O，提升并发处理能力
# Netty
Netty 是一个异步事件驱动的网络框架，适用于高性能、高并发的网络应用。
[Netty官网](https://netty.io/)

## 核心概念
- EventLoop: 负责处理 I/O 事件，每个 EventLoop 绑定一个线程，处理多个 Channel 的事件。
- Channel: 代表一个网络连接，支持异步操作。
- ChannelHandler: 处理 I/O 事件，如数据读取、写入等。
- ChannelPipeline: 包含多个 ChannelHandler，按顺序处理事件。

## 线程池优化
- EventLoopGroup: 管理多个 EventLoop，通常分为 bossGroup（接受连接）和 workerGroup（处理连接）。
- 自定义线程池: 对于耗时操作（如数据库访问），可以使用自定义线程池，避免阻塞 EventLoop。

## 示例代码
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();

try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .childHandler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) {
             ch.pipeline().addLast(new MyChannelHandler());
         }
     });

    ChannelFuture f = b.bind(8080).sync();
    f.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

# Spring WebFlux
Spring WebFlux 是 Spring 5 引入的响应式编程框架，支持异步非阻塞 I/O

## 核心概念
- eactive Streams: 提供异步流处理的标准接口（Publisher、Subscriber、Subscription、Processor）。
- Reactor: Spring WebFlux 的默认实现，提供 Mono（0-1 个元素）和 Flux（0-N 个元素）两种发布者。

## 线程池优化
- Schedulers: Reactor 提供多种调度器（如 `Schedulers.parallel()`、`Schedulers.elastic()`），用于控制异步操作的执行线程。
- 自定义线程池: 通过 `Schedulers.fromExecutorService()` 使用自定义线程池。

## 示例代码
```java
@RestController
public class MyController {

    @GetMapping("/flux")
    public Flux<String> getFlux() {
        return Flux.interval(Duration.ofMillis(100))
                   .map(i -> "Value: " + i)
                   .take(10);
    }

    @GetMapping("/mono")
    public Mono<String> getMono() {
        return Mono.just("Hello, WebFlux!");
    }
}
```

# NIO（Non-blocking I/O）