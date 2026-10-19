---
title: "Kubernetes client-go：Informer、Indexer 与 WorkQueue 入门"
description: "从 Clientset 与资源 API 出发，梳理 Informer、Reflector、DeltaFIFO、Indexer 和 WorkQueue 的协作流程。"
slug: kubernetes-client-go
date: 2026-10-19 16:20:52+0800
image: cover.png
categories:
    - Kubernetes
    - Go
---

### kubernetes调度机制
kubernetes中有很多原生的controller(比如常见的deploymeny controller; node controller; service controller; 还有Garbage Collector, pod autoscale等等)
我们用户在使用kubernetes这个编排工具是很方便的,但是越便利的工具其实背后的工程机制往往越复杂,k8s正是其背后的复杂的工程实现使得用户方感知透明; 个人认为理解了k8s中各种controller的逻辑就是理解了k8s的工作机制. 
因为我们可以通过调度机制串联起来整个相关的知识: 从controller mgr, kube scheduler, kubelet等直接相关组件串联到api server etcd 等等

### 代码与集群交互
最近学习接触到公司内的devopsmcp(接入很多公司后端系统的mcp服务器项目)项目第一次接触到client-go.
client-go是所有原生controller使用的go语言客户端库, 其中包含两个主要的机制, 就是informer机制和lister机制 ; 前者回答了控制器如何感知集群资源的变化, 后者回答了具体如何读这个对象
我们如果要自定义一些controller(也就是做operator开发)的时候, 都是遵守一同相似的思路: 使用inform机制list&watch集群资源(而非一直轮训打api server)然后结合期望和实际状态做调谐reconcile, 其实一般我们新写一个operator的时候要不就直接抄过来一个controller然后改成自己的逻辑, 或者借助社区中的一些脚手架生成的工具

### 说说k8s的资源
我们常说到k8s的设计哲学, 除了声明式之外, 经常听到的还有一个就是“一切皆资源(对象)”; 如果我们在代码中对这些资源进行get, update, 那么要怎么操作呢?
Kubernetes资源的储存是按照不同功能放在不同API Group下的:
- Core API Group (api/v1) pod service node configmap secret namespace PV PVC event
- Apps API Group (apis/app/v1) deployment ReplicaSet StatefulSet DaemonSet ControllerRevision ; 
- Batch API Group(apis/batch/v1) Job CronJob
举个例子,比如我们直接敲命令通过http访问GET /apis/apps/v1/namespaces/default/deployments, 代码中则是访问clientset.AppsV1().Deployments("default")

### client-go大致的编码实现思路
第一看一个通用controller:
```go
type Controller struct {  
    informer cache.Controller  
    indexer  cache.Indexer  
    queue    workqueue.RateLimitingInterface  
}
```

首先会创建个Clientset, 这里会大致读一个kubeconfig然后创建一个clientset, clientset也就是一组k8s API的集合,可以通过这种方式访问k8s各种资源, 比如刚才说的clientset.CoreV1().Pods("default").Get(...)
```go
config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)  
if err != nil {  
    panic(err.Error())  
}  
  
clientset, err := kubernetes.NewForConfig(config)  
if err != nil {  
    panic(err.Error())  
}
```
其中kubernetes.NewForConfig()这个函数是根据config创建一个REST Client, 设置好API server地址, 配置认证和TLS, 创建core batch等资源对应的客户端 ,最后返回一个ClientSet

然后就是根据这个clientset创建informerFactory了. 顾名思义工厂, 其实是一群informer的集合,specificly工厂中同一种informer是尽可能共享缓存和API请求的来实现复用, 这里举例比如拿一个pod的informer
```go
informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*30)  
podInformer := informerFactory.Core().V1().Pods()
```

我们在之前的小节中说到informer是通过list&watch来感知集群中资源的变化, 那么代码这里具体是如何实现的呢? (提供网上找到的两个图来参考)
![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260929222816144.png)
![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260929223102975.png)

Informer内部机制来说 ,其实和API server交互的是reflector, 是Reflector来list获取所有已有的pod, 放入DeltaFIFO这个队列,并且持续watch监听pod资源的变换
DeltaFIFO则是暂存资源的变化类型,如 Added. Deleted等等, 
Informer中的一个常驻go routine则从中不断pop出对象，Indexer看到弹出对象时则直接存在Thread Safe Store中（这个存储存在的意义是减轻查APIserver的压力，后续的controller worker直接在读这个**内存中**的Indexer即可不需要问APIserver）
然后就是之前谈到的一个生产者消费者模型了:
informer更新完indexer之后把事件分发给ResourceEventHandler, 消费者这边则是拿WorkQueue中的key去消费事件

如果说上述的是informer中的一个生产者消费者模型的底层, 那么controller这里也是创建了个controller业务这边的queue, 来消费informer给的事件
```go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
```