---
title: '从DSec的LS/BE分类，学习SCHED_IDLE和Core Scheduling'
date: '2026-10-03T14:09:26+08:00'
lastmod: '2026-10-03T14:09:26+08:00'
keywords: ['linux']
categories: ['linux']
tags: ['linux']
author: '小十一狼'
---

DSec 面对的是大规模 Agent 沙盒，沙盒在等待 LLM 生成下一步动作时，CPU 使用率很低，所以可以进行资源超卖（在一台物理机上跑大量沙盒实例，并且分配出去的虚拟资源总和超过物理资源上限）。

原文如下：

> **Sandboxes must run at high density.** During agent interaction, a sandbox often waits for the LLM to generate the next action, so CPU usage is sparse and naturally suitable for overcommit.

解读：Agent 沙盒大部分时间在等 LLM 返回，CPU 实际占用很稀疏，所以可以在一台机器上挤很多个沙盒——这就是“超卖”的前提。

但高密度意味着不同沙盒会共享同一个节点、同一物理核、同一 SMT 线程。于是出现两类问题：

1. 低优先级任务抢占 CPU 时间；
2. 低优先级任务和高优先级任务若分配在同一物理核上的两个 SMT 线程上将产生资源争用。

论文中把沙盒分为 latency-sensitive（**LS**，延迟敏感）和 best-effort（**BE**，尽力而为）两类，再用 **SCHED_IDLE + Core Scheduling** 做两层隔离，来达到尽量减少这种抢占/争用的问题。

原文如下：

> **QoS-aware CPU scheduling.** To eliminate SMT-level CPU interference, DSec classifies sandboxes into latency-sensitive (LS) and best-effort (BE) classes.

解读：DSec 是按“对延迟的容忍度”分类。LS 沙盒每步延迟必须可控（比如正在和用户交互的 Agent），BE 沙盒可以慢慢跑（比如后台数据处理）。

> BE sandboxes are placed under `SCHED_IDLE` so that they yield the CPU whenever an LS task is runnable.

解读：BE 沙盒被设为 SCHED_IDLE，意思是“只要系统里有 LS 任务想跑，BE 就立刻让出 CPU”。

> Because scheduler priority alone does not prevent interference between sibling hardware threads, we also enable Linux core scheduling ([Zijlstra et al., 2021](https://docs.kernel.org/admin-guide/hw-vuln/core-scheduling.html)) for LS sandboxes, preventing unrelated BE work from running on the sibling thread of the same physical core.

解读：这是比较关键的一句，描述了两件事：

1. **光靠调度优先级不够**——因为 SMT 兄弟线程上的干扰不是“谁先被调度”的问题，而是“两个任务同时跑在同一个物理核上”的问题；
2. **Core Scheduling 补上了这个缺口**——它阻止 BE 任务跑到 LS 任务所在物理核的另一个 SMT 线程上。

> This two-layer policy preserves LS per-step latency budgets while still allowing BE tasks to use idle cycles, reducing SMT-induced latency inflation from 45.2% to 17.3%, as detailed in §8.5.

解读：两层策略的效果是——LS 沙盒的每步延迟预算保住了，BE 沙盒仍然能在系统空闲时利用 CPU 周期。SMT 导致的延迟膨胀从 45.2% 降到了 17.3%。

实现方式上，论文也明确说没有改内核：

> For CPU QoS, we set best-effort tasks to `SCHED_IDLE` and enable core scheduling via `prctl(PR_SCHED_CORE)` to group tasks by QoS class. No kernel modifications are required; the implementation consists entirely of configuration and integration with our sandbox orchestrator.

解读：DSec 没有修改内核代码，只是通过配置+沙盒编排来实现的。这意味着这套方案可以直接用在现有 Linux 内核上，不需要打补丁。

# SCHED_IDLE：让 BE 任务“有空才跑”

SCHED_IDLE 是 Linux CFS 里的最低优先级调度策略。它不是调高 nice 值来降低优先级，而是直接在调度类层面被放在最后。

SCHED_IDLE 和 SCHED_NORMAL 都在同一个调度类 `fair_sched_class` 里，SCHED_IDLE 不是“nice 值更低的 SCHED_NORMAL”，而是 CFS 里一种最低优先级调度策略。nice 值管的是 SCHED_NORMAL 内部的权重分配。

SCHED_IDLE 的行为是，只要系统里有 SCHED_NORMAL、SCHED_BATCH、实时任务、deadline 任务等可运行，SCHED_IDLE 就不该抢 CPU；只有这些任务都不可运行或不需要 CPU 时，它才获得调度机会。

## 设置方式

**chrt 命令**：

chrt 是 Linux 用于查看或修改进程的调度策略核优先级的命令行工具。

```bash
# 将指定进程设为 SCHED_IDLE 策略
# -i 表示 SCHED_IDLE
# -p 表示操作已有进程
chrt -i -p <pid>
```

```bash
# 查看某个进程当前的调度策略和优先级
chrt -p <pid>
```

**cgroup v2 接口**：

cgroup v2 提供了 cpu.idle 文件，写入 1 表示该 cgroup 下所有任务使用 SCHED_IDLE 语义。

```bash
# 将某个 cgroup 下的所有任务设为 idle 调度
echo 1 > /sys/fs/cgroup/be-group/cpu.idle
```

**C 代码调用 sched_setscheduler**：

```c
#include <sched.h>

struct sched_param param = { .sched_priority = 0 };
// 将当前进程（pid=0 表示自身）设为 SCHED_IDLE
// SCHED_IDLE 要求 sched_priority 必须为 0
sched_setscheduler(0, SCHED_IDLE, &param);
```

## SCHED_IDLE 约束

- sched_priority 必须为 0；
- nice 值对它无效；
- 解决的是“BE 任务在 CPU 就绪队列里抢 LS 任务”的问题；
- 不能解决 SMT 兄弟线程上的干扰；
- 适合“可以无限延后”的任务，不适合需要稳定吞吐的任务。

# 为什么 SCHED_IDLE 不够？因为同一物理核上的资源争用管不到

 SMT（Simultaneous Multithreading，Intel 称 Hyper Threading）让一个物理核心对外呈现两个逻辑 CPU。这两个逻辑核共享 L1/L2 缓存、执行单元、分支预测器、Load/Store 队列等资源。

 问题在于：

 - LS 任务跑在某个逻辑 CPU 上；
 - BE 任务跑在该逻辑 CPU 的 SMT 兄弟线程上；
 - 两者共享同一个物理核的硬件资源；
 - 即使 BE 是 SCHED_IDLE，它一旦获得 CPU，就会在硬件层面拖慢 LS。

所以论文说：

> Because scheduler priority alone does not prevent interference between sibling hardware threads, we also enable Linux core scheduling ...

解读：调度优先级只能决定“谁先被选到 CPU 上”，但一旦两个任务分别跑在同一个物理核的两个 SMT 线程上，它们就会共享执行资源——这是优先级机制管不到的层面。

# Core Scheduling：按 Cookie 做同核隔离

Core Scheduling 是 Linux 5.14 引入的机制，需要内核开启 CONFIG_SCHED_CORE。

内核会给任务分配一个内部 Core Scheduling Cookie，这是一个 64 位标识符，代表任务所属的调度信任域。以 LS 和 BE 为例，如果把所有 LS 任务设为同一个 Cookie，所有 BE 任务设为另一个 Cookie，那么 LS 和 BE 就不会同时运行在同一个物理核的 SMT 线程上：

- 同一个物理核上的所有 SMT 线程，同一时刻只能运行具有相同 Cookie 的任务；
- 如果某个 SMT 线程没有匹配 Cookie 的任务可跑，会强制进入空闲；
- 内核的空闲线程被视为全局可信，因此没有匹配 Cookie 的任务时，通常会回退到空闲线程。

## 设置方式

**prctl 接口**：

prctl 是 Linux 系统调用，用于对当前进程执行各种控制操作。PR_SCHED_CORE 是其中一个操作码，专门用于管理 core scheduling cookie：

```c
#include <sys/prctl.h>

/* 为当前进程创建一个新的 core scheduling cookie */
prctl(PR_SCHED_CORE, PR_SCHED_CORE_CREATE, 0, PIDTYPE_PID, 0);

/* 将当前进程的 cookie 共享给目标进程 */
prctl(PR_SCHED_CORE, PR_SCHED_CORE_SHARE_TO, target_pid, PIDTYPE_PID, 0);

/* 从目标进程拉取 cookie 到当前进程 */
prctl(PR_SCHED_CORE, PR_SCHED_CORE_SHARE_FROM, target_pid, PIDTYPE_PID, 0);
```

这三个操作的关系是——先 CREATE 创建一个新 cookie，然后用 SHARE_TO/SHARE_FROM 把其他进程加入同一个 cookie 组。同一组的进程同一时刻可以跑在同一物理核的 SMT 线程上。

第四个参数可以取 PIDTYPE_PID（按单个进程）或 PIDTYPE_TGID（按线程组，即整个进程的所有线程）。

**cgroup v2 接口**：

cgroup v2 下可通过 cpu.core_sched 控制。写入 1 表示该 cgroup 启用 core scheduling，内核会为这个 cgroup 分配一个独立的 cookie：

```bash
# 为 LS 组启用 core scheduling，内核分配 cookie A
echo 1 > /sys/fs/cgroup/ls-group/cpu.core_sched

# 为 BE 组启用 core scheduling，内核分配 cookie B
echo 1 > /sys/fs/cgroup/be-group/cpu.core_sched

# 结果：cookie A <> cookie B，两组任务不能在同一物理核的 SMT 线程上同时运行
```

cgroup 方式比 prctl 更简洁——每个 cgroup 自动获得独立 cookie，不需要手动管理共享关系。

## Core Scheduling 约束

- 需要内核启用 CONFIG_SCHED_CORE，通常 Linux 5.14+ 才支持；
- 只在开启 SMT 的机器上有意义；
- Cookie 不匹配时会触发 Forced Idle，可能降低整体吞吐；
- Core Scheduling 主要隔离 SMT 层面的干扰，不能解决内存带宽、睿频、L3 Cache 等跨核干扰。

# 回到 DSec 的两层 QoS

<table>
<tr>
<th>问题</th>
<th>机制</th>
<th>效果</th>
</tr>
<tr>
<td>BE 任务抢占 LS 任务的 CPU 时间</td>
<td>SCHED_IDLE</td>
<td>BE 在 LS 可运行时让出 CPU</td>
</tr>
<tr>
<td>BE 任务通过 SMT 干扰 LS</td>
<td>Core Scheduling</td>
<td>BE 和 LS 分配不同 core scheduling cookie，不共享同一物理核的 SMT 线程</td>
</tr>
<tr>
<td>剩余延迟</td>
<td>睿频、内存带宽、Last-Level Cache</td>
<td>论文未进一步做内存带宽隔离</td>
</tr>
</table>

论文中也有总结：

> The improvement grows as BE contention increases. These results validate the two-level CPU QoS design in §5.2: `SCHED_IDLE` prioritizes LS tasks, while core scheduling isolates them from BE work on sibling SMT threads.

BE 负载越高，两层策略的收益越明显。SCHED_IDLE 负责“优先级”，Core Scheduling 负责“SMT 隔离”，两者缺一不可。

