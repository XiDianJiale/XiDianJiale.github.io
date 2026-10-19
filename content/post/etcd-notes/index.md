---
title: "etcd 学习笔记：Raft、MVCC 与 Watch 机制"
description: "梳理 etcd 的服务发现用途、Raft 一致性、MVCC 乐观并发控制、Watch 机制与高可用部署思路。"
slug: etcd-notes
date: 2026-10-19 16:37:23+0800
image: cover.png
categories:
    - Kubernetes
    - 运维
---

etc distribution 分布式key value存储 
基于Raft协议保证一致性
设计理念 如何支持watch机制

### 作用
服务发现，共享配置，一致性保障![Screenshot 2026-07-06 at 14.46.08.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-07-06%20at%2014.46.08.png)
### etcd的安装与操作
安装参考： https://github.com/etcd-io/etcd/releases
操作这一块很简单，k-v数据库操作：
- etcdctl member list
- etcdctl get 'key' -wjson
- etcdctl put /a b

### Raft协议原理
1. （前置）分布性一致(distributed consensus)问题: 比如这样一个场景-一个系统中有多个存储节点，后端做一个set命令如何保证多个节点都收到和更新响应？ 而Raft is a protocol for implementing distributed consensus.
2. 三种状态：raft下，每个member都有三种政治“身份”：status of a Node can be : follower, candidate, leader
3. election机制：
	刚开始的时候，所有的follower的节点会设置一个随机时间的election timeout, 在此期间收到leader的heartbeat则会重置这个时间，否则当产生election timeout后，节点将把自己作为选举人发起选举：将自己切换为candidate，并且像集群中其他节点发送请求，询问是否同意自己成为leader
	如果收到集群中半数以上的同意票，则该节点反手揭竿成为集群的leader，接受client的数据并且向其他的follower节点同步日志；相反的，如果没有接受到半数以上的投票，
	一个leader节点会不断向follower发送heartbeat来声明和保证自己的leader地位
![Screenshot 2026-07-06 at 14.34.40.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-07-06%20at%2014.34.40.png)
4. 如何改数据？ 日志复制机制：leader收到client的事物请求后，首先追加到本地的log中，通过heartbeat夹杂私货把该事物请求同步给其他的follower，follower接收到日志后记录日志并且向leader返回ACK，当leader接收到半数以上follower的ACK信息后正式处理该事物请求：把日志设置为committed状态并且追加到本地磁盘中，同时通知所有follower将该日志存储到自己的本地磁盘中![Screenshot 2026-07-06 at 14.36.24.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-07-06%20at%2014.36.24.png)
推荐动画链接： http://thesecretlivesofdata.com/raft/
### 说说某个游戏项目：
Q：zookeeper和etcd的作用重复，为什么还要引入zookeeper？
我们游戏后端一直是有一套强依赖zookeeper的微服务框架，现在业务上云上k8s的时候，业务研发线不想把自己的服务发现业务和云基础设施（k8s绑定），所以依然保留了ZK组件，虽然不够优雅算是一个历史遗留包袱但是跨云部署和虚拟机部署更友好
那么此时的服务发现两端解耦：
1. 基础设施层用etcd做服务发现：比如创建一个k8s service的时候，该Pod的IP发现由etcd这个服务注册中心来帮助完成
2. 游戏业务的服务发现：比如battleserver启动时向ZK注册自己的存在，login服需要找battleserver时是走去ZK里查

### etcd如何支持Kubernetes 强大的并发处理能力、数据一致性？
- MVCC： multi-version Concurrency Control
两个字段：
mod_revison ：全局单调增逻辑时钟，每当etcd任何数据发生变化全局revision就加1（就是你敲get
 po xx -oyaml的`metadata.resourceVersion` 字段，其底层映射的就是 etcd 的 `mod_revision`）
version ： 针对单个key的计数器


有了这个“**数据结构**”的基础  ->  就可以实现乐观锁的“**算法**”：
	每当我们提交一个修改，k8s检查我们提交请求中携带的 `resourceVersion`与数据库里该资源的最新 `mod_revision`是否一致，如果一致的话那就接受这次修改，如果不一致的话说明数据已经有了抢先修改，API server此时会直接报错（“对不起你这个请求是基于老版本上修改的，直接报409conflict” ），要求在你拿着与最新`resourceVersion`一致来进行写请求，
总的来说，就是一种通过 版本号实现锁功能的方法，这种方法巧妙地把数据一致性的保障从
数据库侧责任转移给了用户请求侧，在避免锁带来数据库性能损耗的同时保证了数据的一致性

### Watch机制 
etcd v3支持两种watch模式， 既可以watch某个key，也可以watch一个范围 （keyWatchers & rangeWatchers）
![Screenshot 2026-07-07 at 18.27.58.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-07-07%20at%2018.27.58.png)

### 高可用etcd解决方案
1. etcd-operator: coreos开源 给予kubernetes CRD 完成的etcd集群配置 (archived)
2. etcd statefulset helm chart

从这两种解决方案说说工程师做HA的思路：
前者etcd-operator 是基于 CRD 的 Operator 模式，自定义controller持续监听etcd cluster的健康状态，当一个etcd pod挂掉的时候，operator这里会写调用etcdctl这个API执行除了重启pod之外的逻辑
后者是把etcd作一个Kubernetes原语级部署，让Kubernetes的对statefulset的机制来做持久化和高可用





