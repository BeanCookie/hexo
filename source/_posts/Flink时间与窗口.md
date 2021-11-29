---
title: Flink时间与窗口
date: 2021-11-29 10:24:34
categories:
- Flink
tags: 
- Flink
---

### 流处理中的时间

在流处理中通常会包含**处理时间**与**事件时间**两种时间概念

- **处理时间**

  模拟我们真实世界的时间，其实就算是处理数据的节点本地时间也不一定就是完完全全的我们真实世界的时间，所以说它是用来模拟真实世界的时间。

- **事件时间**

  事件时间是每个单独事件在其产生设备上发生的时间。这个时间通常在记录进入Flink之前嵌入在记录中，并且可以从每条记录中提取该*事件时间戳*。

判断应该使用 Processing Time 还是 Event Time 的时候，可以遵循一个原则：

当你的应用遇到某些问题要从上一个 checkpoint 或者 savepoint 进行重放，是不是希望结果完全相同。如果希望结果完全相同，就只能用 Event Time。如果接受结果不同，则可以用 Processing Time。

Processing Time 的一个常见的用途是，我们要根据现实时间来统计整个系统的吞吐，比如要计算现实时间一个小时处理了多少条数据，这种情况只能使用 Processing Time。

### 时间的特性

**时间的一个重要特性是：时间只能递增，不会来回穿越。** 在使用时间的时候我们要充分利用这个特性。假设我们有这么一些记录，然后我们来分别看一下 Processing Time 还有 Event Time 对于时间的处理。

![https://beancookie.github.io/images/Flink时间与窗口-01.png](https://beancookie.github.io/images/Flink时间与窗口-01.png)

- 对于 Processing Time，因为我们是使用的是本地节点的时间，我们每一次取到的 Processing Time 肯定都是递增的，递增就代表着有序，所以说我们相当于拿到的是一个有序的数据流。
- 而在用 Event Time 的时候因为时间是绑定在每一条的记录上的，由于网络延迟、程序内部逻辑、或者其他一些分布式系统的原因，数据的时间可能会存在一定程度的乱序，比如上图的例子。在 Event Time 场景下，我们把每一个记录所包含的时间称作 Record Timestamp。如果 Record Timestamp 所得到的时间序列存在乱序，我们就需要去处理这种情况。

### 生成watermark
如果单条数据之间是乱序，我们就考虑对于整个序列进行更大程度的离散化。简单地讲，就是把数据按照一定的条数组成一些小批次，但这里的小批次并不是攒够多少条就要去处理，而是为了对他们进行时间上的划分。经过这种更高层次的离散化之后，我们会发现最右边方框里的时间就是一定会小于中间方框里的时间，中间框里的时间也一定会小于最左边方框里的时间。
![https://beancookie.github.io/images/Flink时间与窗口-02.png](https://beancookie.github.io/images/Flink时间与窗口-02.png)

这个时候我们在整个时间序列里插入一些类似于标志位的一些特殊的处理数据，这些特殊的处理数据叫做 watermark。一个 watermark 本质上就代表了这个 watermark 所包含的 timestamp 数值，表示以后到来的数据已经再也没有小于或等于这个时间的了。
![https://beancookie.github.io/images/Flink时间与窗口-03.png](https://beancookie.github.io/images/Flink时间与窗口-03.png)

