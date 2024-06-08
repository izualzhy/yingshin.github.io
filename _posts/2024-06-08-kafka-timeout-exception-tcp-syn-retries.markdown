---
title: "一个环境导致读取 Kakfa TimeoutException 的问题"
date: 2024-06-08 06:50:26
tags: bigdata
---

最近遇到一个比较奇怪的现象：相同 Flink 任务，换个运行环境就会报读取 kafka 失败，但是排查环境跟 Kafka 源的连通性又没有问题。

线上匆忙解决了，今天简化代码验证，感觉值得总结一版。

## 1. TimeoutException

```
org.apache.kafka.common.errors.TimeoutException: Timeout expired while fetching topic metadata
```

这个报错在读取 Kafka 时容易遇到，往往是 client 跟 bootstrap.server 的网络问题，或者 server 本身不可用导致。

但是从我的情况看，任务代码及配置是一致的，报错跟环境相关，唯一的疑点是任务 KafkaConsumer 配置的 bootstrap.server 存在多个已经下线的节点。

## 2. Kafka 及 Flink 代码分析

Kafka 代码里，该报错在`Fetcher.getTopicMetadata`：

```java
public class Fetcher<K, V> implements Closeable {
    public Map<String, List<PartitionInfo>> getTopicMetadata(MetadataRequest.Builder request, Timer timer) {
        do {
            RequestFuture<ClientResponse> future = sendMetadataRequest(request);
            client.poll(future, timer);

            ...

            timer.sleep(retryBackoffMs);
        } while (timer.notExpired());

        throw new TimeoutException("Timeout expired while fetching topic metadata");
    }
}
```

Flink 代码里，调用`Fetcher.getTopicMetadata`方法的栈：

```mermaid
    flowchart TB
    A("FlinkKafkaConsumerBase.open") --> B["partitionDiscoverer.discoverPartitions"] --> C("getAllPartitionsForTopics") --> D(" kafkaConsumer.partitionsFor")
    D --> E("fetcher.getTopicMetadata")

    F("FlinkKafkaConsumerBase.run") --> G["runWithPartitionDiscovery"] --> H("createAndStartDiscoveryLoop") --> B
```

1. open: 用于初始化读取的 offset(比如任务配置了 SPECIFIC_OFFSETS，但是没有指定全部 partition 的情况)  
2. run: 检测 TopicPartition 变化(配置了flink.partition-discovery.interval-millis)

粗看代码，使用比较简单，也没有跟机器环境有关的部分。

## 3. 简化流程

到这思路就卡住了，继续查看 kafka 源码耗时长，flink 调试流程又繁琐。因此尝试简化代码仅调用`KafkaConsumer.partitionsFor`，惊喜的发现跟 flink 任务的行为一致：有的环境正常，有的打印了相同的报错。

于是查看正常执行时 Kafka TRACE 日志(x y 是配置的 broker 地址，x 是已经下线的节点)：

```
[INFO] 2024-06-08 10:14:26.382 org.apache.kafka.clients.consumer.ConsumerConfig:[347] - ConsumerConfig values:
...
[DEBUG] 2024-06-08 10:14:26.432 org.apache.kafka.clients.consumer.KafkaConsumer:[699] - [Consumer clientId=consumer-g-1, groupId=g] Initializing the Kafka consumer
[DEBUG] 2024-06-08 10:14:26.604 org.apache.kafka.clients.consumer.KafkaConsumer:[815] - [Consumer clientId=consumer-g-1, groupId=g] Kafka consumer initialized
[TRACE] 2024-06-08 10:14:26.738 org.apache.kafka.clients.NetworkClient:[700] - [Consumer clientId=consumer-g-1, groupId=g] Found least loaded node x.x.x.x:9092 (id: -18 rack: null) with no active connection
[DEBUG] 2024-06-08 10:14:26.742 org.apache.kafka.clients.NetworkClient:[950] - [Consumer clientId=consumer-g-1, groupId=g] Initiating connection to node x.x.x.x:9092 (id: -18 rack: null) using address /x.x.x.x
[TRACE] 2024-06-08 10:14:26.751 org.apache.kafka.clients.NetworkClient:[697] - [Consumer clientId=consumer-g-1, groupId=g] Found least loaded connecting node x.x.x.x:9092 (id: -18 rack: null)
...
[TRACE] 2024-06-08 10:14:33.745 org.apache.kafka.clients.NetworkClient:[697] - [Consumer clientId=consumer-g-1, groupId=g] Found least loaded connecting node x.x.x.x:9092 (id: -18 rack: null)
[DEBUG] 2024-06-08 10:14:33.763 org.apache.kafka.common.network.Selector:[607] - [Consumer clientId=consumer-g-1, groupId=g] Connection with /x.x.x.x disconnected
java.net.ConnectException: Connection timed out
    at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
    at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:716)
    at org.apache.kafka.common.network.PlaintextTransportLayer.finishConnect(PlaintextTransportLayer.java:50)
    at org.apache.kafka.common.network.KafkaChannel.finishConnect(KafkaChannel.java:216)
    at org.apache.kafka.common.network.Selector.pollSelectionKeys(Selector.java:531)
    at org.apache.kafka.common.network.Selector.poll(Selector.java:483)
    at org.apache.kafka.clients.NetworkClient.poll(NetworkClient.java:547)
    at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:262)
    at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:233)
    at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:212)
    at org.apache.kafka.clients.consumer.internals.Fetcher.getTopicMetadata(Fetcher.java:368)
    at org.apache.kafka.clients.consumer.KafkaConsumer.partitionsFor(KafkaConsumer.java:1926)
    at org.apache.kafka.clients.consumer.KafkaConsumer.partitionsFor(KafkaConsumer.java:1894)
    at cn.izualzhy.SimpleKafkaConsumer.listPartitions(SimpleKafkaConsumer.java:34)
    at cn.izualzhy.SimpleKafkaConsumer.main(SimpleKafkaConsumer.java:59)
[DEBUG] 2024-06-08 10:14:33.764 org.apache.kafka.clients.NetworkClient:[891] - [Consumer clientId=consumer-g-1, groupId=g] Node -18 disconnected.
[WARN] 2024-06-08 10:14:33.765 org.apache.kafka.clients.NetworkClient:[756] - [Consumer clientId=consumer-g-1, groupId=g] Connection to node -18 (/x.x.x.x:9092) could not be established. Broker may not be available.
[WARN] 2024-06-08 10:14:33.766 org.apache.kafka.clients.NetworkClient:[1024] - [Consumer clientId=consumer-g-1, groupId=g] Bootstrap broker x.x.x.x:9092 (id: -18 rack: null) disconnected
[DEBUG] 2024-06-08 10:14:33.813 org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient:[593] - [Consumer clientId=consumer-g-1, groupId=g] Cancelled request with header RequestHeader(apiKey=METADATA, apiVersion=9, clientId=consumer-g-1, correlationId=0) due to node -18 being disconnected
[TRACE] 2024-06-08 10:14:33.914 org.apache.kafka.clients.NetworkClient:[700] - [Consumer clientId=consumer-g-1, groupId=g] Found least loaded node y.y.y.y:9092 (id: -56 rack: null) with no active connection
```

日志里有个非常重要的信息，`尝试连接 x 节点几秒后，连接失败接着尝试 y 节点`{:.info}

而`异常情况，则是会一直尝试连接 x.x.x.x 直到超时失败`{:.error}

使用 telnet 连接 x 节点：

```shell
# time telnet x.x.x.x 9092
Trying x.x.x.x...
telnet: connect to address x.x.x.x: Connection timed out

real    0m7.091s
user    0m0.001s
sys 0m0.000s

% time telnet x.x.x.x 9092
Trying x.x.x.x...
telnet: connect to address x.x.x.x: Connection timed out
telnet x.x.x.x 9092  0.00s user 0.00s system 0% cpu 1:03.14 total
```

跟上述代码一致，执行时间存在较大差别。**因此可以猜测原因跟 socket 配置有关**。

## 4. tcp_syn_retries

以一段代码来说明 socket 连接的超时时间:

```java
public class NonBlockingSocketChannelWithRetry {
    public static void main(String[] args) {
        String host = "192.0.2.1";  // 使用一个无法访问的IP地址来模拟连接超时
        int port = 9092;            // Kafka通常使用的端口
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

        try (Selector selector = Selector.open();
             SocketChannel socketChannel = SocketChannel.open()) {

            socketChannel.configureBlocking(false); // 设置为非阻塞模式
            System.out.println(LocalDateTime.now().format(dtf) + " - Attempting to connect to " + host + ":" + port);

            if (!socketChannel.connect(new InetSocketAddress(host, port))) {
                socketChannel.register(selector, SelectionKey.OP_CONNECT);
                while (selector.select() > 0) {  // 无超时，直到有事件发生
//              while (selector.select(10000) > 0) {  // 超时10s
                    Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();
                    while (keyIterator.hasNext()) {
                        SelectionKey key = keyIterator.next();
                        keyIterator.remove();
                        if (key.isConnectable()) {
                            if (socketChannel.finishConnect()) {
                                ...
    }
}
```

*完整代码在 [NonBlockingSocketChannelWithRetry](https://github.com/izualzhy/Bigdata-Systems/blob/main/java/java/src/main/java/cn/izualzhy/NonBlockingSocketChannelWithRetry.java)*
1. `select`指定时间时，超时跟该时间一致；  
2. 如果`select`没有指定超时时间，则跟`tcp_syn_retries`有关

![TCP_SYN_RETRY](/assets/images/tcp_syn_retry.webp){:height="200"}

TCP 建立连接，如果未收到 SYN+ACK，则 client 会一直尝试发送 SYN，直到达到`tcp_syn_retries`次数，每次重试间隔是2的幂次方([RFC 6298](https://www.rfc-editor.org/rfc/rfc6298.txt))<sup>1</sup>，测试机器上：

```
net.ipv4.tcp_syn_retries = 5
```

这样就解释了为什么前面 telnet 在 63s 秒后超时退出。

因此修改机器配置可以解决，实际上 Kafka 在高版本也引入了 socket.connection.setup.timeout.ms socket.connection.setup.timeout.max.ms<sup>3</sup>，来避免超时时间跟机器环境强相关。进一步，当我们自己使用 RPC 时，应当显示设置超时时间；读写 Kafka 时，使用 LB 而不是 broker 列表也是一个好习惯。

## 5. Ref

1. [TCP Syn Retries](https://medium.com/@avocadi/tcp-syn-retries-f30756ec7c55)
2. [Kafka client is not traversing through an entire bootstrap list](https://community.streamsets.com/advanced-debugging-48/kafka-client-is-not-traversing-through-an-entire-bootstrap-list-1989)
3. [KIP-601: Configurable socket connection timeout in NetworkClient](https://cwiki.apache.org/confluence/display/KAFKA/KIP-601%3A+Configurable+socket+connection+timeout+in+NetworkClient)