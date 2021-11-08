---
title: 运行Flink程序
date: 2021-11-08 11:15:34
categories:
- Flink
tags: 
- Flink
---

### 基本概念
运行 Flink 应用其实非常简单，但是在运行 Flink 应用之前，还是有必要了解 Flink 运行时的各个组件，因为这涉及到 Flink 应用的配置问题。图 1 所示，这是用户用 DataStream API 写的一个数据处理程序。可以看到，在一个 DAG 图中不能被 Chain 在一起的 Operator 会被分隔到不同的 Task 中，也就是说 Task 是 Flink 中资源调度的最小单位。

![https://beancookie.github.io/images/运行Flink程序-01.png](https://beancookie.github.io/images/运行Flink程序-01.png)

Flink 实际运行时包括两类进程
1. JobManager（又称为 JobMaster）：协调 Task 的分布式执行，包括调度 Task、协调创 Checkpoint 以及当 Job failover 时协调各个 Task 从 Checkpoint 恢复等。

2. TaskManager（又称为 Worker）：执行 Dataflow 中的 Tasks，包括内存 Buffer 的分配、Data Stream 的传递等。

![https://beancookie.github.io/images/运行Flink程序-02.png](https://beancookie.github.io/images/运行Flink程序-02.png)

Task Slot 是一个 TaskManager 中的最小资源分配单位，一个 TaskManager 中有多少个 Task Slot 就意味着能支持多少并发的 Task 处理。需要注意的是，一个 Task Slot 中可以执行多个 Operator，一般这些 Operator 是能被 Chain 在一起处理的。

![https://beancookie.github.io/images/运行Flink程序-03.png](https://beancookie.github.io/images/运行Flink程序-03.png)

### 命令行运行jar

1. 连接到K8S环境
```bash
kubectl get pod

NAME                                READY   STATUS    RESTARTS   AGE
flink-jobmanager-7f675c8869-fhb9m   1/1     Running   0          45h
flink-taskmanager-8869bf445-4vdf4   1/1     Running   0          46h
flink-taskmanager-8869bf445-bqn6x   1/1     Running   0          46h


kubectl exec -it flink-taskmanager-8869bf445-4vdf4 bash

./bin/flink run examples/streaming/WordCount.jar

./bin/flink run examples/streaming/WordCount.jar --input ./README.txt  --output ./word_count_out.txt

cat ./word_count_out.txt/1
```
### UI界面上传jar

```bash
# 拷贝容器中的jar包
docker ps

CONTAINER ID   IMAGE          COMMAND                  CREATED        STATUS        PORTS     NAMES
273f4ed76172   538a41938a2c   "/docker-entrypoint.…"   46 hours ago   Up 46 hours             k8s_jobmanager_flink-jobmanager-7f675c8869-fhb9m_default_c2a6453a-43ad-4f01-bf9f-362ee2d32e1a_0
f7d1a54d374f   apache/flink   "/docker-entrypoint.…"   46 hours ago   Up 46 hours             k8s_taskmanager_flink-taskmanager-8869bf445-4vdf4_default_a4c32e5e-d4bd-484f-b5ba-a6c32bda34e1_0
cec1f4dc93fa   apache/flink   "/docker-entrypoint.…"   46 hours ago   Up 46 hours             k8s_taskmanager_flink-taskmanager-8869bf445-bqn6x_default_27370fb4-16c0-4d60-8972-b66b891c07e0_0

docker cp f7d1a54d374f:/opt/flink/examples/streaming/WordCount.jar .
```

#### 通过UI界面上传jar包
![https://beancookie.github.io/images/运行Flink程序-04.png](https://beancookie.github.io/images/运行Flink程序-04.png)

![https://beancookie.github.io/images/运行Flink程序-05.png](https://beancookie.github.io/images/运行Flink程序-05.png)
### 开发工具调试开发