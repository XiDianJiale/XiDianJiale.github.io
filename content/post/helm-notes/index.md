---
title: 知识点：helm
description: Helm 安装 Prometheus 与 Agones 配置速记
slug: helm-notes
date: 2026-05-21 20:34:27+0800
image: helm.png
categories:
    - Kubernetes
    - Helm
---

### helm install之后的log be like：
```bash
(base) savilahao@bogon ~ % helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
NAME: prometheus
LAST DEPLOYED: Wed May 20 14:54:39 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=prometheus"
  # ======================Grafana登陆与port转发教程===============
Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:
  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus" -oname)

  kubectl --namespace monitoring port-forward $POD_NAME 3000

Get your grafana admin user password by running:

  
  kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo


Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```


如helm创建后指导所示看密码：
`kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo` ^ebfc61


### 查看helm起的agons的配置：
1. helm list -A看release名：
```bash
NAME      	NAMESPACE     	REVISION	UPDATED                             	STATUS  	CHART                       	APP VERSION
agones    	agones-system 	1       	2026-05-19 13:07:59.439179 +0800 CST	deployed	agones-1.57.0               	1.57.0
metallb   	metallb-system	1       	2026-05-19 13:01:18.107293 +0800 CST	deployed	metallb-0.15.3              	v0.15.3
prometheus	monitoring    	1       	2026-05-20 14:54:39.049677 +0800 CST	deployed	kube-prometheus-stack-85.2.0	v0.90.1
```
2. 看具体配置（也就是value

> [!注意默认不加“a参数” 的话只能看到你手动set过的value]
> helm get values agones -n agones-system -a

```bash
(base) savilahao@bogon ~ % helm get values agones -n agones-system -a | grep prome
    prometheusEnabled: true
    prometheusServiceDiscovery: true
```
Q1:`prometheusEnabled: true`说明什么？
在 Agones 中开启这个配置，它的实际意思是：Agones Controller 在代码层面启动了一个内置的 HTTP Server（通常是 8080 端口的 `/metrics` 路径），并且开始以 Prometheus 能看懂的纯文本格式往外“吐”数据（比如当前有多少个游戏服）。
Q2:`prometheusServiceDiscovery: true` 是干嘛的？
简答：做服务发现——
原生 Kubernetes 早期是用 Annotations（注解）来做服务发现的。开启这个，Agones 会给自己的 Pod 打上类似 `prometheus.io/scrape: "true"` 的标签。意思是：“嗨，如果有谁在收集监控数据，请来这里抓我。”
