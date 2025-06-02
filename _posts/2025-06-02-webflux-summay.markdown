---
title: "Spring Cloud Gateway : WebFlux "
date: 2025-06-02 01:06:21
tags: microservice
---

这篇笔记，继续沿着 Spring Cloud Gateway，来聊一聊 WebFlux.

# 1. 响应式编程

优秀的系统实现，特点之一就是对于线程的控制，能够让线程专注在一件事情上，降低系统线程间切换的开销。

比如当我们发起一次耗时的 IO 操作：

```java
String foo() {
  String s = fetch_from_remote_long_long_time();

  return s;
}
```

当前线程会阻塞，直到 IO 返回。当调用变多，就会有大量线程阻塞在这个位置，由于线程本身需要栈空间、上下文切换等操作，当面临高并发时，就会出现性能问题。

响应式编程会优化为非阻塞的过程，当前线程在请求完成后能够立刻“释放”，不会阻塞在当前位置。

举个[访问 redis 的例子](https://github.com/izualzhy/Microservice-Systems/blob/main/webflux/src/main/java/cn/izualzhy/webflux/controller/RedisTestController.java)：

```java
    @GetMapping(value = "/test/redis")
    public void testRedis() {
        System.out.println("current thread: " + Thread.currentThread());

        try (Jedis jedis = new Jedis("localhost", 6379)) {
            jedis.auth("foobared");  // 认证密码
            String value = jedis.get("hello");  // 阻塞调用
            System.out.println("Got from Jedis: " + value +  " in thread: " + Thread.currentThread());
        }

        RedisClient client = RedisClient.create("redis://:foobared@localhost:6379");
        RedisReactiveCommands<String, String> reactiveCommands = client.connect().reactive();

        Mono<String> valueMono = reactiveCommands.get("hello");

        valueMono.subscribe(value -> System.out.println("Got from Reactive: " + value + " in thread: " + Thread.currentThread()));
    }
```

输出：
```
current thread: Thread[reactor-http-nio-3,5,main]
Got from Jedis: world in thread: Thread[reactor-http-nio-3,5,main]
Got from Reactive: world in thread: Thread[lettuce-nioEventLoop-8-1,5,main]
```

可以看到最大的区别，就是返回结果执行线程不同。原来的线程，在发起`subscribe`后就“释放”了。

我在看 SCG 的代码，以及使用 Reactive 时，最直观的感受就是可以更优雅地嵌入回调了，比如代码里的`System.out.println`。总结 Reactive 性能之所以强大，是在于线程能够专注，减少了频繁的上下文切换。
{:.success}

*注：实际上我在实际场景里，也很少会需要用到 lettuce-core 访问 redis，这里只用来方便对比说明。*

# 2. WebFlux、Project Reactor、ReactiveStream

ReactiveStream 是一套规范<sup>1</sup>，定义了像`Publisher Subscriber Subscription Processor`的接口及行为方式。

Project Reactor 是 ReactiveStream 的其中一种实现，提供了`Flux Mono`等定义。

WebFlux 则是基于 Project Reactor 实现了响应式 Web 框架。

![Spring MVC and WebFlux](https://docs.spring.io/spring-framework/reference/_images/spring-mvc-and-webflux-venn.png)


SCG 里我们主要关注后两者即可。由于 WebFlux 里本身就有`RouteFunction`的定义，同时加上`Mono Flux`、`subscribeOn publishOn`等语法，写法跟 MVC 差别很大。对于初学者还是比较容易弄混的。接下来的章节简单梳理一下。

# 3. Flux Mono

```java
package reactor.core.publisher;
public abstract class Flux<T> implements CorePublisher<T> {

package reactor.core.publisher;
public abstract class Mono<T> implements CorePublisher<T> {

package reactor.core;
  public interface CorePublisher<T> extends Publisher<T> {

package org.reactivestreams;
public interface Publisher<T> {
```

可以看到两者都是`org.reactivestreams.Publisher`的一种实现，之所以需要有两种，主要是：
1. `Flux`: 流式处理，数据源产出的数据有 0~N 个  
2. `Mono`: 常用于返回值，比如响应式框架里`@RestController`注解的方法，就可以返回`Mono<String>`，因此用于表示 0~1 个的数据

以`Flux`为例，可以从已有数据创建；也可以通过程序构建，用于创建无限流。既然是流，也就有类似 java 里 Stream 的操作，比如`map`

```java
        Flux<String> flux1 = Flux.just("foo", "bar", "foobar");
        flux1.map(s -> s.toUpperCase())
                .subscribe(System.out::println);

        Flux<String> flux2 = Flux.generate(
                () -> 0,
                (state, sink) -> {
                    sink.next("3 x " + state + " = " + 3*state);
                    if (state == 100000) sink.complete();
                    return state + 1;
                });
        flux2.subscribe(System.out::println);
```

比如 WebFlux 里这段`DispatcherHandler`的源码：

```java
public class DispatcherHandler implements WebHandler, PreFlightRequestHandler, ApplicationContextAware {

  @Override
  public Mono<Void> handle(ServerWebExchange exchange) {
    if (this.handlerMappings == null) {
      return createNotFoundError();
    }
    if (CorsUtils.isPreFlightRequest(exchange.getRequest())) {
      return handlePreFlight(exchange);
    }
    return Flux.fromIterable(this.handlerMappings)
        .concatMap(mapping -> mapping.getHandler(exchange))
        .next()
        .switchIfEmpty(createNotFoundError())
        .onErrorResume(ex -> handleResultMono(exchange, Mono.error(ex)))
        .flatMap(handler -> handleRequestWith(exchange, handler));
  }
```

`concatMap`的作用是将 Flux 转为 Mono，说白话就是将多个数据，转为单条数据，对应到业务效果上，就是筛选出`mapping.getHandler(exchange)`返回`handler`而不是`Mono.empty()`的那条。

举个例子：

```java
        Flux<Mono<String>> handlerMonos = Flux.just(
                Mono.empty(),
                Mono.empty(),
                Mono.just("matchHandler-1"),
                Mono.just("matchHandler-2")
        );

        Flux<String> handlerFlux = handlerMonos.concatMap(m -> m);
        // concatMap: matchHandler-1
        // concatMap: matchHandler-2
        handlerFlux.subscribe(result -> System.out.println("concatMap: " + result));

        handlerFlux.next()
                .defaultIfEmpty("Error")
                // next: matchHandler-1
                .subscribe(result -> System.out.println("next: " + result));
```

# 4. 线程

既然非阻塞解决了当前线程阻塞的问题，那么首要的疑问，就是回调是在哪里执行的？
{:.warning}

看个例子：

```java
    @GetMapping(value = "/test/thread")
    public String testThread() {
        Flux.just(1)
                // .publishOn(Schedulers.parallel()) //指定在parallel线程池中执行
                .map(i -> {
                    System.out.println("map1: " + Thread.currentThread().getName());
                    return i;
                })
                // .publishOn(Schedulers.boundedElastic()) // 指定下游的执行线程
                .map(i -> {
                    System.out.println("map2: " + Thread.currentThread().getName());
                    return i;
                })
                // .subscribeOn(Schedulers.single())
                .subscribe(i -> System.out.println("subscribe: " + Thread.currentThread().getName()));

        return "success";
    }
```

当前都是在一个线程(reactor-http-nio-x)里执行的。

如果去掉注释：

```java
map1: custom-thread-1
map2: parallel-1
map3: boundedElastic-2
map4: boundedElastic-2
subscribe: boundedElastic-2
```

1. `publishOn`：对之后的算子生效，直到遇到新的`publishOn subscribeOn`  
2. `subscribeOn`: 从流开始生效，直到遇到新的`publishOn subscribeOn`

就能够理解上述的输出了。

WebFlux 内置的几种线程池：

| 方法  | 用途 | 特点 |
| -- | -- | -- |
| `Schedulers.parallel()`                                                       | CPU 密集型任务  | 固定线程池，线程数 = CPU 核数，避免上下文切换 |
| `Schedulers.immediate()`                                                      | 当前线程执行     | 不切换线程，直接在调用线程上执行           |
| `Schedulers.newBoundedElastic(int threadCap, int queuedTaskCap, String name)` | 自定义弹性线程池   | 指定最大线程数、队列任务数和线程名称前缀       |
| `Schedulers.single()`                                                         | 所有任务共用一个线程 | 顺序执行任务，适合串行操作              |

从这里就可以看出在 reactive 对于线程的切换极其方便，同时对语义没有影响，结果保持一致。

# 5. ThreadLocal

讲了线程，就离不开`ThreadLocal`，在[DolphinScheduler-2: 日志](https://izualzhy.cn/ds-logger)里介绍过 MDC。使用 MDC 可以更方便的追踪一个任务不同阶段的执行记录。

MDC 的实现就依赖于`ThreadLocal`.

对于上面的代码，如果在`map`方法里打印日志，肯定就会出错了。举个更简化的例子看看：
```java
    @GetMapping(value = "/test/MDC")
    public String testMDCMultiThread() throws InterruptedException {
        MDC.put("trace_id", "multiThread");

        Flux.range(1, 3)
                .map(i -> {
                    System.out.println("[before publishOn] " + i + " on thread: " + Thread.currentThread().getName() + ", trace_id=" + MDC.get("trace_id"));
                    return i;
                })
                .publishOn(Schedulers.boundedElastic()) // 切换线程池
                .map(i -> {
                    System.out.println("[after publishOn] " + i + " on thread: " + Thread.currentThread().getName() + ", trace_id=" + MDC.get("trace_id"));
                    return i;
                })
                .subscribe();

        System.out.println("testMDC in Thread: " + Thread.currentThread().getName());

        return "testMDC";
    }
```

通过输出可以看到在第二个`doOnNext`方法里，线程和`trace_id`都变了：

```
[before publishOn] 1 on thread: reactor-http-nio-3, trace_id=multiThread
[before publishOn] 2 on thread: reactor-http-nio-3, trace_id=multiThread
[before publishOn] 3 on thread: reactor-http-nio-3, trace_id=multiThread
testMDC in Thread: reactor-http-nio-3
[after publishOn] 1 on thread: boundedElastic-1, trace_id=null
[after publishOn] 2 on thread: boundedElastic-1, trace_id=null
[after publishOn] 3 on thread: boundedElastic-1, trace_id=null
```

**怎么解决呢？**

我想到的一个解法，是使用 Project Reactor 里的 context<sup>2</sup>:

```java
        Flux.range(1, 3)
                .map(i -> {
                    System.out.println("[before publishOn] " + i + " on thread: " +
                            Thread.currentThread().getName() + ", trace_id=" + MDC.get("trace_id"));
                    return i;
                })
                .publishOn(Schedulers.boundedElastic())
                .flatMap(i ->
                        Mono.deferContextual(ctx -> {
                            String traceId = ctx.getOrDefault("trace_id", "N/A");
                            MDC.put("trace_id", traceId);
                            System.out.println("[after publishOn] " + i + " on thread: " +
                                    Thread.currentThread().getName() + ", trace_id=" + MDC.get("trace_id"));
                            MDC.clear(); // 避免污染线程池
                            return Mono.just(i);
                        })
                )
                .contextWrite(Context.of("trace_id", "multiThread"))
                .subscribe();
```

代码丑陋了很多，“What Is a Good Pattern for Contextual Logging?(MDC)”<sup>3</sup>这里也提到了`Context`，感觉是一回事。

实际上在之前的笔记里，我发现 DolphinScheduler 手动维护`MDC.put clear`时，由于模块设计了几组线程池和回调，也总会有出错的地方。我觉得最好的方式，是在模块里自定义`Context`传递，这样最为稳妥，比如对于 Spring Cloud Gateway，可以将这类数据都写入到`exchange attribute`里。

进一步的，可以考虑自定义`MDCAdapter`的实现。

# 6. WebFlux 的编程模型

SCG 里用的不多，不过编程模型是 谈 WebFlux 时绕不开的，主要有两种：
1. 注解式：`@RestController`，这个跟 Spring MVC 很像，返回诸如`Mono<String> Flux<String>`的结构。
2. 函数式：通过`route`指定 url 和处理方法，`HandlerFunction`则用来接收`ServerRequest`，返回`ServerResponse`

这块实现起来偏 CRUD，主要熟悉用法即可，重点关注避免耗时操作打满 controller 线程。我把一些 demo 上传到了[Microservice-Systems/webflux/](https://github.com/izualzhy/Microservice-Systems/tree/main/webflux).

# 7. 总结

目前为止，我大概接触了 3 种流模式的数据处理。

第一类最为简单，是编程语言支持的，比如 java 里的 stream，python 里的 list 等，顺序处理，重点关注其不同方法的效果就行。

第二类则是 ReactiveStream ，除了函数的效果，有了更多关注点：比如`flatMap`可能会并发执行，而`concatMap`固定单个线程执行；比如`Mono.defer`能够起到 lazy 的效果。

同时这里已经有点声明式语法那味了，只有`subscribe`之后，定义的各个方法才会真正执行。到了第三类，即大数据里的流式处理，典型的如 flink spark。类似于`subscribe`，只有调用`execute`才会生成算子拓扑图，分发任务启动，不只是线程不同，算子已经是通过 YARN K8S 部署在 Container 里了。比如在[之前的笔记](https://izualzhy.cn/bigdata-and-backend#2-%E5%A4%A7%E6%95%B0%E6%8D%AE%E7%9A%84%E6%8A%80%E6%9C%AF%E4%BD%BF%E7%94%A8%E4%B8%8A%E6%9B%B4%E5%8A%A0%E7%AE%80%E5%8D%95)里，刚接触大数据时，还是非常诧异的。  

回到 WebFlux，之所以有这篇笔记，也是缘于 Spring Cloud Gateway。响应式编程，相比原来性能提升有多少，我没有太多实际项目经验。不过从网关的角度，每个连接一个线程的做法，毫无疑问是不合理的。

# 8. 参考

1. [reactive-streams-jvm/README.md](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.4/README.md)
2. [Adding a Context to a Reactive Sequence](https://projectreactor.io/docs/core/release/reference/advancedFeatures/context.html)
3. [What Is a Good Pattern for Contextual Logging? (MDC)](https://projectreactor.io/docs/core/release/reference/faq.html#faq.mdc)
