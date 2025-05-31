---
title: "Spring Cloud Gateway : 一条请求的处理流程"
date: 2025-05-31 01:47:22
tags: microservice
---

# 1. 什么时候需要网关？

简单的系统并不需要网关，只有当系统变得越来越复杂，就不得不面临以下问题：
1. 流量难以控制：系统里有多个模块，每个模块都对系统外部开放了入口。入口多导致系统频出问题；为了控制流量，每个模块都不得不做重复的工作。  
2. 外部调用成本高：久而久之，内部单个模块迁移，也需要协调大量外部工作。更别说系统级别的重构了。
3. 重复工作：除了流量控制，重复工作还有安全认证、服务鉴权、日志监控、协议转换与统一等基础能力。

上述问题单独看，也可以通过统一基础库等方式解决。但是整体解决的话，就需要网关，同时无论从开发还是部署，都与原来的模块是解耦的，在实际推进时也更加容易落地。

不止微服务，业务架构、大数据架构，很多时候面临多种系统的输入、输出时，网关是通用的解决方案。比如 Apache Linkis<sup>1</sup>是对接多种数据引擎的网关，其底层也是通过 Spring Cloud Gateway 实现的。

对应的，对于网关也会有更高的要求：

1. 性能：增加这一层，如何尽可能的降低性能影响？比如 WebFlux、连接池 等的使用  
2. 协议：实现待定，协议先行。支持哪些协议，协议如何转换  
3. 能力：流量控制、安全、日志、监控等，能够对各个服务开放、可配置  

今天这篇笔记，看看 HTTP 请求在 Spring Cloud Gateway 的处理流程：  
收到请求是如何处理的，如何一步步发送到了 downstream，在 downstream 返回结果后又是如何发送回 Client.
{:.success}

# 2. 例子、架构和数据流

用 SCG 的 AddRequestHeader GatewayFilter Factory<sup>2</sup>搭建一个简单例子。  
`application.yaml`配置：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: add-request-header-example
          uri: http://127.0.0.1:6001/user/addRequestHeaderExample
          predicates:
            - Path=/user/addRequestHeaderExample
          filters:
            - AddRequestHeader=X-Request-red, blue
```
收到客户端请求后，如果 path 匹配`/user/addRequestHeaderExample`，则添加 header，发送到指定的 uri.  
响应后，透传回客户端。  

`Route`是 SCG 里的路由单元，从配置里能看到`Route`的几个要素：`id predicates filters`:
1. `id`表示唯一性，用于查找`route`对象  
2. `predicates`用于判断当前`route`能否处理该请求
3. 能处理，则调用`filters`处理本次请求  

对应到`Route`的定义：

```java
public class Route implements Ordered {
  private final String id;
  private final URI uri;
  private final int order;
  private final AsyncPredicate<ServerWebExchange> predicate;
  private final List<GatewayFilter> gatewayFilters;
  private final Map<String, Object> metadata;

  ...
}  
```

这里参与处理的 filter 一共有这些，其中大部分是`GlobalFilter`:

![AddRequestHeader-combined-filters](/assets/images/SpringCloudGateway/AddRequestHeader-combined-filters.png)

各个 filter 的处理流程可以简化为：

![filter-process.svg](/assets/images/SpringCloudGateway/filter-process.svg)

*注：我在这里忽略了`ForwardRoutingFilter`，具体原因可见代码-ForwardRoutingFilter章节*  

更多例子可以参考个人仓库[Microservice-Systems/gateway](https://github.com/izualzhy/Microservice-Systems/tree/main/gateway)

下一节开始顺着这个流程图梳理代码，看看请求是如何被修改和转发的。

# 3. 代码

## 3.1. DispatcherHandler

SCG 基于 WebFlux 实现，WebFlux 的入口是`org.springframework.web.reactive.DispatcherHandler::handle`方法。

`handlerMappings`是`List<HandlerMapping>`结构，`handler`方法遍历该结构，判断当前元素是否可以处理该`exchange`，能处理则返回`handler`，否则为`Mono.empty()`. `concatMap().next()`获取第一个不为空的`handler`.

然后调用`handleRequestWith(exchange, handler)`传入返回的`handler`，处理该请求。

```java
public class DispatcherHandler implements WebHandler, PreFlightRequestHandler, ApplicationContextAware {

  private List<HandlerMapping> handlerMappings;

  private List<HandlerAdapter> handlerAdapters;

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

Spring Cloud Gateway 里，实现了自己的`RoutePredicateHandlerMapping` 
{:.warning}

## 3.2. SimpleHandlerAdapter

`handleRequestWith`的实现比较简单，这里会引入 webflux 里`HandlerAdapter`的概念，

```java
  private Mono<Void> handleRequestWith(ServerWebExchange exchange, Object handler) {
    if (ObjectUtils.nullSafeEquals(exchange.getResponse().getStatusCode(), HttpStatus.FORBIDDEN)) {
      return Mono.empty();  // CORS rejection
    }
    if (this.handlerAdapters != null) {
      for (HandlerAdapter adapter : this.handlerAdapters) {
        if (adapter.supports(handler)) {
          Mono<HandlerResult> resultMono = adapter.handle(exchange, handler);
          return handleResultMono(exchange, resultMono);
        }
      }
    }
    return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
  }
```

webflux 提供了几种`HandlerAdapter`，在 SCG 里我们暂时只关注`SimpleHandlerAdapter`即可。

```java
public class SimpleHandlerAdapter implements HandlerAdapter {

  @Override
  public boolean supports(Object handler) {
    return WebHandler.class.isAssignableFrom(handler.getClass());
  }

  @Override
  public Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler) {
    WebHandler webHandler = (WebHandler) handler;
    Mono<Void> mono = webHandler.handle(exchange);
    return mono.then(Mono.empty());
  }

}
```
上面`SimpleHandlerAdapter`的代码，会调用`WebHandler.handle`。因此，我们只要确保第一步返回的类型为`WebHandler`，就可以复用该流程。这里也是调用 Spring Cloud Gateway 实现的`FilteringWebHandler`处理的入口。 

## 3.3. RoutePredicateHandlerMapping

先接着看前面的`HandlerMapping`，Webflux 提供了一些原生的`HandlerMapping`，Gateway 则定义了自己的`RoutePredicateHandlerMapping`:

```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {

  private final FilteringWebHandler webHandler;

  @Override
  protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
    // don't handle requests on management port if set and different than server port
    if (this.managementPortType == DIFFERENT && this.managementPort != null
        && exchange.getRequest().getLocalAddress() != null
        && exchange.getRequest().getLocalAddress().getPort() == this.managementPort) {
      return Mono.empty();
    }
    exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());

    return Mono.deferContextual(contextView -> {
      exchange.getAttributes().put(GATEWAY_REACTOR_CONTEXT_ATTR, contextView);
      return lookupRoute(exchange)
        // .log("route-predicate-handler-mapping", Level.FINER) //name this
        .map((Function<Route, ?>) r -> {
          exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
          if (logger.isDebugEnabled()) {
            logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
          }

          exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
          return webHandler;
        })
        .switchIfEmpty(...);
    });
  }

  protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
    return this.routeLocator.getRoutes()
      // individually filter routes so that filterWhen error delaying is not a
      // problem
      .concatMap(route -> Mono.just(route).filterWhen(r -> {
        // add the current route we are testing
        exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
        return r.getPredicate().apply(exchange);
      })
        // instead of immediately stopping main flux due to error, log and
        // swallow it
        .doOnError(e -> logger.error("Error applying predicate for route: " + route.getId(), e))
        .onErrorResume(e -> Mono.empty()))
      // .defaultIfEmpty() put a static Route not found
      // or .switchIfEmpty()
      // .switchIfEmpty(Mono.<Route>empty().log("noroute"))
      .next()
      // TODO: error handling
      .map(route -> {
        if (logger.isDebugEnabled()) {
          logger.debug("Route matched: " + route.getId());
        }
        validateRoute(route, exchange);
        return route;
      });

    /*
     * TODO: trace logging if (logger.isTraceEnabled()) {
     * logger.trace("RouteDefinition did not match: " + routeDefinition.getId()); }
     */
  }
```

上面这段代码有两处关键点：
1. 无论匹配到哪个 Route, 返回的一定是`webHandler`，即`FilteringWebHandler`。WebFlux 原生的`SimpleHandlerAdapter`提供的`supports handle`方法，支持调用`WebHandler.handle`。SCG 正是通过这点，调用到`FilteringWebHandler.handle(exchange)`继续后续的 HTTP 处理流程。
2. `lookupRoute`遍历定义的`Route`集合，找到能够处理该`exchange`的`Route`，赋值到`exchange.getAttributes()`，key为`GATEWAY_ROUTE_ATTR`，通过这种方式记录了处理该请求的`route`.


## 3.4 FilteringWebHandler

`FilteringWebHandler`是 Gateway 里真正处理请求的地方，该类定义成`WebHandler`，所以可以被`SimpleHandlerAdapter`发现和调用。
{:.warning}

`FilteringWebHandler.handle`的实现：  
1. 组合全局的`globalFilters`和当前的`route.getFilters()`，记录到`combined`  
2. 初始化`DefaultGatewayFilterChain`，递归调用`filter.filter(exchange, chain)`, 注意是 lazy 模式。  

```java
public class FilteringWebHandler implements WebHandler {

  @Override
  public Mono<Void> handle(ServerWebExchange exchange) {
    Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
    List<GatewayFilter> gatewayFilters = route.getFilters();

    List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
    combined.addAll(gatewayFilters);
    // TODO: needed or cached?
    AnnotationAwareOrderComparator.sort(combined);

    if (logger.isDebugEnabled()) {
      logger.debug("Sorted gatewayFilterFactories: " + combined);
    }

    return new DefaultGatewayFilterChain(combined).filter(exchange);
  }

  private static class DefaultGatewayFilterChain implements GatewayFilterChain {

    private final int index;

    private final List<GatewayFilter> filters;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange) {
      return Mono.defer(() -> {
        if (this.index < filters.size()) {
          GatewayFilter filter = filters.get(this.index);
          DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(this, this.index + 1);
          return filter.filter(exchange, chain);
        }
        else {
          return Mono.empty(); // complete
        }
      });
    }
```

注：看到很多文档，会强调`DispatcherHandler` `FilteringWebHandler`等都是`WebHandler`，其实原因是不同的，后者的原因如上所述。

## 3.5. DefaultGatewayFilterChain

`DefaultGatewayFilterChain`是有状态的(记录了index)。在各个 `filter.filter`的实现里，都会调用`chain.filter(exchange)`。上面方法会在传入当前的`index+1`，效果上也就是调用链表里下一个`filter.filter`.

该链式调用的效果：
```
# DefaultGatewayFilterChain.filter：
FilterA -> FilterB -> FilterC 
-> 
... 
-> 
# then(xxx)里调用
FilterC -> FilterB -> FilterA
```

Filter 的顺序则是通过`Order`设置的。  
> All “pre” filter logic is executed. Then the proxy request is made. After the proxy request is made, the “post” filter logic is run.<sup>3</sup>

Filter 按照生效范围，可以分为`GlobalFilter`和`GatewayFilter`，效果相同，只是后者只作用的对应的 route 上。

按照流程，我觉得 Filter 也可以分为如下两类：  
1. 网络：调用 netty，发送和接收数据。不可配置，是网关管道的必经之路。  
2. 功能：调整 header、目标 uri 等。可配置，根据业务场景调整。  

接下来分别介绍。  

## 3.6. Filter-网络

### 3.6.1. NettyRoutingFilter(Order=2147483647)

这里取到转发的`GATEWAY_REQUEST_URL_ATTR`(即用户设置的目标 uri)，通过`reactor.netty.http.client.HttpClient`发起请求。downstream 返回响应后，响应头存到`CLIENT_RESPONSE_ATTR`, connection 存到`CLIENT_RESPONSE_CONN_ATTR`.

注意`chain.filter(exchange)`是在`then`里调用，也就是从这个 filter 开始处理响应。
{:.warning}

```java
public class NettyRoutingFilter implements GlobalFilter, Ordered {
  @Override
  @SuppressWarnings("Duplicates")
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);

    ...

    boolean preserveHost = exchange.getAttributeOrDefault(PRESERVE_HOST_HEADER_ATTRIBUTE, false);
    Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);

    Flux<HttpClientResponse> responseFlux = getHttpClientMono(route, exchange)
      ...
      }).request(method).uri(url).send((req, nettyOutbound) -> {
        ...
      }).responseConnection((res, connection) -> {

        // Defer committing the response until all route filters have run
        // Put client response as ServerWebExchange attribute and write
        // response later NettyWriteResponseFilter
        exchange.getAttributes().put(CLIENT_RESPONSE_ATTR, res);
        exchange.getAttributes().put(CLIENT_RESPONSE_CONN_ATTR, connection);

    return responseFlux.then(chain.filter(exchange));
  }        
```

### 3.6.2. NettyWriteResponseFilter(Order=-1)

`NettyRoutingFilter`将 downstream connection 放到了`CLIENT_RESPONSE_CONN_ATTR`，该 Filter 则是取出connection, 读取 downstream 的 response，返回给 SCG 的客户端。这也是为什么代码里逻辑需要在`then`，也就是 post 阶段实现的。

注意这里区分了是否是流式输出：
1. 流式输出，调用`writeAndFlushWith`  
2. 非流式输出，调用`writeWith`  

这里的`connection`连接到 downstream，因此 SCG 作为 inbound 接收数据。`exchange.getResponse()`是连接到 client，使用该对象写入返回给 client 的响应。

```java
public class NettyWriteResponseFilter implements GlobalFilter, Ordered {
  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return chain.filter(exchange)
        .then(Mono.defer(() -> {
          Connection connection = exchange.getAttribute(CLIENT_RESPONSE_CONN_ATTR);

          ...
          ServerHttpResponse response = exchange.getResponse();

          // TODO: needed?
          final Flux<DataBuffer> body = connection
              .inbound()
              .receive()
              .retain()
              .map(byteBuf -> wrap(byteBuf, response));

          MediaType contentType = null;
          try {
            contentType = response.getHeaders().getContentType();
          }
          catch (Exception e) {
            if (log.isTraceEnabled()) {
              log.trace("invalid media type", e);
            }
          }
          return (isStreamingMediaType(contentType)
              ? response.writeAndFlushWith(body.map(Flux::just))
              : response.writeWith(body));
        })).doOnCancel(() -> cleanup(exchange))
        .doOnError(throwable -> cleanup(exchange));
  }
```

按照`Order`大小，调用`NettyWriteResponseFilter`早于`NettyRoutingFilter`，我在最开始看到的时候觉得非常奇怪，也是知道看到`then`这个方法才恍然大悟。

## 3.7. Filter-功能

### 3.7.1. GatewayMetricsFilter(Order=0)

收集网关请求的度量指标（如响应时间），`GatewayMetricsFilter.filter`实现比较简单。

`Order`设置为 0 比较有意思，我们可以这么推测：设置了这个 Order 值，要统计的时间，就是在交给`NettyWriteResponseFilter`之前。同时所有 Order > 0 的 filter 处理性能，都可以统计到。0 作为一个分界点，从这个角度，设置为 0 就比较合理了。

```java
public class GatewayMetricsFilter implements GlobalFilter, Ordered {

  @Override
  public int getOrder() {
    // start the timer as soon as possible and report the metric event before we write
    // response to client
    return NettyWriteResponseFilter.WRITE_RESPONSE_FILTER_ORDER + 1;
  }

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    Sample sample = Timer.start(meterRegistry);

    return chain.filter(exchange)
      .doOnSuccess(aVoid -> endTimerRespectingCommit(exchange, sample))
      .doOnError(throwable -> endTimerRespectingCommit(exchange, sample));
  }
```


### 3.7.2 RouteToRequestUrlFilter(Order=10000)

用于 route 到指定的 url，例如该配置：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: route-example
          uri: http://127.0.0.1:6001/user/routeExample
          predicates:
            - Path=/user/routeExample
```

就是通过这段代码生效的：

```java
public class RouteToRequestUrlFilter implements GlobalFilter, Ordered {
  public static final int ROUTE_TO_URL_FILTER_ORDER = 10000;

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
    if (route == null) {
      return chain.filter(exchange);
    }
    log.trace("RouteToRequestUrlFilter start");
    URI uri = exchange.getRequest().getURI();
    boolean encoded = containsEncodedParts(uri);
    URI routeUri = route.getUri();

    ...

    URI mergedUrl = UriComponentsBuilder.fromUri(uri)
      // .uri(routeUri)
      .scheme(routeUri.getScheme())
      .host(routeUri.getHost())
      .port(routeUri.getPort())
      .build(encoded)
      .toUri();
    exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, mergedUrl);
    return chain.filter(exchange);
  }
```

注意我们自定义的 route 转发时，可能会在自定义的 filter 通过指定`GATEWAY_REQUEST_URL_ATTR`转发。观察上述代码也会设置该 key，所以为了避免被覆盖，自定义 filter 的 Order 应当小于`ROUTE_TO_URL_FILTER_ORDER`.
{:.warning}

### 3.7.3. ForwardRoutingFilter(Order=2147483647)

转发请求，比如可以将 /metrics 指向 /actuator/metrics

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: custom-metrics
          uri: forward:/actuator/metrics
          predicates:
            - Path=/metrics
```

从源码看，`ForwardRoutingFilter`和`NettyRoutingFilter`有着相同的`Order`，处理的`url`有所不同：

```java
public class ForwardRoutingFilter implements GlobalFilter, Ordered {
  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);

    String scheme = requestUrl.getScheme();
    if (isAlreadyRouted(exchange) || !"forward".equals(scheme)) {
      return chain.filter(exchange);
    }
    ...


public class NettyRoutingFilter implements GlobalFilter, Ordered {
  @Override
  @SuppressWarnings("Duplicates")
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);

    String scheme = requestUrl.getScheme();
    if (isAlreadyRouted(exchange) || (!"http".equalsIgnoreCase(scheme) && !"https".equalsIgnoreCase(scheme))) {
      return chain.filter(exchange);
    }
    setAlreadyRouted(exchange);
```

这样也就能解释为什么两者`Order`相同而没有问题了：两者处理的 schema 互斥。我们也可以猜测 SCG 的设计上，发起请求的 filter 被固定为 pre 阶段最后一环。

该类似乎是需要配合`ForwardPathFilter`使用，没有细看。

功能类的 filter 较多，例如 ReactiveLoadBalancerClientFilter(Order=10150)，用于处理`GATEWAY_REQUEST_URL_ATTR`为`lb:myservice`scheme 的 URI；以及官方提供的非常多 GatewayFilter Factories.

# 4. 总结

SCG 内置的 predicates、filters 很多，这篇笔记并不想一一罗列，更想通过“知其所以然”的方式串联起来。这样在实际场景里，把各类已知 filter 工具也可以用的更好。

这是 Spring Cloud Gateway 引用比较广泛的一张架构图<sup>3</sup>：

![spring_cloud_gateway_diagram](https://docs.spring.io/spring-cloud-gateway/reference/_images/spring_cloud_gateway_diagram.png)

初看的时候我还是觉得过于简洁了，经过这篇笔记，不妨再把架构图里的组件，对应到：
1. Gateway Handler Mapping: `RoutePredicateHandlerMapping`
2. Gateway Web Handler: `FilteringWebHandler`
3. Filter: 预置及自定义的各个 Filter

而之所以强调 Gateway，是因为 WebFlux 也提供了诸多默认的 HandlerMapping WebHandler.   

这些 Bean 的注册都是在`GatewayAutoConfiguration`注册的，根据配置决定是否开启。      
关于自定义的 Filter 实现，我也还有诸多疑问，比如 Filter 是链式执行的，那么自定义的 Filter，是否可以执行耗时的比如 IO 操作，IO 超时时，是否会耗尽线程池 hang 住其他转发请求。这些问题，大概需要到 WebFlux 里寻找答案了。

# 5. 参考

1. [Apache Linkis](https://linkis.apache.org/zh-CN/docs/latest/architecture/service-architecture/overview)
2. [AddRequestHeader GatewayFilter Factory](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories/addrequestheader-factory.html)
3. [How It Works](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/how-it-works.html)
