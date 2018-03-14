---
title: Kubernetes部署heapster插件
---

# 介绍
Kubernetes dashboard安装完成之后，界面并不会图形化展示集群的计量信息。

![no heapster](img/noheapster.png)

Heapster插件就是专门用来进行计量信息的图形化展示的。安装完heapster之后k8s的dashboard效果如下图所示。

![heapster](img/heapster.png)

Heapster 包含两个组件，heapster core 和Eventer。Heapster core负责从k8s集群中读取计量数据然后存储起来，同时也会通过Model API提供一些计量数据。
Eventer从k8s主节点读取事件然后保存到后端存储中。

k8s本身会手机计量数据的，heapster会去采集这些数据然后存储起来。Heapster支持的后端存储有InfluxDB、Kafka、Elasticsearch、StatsD、Graphite等，详见[Heapster支持的后端列表](https://github.com/kubernetes/heapster/blob/master/docs/sink-configuration.md)

# 配置

本文采用InfluxDB作为存储后端，配置步骤可以参考[Heapster InfluxDB配置](https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md)

    kubectl create -f deploy/kube-config/influxdb/
    kubectl create -f deploy/kube-config/rbac/heapster-rbac.yaml

#  安装过程中存在的问题

Pod一直处在创建中，镜像无法下载
* 在docker Hub上搭建代理镜像 [解决gcr.io/google_container/***镜像下载失败的解决方案](http://blog.csdn.net/chenyufeng1991/article/details/79118330)

* 使用docker-get命令拉取镜像 见[docker-get](http://ss.samblade.ml/docker-get.html)
* 使用网页下载镜像 见[docker-tar](http://ss.samblade.ml/docker-tar.html)

