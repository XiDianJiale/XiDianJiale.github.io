---
title: NAT配置与路由
description: 通过真实的一个网络拓扑理解nat的工作方式，内网外网如何安全通信？
slug: 计算机网络 
date: 2026-05-24 22:59:27+0800
image: 网络拓扑.png
categories:
    - 计算机网络
---

# 事情起因
今天May 24是我院计算机通信网络的实验考试，考试要求设计在Cisco仿真软件中配置单臂路由，DHCP，静态IP，NAT，ACL；同组的一个同学做的比我快，但是最后卡在了NAT配置上，导致最后以我的实验成果作为最好的标准完成了考核目标
该同学具有刨根问底的精神，哪怕考试后依然虚心向我请教学习和思考，我被我校同学的探究精神打动，耐心回复与解答，并且打算成文一篇，把我们今天遇到的问题公布于互联网上一起讨论：
# A Quick glance at 网络拓扑
![网络拓扑.png|695](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/%E7%BD%91%E7%BB%9C%E6%8B%93%E6%89%91.png)

我们的网络拓扑大致分为左、中、右三块：

- **左侧（内网/私网）：** 由 Router0 作为边界网关，下挂交换机和路由器（Router2, Router3），连接 PC0-PC3。采用单臂路由（Router-on-a-stick）划分了多个 VLAN，IP 网段规划为 `192.168.14.0/24` 等私网地址。
- **中间（骨干网/公网）：** 由 Router1 充当 ISP（运营商）路由器。它与 Router0 之间是公网网段 `200.14.1.0`。
- **右侧（外部网络）：** 由 Router4、Router5、Router6 组成的外部目标网络，连接 PC4-PC7。
Router0 路由表：
![router0-routetable.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/router0-routetable.png)
Router1 路由表：
![router1-routetable.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/router1-routetable.png)
# 问题聚焦
假设配置好所有VLAN，单臂路由，DHCP等等之后，思考几个问题：

### 1. 为什么本网络老师要求用静态路由配置，写很多规则，难道是老师折磨学生的变态心理？

很多同学的第一反应是为了让左侧的 PC0 能够 Ping 通右侧的 PC7，干脆在 Router0、Router1 和右侧的所有路由器上全部开启 RIP，互相学习路由。全网路由全覆盖，自然就通了。然后做一个NAT限制，外网访问不了PC0等内网设备就可以了，**我的组员队友就是踩在了这个坑里**
真相是：
- 这样做在模拟器里确实能通，但在逻辑上**完全违背了真实世界的网络架构，也让 NAT 彻底失去了意义。** 在真实情况中，Router1 是运营商的路由器，公网路由器**绝对不可能**包含你家内网（`192.168.x.x`）的路由表。如果用了动态路由，左右两边完全不需要做 NAT 转换就能互相访问，公网私网的边界就被打破了。路由器同样学会了外网怎么打入内网设备的路由，那这个NAT还有什么意义？

### 2. 公网不知道私网的IP，那么Ping的回包是怎么传回的？
假设 PC0（`192.168.14.1`）去 Ping 远端的 PC7，流程如下：
1. **去程：** PC0 发出 ICMP 包（源:`192.168.14.1`，目的:`PC7 IP`）。包到达边界 Router0。
2. **NAT 介入：** Router0 发现配置了 NAT，于是将包的**源 IP** 修改为自己的公网 IP（`200.14.1.2`），目的 IP 不变，丢给下一跳 Router1。
3. Router1 查路由表，找到去往 PC7 的路，将包顺利送达。
4. **回程：** PC7 收到包，准备回包。此时回包的**目的 IP 是 `200.14.1.2`**（Router0 的公网口 IP），而不是 PC0 的私网 IP！
5. **NAT 映射开始作用：** Router1 知道怎么去 `200.14.1.0` 网段（直连路由），把包交还给 Router0。Router0 查自己的 NAT 映射表，把包的目的 IP 还原成 `192.168.14.1`，最终送回 PC0。
而相反的，如果是PC7直接pingPC1的话，数据包到达Router1之后就不知道去哪里了，此时路由器会直接丢包并向源地址PC7地址发送一个 ICMP 错误消息（如“目的不可达”或“超时”）。