---
title: "[译]粗略估算Flink集群的容量"
date: 2022-05-11T13:49:01+08:00
lastmod: 2022-05-11T13:49:01+08:00
keywords: ["flink"]
categories: ["flink"]
tags: ["flink"]
author: "小十一狼"
---

原文：[How To Size Your Apache Flink® Cluster: A Back-of-the-Envelope Calculation](https://www.ververica.com/blog/how-to-size-your-apache-flink-cluster-general-guidelines)

一个常被问到的问题是当从开发环境到线上环境的过程中如何估算Flink集群大小。一般得到的回答都是"视情况而定"，但这毫无帮助。这篇文章罗列一些问答，给出一些数字来帮助我们进行Flink集群的估算。

# 运用数学，建立基线

第一步则是思考通过应用指标来建立所需资源的基线。

需要考虑的关键指标如下：

- 每秒记录数 与 每条记录大小
- 去重的key数量 与 每个key的state大小
- state更新次数 与 state backend的访问模式

当然更实际的关注点是你与客户之间关于停机时间、延迟、最大吞吐量的SLA，将直接影响你的容量规划。

接下来，看一下你的预算会被消耗在哪些资源上：

- 网络容量。考虑到任何外部服务都会使用网络，比如Kafka，HDFS，等等。
- 磁盘带宽。如果你依赖一个基于磁盘的state backend，比如RocksDB。（也要考虑到Kafka和HDFS对磁盘的使用）
- 机器数，以及可用的CPU和内存。

基于以上因素来为常见算子构建基线，并预留一些资源buffer用于任务恢复时追数或者处理负载峰值。建议在建立基线时也考虑checkpointing消耗的资源。

# 示例：跑一些数字看看

为了演示建立资源使用基线，现在计划在假定集群上进行作业部署。示例中的数字仅是粗略且不全面地演算值，文末还会专门讲出一些没有考虑到的方面。

## Flink流作业示例和硬件

![Flink流作业示例](/粗略计算Flink集群的容量/typical_streaming_job.png)

看这个示例，部署一个典型的流作业，利用Flink Kafka Consumer从Kafka topic读取数据，后面接keyBy和Sliding Window算子做聚合。滑动窗口size设为5分钟，slide设为1分钟。

这意味着将得到每分钟就会更新过去5分钟的聚合数据，该流作业按userId进行聚合。消费Kafka topic的消息平均大小为2KB。

作业吞吐量是1,000,000 msg/sec。为了了解窗口算子的state大小，需要知道去重的key数量。在这个例子中，userId即为key，有500,000,000个不同的用户。对每一个用户，计算4个数值，存储为long型（8字节）。整理下作业的关键指标：

- **消息大小**：2KB
- **吞吐量**：1,000,000 msg/sec
- **去重的key数量**：500,000,000 （在窗口里按key聚合4个long数值）
- **Checkpointing**：一分钟一次

![假定的硬件配置](/粗略计算Flink集群的容量/hypothetical_hardware_setup.png)

五台机器用于运行作业，每个机器运行一个Flink TaskManager（Flink的工作节点）。磁盘是网络附属（通常在云上），主交换机利用10吉比特以太网与每台TM连接。Kafka broker运行在独立的集群上。

每个机器有16核CPU。简化点的话，先不考虑CPU和内存。但真实情况中，基于你的应用逻辑和所使用的state backend，你需要关注内存。本例中使用基于RocksDB的state backend，鲁棒性比较强且对内存有较低需求。

## 单机视角

为了简单点理解作业部署所需资源，先聚焦于单机单TM。你可以利用单机产出的数字计算全部资源所需量。

默认情况下，如果所有算子有着相同的并行度且没有特殊调度限制，那么一个流作业的所有算子将会运行在每个机器上。

本例中，Kafka source（消息消费者），窗口算子，Kafka sink（消息生产者），5台机器中每台机器上都包含这些算子。

![单机视角](/粗略计算Flink集群的容量/operators_on_tm.png)

现在自上而下过一遍每个算子的网络资源需求量。

## Kafka Source

计算单个Kafka source接收的数据量，首先看一下Kafka输入的聚合情况。Sources接收1,000,000 msg/sec，每个消息2KB。

```
2KB × 1,000,000/s = 2GB/s
```

2GB/s除以机器数得到：

```
2GB/s ÷ 5 machines = 400MB/s
```

集群中5个Kafka source，每个接收数据的吞吐为400MB/s。

![Kafka source的计算](/粗略计算Flink集群的容量/kafka_source_calc.png)

## Shuffle / keyBy

接下来，需要确保相同key（userId）的事件在同一台机器上。从Kafka topic读取的数据可能是按另一个不同的字段进行的分区。

Shuffle进程将相同key的数据发送至同一台机器，所以相当于将来自Kafka的400MB/s的数据流拆分成按userId分区的数据流：

```
400MB/s ÷ 5 machines = 80MB/s
```

平均下来，会按80MB/s发送数据到每台机器上。这个分析是从单机视角出发，即有1/5的数据已经在选定的机器上了，所以减去80MB/s：

```
400MB/s - 80MB/s = 320MB/s
```

每个机器按320MB/s的吞吐接发数据。

![Shuffle的计算](/粗略计算Flink集群的容量/shuffle_calc.png)

## Window Emit 和 Kafka Sink

下一个问题是计算窗口算子和Kafka sink发送的数据量。先说答案是67MB/s，接下来解释这个数字是怎么产生的。

窗口算子每个key聚合4个long型值，每分钟一次，算子会发送当前聚合后的数值。每个key发送2个int型值`(user_id, window_ts)`和4个long型聚合值：

```
(2 × 4 bytes) + (4 × 8 bytes) = 40 bytes per key
```

500,000,000个单独的key被分配到各机器上，假设key被均分到各机器上，即除以5：

```
500,000,000 keys ÷ 5 = 100,000,000 keys
```

```
100,000,000 keys × 40 bytes ≈ 4GB
```

接着计算每秒数据量：

```
4GB/min ÷ 60 = 67MB/s
```

这意味着每个TaskManager上窗口算子发送用户数据的吞吐约67MB/s。因为每个TaskManager上紧接着窗口算子有一个Kafka sink，也不会再发生重新分区了，所以这个速率即就是Flink到Kafka的数值。

![用户数据经Kafka发出，shuffle到window算子，输出到Kafka](/粗略计算Flink集群的容量/window_to_kafka.png)

从窗口算子发出的数据是"间歇性的"，因为它们每隔一分钟才发出一次数据。实际生产中，算子不会一直以恒定速率67MB/s发出数据，而是每分钟内有那么几秒钟会最大限度地利用带宽发送数据。

计算一下每个机器输入/输出的吞吐总值：

- 输入：每台机器720MB/s   （`400 + 320`）
- 输出：每台机器387MB/s   （`320 + 67`）

![第一次汇总输入/输出的吞吐总值](/粗略计算Flink集群的容量/total_in_out.png)

## State Access 和 Checkpointing

截至目前，只看了Flink处理用户数据的数字。还需要考虑访问RocksDB用于存储状态和Checkpointing的磁盘负载。为了理解磁盘访问的成本，来看看窗口算子是怎么访问状态的。Kafka source也会保存一些状态，但与窗口算子对比起来是很微不足道的。

从一个不同的角度看可以理解窗口算子的大小。Flink计算一个5分钟大小的窗口，滑动步长是1分钟。Flink实现这个滑动窗口会同时维持5个窗口，每个"slide"一个。如之前提到的，维持了每个窗口下的每个key状态40bytes，做eager方式的窗口聚合实现。对于每个到来的事件，首先从磁盘读取聚合值（读40 bytes），更新这个聚合值，然后写回到磁盘（写40 bytes）。

![窗口状态](/粗略计算Flink集群的容量/window_state.png)

这意味着：

```
40 bytes of state × 5 windows × 200，000 msg/s per machine = 40MB/s
```

每个机器按该速率读写磁盘。前面提到，磁盘是网络附属存储器，所以把这个数字加到总吞吐中：

- 输入：760MB/s   （`400MB/s data in + 320MB/s shuffle + 40MB/s state`）
- 输出：427MB/s   （`320MB/s shuffle + 67MB/s data out + 40MB/s state`）

![第二次汇总输入/输出的吞吐总值](/粗略计算Flink集群的容量/total_in_out_2.png)

以上我们讨论了状态存取，当有新事件到达窗口算子时就会发生。另外需要开启Checkpointing进行容错，帮助在机器或作业出错后可以恢复窗口的内容继续处理。

Checkpointing每隔一分钟制作一次，并拷贝整个作业的状态到NAS上。

来看下单个机器上整个状态的大小：

```
40 bytes of state × 5 windows × 100,000,000 keys = 20GB
```

按秒算一个速率：

```
20GB ÷ 60min = 333MB/s
```

和窗口算子类似。Checkpointing也是一个"间歇性"的动作，一分钟触发一次，将状态数据发至外部存储器。Checkpointing额外为RocksDB引入状态存储的开销，需要访问NAS。从Flink 1.3开始RocksDB支持增量Checkpointing，减小网络上传输的checkpoint大小，只需要最近一次checkpoint不同的增量部分，但在本例中假定不用这个特性。

再来算一下总吞吐：

- 输入：760MB/s   （`400 + 320 + 40`）
- 输出：760MB/s   （`320 + 67 + 40 + 333`）

![第三次汇总输入/输出的吞吐总值](/粗略计算Flink集群的容量/total_in_out_3.png)

这意味着总网络流量是：

```
(760 + 760) × 5 + 400 + 2335 = 10335 (MB/s)
```

400表示5台机器按80MB/s进行状态存取，2335表示5台机器对Kafka消费（400MB/s）和生产（67MB/s）的数据量。

只用了网络容量的一半多一些。

![所需网络容量](/粗略计算Flink集群的容量/network_requirements.png)

有一点需要说明的是，以上所有的计算都没包含协议本身的开销，比如TCP，以太网，Flink、Kafka、文件系统之间的RPC调用。当然上面的工作仍然是一个好的起点来帮助你了解需要什么样的硬件资源并可以清楚一些性能指标。

# 扩容你的集群

基于以上示例的分析，一个5节点集群，一些常见的算子，每个机器需要分别处理760MB/s的输入/输出数据，另外网络总容量是1250MB/s，还预留了约40%的网络容量剩余来帮助我们解决前面示例掩盖的复杂性，比如网络协议开销，或者作业恢复时进行数据追溯时的负载，又或者由数据倾斜带来的集群负载不均衡。

40%当然不是一个标准的余量指标，但以上讨论仍然是一个好的容量估算算法的起点。尝试上面的计算，替换机器数量、key的数量、每秒消息数、当然还要考虑你的资源预算和运营因素，从而选择合适的估算值。现在，扩容你的集群吧。
