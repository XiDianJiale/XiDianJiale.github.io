---
title: k8s生产化项目-Agones导学（May19）
description: Minikube + Agones 安装、测试与监控实践记录
slug: agones-may19-notes
date: 2026-05-19 00:00:00+0800
image:
categories:
    - Kubernetes
    - Agones
    - Kubernetes生产化项目-Agones
---

### 安装与配置
```bash
# 安装
brew install kubectl
brew install helm
brew install minikube

# 创建minukube k8s cluster
minikube start \
--kubernetes-version v1.34.6 \
-p agones \
--driver docker \
--cpus 4 \
--memory 8192
```

配置MetalLB
```bash
# 安装 MetalLB
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb \
--namespace metallb-system \
--create-namespace
# 安装好都就绪后先minikube ip -p agones看IP然后配置IP地址池
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
name: default-pool
namespace: metallb-system
spec:
addresses:
- 192.168.49.50-192.168.49.250 # 从 minikube ip 的⽹段分配
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
name: default
namespace: metallb-system
spec:
ipAddressPools:
- default-pool
EOF
```
安装**Agones**
helm repo add ; helm repo update ; helm install agones xx

部署服务
kubectl apply -f https://raw.githubusercontent.com/agones-dev/agones/release-1.57.0/examples/simple-game-server/gameserver.yaml

`minikube tunnel -p agones`
修改macos路由表，让所有发往整个 LoadBalancer 网段的包都走特定网关IP，从而可以转发到k8s集群内部

外部可以ping通但是无法udp连接(我用` minikube tunnel -p agones`之后也不行)，进入`minikube ssh -p agones`后发现可以ACK返回测试正常

停止集群运行：保留磁盘但是释放内存和cpu
`minikube stop -p agones`

看minikube整体情况
`minikube profile list`

看某个profile的局域网IP
`minikube ip -p agones`
看Agones GameServer 的分配端口：
kubectl get gs -n \<namespace>

### 安装成功后测试
```bash
# 看gameserver的IP和端口后
docker@agones:~$ nc -u 192.168.58.2 7621
kk
ACK: kk
```
这里证明 UDP 包已经成功打通了 Minikube 的网络，并被游戏服接收并回包。

### 体会agones项目的工程魅力
` kubectl describe gs simple-game-server-pwdww`
1. 可以看到events：
```bash
Events:
  Type    Reason          Age    From                   Message
  ----    ------          ----   ----                   -------
  Normal  PortAllocation  7m38s  gameserver-controller  Port allocated
  Normal  Creating        7m38s  gameserver-controller  Pod simple-game-server-pwdww created
  Normal  Scheduled       7m37s  gameserver-controller  Address and port populated
  Normal  RequestReady    7m36s  gameserver-sidecar     SDK state change
  Normal  Ready           7m36s  gameserver-controller  SDK.Ready() complete
```
2. Status部分，看看 Agones 是如何把端口映射和宿主机 IP 同步到资源状态里的
```bash
Status:
  Address:  192.168.58.2
  Addresses:
    Address:  192.168.58.2
    Type:     InternalIP
    Address:  agones
    Type:     Hostname
    Address:  10.244.0.25
    Type:     PodIP
  Eviction:
    Safe:              Never
  Immutable Replicas:  1
  Node Name:           agones
  Players:             <nil>
  Ports:
    Name:          default
    Port:          7621
  Reserved Until:  <nil>
  State:           Ready
```
#### 一个实验，体会pod优雅退出（gameserver业务逻辑：自己决定自己的生死，不由k8s调度机制杀掉）：
实验：在nc -u中发送EXIT信号，观察容器状态编程退出
![Screenshot 2026-05-20 at 14.03.50.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-05-20%20at%2014.03.50.png)
观察这个gm完整的生命周期：
![Screenshot 2026-05-20 at 14.12.21.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-05-20%20at%2014.12.21.png)
给pod发送EXIT信号这个pod就自动把自己删除了，k get pod -A 不会看到 0/1 ，这是gameserve的业务逻辑，一个游戏结束后该服务器就完全释放掉了

#### 理解agones重要工程机制
1. `kubectl logs simple-game-server-rcsxb -c agones-gameserver-sidecar `
密密麻麻的输出中，
- `"grpcEndpoint":"localhost:9357"`
- `"httpEndpoint":"localhost:9358"`
这说明 Sidecar 启动后，在 Pod 的内部网络空间（`localhost`）死死地守住了这两个端口。监听信号：

	游戏进程启动时，它其实是向 `localhost:9357`（Sidecar 的端口）发送了一个 `/Ready` 请求。 当你敲下 `EXIT` 时，你的程序也是向 `localhost` 发送了 `/Shutdown` 请求。 
	业务代码完全不需要懂怎么调用复杂的 Kubernetes API。它只需要和同 Pod 内的本地 Sidecar 通信。Sidecar 收到指令后，替你去和 K8s 的 API Server 交涉，修改 CRD 的状态。这就是典型的解耦设计。

2. `kubectl get pod simple-game-server-rcsxb -o yaml | grep hostPort`
```bash
# 输出
    hostPort: 7078
```
Agones 暴躁地跳过了 K8s 标准的 Service 负载均衡，直接将宿主机的网卡端口和容器内的 UDP 端口硬绑死。刚才用 `nc` 连的 `192.168.58.2` 根本不是一个虚拟 IP，那是 Minikube 虚拟机的真实物理 IP。Agones Controller 在后台帮你维护了一个庞大的端口池，自动避开端口冲突，实现了网络直通（Direct Routing）。

在原生 K8s 中，是 Kubelet 定期去Probe你的 Pod 活没活着。但在 Agones 中，这个逻辑被反转了。日志会看到游戏程序在疯狂地、周期性地向 Sidecar 发送心跳包。 
机制揭秘： 游戏服一旦卡死（比如死锁），Kubelet 的外部探测可能无法察觉业务逻辑层的假死。Agones 要求业务进程主动调用 SDK 上报 `Health()`。一旦 Sidecar 一定时间内没收到心跳，Sidecar 就会直接向 K8s 宣告这个游戏服处于 `Unhealthy` 状态，触发后续的回收逻辑。
该心跳机制可以看日志：
```bash
(base) savilahao@bogon ~ % k logs simple-game-server-rcsxb
Defaulted container "simple-game-server" out of: simple-game-server, agones-gameserver-sidecar (init)
2026/05/20 06:15:09 Creating SDK instance
2026/05/20 06:15:09 Starting Health Ping
2026/05/20 06:15:09 Marking this server as ready
2026/05/20 06:15:09 Starting UDP server, listening on port 7654
2026/05/20 06:15:09 Health Ping
2026/05/20 06:15:11 Health Ping
2026/05/20 06:15:13 Health Ping
```

#### 做下observability(**Prometheus + Grafana**)
[[知识点：生产环境observability做法]]
agone项目暴露了很多观测借口，
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 
helm repo update 
# 在 monitoring 命名空间下安装全家桶 
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```
创建一个 `ServiceMonitor` 资源，把 Agones Controller 的指标端点暴露给它:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
name: agones-all-metrics
namespace: agones-system
labels:
release: prometheus
spec:
namespaceSelector:
matchNames:
- agones-system
selector:
matchLabels:
  # 靠这个标一网打尽三个 Service
  app: agones
endpoints:
# 匹配 Controller 和 Extensions
- port: metrics
# 匹配 Allocator
- port: http
EOF
```
可以查到这个serviceMonitor对象创建成功：
```bash
(base) savilahao@bogon ~ % kubectl get servicemonitor -n agones-system
NAME                 AGE
agones-all-metrics   4m12s
```

端口转发+登陆Grafana：
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
看密码![[知识点：helm#^ebfc61]]

进入grafana后import json, json哪里来？ 从官方项目复制
curl -LO https://raw.githubusercontent.com/agones-dev/agones/main/build/grafana/dashboard-gameservers.yaml


最终看板效果![[知识点：生产环境observability做法#^b3ca9b]]

### 写在实验结束：
minikube资源生命管理层级：
- minikube pause -p agones ->不占CPU了，但是内存依然在吃
- minikube stop -p agones ->Graceful Shutdown，给Kubelet 和所有的 Pod 发送 SIGTERM 信号，CPU和占用内存都释放掉
- minikube delete -p agones -> 毁灭整个集群，释放CPU和内存，同时连带之前写入的磁盘资源一并释放

Q：下次如何恢复之前的状态？
Kubernetes 是Declarative的架构，依赖`etcd` 的分布式键值数据库，所有数据都做了持久化，理论上不需要我做什么minikube start -p agones就可以恢复，不过监测这里我还要输port-forward命令连到grafana server，另外就是Grafana 面板可能需要在dashboards中重导个那个json文件![[知识点：生产环境observability做法#^f070bd]]




### 总结
我的暴露出来的一些弱点：
1. 对网络层级的理解一般，本次ICMP可通但是TCP不通其实可以用cn的知识很好理解但是i cant
2. 对Kubernetes的网络入口机制不清楚，不知道NodePort Ingress 这些都是干嘛的，概念都没有，not to mention结合这些设计原理进行网络排查 
3. 

