---
title: k8s生产化项目-Agones导学（May20）
description: Agones 核心功能：模拟玩家进场与动态扩缩容笔记
slug: agones-may20-notes
date: 2026-05-20 00:00:00+0800
image:
categories:
    - Kubernetes
    - Agones
    - Kubernetes生产化项目-Agones
---

书接上回，现在我们部署好了gameserver和监控系统，现在还要研究agones的两个核心功能：模拟玩家进场和动态扩缩容
依赖的组件：
- FleetAutoscaler：基于缓冲区的自动扩缩容
- GameServerAllocation：模拟Matchmaker分配房间服务器![[参考文档#^6dde12]] 


