# 序言




## RoadMap

本书致力于**可观测性**,主要包括 指标监控, 链路追踪, 日志, 但现在正处于初级阶段。内容正在逐步规划中。

## 致用 PromQL

* [查询应用的所有实例指标, 善用group_*() on()](/promql/kubernetes_application_promql.md)

## 部署指南

Prometheus 可以支持多种部署方式, 这里将列举 二进制部署, Docker 容器部署, Kubernetes 部署。后面将会针对各种场景的脚本部署, Operator部署。

* 部署方式
* [二进制部署](/deploy/method/binary.md)
* Docker 容器部署
* Kubernetes 部署

## 配置管理
* prometheus 配置结构分析
* 采集目标
* Label
    * 重命名Label
    * 删除Label
    * 添加Label
* 服务发现
    * 静态文件服务发现
    * 动态文件服务发现
    * Kubernetes服务发现 
* Action

## Exporter

* node-exporter 部署
* kube-state-metrics 部署
* 开发 exporter

## Pushgateway

## PromQL 
* Metrics类型
* 常用 PromeQL

##  数据存储
* 数据存储分析
* Local Store
* Remote Store

## 数据可视化

* Grafana

## AlertManager 
* 部署
* 触发逻辑
*  告警方式
    * Wechat
    * Slack
    * Email
    * 钉钉
    * Webhook 

## 集群

* 联邦
* Thanos

## Kubernetes 监控方案
* node-exporter
* kube-state-metrics
* cAdvisor

## 常见应用监控

* Mysql
* Redis
* Nginx

## 性能调优

## 常见问题

# Kubernetes 监控组件推荐


| 组件名 |项目地址| 状态 | 场景|安装手册|
|---|---|---|---|---|---|---|---|
|Heapster| github.com/kubernetes-retired/heapster (Archived)|于Kubernetes v1.11后废弃|Kubernetes 监控数据聚合工具,节点监控, Kubernetes 组件状态监控, 编排级指标|暂无|
|metrics-server|github.com/kubernetes-sigs/metrics-server|推荐| Kubernetes监控的数据聚合工具,Heapster的替代品|暂无|
|cAdvisor|github.com/google/cadvisor| 推荐| Google开源的**容器资源**监控和性能分析工具,为容器而生| 暂无|
|kube-state-metrics|github.com/kubernetes/kube-state-metrics|推荐| 通过APIServer生产有关**资源对象**的状态指标元数据 |暂无|
| node_exporter|github.com/prometheus/node_exporter| 推荐| 节点指标监控|  暂无|
