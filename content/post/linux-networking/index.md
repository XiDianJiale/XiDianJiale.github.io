---
title: "Linux 网络：veth、网桥与容器 Overlay 网络"
description: "从虚拟网卡、network namespace 和 veth pair 出发，理解 Docker 网桥、NAT 与跨主机 Overlay 网络。"
slug: linux-networking
date: 2026-10-19 16:36:32+0800
image: cover.png
categories:
    - 计算机网络
    - 运维
---

docker这边说的:
![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260924143851537.png)
其实veth docker0 endpoint就是对应linux里用软件做出来的网卡, 网线和交换机 
### 网卡
对于电脑来说,想要上网就需要网卡, 网卡中得有我们的MAC地址和IP地址,来方便我们数据包接受来实现通信


### linux通过软件的方式造出这些虚拟设备,
linux通过新建一个network namespace在逻辑上创建一个独立的网络空间(可以有自己的网卡,ip,路由表)
创建veth pair相当于虚拟的网线,该虚拟网线的两头分别是一块逻辑虚拟网卡, 从A进的数据从B出来, 不涉及IP MAC等等, 只是数据的搬运工
如果说想要让新建的容器和宿主机通信, veth pair一头是容器的网卡eth0,一头是宿主机网卡vethxxxx(不配IP网卡)


这里重点可以看下docker的网络:
[[docker技术理解#网络]]

### 网络
网络这里强调引申一下, Q:容器如何和宿主机通信?
每个容器有自己的network namespace, 和宿主机的协议栈是分开的, 通过虚拟机网线veth连接 :
![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260924150527100.png)
- container 1访问container2: 容器1eth0 -> 宿主机vethxx -> docker0虚拟交换机 -> 宿主机vethxx -> 容器2eth0
- 容器访问外网: 比如容器1eth0的包送到docker0, 宿主机发现要访问的这个外网不在172.17.0.0/16, 于是走网关,并且摇身把容器这边网卡的ip`172.17.0.2` 换成宿主机网卡ip `192.168.0.101`, 然后发给外网
- 容器访问docker网关(比如容器访问宿主机中的一个程序): 比如容器1包的目标地址是172.17.0.1, 该包到达docker0就停下给宿主机了,不会出网或者说转给其他容器 ; 这里给个例子,比如宿主机有一个服务监听0.0.0.0:8080, 容器内执行curl http://172.17.0.1:8080, 此时包的目的就是docker0的地址

引申一下,问你一个致命的问题, 假设对于一个cluster, 不同node上中的pod和pod之间要访问的话,是不是就只能走网关NAT为node网卡的IP来进行通信吗?
因为如果走网桥通过NAT一层映射后自己原来pod容器网卡IP就变了, 这样通信很不方便,于是就有了Overlay这种跨主机的通信方式, 道理很简单就是再套一层, 把容器的IP套在网络包中,只是多一层包头的开销而已
举个例子:
![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260924160026745.png)
- 同一个主机的不同pod可以使用网桥进行通信
- 不同主机上的pod之间流量是flanneld封装在UDP数据包中的 
所以网络抓包大概拿到的flannel包大概长这样:
```txt
外层以太网   源 MAC：本机 eth0
            目标 MAC：下一跳网关或对端宿主机

外层 IP     源：192.168.0.101（本机节点 IP）
            目标：192.168.0.102（对方节点 IP）
            协议：UDP

外层 UDP    目标端口：8472（Flannel 默认，不是标准 VXLAN 的 4789）
            源端口：按内层包算出来的，用来做负载分担

VXLAN 头    8 字节，VNI 默认是 1
            常见字节：08 00 00 00 00 00 01 00

内层以太网   源 MAC：本机 flannel.1 的 MAC
            目标 MAC：对方 flannel.1 的 MAC

内层 IP     源：10.244.0.5（Pod A，原样保留）
            目标：10.244.1.8（Pod B，原样保留）

内层 TCP/UDP  原来的端口和数据

```

