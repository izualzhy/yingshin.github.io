---
title: "Istio 初探"
date: 2025-03-09 06:25:45
tags: microservice
---

这篇笔记是对 Istio 文档<sup>1</sup>的一个初步总结。

## 1. Istio 是什么?

Istio 是服务网格的一种实现。AWS 对 Service Mesh 是这么描述的<sup>2</sup>：

> 服务网格是一个软件层，用于处理应用程序中服务之间的所有通信。该层由容器化微服务组成。随着应用程序的扩展和微服务数量的增加，监控服务的性能变得越来越困难。为了管理服务之间的连接，服务网格提供了监控、记录、跟踪和流量控制等新功能。它独立于每项服务的代码，这使它能够跨网络边界和多个服务管理系统工作。

即服务网格托管了以下功能：  
1. 负载均衡
2. 服务发现
3. 熔断
4. 动态路由
5. 安全通信
6. 多语言、多协议支持
7. 指标和分布式跟踪
8. 支持访问控制、限流和配额的可插入策略层和配置 API

我的理解，可以简单认为 Istio 就是 K8S 上更强大的的 Nginx.

## 2. 为什么诞生了 Istio?

软件架构，有一句非常常见的话：**让 xxx 更专注在 xxx 上**。

比如大数据领域，为了让分析师更专注在实现业务分析上，因此提供了非常强大的 SQL 能力。而在微服务领域，为了让开发人员更专注在实现业务逻辑，而不是负载均衡、日志追踪等通用能力上，因此有了 Istio.

之前写过一篇文章，是[Dolphin里的网络模型](https://izualzhy.cn/ds-net-model)，其中 Worker 上报自身状态，Master 分发任务时自行实现了负载均衡。这种自定义的方式，我在迁移到 K8S 上时发现存在诸多不变。因此随着服务网格的强大，也在逐步影响业务模块自身的架构设计。

## 3. Istio 架构

![Istio arch](/assets/images/istio/istio-arch.svg)

1. 控制平面 Istiod 提供服务发现、配置和证书管理。   
2. 数据平台 Envoy 部署为服务的 Sidecar，正如命名，Istio 对流量的控制是在这一层设计生效的。    

注意 Istio 启动后，除了 istiod 还有其他组件：

```bash
$ kubectl get deployments -n istio-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
istio-egressgateway    1/1     1            1           33d
istio-ingressgateway   1/1     1            1           33d
istiod                 1/1     1            1           33d
```

ingressgateway、egressgateway 分别是入口网关和出口网关，负责流量的输入输出管理。

## 4. Istio 自定义资源

Istio 引入了三类重要资源：
- VirtualService: 定义了流量分发的规则，比如根据 header字段、url pattern 发往不同的 K8S Service.  
- DestinationRule: 允许将不同标签的 pod 定义为不同的 subset。然后在 VS 里细化流量发往不同的 subset，以实现灰度发布类似的能力。
- Gateway: 网格最外层的定义，作为唯一的入口，然后通过 VS 将流量分发到内部服务。

简言之，VirtualService 绑定在 Gateway 上，VS 里可以配置了 不同的 uri 指向不同的 Service. 对于同一个 Service，也可以通过 DestinationRule 继续细化。

## 5. Istio 怎么用?

### 5.1. 流量控制

以 Bookinfo 应用为例：
![Bookinfo withistio](/assets/images/istio/bookinfo-withistio.svg)

初始版本，只定义了[name=bookinfo 的 Gateway](https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/bookinfo-gateway.yaml)，没有引入 Istio 的其他能力。 

由于 kube-proxy 默认采用 RR 策略将流量分配到全部的 pod 上，因此`productpage`页面会以轮询的方式展示不同版本的 reviews 服务 （红色星形、黑色星形或者没有星形）。

可以看到流量是均匀的发送到 review 各个版本的服务上：  
![Bookinfo-only-gateway](/assets/images/istio/bookinfo-only-gateway.png)

接下来，我们先定义[DestinationRule](https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/destination-rule-all.yaml)，然后定义[VirtualService](https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-all-v1.yaml)，注意此时reviews所有的流量都发送到了`subset=v1`，即`version=v1`版本的pod.

可以看到流量只被控制发送到了 review v1 版本的服务上：

![Bookinfo-DR-v1](/assets/images/istio/bookinfo-DR-v1.png)

进一步，我们可以设置更精确的路由规则，例如[virtual-service-reviews-test-v2](https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml):

```yaml
$ kubectl get virtualservice reviews -o yaml

apiVersion: networking.istio.io/v1
kind: VirtualService
...
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

当 headers 里包含`end-user:jason`时，流量就会发送到 v2 版本的 reviews 服务。Istio 支持了丰富的特性，例如[weight](https://preliminary.istio.io/latest/zh/docs/tasks/traffic-management/traffic-shifting/)，**使用这些配置，也就能起到灰度上线、AB 测试的效果**。

### 5.2. 注入延迟与故障

修改 ratings 的 VirtualService ，访问 ratings 服务耗时在 2s :

```yaml
  spec:
    hosts:
    - ratings
    http:
    - fault:
        delay:
          fixedDelay: 2s
          percentage:
            value: 100
      route:
      - destination:
          host: ratings
          subset: v1
```

可以看到修改生效:

```bash
$ kubectl exec -it reviews-v2-86d5db4bd6-bn6xg -- /bin/bash
default@reviews-v2-86d5db4bd6-bn6xg:/$ time curl http://ratings:9080

real    0m2.083s
user    0m0.006s
sys 0m0.014s
```

观测通路的延迟也达到了 2s+：

![bookinfo-ratings-delay2s](/assets/images/istio/bookinfo-ratings-delay2s.png)

*注：图里 ratings 的延迟在之后会变回 x ms ，感觉跟 reviews 的代码实现有关*

当然也可以通过配置 httpStatus 注入访问故障，不再赘述。

### 5.3. redirect and rewrite

实现 uri 的重定向和改写：

```
http:
- match:
    - uri:
        exact: /old-path
  redirect:
    uri: /new-path
    redirectCode: 301
- match:
    - uri:
        prefix: /final-path
  rewrite:
    uri: /final-path
  route:
    - destination:
        host: service.example.svc.cluster.local
        port:
          number: 80
```

### 5.4. 熔断

[熔断](https://preliminary.istio.io/latest/zh/docs/tasks/traffic-management/circuit-breaking/)能力使得我们可以在连接、请求达到预置或者异常检测时，不再提供服务，以避免系统的雪崩问题。

例如:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF

```

### 5.5. 认证与授权

Istio 定义了 RequestAuthentication 来实现认证。例如验证 JWT 是否合法：

```bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: ingress-jwt
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/master/security/tools/jwt/samples/jwks.json"
EOF
```

RequestAuthentication 对 JWT 的鉴权简单，但是表达能力有限。也可以引入[外部授权](https://raw.githubusercontent.com/istio/istio/master/samples/extauthz/ext-authz.yaml
service/ext-authz)，对接 OPA authorization、 oauth2-proxy 或定制的外部授权服务器。

### 5.6. 负载均衡

常用的方式有：  

- ROUND_ROBIN：默认，轮询算法，将请求依次分配给每一个实例。
- LEAST_CONN：最小连接数，随机选择两个健康实例，将请求分配给两个健康实例中连接数少的那个。
- RANDOM：随机算法，将请求随机分配给其中一个实例。

```
spec:
  trafficPolicy:
    loadBalancer: 
      simple: RANDOM
```

事实上，我在学习 Istio 的过程中，愈加觉得 Istio 功能丰富、系统目标很清晰（记得之前还认为服务网格只是炒作概念），而且 plugin 的设计也能扬长避短。这篇笔记谈不上生产环境的实践，更多的是对于常用能力的总结。

## 6. 参考

1. [Istio官方文档](https://preliminary.istio.io/latest/docs)
2. [什么是服务网格？](https://aws.amazon.com/cn/what-is/service-mesh)
3. [任务](https://preliminary.istio.io/latest/docs/tasks/)_

