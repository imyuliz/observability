# Kubernetes 监控组件推荐


| 组件名 |项目地址| 状态 | 场景|安装手册|
|---|---|---|---|---|---|---|---|
|Heapster| github.com/kubernetes-retired/heapster (Archived)|于Kubernetes v1.11后废弃|Kubernetes 监控数据聚合工具,节点监控, Kubernetes 组件状态监控, 编排级指标|暂无|
|metrics-server|github.com/kubernetes-sigs/metrics-server|推荐| Kubernetes监控的数据聚合工具,Heapster的替代品|暂无|
|cAdvisor|github.com/google/cadvisor| 推荐| Google开源的**容器资源**监控和性能分析工具,为容器而生| 暂无|
|kube-state-metrics|github.com/kubernetes/kube-state-metrics|推荐| 通过APIServer生产有关**资源对象**的状态指标元数据 |暂无|
| node_exporter|github.com/prometheus/node_exporter| 推荐| 节点指标监控|  暂无|
