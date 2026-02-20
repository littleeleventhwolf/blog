---
title: '关于优化Flink状态恢复和制作的一些工作'
date: '2026-02-20T13:45:55+08:00'
lastmod: '2026-02-20T13:45:55+08:00'
keywords: ['flink']
categories: ['flink']
tags: ['flink']
author: '小十一狼'
---

过去一年，我们针对 Flink 状态的恢复和制作进行了一些优化工作，内部目前主要维护版本是 Flink 1.16.1，其中包括社区优秀工作回迁到 Flink 1.16.1 以及我们在公司内部版本上开发的一些特性。

我会结合简要的描述和图片展示，快速体现这部分工作的内容和结果。

# 优化 rescale 恢复

引入 Ingest 恢复模式替换 Iterator 模式，将数据行级别的遍历与拷贝转变为文件级别的硬链接，提升 RocksDB 重建速度。具体参见：[FLINK-31238](https://issues.apache.org/jira/browse/FLINK-31238)、[FLINK-35581](https://issues.apache.org/jira/browse/FLINK-35581)、[FLINK-35580](https://issues.apache.org/jira/browse/FLINK-35580)。Ingest 恢复模式的主要思路如下：

![Ingest恢复模式](/关于优化Flink状态恢复和制作的一些工作/1-ingest.png)

实际测试效果：单并发 100GB 状态重建从 17min± 降至 40s±，单并发 500GB 状态重建从 120min± 降至 4min±。

# 优化 non-rescale 下载

提出 LazyLoad 能力，不等数据文件全部下载到本地即可快速恢复 RocksDB，运行阶段并行执行 Download 与 Read/Write，互相利用对方等待数据/指令的空闲时间，充分利用资源提升效率。

这个优化尤其对状态中冷热分明的访问友好，初期可以优先下载热数据（一般就是 L0/L1 层数据），不阻碍 Task 运行的访问，然后再异步后台下载冷数据。整体方案如下：

![引入 HDFS 远程访问](/关于优化Flink状态恢复和制作的一些工作/2-lazyload-1.png)

借助 RocksDB 中 Env/FileSystem 暴露的接口，可以实现 HdfsEnv 用于访问远程 HDFS 上的文件，并结合 HdfsEnv 和 PosixEnv 得到 UnionEnv，在 UnionEnv 内部决定是访问远程、或者是先下载到本地后再访问。我们称这个功能为 LazyLoad。实现思路如下：

![LazyLoad 实现逻辑](/关于优化Flink状态恢复和制作的一些工作/2-lazyload-2.png)

实际测试效果：追数 5 百万时间减少 10min±；单并发 455 GB 状态，restore 时间从 20min± 降至 2s。

# 结合 ingest 和 lazyload

结合 Ingest 和 LazyLoad，主要是解决 Link/Rename/Delete 在 LazyLoad 下的工作正确性。因为 Ingest 能力依赖这三个文件操作。

在 UnionEnv 中处理这三个文件操作时，若文件在远程，则通过维护内存中的元数据来表达相应操作即可，直到文件下载到本地，才开始真正的执行文件系统上的 Link/Rename/Delete。实现思路如下：

![处理link-rename-delete](/关于优化Flink状态恢复和制作的一些工作/3-lazyload-ingest.png)

实际测试效果：追数 5 百万时间减少 100min±；单并发 455 GB 状态，restore 时间从 100min± 降至 5min±。

# 优化第一次 checkpoint 全量上传问题

选择增量 checkpoint 制作模式，在作业重启后的第一次状态上传过程中，仍然是全量上传。为解决这个问题，引入 FastCopy，通过远程 DataNodes 在 favorite nodes 上并行高效数据块级别的 hardlink 来减少 Flink 资源消耗以及网络传输。实现思路如下：

![hdfs fastcopy 状态文件](/关于优化Flink状态恢复和制作的一些工作/4-fastcopy.png)

注意在实现上我们让 HadoopFileSystem 实现了 DuplicatingFileSystem，因为我们使用的是 Flink 1.16.1，但是发现高版本中已经将 DuplicatingFileSystem 移除掉了，不过这不影响开发思路的实现。

实际测试效果：总 1.83 TB 的状态量，单 Task 468GB、 8000+ 文件，Checkpoint 耗时从 23min± 降低至 1min±。

# 解决极端场景因无法 compact 导致状态累积变大

在使用 Window 算子时，如果 Window Size 远小于 Checkpoint Interval，此时会出现很多小文件，这些小文件里存的多是 timer state，且由于这些文件的 key 和其他文件不产生交集，所以 compact 也无法真正意义地去删除或合并其中的 key，这在 RocksDB 的官方文档里也有提到，称之为 Trivial Move。

比如，通过构造 TumblingProcessingTimeWindow，让 KeyedStream 一直维持仅一个 key，Checkpointing Interval 为 20s，Window Size 为 100ms，这时 RocksDB 中会逐步累积很多文件：

![small-window-rocksdb-files](/关于优化Flink状态恢复和制作的一些工作/small-window-rocksdb-files.png)

文件数量持续增长。都是随着时间推移，各 sst 文件间不互相相交，触发 compaction 也只会被移至下一层，并不会去除墓碑来减小体积，另外因为 Time Window 的访问性质，这些 sst 文件也不会再被访问到。因此无论是 size compaction 亦或是 seek compaction 都无法达到真正的文件合并效果：

![small-window-rocksdb-file-list](/关于优化Flink状态恢复和制作的一些工作/small-window-file-list.png)

查看文件内容全是 deletion 墓碑数据（type=0 表示 kTypeDeletion）：

![deletion-type-content](/关于优化Flink状态恢复和制作的一些工作/deletion_type_content.png)

追踪 WindowOperator 出现 delete 的调用链路：

![arthas-trace-window-operator-1](/关于优化Flink状态恢复和制作的一些工作/arthas-trace-window-operator-1.png)

![arthas-trace-window-operator-2](/关于优化Flink状态恢复和制作的一些工作/arthas-trace-window-operator-2.png)

可以发现是 Processing Timer 最终清理产生的 RocksDB deletion。

再回顾一下，Trivial Move 发生在 Compact 时没有与其他文件 overlapping 的情况，如下图的文件 2，直接移至下一层，目的是减少读写放大。

![trivial-move](/关于优化Flink状态恢复和制作的一些工作/5-trivial-move.png)

但过多的 Trivial Move 会带来以下危害：

- 产生大量未经 compact 的小文件；

- 这些文件均无法执行 compaction filter，在 Flink 中导致的结果就是无法 ttl 过期。

通过引入定期 Manual Compact 解决这些文件无法真正合并的问题，这个功能来自社区 [FLINK-26050](https://issues.apache.org/jira/browse/FLINK-26050)，逻辑如下：

![manual-compact](/关于优化Flink状态恢复和制作的一些工作/6-manual-compact.png)

等待上面例子中复现到积累文件数为 1w+ 时：

![trivial-move-1w-files](/关于优化Flink状态恢复和制作的一些工作/trivial-move-1w-files.png)

我们通过设置 state.backend.rocksdb.manual-compaction.min-interval:3min 开启 manual compaction，再观察文件情况：

![manual-compact-decrease-num-of-files](/关于优化Flink状态恢复和制作的一些工作/manual-compact-decrease-num-of-files.png)

可以观察到文件量在逐步减少。最后文件数只剩下个位数。

从火焰图上可以看到 manual compact 在触发执行：

![flamegraph-of-manual-compaction](/关于优化Flink状态恢复和制作的一些工作/flamegraph-of-manual-compaction.png)

从 RocksDB 日志中也能发现是 processing_window-timers 的 sst 文件被 manual compact，既说明了是 Processing Timer 产生的墓碑，也说明 Manual Compaction 起效的结果：

![rocksdb-log-of-manual-compaction](/关于优化Flink状态恢复和制作的一些工作/rocksdb-log-of-manual-compaction.png)
