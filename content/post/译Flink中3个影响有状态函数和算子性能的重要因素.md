---
title: "[译]Flink中3个影响有状态函数和算子性能的重要因素"
date: 2022-05-21T17:28:50+08:00
lastmod: 2022-05-21T17:28:50+08:00
keywords: ["flink"]
categories: ["flink"]
tags: ["flink"]
author: "小十一狼"
---

原文：[3 important performance factors for stateful functions and operators in Flink](https://www.ververica.com/blog/performance-factors-stateful-functions-operators-flink)

这篇博客关注在状态流应用中，当使用Flink的Keyed State时，3个影响有状态函数或算子访问性能的因素。

Keyed State是Flink两类状态之一，另一类是Operator State。见名知意，Keyed State绑定到了key上，并且只当在函数和算子中处理`KeyedStream`时可访问。Operator State和Keyed State的区别在于，Operator State作用于算子并行的每一个实例上（sub-task），而Keyed State是按key进行状态分区的。

想了解更多关于State的信息请前往"[Working with State](https://ci.apache.org/projects/flink/flink-docs-stable/dev/stream/state/state.html)"，其中描述了如何使用State开发流应用。

# 3个影响Keyed State访问性能的因素

下面逐一讨论这3个影响Keyed State访问性能的因素：

## 所选的State Backend

[State Backend](https://www.ververica.com/blog/stateful-stream-processing-apache-flink-state-backends)的选择对有状态函数或算子的性能影响特别明显。最独特的原因便是每个State Backend如何以不同方式处理状态序列化以实现持久性。

例如，当使用`FsStateBackend`或`MemoryStateBackend`，本地状态在运行时是以堆上对象的形式进行维护，因此访问或更改状态的开销较低。序列化开销仅在生成状态快照以创建checkpoint和savepoint时发生。使用此类State Backend的缺点是状态大小受限于JVM的堆大小，并且可能会发生`OutOfMemory`错误或垃圾回收时的长暂停。

相反，`RocksDBStateBackend`因为将本地状态放在磁盘，所以允许存储更大的状态。当然，这带来的代价是读写每个状态都需要序列化/反序列化。

总结一下，如果状态很小，且预计不会超过堆大小，则使用堆上的State Backend是很明显的选择，这避免了序列化开销。要不然，`RocksDBStateBackend`则是具有大状态应用的首选。

## Key对应的状态原语

### `ValueState` / `ListState` / `MapState`

另一个重要的因素是选择正确的状态原语。Flink当前支持3个主要的Keyed State原语：`ValueState`、`ListState`、`MapState`。

新的开发者常犯的错误是使用`ValueState<Map<String, Integer>>`来存取map型的状态，这样map中的数据项只能被随机访问。在这种场景中，更好的方式是使用`MapState<String, Integer>`，尤其是考虑到使用`RocksDBStateBackend`时，`ValueState`的形式会导致key下的状态整个被序列化/反序列化，而`MapState`则只需要序列化key下的状态中的某一项。

## 访问模式

上一小节讨论了状态原语的内容，其实也彻底说明了需要考虑清楚应用程序对状态的访问模式，这将帮助你选择正确的数据结构。开发者应该预想到，当设计应用时若选择了不适合的数据结构，那么访问特定数据时将会产生很严重的性能影响。

# 总结

开发者应该谨记以上三个因素，因为它们在很大程度上影响了有状态函数和算子的访问性能。
