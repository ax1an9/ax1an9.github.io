---
title: G1 Collector
date: 2026-09-26 12:26:42
tags:
  - GC(garbage collection)
  - JVM(Java Virtual Machine)
categories:
  - Java
---
# 参考内容

1. https://tech.meituan.com/2016/09/23/g1.html
2. https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html

# 简介

GC 总是需要在三者之间取得平衡：

1. 吞吐量（工作线程完成任务的效率是否因为GC减少）
2. 暂停时间（GC在特定阶段需要触发STW，时间越少越好）
3. 内存开销（运行GC的额外内存开销）

G1即Garbage-First (G1) Garbage Collector，它每次优先选择“回收收益比较高”的老年代区域，能在可估计的时间内完成垃圾清理，保证较高的吞吐量。

> It attempts to meet garbage collection pause-time goals with high probability while achieving high throughput with little need for configuration.



```java
                Java Heap
                    │
          划分为大量固定大小 Region
                    │
       ┌────────────┴────────────┐
       │                         │
     Young                     Old					
 Eden + Survivor          Old + Humongous
       │                         │
       │                         │
       │              Concurrent Marking
       │                         │
       │              统计 Region 存活率
       │                         │
       │                  Mixed 候选集
       │                         │
       └───────────┬─────────────┘
                   │
                   ▼
              构造候选集合 CSet

Young GC:
CSet = 全部 Young Region

Mixed GC:
CSet = 全部 Young + 部分高收益 Old Region
```



# 关键概念

## Memory Layout

G1不把连续内存分配Young和Old区。而是分多个Region（区域化）。每个Region的属性表明自己是Eden、Survivor、Old、Humorous区域。

```java
传统 GC：
Young 连续区域 | Old 连续区域

G1：
[R][R][R][R][R][R][R]...
 E  O  S  O  E  H  O
```

# 分代GC模式

## 年轻代回收 Young GC

年轻代回收是高频的事件。年轻代回收仅需针对年轻代的局部可达性分析。可达性分析从常规的GC Root对象出发。因为不分析老年代引用的对象，所以通过区域的Remember Set（RSet）确认区域被哪些老年代区域引用。

> 基本假设：**老年代对象默认都是存活的**（除非发生老年代回收）。所以只要一个年轻代对象被老年代对象引用，它就通过 RSet 直接被视为可达，**不需要去追溯这个老年代对象本身是否被其他老年代对象引用**。

1. **选择回收集合 (Choose CSet)**
   G1 会根据用户设置的**最大 GC 暂停时间目标（`-XX:MaxGCPauseMillis`）**，预测并选择一个合适数量的年轻代 Region 组成回收集合（Collection Set, CSet）。
2. **根扫描 (Root Scanning)**
   从 GC Roots（如静态变量、线程栈中的局部变量等）出发，扫描并标记直接可达的对象。这些对象如果位于 CSet 中，会被立即复制到 Survivor 区或晋升到老年代，同时将其引用的其他对象加入标记栈。
3. **更新记忆集 (Update RSet)**
   确保 RSet 数据是最新且完整的，为后续扫描做准备。
4. **扫描记忆集 (Scan RSet)**
   将 RSet 作为根的一部分进行遍历。RSet 记录了**老年代对象对年轻代对象的引用**。通过扫描 RSet，G1 可以高效地找到那些被老年代引用的年轻代存活对象，而**无需扫描整个老年代**，从而大幅减少扫描开销。
5. **对象复制与晋升 (Evacuation/Object Copy)**
   遍历标记栈，将其中所有存活的对象**并行复制**到新的、空的 Survivor Region 中。在复制过程中，会根据对象的年龄和 Survivor 区的填充情况（如 `-XX:TargetSurvivorRatio`）决定：
   - **晋升 (Promotion)**：年龄超过阈值的对象，会直接**晋升到老年代**的 Region。
   - **复制到 Survivor**：未达到晋升年龄的对象，则被复制到新的 Survivor Region，其年龄加 1。
6. **收尾工作**
   完成复制后，被清理的 CSet 中的所有 Region 会被整体释放，并归还到空闲 Region 列表中，供后续分配使用。

## 混合回收 Mixed GC

当老年代占用率达到某个阈值——初始堆占用阈值（Initiating Heap Occupancy threshold）——时，仅年轻代阶段到空间回收阶段（Space-reclamation phase）的转换开始。此时，G1 会调度一次并发开始（Concurrent Start）年轻代收集，而不是普通年轻代收集。

1. **并发开始（Concurrent Start）：** 这种收集除了执行普通年轻代收集外，还会启动标记过程。并发标记确定老年代区域中当前所有可达（存活）对象，以便在后续空间回收阶段保留。在标记尚未完全完成期间，可能会发生普通年轻代收集。标记以两个特殊的STW暂停结束：重新标记（Remark）和清理（Cleanup）。
2. **重新标记（Remark）：** 完善标记结果。在重新标记和清理之间，G1 会计算信息，以便之后能够并发地回收选定的老年代区域中的可用空间；这将在清理暂停中最终完成。
3. **清理（Cleanup）：** 该暂停决定是否实际会进行空间回收阶段。如果随后要进行空间回收阶段，则仅年轻代阶段以一次准备混合（Prepare Mixed）年轻代收集结束。
4. **空间回收阶段（Space-reclamation phase）：** 该阶段由多次混合收集（Mixed collection）组成，会清理年轻代和老年代。
5. 作为后备，如果应用在收集存活信息时耗尽内存，G1 会像其他收集器一样执行一次 Full GC。