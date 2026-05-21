---
title: 知识点：生产环境observability做法
description: Agones 生产环境监控链路与 Grafana 实战记录
slug: observability-practice
date: 2026-05-21 20:34:27+0800
image: grafana.png
categories:
    - Kubernetes
    - Observability
---

### 观测逻辑链条
```bash
Prometheus
    ↓ select
ServiceMonitor
    ↓ select
Service
    ↓ endpoint
Pod /metrics
```


### 1.首先helm add:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 
helm repo update 
# 在 monitoring 命名空间下安装全家桶 
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

### 2.查你自己生产集群对象字段再做连接prometheus
`kubectl get prometheus -n monitoring -o yaml | grep -A 5 serviceMonitorSelector`

```bash
(base) savilahao@bogon ~ % kubectl get prometheus -n monitoring -o yaml | grep -A 5 serviceMonitorSelector
    serviceMonitorSelector:
      matchLabels:
        release: prometheus #你通过helm拉的一般都叫这个名字
    shards: 1
    tsdb:
      outOfOrderTimeWindow: 0s
```
1) Prometheus Operator只看有特定 Label 的资源，`matchLabels`这个key下是你需要告诉它的内容，后续写到k apply YAML 里 `metadata.labels`对象中
2) 查Agones 暴露指标的靶子（定位 Service 和 Port Name）
```bash
# 先看srv对象列表
(base) savilahao@bogon ~ % kubectl get svc -n agones-system | grep metrics
agones-allocator-metrics-service    ClusterIP      10.101.11.151    <none>          8080/TCP           26h
agones-controller-metrics-service   ClusterIP      10.109.209.46    <none>          8080/TCP           26h
agones-extensions-metrics-service   ClusterIP      10.105.232.133   <none>          8080/TCP           26h
# 看对应srv对象的Port Name
(base) savilahao@bogon ~ % kubectl get svc agones-controller-metrics-service -n agones-system -o jsonpath='{.spec.ports[*].name}{"\n"}'
	metrics  #同样的命令抓allocator和extensions的
# 看Agones 的 Service 自己带了什么标签
(base) savilahao@bogon ~ % kubectl get svc agones-controller-metrics-service -n agones-system --show-labels
NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   LABELS
agones-controller-metrics-service   ClusterIP   10.109.209.46   <none>        8080/TCP   26h   agones.dev/role=controller,app.kubernetes.io/managed-by=Helm,app=agones,chart=agones-1.57.0,heritage=Helm,release=agones
```


kubectl get svc agones-allocator-metrics-service -n agones-system --show-labels
### 根据信息组成yaml：
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: agones-metrics
  namespace: agones-system #注意这里最好写业务的ns而不是我们刚才创建的monitoring ns
  labels:
    # 替换为你从第一步查到的 Prometheus Selector
    release: prometheus 
spec:
  endpoints:
  - port: metrics # 替换为你从第二步查到的 Port Name (可能是 http web)
  - port: http #我抓到allocator的是这个
  namespaceSelector:
    matchNames:
    - agones-system
  selector:
    matchLabels:
      # 替换为你从第三步查到的 Service Label
      app: agones
```

做check：
```bash
(base) savilahao@bogon ~ % (base) savilahao@bogon ~ % kubectl get servicemonitor -n agones-system
NAME                 AGE
agones-all-metrics   4m12s
```
### 进入 Grafana 监控室
`kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80`

输入user和pw
![Screenshot 2026-05-20 at 16.00.12.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-05-20%20at%2016.00.12.png)

补充，转发进入服务器指令：
-  `kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80是Grafana `可视化界面的server ^f070bd
- `kubectl port-forward -n monitoring svc/prometheus-operated 9090`用于Prometheus 服务器本身

import json 到dashboards后：
![Screenshot 2026-05-20 at 16.40.26.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-05-20%20at%2016.40.26.png)
当然这里有个小插曲，开始的时候都是红色三角no data, fix: add Data Source-Prometheus mannually
看板信号分析：
- **GameServers count per type (饼图) & count overview (折线图):** 这两个图表显示，当前集群里**只有 1 个**状态为 `Ready` 的游戏服。结合下方的折线图，这条绿线一直平稳维持在 1，说明这个游戏服已经存活了一段时间。
- **GameServers per node & Node availability:** 这两个底部的图表监控的是底层的物理资源。它告诉你，当前有一个 Node（也就是你的 Minikube 虚拟机）正在被“使用（used）”，并且上面跑着这个孤零零的游戏服。
Q:为什么`Fleet RollOut Percentage`, `Fleet Replicas Count`是No Data？
当前集群里只有一个散养的单体 `GameServer`,并没有部署任何 `Fleet`资源
证据：
```bash
(base) savilahao@bogon ~ % kubectl get fleet
No resources found in default namespace.
```

### Grafana监测实验：部署gameServer舰队！(Fleet对象)
kubectl apply -f :
```bash
cat <<EOF | kubectl apply -f -
apiVersion: "agones.dev/v1"
kind: Fleet
metadata:
  name: simple-game-fleet
spec:
  # 我们要求集群里永远保持 5 个 Ready 状态的游戏服
  replicas: 5
  template:
    spec:
      ports:
      - name: default
        containerPort: 7654
      template:
        spec:
          containers:
          - name: simple-game-server
            image: us-docker.pkg.dev/agones-images/examples/simple-game-server:0.35
EOF
```
我们的舰队all set!!! 
```bash
(base) savilahao@bogon ~ % kubectl get fleet
NAME                SCHEDULING   DESIRED   CURRENT   ALLOCATED   READY   AGE
simple-game-fleet   Packed       5         5         0           5       5m23s
```
爽看Grafana：
![Screenshot 2026-05-20 at 16.52.14.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-05-20%20at%2016.52.14.png) ^b3ca9b

Q：“General，鄙人突然想把这支新组建的舰队都干掉怎么办？”
```bash
kubectl delete fleet simple-game-fleet
```
