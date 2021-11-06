---
title: Flink开发环境搭建
date: 2021-11-06 15:24:40
categories:
- Flink
tags: 
- Flink
---

### 安装Flink
1. Kubernetes
Flink会话集群作为长期运行的Kubernetes Deployment执行。
Kubernetes 中的基本 Flink 会话集群部署包含三个组件：
- JobManager
- TaskManagers
- 暴露JobManager的REST和UI端口的服务

> jobmanager-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-jobmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      component: jobmanager
  template:
    metadata:
      labels:
        app: flink
        component: jobmanager
    spec:
      containers:
      - name: jobmanager
        image: flink:latest
        args:
        - jobmanager
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob
        - containerPort: 6125
          name: query
        - containerPort: 8081
          name: ui
        env:
        - name: JOB_MANAGER_RPC_ADDRESS
          value: flink-jobmanager
```

> taskmanager-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-taskmanager
spec:
  replicas: 2
  selector:
    matchLabels:
      component: taskmanager
  template:
    metadata:
      labels:
        app: flink
        component: taskmanager
    spec:
      containers:
      - name: taskmanager
        image: flink:latest
        args:
        - taskmanager
        ports:
        - containerPort: 6121
          name: data
        - containerPort: 6122
          name: rpc
        - containerPort: 6125
          name: query
        env:
        - name: JOB_MANAGER_RPC_ADDRESS
          value: flink-jobmanager
```

> jobmanager-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager
spec:
  ports:
  - name: rpc
    port: 6123
  - name: blob
    port: 6124
  - name: query
    port: 6125
  - name: ui
    port: 8081
  selector:
    app: flink
    component: jobmanager
```

### 启动Flink
```bash
kubectl create -f jobmanager-service.yaml
kubectl create -f jobmanager-deployment.yaml
kubectl create -f taskmanager-deployment.yaml

kubectl proxy
```

### 访问Flink UI

> ![http://localhost:8001/api/v1/namespaces/default/services/flink-jobmanager:ui/proxy/#/overview](http://localhost:8001/api/v1/namespaces/default/services/flink-jobmanager:ui/proxy/#/overview)