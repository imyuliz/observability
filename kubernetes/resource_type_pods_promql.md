# 获取Kubernetes 资源对象的Pod列表PromQL

### 获取某 Deployment 的所有 Pod 列表

PromQL:
```
label_join(kube_pod_owner{namespace="命名空间",owner_kind="ReplicaSet",job="kube-state-metrics"},"replicaset",",","owner_name") * on(replicaset) 
 group_left kube_replicaset_owner{namespace="命名空间",owner_name="应用名字",job="kube-state-metrics"}
 ```

**注意:** 在 PromQL 中指定了 job,由于 pod_owner 指标在job=kubernetes-service-endpoints也存在,避免重复计算, 请根据实际情况改动。

获取default下的 web-app 的 Pod 列表:
```
label_join(kube_pod_owner{namespace="default",owner_kind="ReplicaSet",job="kube-state-metrics"},"replicaset",",","owner_name") * on(replicaset) 
 group_left kube_replicaset_owner{namespace="default",owner_name="web-app",job="kube-state-metrics"}
 ```

 ### 获取某 Statefulset 的所有 Pod 列表

PromQL:

```
kube_pod_owner{owner_kind="StatefulSet",namespace="命名空间",owner_name="应用名字",job="kube-state-metrics"}
```

获取 observability 下的 prometheus-alertmanager 的 Pod 列表:
```
kube_pod_owner{owner_kind="StatefulSet",namespace="observability",owner_name="prometheus-alertmanager",job="kube-state-metrics"}
```

### 获取某 Daemonset 的所有 Pod 列表

PromQL:
```
kube_pod_owner{namespace="命名空间",owner_name="应用名字",owner_kind="DaemonSet",job="kube-state-metrics"}
```

获取 observability 下 prometheus-node-exporter 的 Pod 列表:
```
kube_pod_owner{namespace="observability",owner_name="prometheus-node-exporter",owner_kind="DaemonSet",job="kube-state-metrics"}
```

### 获取某 Cronjob 的所有 Pod 列表

PromQL:

```
label_join(kube_pod_owner{namespace="命令空间",owner_kind="Job",job="kube-state-metrics",},"job_name",",","owner_name") * on(job_name) group_left() kube_job_owner{job="kube-state-metrics",namespace="命令空间",owner_name="应用名字",owner_kind="CronJob"}
```

获取 default 下 的 helloworld 的 Pod 列表:
```
label_join(kube_pod_owner{namespace="default",owner_kind="Job",job="kube-state-metrics",},"job_name",",","owner_name") * on(job_name) group_left() kube_job_owner{job="kube-state-metrics",namespace="default",owner_name="helloworld",owner_kind="CronJob"}
```

### 获取某 Job 的所有 Pod 列表

```
kube_pod_owner{namespace="命名空间",owner_name="应用名字",owner_kind="Job",job="kube-state-metrics"}
```

获取 default 下 job-demo 的 Pod 列表:

```
kube_pod_owner{namespace="default",owner_name="job-demo",owner_kind="Job",job="kube-state-metrics"}
```

### 相关链接

1. [Prometheus 最佳实践](https://prometheus.io/docs/practices/naming/)
2. [PromQL内置函数:动态标签替换](https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/promql/prometheus-promql-functions#dong-tai-biao-qian-ti-huan)