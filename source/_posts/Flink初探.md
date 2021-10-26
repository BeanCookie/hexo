---
title: Flink初探
date: 2020-06-14 17:36:26
categories:
- 实时计算
tags: 
- Flink
---

## 批处理与流计算

- #### SQL时代

  > 从数据工程师的角度看待问题，软件系统核心价值在于数据，系统中的一系列功能业务都是围绕着如何对数据进行这种关联和聚合操作，最终总结出有价值的数据展现给用户。在RDBMS时期一系列围绕着SQL操作构建了最初的批处理时代。

- #### MapReduce时代

  > 随着信息时代的到来，人类产生的信息数据不断呈指数级增长，RDBMS已经越来越难以满足大量的非结构化数据的存储要求，如何实现一个可靠的分布式文件系统成为当务之急，以Google GFS为核心实现的各种分布式文件系统成为现代数据仓库的标配，有老牌的Hadoop体系全家桶，也有Mongo为代表的各种NoSQL。有了可靠的分布式文件系统之后如何进行数据计算呢，依旧是Google提出的MapReduce成为了大数据蛮荒时代的第一束光，随之而来的还有大数据时代中一个很重要的思想**移动计算比移动数据更划算**，这个思想可以简单的理解为*我们会把编写的程序分发到数据的各个节点，然后经过一系列的操作再将计算结果汇总到某个主节点*。Spark也是在这个思想的基础上使用DAG模型简化了MapReduce的操作，使用迭代式的内存计算优化了计算性能。Mongo作为NoSQL的杰出代表也提供了pipeline从而实现了更强的聚合能力。

- #### 流处理时代

  > 业务需求是永无止境的，或者是业务需求推动了计算进步
  >
  > 1. 当淘宝上某个店家上架了某款商品后期望在三分钟内就可以在搜索框中检索出最新的商品时
  > 2. 当滴滴用户想在五秒钟之内就可以搜索到附件的车辆时
  > 3. 当蚂蚁金服风控团队想把异常交易的响应时间缩短到秒级时
  > 4. 当个个电商想在活动促销时实时展现各种销售数据时
  > 5. 当智鹤的产品团队想展现各类机械的实时运转情况，想将各类报表的实时性提高到分钟级别时

## Stream与DAG

- ### 编程语言中的容器

  > 任何一门编程语言都会将容器融入到自身的血脉之中，例如在我们学习一门新的编程语言时当了解我if else for之后大概率就会接着学习数组、字典之类的内置数据容器。最初大多数编程语言只是封装了一些通用数据结构例如可变长数组、Set、Map等，但是随着时间的推移大家发现单纯的数据结构难以解决稍微复杂一点的数据统计问题。纵观多数编程语言以及很多函数式编程的概念，大家都异曲同工的选择了一组Stream风格的API来进行复杂的数据统计操作。

  - #### Filter

    > 数据过滤操作，对应SQL中的Where条件

<img src="https://beancookie.github.io/images/Flink初探-04.png" alt="Flink初探-04" style="zoom: 80%;" />

  - #### Map

    > 数据转换操作，对应SQL中的Select

<img src="https://beancookie.github.io/images/Flink初探-03.png" alt="Flink初探-03" style="zoom:80%;" />

  - #### FlatMap

    > 获取一个元素然后产生0个、1个或多个输出

<img src="https://beancookie.github.io/images/Flink初探-02.png" alt="Flink初探-02" style="zoom: 50%;" />

  - #### Average

    > 获取一组数据平均数的聚合操作

<img src="https://beancookie.github.io/images/Flink初探-06.jpg" alt="Flink初探-06" style="zoom:80%;" />

  - #### Reduce

    > 这是一种聚合操作的抽象表达方式，可以通过**Reduce**实现**Sum**、**Average**、**Min**和**Max**等聚合操作

<img src="https://beancookie.github.io/images/Flink初探-05.jpg" alt="Flink初探-05" style="zoom:80%;" />

- ### 并行计算中的操作单元

> 随着数据量的不断增长，当到了单机内存无法无法存储的时候就需要对Stream的模型进行改造。首先在逻辑上会采用DAG（Directed Acyclic Graph、有向无环图），一般情况下数据计算引擎会采用数据驱动的方式，它会提前设置一些算子，然后等到数据到达后对数据进行处理，为了表达复杂的计算逻辑，包括 Flink 在内的分布式流处理引擎一般采用 DAG 图来表示整个计算逻辑，其中 DAG 图中的每一个点就代表一个基本的逻辑单元，也就是前面说的算子。由于计算逻辑被组织成有向图，数据会按照边的方向，从一些特殊的 Source 节点流入系统，然后通过网络传输、本地传输等不同的数据传输方式在算子之间进行发送和处理，最后会通过另外一些特殊的 Sink 节点将计算结果发送到某个外部系统或数据库中。计算引擎内部的物理结构通常会更复杂，DAG中的每个算子都可以是多个物理节点。

  

![Flink初探-07](https://beancookie.github.io/images/Flink初探-07.png)

## 把大象装进冰箱

> 上文中提到的数据驱动方式，数据处理过程可以抽象为数据从Source然后经过一系列的计算最后通过Sink输出到外部系统。在这个前提下大多数的带有简单分析聚合的ETL系统，都可以把分析工作分为三步，从数据源获取原始数据，经过计算引擎的分析处理，最后输出到外部系统。

<img src="https://beancookie.github.io/images/Flink初探-08.png" alt="Flink初探-08" style="zoom:67%;" />

![Flink初探-09](https://beancookie.github.io/images/Flink初探-09.jpg)

## 三个时间

> 分布式环境中 Time 是一个很重要的概念，Event-Time 表示事件发生的时间，Processing-Time 则表示处理消息的时间，Ingestion-Time 表示进入到系统的时间。

![Flink初探-10](https://beancookie.github.io/images/Flink初探-10.jpg)



## 时间窗口

> 真实世界中的大多数数据都是已流的形式输出的，例如终端上传的数据、用户操作日志和电商交易记录等。所以真实的数据通常是没有边界的，在批处理时代通常会将无线的数据人为的划分成多个有边界的数据然后进行离线处理。

<img src="https://beancookie.github.io/images/Flink初探-13.png" alt="Flink初探-13" style="zoom:67%;" />

> 因为大多数的业务需求都是要对一段时间的数据进行统计，比如多种时间维度的工时日报、传感器的日增长量和机械开工率变化曲线等。流计算中入了Window的概念，这里的Window可以是按时间间隔定义的，也可以是按事件数量定义的。Window的移动方式又可以分为滚动窗口和滑动窗口。

![Flink初探-12](https://beancookie.github.io/images/Flink初探-12.jpg)

> 如何处理迟到的时间，因为多种原因数据源产生的数据可能存在延时的情况，这样就会导致当前的Window可能接收到上个Window的数据，但是此时前一个Window已经触发的计算操作，这样就会导致Window数据丢失。这种情况下流计算引入了水位线的概念，设置了水位线之后呢可以延长窗口计算的时间，从而保障数据完整性，当然数据完整性和实时性很多情况下不能兼得。

![Flink初探-11](https://beancookie.github.io/images/Flink初探-11.jpg)

## 无状态与有状态

- #### 无状态

  > 大多数CURD的程序，其程序本身没有状态，每次接口调用都是从外部数据库获取状态。在这种业务场景中我们并没有觉得这有什么不妥，因为这种场景下的核心业务就是围绕着外部状态展开的。

- #### 有状态

  > 但是在实时流计算中很多窗口数据都需要长期保持或者定期更新，这种情况下将状态数据保持在外部数据库诸如Redis之类，虽然一定程度上可以保证实时性，但是手动管理大量的外部数据对于开发人员将会是很大的负担。另一方面流计算是必须保证7*24不间断运行的，任何的版本上线和bug修复都不容许数据丢失。此时我们再看Flink的官方定义 *Apache Flink is a framework and distributed processing engine for stateful computations over unbounded and bounded data streams.* *Apache Flink是一个框架和分布式处理引擎，用于对无限制和有限制的数据流进行有状态的计算。*回顾Flink的诸多特性安全可靠的内部状态管理机制绝对是Flink对标其他竞品时的杀手锏。