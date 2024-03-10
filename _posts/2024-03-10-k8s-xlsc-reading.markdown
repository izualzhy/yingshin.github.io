---
title: "《Kubernetes修炼手册》读书笔记"
date: 2024-03-10 01:05:45
tags: read
---
![Kubernetes修炼手册](https://izualzhy.cn/assets/images/book/s34078881.jpg)

由于最近开始落地 Native Flink On Kubernetes，但是公司的容器团队支持力度很小。因此开始看 Kubernetes 相关书籍，这本书是从春节前开始看的，早上、周末、陆陆续续持续了一个月左右，收获很大。适合入门，非常推荐。

## 1. Kubernetes系统

Kubernetes 集群由主节点和工作节点组成：
1. 主节点(master): 即控制平面，包含了   
	1. API Server: 组件通信，通过HTTPS的方式提供了RESTful风格的API接口  
	2. 集群存储: etcd  
	3. controller管理器: controller 管理器是 controller 的管理者，例如工作节点controller、终端controller，以及副本controller，保证集群的当前状态（current state）可以与期望状态（desired state）相匹配  
	4. 调度器: 通过监听API Server来启动新的工作任务，分配到适合的且处于正常运行状态的节点中  
	5. 云 contoller 管理器: 云controller管理器负责集成底层的公有云服务，例如实例、负载均衡以及存储等  
2. 工作节点(node): 包含了 
	1. Kubelet: Kubelet负责将当前工作节点注册到集群当中，集群的资源池就会获取到当前工作节点的CPU、内存以及存储信息，并将工作节点加入当前资源池   
	2. CRI: Kubelet需要一个容器运行时（container runtime）来执行依赖容器才能执行的任务，例如拉取镜像并启动或停止容器
	3. kube proxy: kube-proxy保证了每个工作节点都可以获取到唯一的IP地址，并且实现了本地IPTABLE以及IPVS来保障Pod间的网络路由与负载均衡  
3. DNS: 每个Kubernetes集群都有自己内部的DNS服务，确保集群内的通信。  


## 2. Pod/Deployment/Service

VM 是 VMware 调度的原子单位，容器是 Docker 调度的原子单位，Pod 则是 Kubernetes 调度的原子单位。简单的使用方式，就是一个 Pod 里只运行一个容器；如果有共享 IPC 命名空间、内存、磁盘、网络等的场景，也可以在一个 Pod 里运行多个容器。

1. Pod 没有自愈能力，不能扩缩容，也不支持方便的升级和回滚。而 Deployment 可以。大多数时间，用户通过更上层的控制器来完成 Pod 部署。上层的控制器包括 Deployment、DaemonSet 以及 StatefulSet。  
2. Pod 是最基本的用来部署微服务应用的单元，而 Deployment 增加了诸如扩缩容、自愈和滚动升级等特性。但我们不能仅仅依靠各 Pod 的 IP 来访问它们。Service 对象能够为一组动态的 Pod 提供稳定可靠的网络管理能力。  
3. Service 依靠 Label 匹配 Pod，只要 Pod 拥有这些 Label 即可。Pod 的 Label 不要求完全一样，即使 Pod 有额外的 Label，不受影响。Service 有 ClusterIP、NodePort、LoadBalancer 三种方式，LB 即云厂商的负载均衡服务，使用要注意。   

举个 Native Flink On Kubernetes 的例子：
1. JobManager 是通过 Deployment 形式部署，定义了 replicas=1，即需要有 1 个 pod. Deployment、Pod 上都定义了 type=flink-native-kubernetes component=jobmanager app=Deployment名字。    
2. TaskManager 是通过 Pod 形式部署，也定义了 type、app 等 label，以及 component=taskmanager   
3. RestService: JobManager 会同时创建 Service，用于对外提供 RestEndpoint 服务；如果是非 HA 模式，还会定义 clusterIP=None 的 headless service，用于 Flink 集群通信。service 的 spec.selector 字段定义了用于筛选 Pod 的 lable 集合。例如：
```
% kubectl describe svc my-first-application-cluster-rest
Name:                     my-first-application-cluster-rest
Namespace:                default
Labels:                   app=my-first-application-cluster
                          type=flink-native-kubernetes
Annotations:              &lt;none&gt;
Selector:                 app=my-first-application-cluster,component=jobmanager,type=flink-native-kubernetes
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       x
IPs:                      x
Port:                     rest  8081/TCP
TargetPort:               8081/TCP
NodePort:                 rest  31492/TCP
Endpoints:                x:8081
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason           Age   From                Message
  ----    ------           ----  ----                -------
  Normal  EnsuringService  13m   service-controller  Deleted Loadbalancer
```   

## 3. 服务发现

每个 Service 对象维护了一个 Endpoint 对象，Endpoint 内部维护了会变化的 Pod IP，这个列表是通过 selector 筛选的。

每一个节点上都运行着一个 kube-proxy，它能够为新的 Service 和 Endpoint 创建 IPVS 规则，从而到达 Service 的 ClusterIP 的流量会被转发至匹配 Label 筛选器的某一个 Pod 上：
![kubeprox-service](/assets/images/kubernetes-xlsc/kubeprox-service.jpeg)

Kubernetes 将集群 DNS 作为服务注册中心使用，几个重要的组件:   
1. Pod：由coredns Deployment管理   
2. Service：一个名为kube-dns的ClusterIP Service，其监听端口为TCP/UDP53     
3. Endpoint：也叫kube-dns。所有与集群DNS相关的对象都有K8s-app=kube-dns的Label   
这一点在筛选kubectl输出的时候很有用。

FQDN 里包含：$object-name.$namespace.svc.cluster.local，例如 ent.prod.svc.cluster.local

举个例子：  
1. curl env:8080 ，访问的是 ns=dev 的 env 服务   
2. curl env.prod.svc.cluster.local，访问的是 ns=prod 的 env 服务   

这是 dns 解析到了不同 ip.同样的，容器外部访问不了该名字，但是容器内部机器可以：
```
% curl my-first-application-cluster-rest:8081
curl: (6) Could not resolve host: my-first-application-cluster-rest; Unknown error

% kubectl exec -it -n default my-first-application-cluster-taskmanager-1-1 -- /bin/bash
root@my-first-application-cluster-taskmanager-1-1:/opt/flink# curl my-first-application-cluster-rest:8081
```

dns 修改了 pod 的 /etc/resolve.conf，比如同一个 flink 任务的 jobmanager、taskmanager：

```
root@my-first-application-cluster-6647497579-79dr5:/opt/flink# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver x.y.z
options ndots:5

root@my-first-application-cluster-taskmanager-1-1:/opt/flink# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver x.y.z
options ndots:5
```

这个 x.y.z 正是 kube-dns service 的 CLUSTER-IP

## 4. 存储

kind=PersistentVolumen kind=PersistentVolumeClaim

挂载外部存储的方式，预计当前阶段用的不多，没有细看。后续实践里，打算用于挂载用户 Flink 任务未打包到镜像里的文件。

## 5. ConfigMap

应用和配置解耦，其优点如下:

+ 可重用的应用镜像  
+ 更容易测试  
+ 更简单、更少的破坏性改动  

ConfigMap 包含了多个 key/value 格式的数据。具体创建和使用的流程

通过卷导入ConfigMap：
1. 创建 ConfigMap: 名为 multimap，具体略   
2. 创建基于 ConfigMap multimap 的名为 volmap 的卷   
```
spec:
  volumes:
    - name: volmap
      configMap:
        name:multimap
```
3.将 volmap 挂载到 /etc/name
```
spec:
  containers:
    - name: ctr
      image: nginx
      volumeMounts:
        - name: volmap
          mountPath: /etc/name
```
这样的效果，就是 /etc/name 目录下有了文件，文件名是 multimap 的 key，文件内容是对应的 value

再看看在 flink jobmanager 是如何使用 ConfigMap 的：   
```
spec:
  containers:
  - args:
    - native-k8s
    - ...
    volumeMounts:
    - mountPath: /opt/flink/conf
      name: flink-config-volume
  volumes:
  - configMap:
      defaultMode: 420
      items:
      - key: logback-console.xml
        path: logback-console.xml
      - key: log4j-console.properties
        path: log4j-console.properties
      - key: flink-conf.yaml
        path: flink-conf.yaml
      name: flink-config-zlink-202402201833
    name: flink-config-volume
```

1. 定义了 volume = flink-config-volume, 挂载点=/opt/flink/conf   
2. flink-config-volume 这个volume，引用了 configmap=flink-config-zlink-202402201833 里的 3 个 entry  
3. 效果上：/opt/flink/conf/$key 这个文件内容为 $value，key/value 即来自于 2 里的 entry 定义；当 configmap 的值更新了，文件内容也会异步更新。  
4. 挂载点是只读的，通过更新 ConfigMap 对象来更新文件。  

## 6. 集群运维

集群运维是最为复杂和考验熟悉程度的：

1. kubectl get/describe/apply/delete 不同资源的命令都是类似的，需要熟练。   
2. pod 里支持的命令很少，可以启动一个用于排查问题的Pod，其中需安装常用的的网络工具（ping、traceroute、curl、dig、nslookup等）。例如常见的测试 DNS解 析的方法是使用 nslookup 来解析用于代理 API Server 的 kubenetes.default Service，测试请求将返回一个 IP 地址和名称 kubernetes.default.svc.cluster.local。