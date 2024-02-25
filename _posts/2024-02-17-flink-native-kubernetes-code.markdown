---
title: "Flink - fabric8 的使用"
date: 2024-02-17 06:13:37
tags: flink
---

Flink 使用 Fabric8 Kubernetes client<sup>1</sup>作为 Kubernetes 的客户端，本文结合 Flink 提交 JobManager、TaskManager 的代码介绍 Fabric8 的使用。

## 1. Flink 使用 Fabric8 相关源码

Fabric8 是一个 Java 的 Kubernetes 客户端，使用一套自定义的 DSL 跟 REST API 交互。

我们可以使用链式调用方式访问和操作集群资源，例如：

```java
            ListOptions options = new ListOptions();
            options.setLabelSelector("type=flink-native-kubernetes");
            client.pods().inNamespace("default")
                    .list(options)
                    .getItems().forEach(pod -> {
                        System.out.println("pod name: " + pod.getMetadata().getName());
                        System.out.println("pod: " + pod);
                    });
```

这段代码会查询 namespace=default 下的所有 flink pod.

提交 JobManager TaskManager 流程大致相同，因此 Flink 封装了一些公共类：
1. `KubernetesParameters`: 有两个子类`KubernetesJobManagerParameters KubernetesTaskManagerParameters`，存储 JobManager Pod、TaskManager Pod 的参数。
2. `KubernetesJobManagerFactory KubernetesTaskManagerFactory`: 这两个工厂类分别提供了 JobManager Pod、 TaskManager Pod 的定义
3. `Fabric8FlinkKubeClient`: 提供`createJobManagerComponent createTaskManagerPod`创建 JobManagerPod、TaskManager Pod，实现时通过成员变量`io.fabric8.kubernetes.client.KubernetesClient internalClient` 跟 Kubernetes 集群交互。

### 1.1. 提交 JobManager

提交 JobManager 入口在`KubernetesClusterDescriptor.deployClusterInternal`:

```java
public class KubernetesClusterDescriptor implements ClusterDescriptor<String> {
    private ClusterClientProvider<String> deployClusterInternal(
            String entryPoint, ClusterSpecification clusterSpecification, boolean detached)
            throws ClusterDeploymentException {
        // ...
        final KubernetesJobManagerParameters kubernetesJobManagerParameters =
                new KubernetesJobManagerParameters(flinkConfig, clusterSpecification);

        final KubernetesJobManagerSpecification kubernetesJobManagerSpec =
                KubernetesJobManagerFactory.buildKubernetesJobManagerSpecification(
                        kubernetesJobManagerParameters);

        client.createJobManagerComponent(kubernetesJobManagerSpec);

        return createClusterClientProvider(clusterId);
    }
}
```
主要分为三步：
1. `KubernetesJobManagerFactory.buildKubernetesJobManagerSpecification`: 初始化 JM 配置
2. `client.createJobManagerComponent(kubernetesJobManagerSpec)`: 创建 JM Pod
3. `createClusterClientProvider(clusterId)`: 封装了 service，即 Flink WebUI

初始化 JM 使用了装饰器模式:

```java
public class KubernetesJobManagerFactory {
    private static final Logger LOG = LoggerFactory.getLogger(KubernetesJobManagerFactory.class);


    public static KubernetesJobManagerSpecification buildKubernetesJobManagerSpecification(
            KubernetesJobManagerParameters kubernetesJobManagerParameters) throws IOException {
        FlinkPod flinkPod = new FlinkPod.Builder().build();
        List<HasMetadata> accompanyingResources = new ArrayList<>();

        final KubernetesStepDecorator[] stepDecorators =
                new KubernetesStepDecorator[] {
                    new InitJobManagerDecorator(kubernetesJobManagerParameters),
                    new EnvSecretsDecorator(kubernetesJobManagerParameters),
                    new MountSecretsDecorator(kubernetesJobManagerParameters),
                    new JavaCmdJobManagerDecorator(kubernetesJobManagerParameters),
                    new InternalServiceDecorator(kubernetesJobManagerParameters),
                    new ExternalServiceDecorator(kubernetesJobManagerParameters),
                    new HadoopConfMountDecorator(kubernetesJobManagerParameters),
                    new KerberosMountDecorator(kubernetesJobManagerParameters),
                    new FlinkConfMountDecorator(kubernetesJobManagerParameters)
                };

        for (KubernetesStepDecorator stepDecorator : stepDecorators) {
            flinkPod = stepDecorator.decorateFlinkPod(flinkPod);
            List<HasMetadata> hasMetadataList = stepDecorator.buildAccompanyingKubernetesResources();
            accompanyingResources.addAll(hasMetadataList);
        }

        final Deployment deployment =
                createJobManagerDeployment(flinkPod, kubernetesJobManagerParameters);

        return new KubernetesJobManagerSpecification(deployment, accompanyingResources);
    }

    private static Deployment createJobManagerDeployment(
            FlinkPod flinkPod, KubernetesJobManagerParameters kubernetesJobManagerParameters) {
        final Container resolvedMainContainer = flinkPod.getMainContainer();

        final Pod resolvedPod =
                new PodBuilder(flinkPod.getPod())
                        .editOrNewSpec()
                        .addToContainers(resolvedMainContainer)
                        .endSpec()
                        .build();

        return new DeploymentBuilder()
                .withApiVersion(Constants.APPS_API_VERSION)
                .editOrNewMetadata()
                .withName(
                        KubernetesUtils.getDeploymentName(
                                kubernetesJobManagerParameters.getClusterId()))
                .withLabels(kubernetesJobManagerParameters.getLabels())
                .endMetadata()
                .editOrNewSpec()
                .withReplicas(1)
                .editOrNewTemplate()
                .withMetadata(resolvedPod.getMetadata())
                .withSpec(resolvedPod.getSpec())
                .endTemplate()
                .editOrNewSelector()
                .addToMatchLabels(labels)
                .endSelector()
                .endSpec()
                .build();
    }
}
```

`buildAccompanyingKubernetesResources`方法在传入 Pod 的基础上，通过 fabric8/kubernetes-client 的方法添加 Pod、Service、Secret、ConfigMap 等资源。例如:  
1. `InitJobManagerDecorator`使用`PodBuilder`
定义了 Image、ImagePullPolicy
2. `ExternalServiceDecorator`使用`ServiceBuilder`定义了 RestServiceExposedType 
3. `HadoopConfMountDecorator FlinkConfMountDecorator`使用`VolumeBuilder`加载 Hadoop、Flink 等配置。

常见的属性，例如 command、args、env、ports、resources、volumeMounts 都是在这里配置的，经过层层装饰，完成 JobManager Pod 的目标定义。

`createJobManagerDeployment`创建 JobManager 部署的 deployment，例如 Pod 的副本数(1)、labels 等。

`Fabric8FlinkKubeClient.createJobManagerComponent`完成创建 JobManager Deployment: 
```java
public class Fabric8FlinkKubeClient implements FlinkKubeClient {
    public void createJobManagerComponent(KubernetesJobManagerSpecification kubernetesJMSpec) {
        final Deployment deployment = kubernetesJMSpec.getDeployment();
        final List<HasMetadata> accompanyingResources = kubernetesJMSpec.getAccompanyingResources();

        // create Deployment
        final Deployment createdDeployment =
                this.internalClient
                        .apps()
                        .deployments()
                        .inNamespace(this.namespace)
                        .create(deployment);

        // Note that we should use the uid of the created Deployment for the OwnerReference.
        setOwnerReference(createdDeployment, accompanyingResources);

        this.internalClient
                .resourceList(accompanyingResources)
                .inNamespace(this.namespace)
                .createOrReplace();
    }    
}
```

### 1.2. 提交 TaskManager

提交 TaskManager 入口是在`KubernetesResourceManagerDriver.requestResource`

```java
public class KubernetesResourceManagerDriver
        extends AbstractResourceManagerDriver<KubernetesWorkerNode> {
    @Override
    protected void initializeInternal() throws Exception {
        kubeClientOpt =
                Optional.of(kubeClientFactory.fromConfiguration(flinkConfig, getIoExecutor()));
        log.info("initializeInternal labels:{}", KubernetesUtils.getTaskManagerLabels(clusterId));
        podsWatchOpt =
                Optional.of(
                        getKubeClient()
                                .watchPodsAndDoCallback(
                                        KubernetesUtils.getTaskManagerLabels(clusterId),
                                        new PodCallbackHandlerImpl()));
        recoverWorkerNodesFromPreviousAttempts();
    }

    @Override
    public CompletableFuture<KubernetesWorkerNode> requestResource(
            TaskExecutorProcessSpec taskExecutorProcessSpec) {
        final KubernetesTaskManagerParameters parameters =
                createKubernetesTaskManagerParameters(taskExecutorProcessSpec);
        final KubernetesPod taskManagerPod =
                KubernetesTaskManagerFactory.buildTaskManagerKubernetesPod(parameters);
        // ...

        // When K8s API Server is temporary unavailable, `kubeClient.createTaskManagerPod` might
        // fail immediately.
        // In case of pod creation failures, we should wait for an interval before trying to create
        // new pods.
        // Otherwise, ActiveResourceManager will always re-requesting the worker, which keeps the
        // main thread busy.
        final CompletableFuture<Void> createPodFuture =
                podCreationCoolDown.thenCompose(
                        (ignore) -> getKubeClient().createTaskManagerPod(taskManagerPod));
        
        // ...
    }
}
```

可以看到 TaskManager Pod 的创建，跟 JobManager 流程是类似的：先初始化 parameters，然后定义 Pod，最后通过 kubeClient 创建。

`KubernetesTaskManagerFactory.buildTaskManagerKubernetesPod`跟`KubernetesJobManagerFactory.buildKubernetesJobManagerSpecification`类似，也是采用装饰器的模式；但是相对简单一些，因为 TaskManager 不需要对外暴露服务，也就不需要`InternalServiceDecorator ExternalServiceDecorator`了。

`Fabric8FlinkKubeClient.createTaskManagerPod`完成创建 TaskManger Pod:
```java
public class Fabric8FlinkKubeClient implements FlinkKubeClient {
    @Override
    public CompletableFuture<Void> createTaskManagerPod(KubernetesPod kubernetesPod) {
        return CompletableFuture.runAsync(
                () -> {
                    final Deployment masterDeployment =
                            this.internalClient
                                    .apps()
                                    .deployments()
                                    .inNamespace(this.namespace)
                                    .withName(KubernetesUtils.getDeploymentName(clusterId))
                                    .get();

                    // Note that we should use the uid of the master Deployment for the
                    // OwnerReference.
                    setOwnerReference(
                            masterDeployment,
                            Collections.singletonList(kubernetesPod.getInternalResource()));

                    this.internalClient
                            .pods()
                            .inNamespace(this.namespace)
                            .create(kubernetesPod.getInternalResource());
                },
                kubeClientExecutorService);
    }
}
```

注意`initializeInternal`方法里，还监听了 TaskManager Pod 的变化。当 Pod 变化时，回调`PodCallbackHandlerImpl`方法。

## 2. Fabric8 的使用

由于 Fabric8 提供了一套 DSL 用于操作 Kubernetes 资源，因此使用是比较简便的。但是同时也注意，由于 DSL 的限制，有些接口报错只有在运行态才会观察到。这里举一个创建 Pod 的例子，更多可以参考kubernetes-examples<sup>2</sup>

```java
public class CreatePodExample {
    void createPodSample(String fileName) {
        System.setProperty(KUBERNETES_KUBECONFIG_FILE, "/data/homework/.kube/config");
        try (final KubernetesClient client = KubernetesUtils.initKubernetesClient()) {
            logger.info("namespace:{}", client.getNamespace());
            client.pods().list().getItems().forEach(pod -> {
                logger.info("pod name:{}", pod.getMetadata().getName());
            });

            logger.info("config:{} {} {}", client.getConfiguration().getFile()
                    , client.getConfiguration().getClientCertFile()
                    , client.getMasterUrl());
            // created by yml
            List<HasMetadata> resources = client.load(Files.newInputStream(Paths.get(fileName)))
                    .items();
            logger.info("resources'len:{}", resources.size());
            logger.info("resources:{}", resources);

            HasMetadata resource = resources.get(0);
            if (resource instanceof Pod) {
                Pod pod = (Pod) resource;

                Pod createdPod = client.pods()
                        .inNamespace(client.getNamespace())
                        .resource(pod)
                        .create();

                logger.info("created pod name:{}", createdPod.getMetadata().getName());
                logger.info("created pod:{}", createdPod);
            }

            // created by builder
            Pod pod = new PodBuilder()
                    .withNewMetadata()
                    .withGenerateName("example-pod-")
                    .addToLabels("app", "example-pod")
                    .addToLabels("version", "v1")
                    .addToLabels("role", "backend")
                    .endMetadata()
                    .withNewSpec()
                    .addNewContainer()
                    .withName("nginx")
                    .withImage("nginx:1.7.9")
                    .withPorts(Collections.singletonList(new ContainerPort(80, null, null, "http", null)))
                    .endContainer()
                    .endSpec()
                    .build();

            Pod createdPod = client.pods().inNamespace("default").create(pod);
            logger.info("created pod name:{}", createdPod.getMetadata().getName());
        } catch (Exception e) {
            logger.info("exception", e);
        }
    }
}
```

## 3. 总结

Flink 在跟 Kubernetes 集群交互时，底层使用 Fabric8. 创建 JobManager、TaskManager 代码有一些相似之处，Pod 资源的配置，使用了装饰器模式。这段代码定义了 JM/TM 的镜像、环境变量、启动命令、参数、配置，之后就是通过 Fabric8 创建 Pod 了。

JM 和 TM 的 label:

|   label   |       JobManager        |       TaskManager       |
|:---------:|:-----------------------:|:-----------------------:|
|    app    |      ${clusterId}       |      ${clusterId}       |
| component |       jobmanager        |       taskmanager       |
|   type    | flink-native-kubernetes | flink-native-kubernetes |

同时可以观察到，JobManager 是通过 deployment 创建，能够“自愈”；而 TaskManager 是通过 pod 创建的，失败后依赖 restart-strategy 配置恢复。

## 4. 参考资料
1. [Fabric8 Kubernetes client](https://nightlies.apache.org/flink/flink-docs-release-1.12/deployment/resource-providers/native_kubernetes.html#configuring-flink-on-kubernetes)
2. [https://github.com/fabric8io/kubernetes-client/tree/main/kubernetes-examples](https://github.com/fabric8io/kubernetes-client/tree/main/kubernetes-examples)