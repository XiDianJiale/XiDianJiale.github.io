---
title: "游戏破解：本地实验中的存档修改与逆向分析"
description: "通过本地模拟游戏实验，理解存档校验、内存数值修改与二进制逆向中的客户端信任问题。"
slug: game-security-lab
date: 2026-10-19 16:37:23+0800
image: cover.png
categories:
    - 安全
---

### 背景
这篇笔记源于一次公司安全事件后的学习与思考，下面通过本地模拟游戏实验，理解客户端数据可信性与常见防护思路。

### 谈谈游戏破解
记得小时候玩很多游戏的时候, 很多游戏支持破解, 作为小孩本来是很馋xx的手冲, 或者一个游戏的角色和装备, 直到有一天我逛到了一些 xx助手,xx破解版的社区, 当时还玩了个小心机想着一定不能给其他小朋友们说这个渠道, 我拿着内购破解/金币无限的游戏在朋友们中是风光无限.
不过这种破解对单机游戏比较好使(也有部分联网的游戏,比如一些斗地主和植物大战僵尸2), 旧时的很多游戏的充值校验, 虚拟资产其实都是放在用户客户端中算的, 思路其实就是把客户端的充值校验逻辑改一下, 或者用修改器直接修改在内存中的金币钻石数量等就可以实现, 随着时间的发展, 现在的游戏这部分的信任已经逐渐从客户端移走了, 客户端只是对服务端后台数据库的一个显示, 但是黑客攻击者们又有动态注入等其他的思路......

### 我是攻击者
接下来进行role-play我们模拟攻击者对游戏进行不同阶段的破解操作,来体会攻防的技术和发展(ethic声明: 环境为本地模拟, 不提供对任何第三方游戏的违法攻击方法)











#### 涉及的环境, 工具与操作 
- C编译执行环境 
- 网络抓包
- eBPF(需要linux环境)
- 操作: 反向编译重签名, 改存档, 内存修改器
文件位置: /Volumes/WD1T/work/lab/cyber-security
### 先搭建起来个实验的框架 
这里使用一个及其简陋的C game server ,
![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260901214542738.png)
dig可以挖矿, 金币+1
save为保存存档,
同时有内购buy 和 CRC校验支持


### 从最简单的开始:扫描内存修改数值
记得之前玩ns1的时候有一个作弊工具, 是支持比如修改你塞尔达游戏中物品的数量, 可以按照要求扫描值, 定位到一个物品在内存中的具体地址之后, 修改这个地址对应的数据就可以修改你游戏里物品的数量啦. 

![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260901213932431.png)

这里是最简单的一种方法,就是直接修改存档文件, 所需要做的就是加载存档中的数值到内存,修改, 然后保存回原存档, 当然很多游戏会有校验的逻辑,这里就模拟了个crc, 修改的思路如下,很简单
```python
#!/usr/bin/env python3

import struct, zlib
from pathlib import Path
p = Path("/tmp/goldminer.save")
magic, gold, mined, vip, pad, crc = struct.unpack("<4sIIB3sI", p.read_bytes())
print("before changed", magic, gold, mined, vip, pad, hex(crc))
  

# Modify the values as needed
gold = 1000
vip = 1
  

# Pack the modified values back into a struct
new_data = struct.pack("<4sIIB3s", magic, gold, mined, vip, pad)
  

# Write the modified data back to the file
p.write_bytes(new_data + struct.pack("<I", zlib.crc32(new_data) & 0xFFFFFFFF))
  
print("after changed", magic, gold, mined, vip, pad)
```
这里crc直接调用zlib.crc32()就撞上了客户端代码的校验逻辑, 实际的游戏中校验很多是签名的方法, 如果是签名的私钥放在服务器, 那么客户端这里就没法用模拟算法的方法不侵入客户端代码的情况下成功修改了, 此时就依赖对客户端代码的修改, 比如校验不通过return -1的逻辑直接给他修改掉(内购破解也是这个逻辑就不在重复了)

> 如果说上面的本质是改存档文件的方法,接下来我们就看看改内存的方法

![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260902105739122.png)

### 二进制产物逆向工程方法
对于游戏来说, 玩家总是能拿到客户端代码的产物的, 比如一个二进制文件总是能够通过反编译, 动态调试等方法拿到一些密钥(无论是直接明文还是做了一些简单的混淆), 只要程序运行时需要在本地把密钥算出来做校验，这个密钥迟早会被逆向出来, 接下来我们就试试直接用strings一个简单的工具拔出来藏在二进制中的明文密钥


编译原理就决定了,对于代码中的一个const char* xx=xx,编译器因为要在程序运行的时候做计算, 所以就必须把这个东西完整放在产物中, 所以没有加任何防护的程序就直接可以用strings拿出来, 
补充: `strings` 是个非常朴素的工具，他扫描一个文件的所有字节，找其中连续的ASCII字符序列, 把看起来像是文字的都拿出来
这里做类似于之前crc的操作
![image.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/20260902165337446.png)
