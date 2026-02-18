---
title: "Flink的TM内存计算逻辑"
date: 2023-08-06T17:32:21+08:00
lastmod: 2023-08-06T22:27:10+08:00
keywords: ["flink"]
categories: ["flink"]
tags: ["flink"]
author: "小十一狼"
---

# TM 内存模型

![Flink TM内存模型](/Flink的TM内存计算逻辑/tm_memory.png)

# 描述 TM 内存配置的类

![描述 TM 或 JM 的 JVM 进程内存配置的公共配置类](/Flink的TM内存计算逻辑/ProcessMemoryOptions.png)

由上图可知，JM 也是使用该类来描述内存配置。

在 TaskExecutorProcessUtils 中定义了 TM 的内存配置实例：

```java
public class TaskExecutorProcessUtils {

    static final ProcessMemoryOptions TM_PROCESS_MEMORY_OPTIONS =
            new ProcessMemoryOptions(
                    Arrays.asList(
                            TaskManagerOptions.TASK_HEAP_MEMORY,
                            TaskManagerOptions.MANAGED_MEMORY_SIZE),
                    TaskManagerOptions.TOTAL_FLINK_MEMORY,
                    TaskManagerOptions.TOTAL_PROCESS_MEMORY,
                    new JvmMetaspaceAndOverheadOptions(
                            TaskManagerOptions.JVM_METASPACE,
                            TaskManagerOptions.JVM_OVERHEAD_MIN,
                            TaskManagerOptions.JVM_OVERHEAD_MAX,
                            TaskManagerOptions.JVM_OVERHEAD_FRACTION));

	... ...
}
```

可以看出：

- requiredFineGrainedOptions 对应 Task Heap（`taskmanager.memory.task.heap.size`）和 Managed Memory（`taskmanager.memory.managed.size`）
- totalFlinkMemoryOption 即 Total Flink Memory（`taskmanager.memory.flink.size`）
- totalProcessMemoryOption 即 Total Process Memory（`taskmanager.memory.process.size`）
- jvmOptions 即 JVM Metaspace（`taskmanager.memory.jvm-metaspace.size`） 和 JVM Overhead（`taskmanager.memory.jvm-overhead.min`、`taskmanager.memory.jvm-overhead.max`、`taskmanager.memory.memory.jvm-overhead.fraction`）

# TM 内存计算逻辑所在方法

ProcessMemoryUtils#memoryProcessSpecFromConfig(Configuration config)

处理逻辑优先级如下：

![TM 内存计算逻辑梳理](/Flink的TM内存计算逻辑/ProcessMemoryUtilsmemoryProcessSpecFromConfig.png)
