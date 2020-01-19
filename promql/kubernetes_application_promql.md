# 查询应用的所有实例指标


### 主题: 以优雅的方式查询到某应用下所有实例的监控指标

在PaaS 平台的研发过程中, 我们通常需要查找某应用所有实例的监控指标。为了在Dashboard中能够很好的展示应用的运行状态,这个需求相对容易, 但在实现时需要考虑一些细节点:

eg: 

1. 由于某些应用的实例数的确太多, 实例数太多会导致PromQL执行慢,数据返回不及时, 影响用户体验。
2. 由于某些应用是以sidecar方式部署的, 除了监控Pod内的主容器以外,还需要监控Pod内的副容器。
3. 是否支持此应用历史Pod指标查找。
4. 未来有没有TopN需求等待。

稍加思考,你可能会使用一些暴力的方式实现, 也有可能会使用一些比较自然的方式来实现, 这里我列举了两种实现方式。

已知参数:
```
0. application resource type (eg: deployment,sts,ds,cronjob and so on...)
1. applicationname
2. namespace

```

### 需求实现方式
#### 实现方式1:

已知应用类型和应用名称, 通过K8s clientset 查询到这个应用下的所有的Pod列表, 然后为每个Pod构造 PromQL 以for-range的方式请求prometheus的API执行PromeQL语句, 然后返回结果。

伪代码:
```
func GetMetrics() {
	r:=[]interface{}{}
	podList,_: = k8sclient.CoreV1().Pod(namespace).List(meta.ListOptions{LabelSelector: labels.FormatLabels(labelsmap))
	for _,v := podList.Item{
		podPromQL:=fmt.Sprintf("promql ql")
		sample:= prometheusapi.QueryRange(podPromQL)
		r=append(r,sample)
	}
}
```

缺点:

1. 若Pod数为n, 则发起请求数为n,对Prometheus服务器发起了太多的HTTP请求且没有必要。
2. 若不同应用的Pod label一样, 会导致应用的Pod列表异常/错乱。
2. 无法查询到已经挂掉的实例指标, 由于K8S的client返回的是当前时间的实例列表, 自然而然, 也就无法查询到已经挂掉的Pod监控指标, 但是这在实际工作中, 查询N小时,N分钟前应用的监控指标这是非常常见的操作。
3. 从代码角度来说, for-range构造PromeQL 发起N个请求是没必要的。首先, 如果for-range 是同步执行,一个http request 需要1s, 你有10个Pod, 也就是在10s后你才能返回, 显然用户是不能接受的; 如果你使用异步执行, 对比于同步来说, 稍微好点, 用户不用等待那么长时间, 但一瞬间有n个请求到prometheus服务端, 如果Prometheus的查询性能不太好, 有获取不到数据的可能, 同时也很没有必要。


#### 实现方式2:(推荐)

结合```cAdvisor```和```kube-state-metrics```的指标构造合理合理的PromQL, 一次性返回某应用下某Pod的指标。如果Pod数较多, 可以使用Topk 取前几条, 以便快速返回结果。

理由:

1. cAdvisor 提供Pod和container的指标
2. kube-state-metrics 提供指标 kube_pod_owner, kube_pod_owner会返回应用的Pod列表
3.  PromQL on() 指定匹配标签, group_left/group_right 做一对多或者多对一的关联查询。

优点: 

1. 完全避开了实现方式上的所有不足。
2. 可以查看历史pod的监控指标, 只要合理的设置prometheus 的查询条件Range


示例PromQL: 获取 prometheus-node-exporter 所有Pod的memory_working_set_bytes 指标。
```
container_memory_working_set_bytes{container!="POD",container!=""} / on(namespace,pod) group_left()  kube_pod_owner{owner_kind="DaemonSet",owner_name="prometheus-node-exporter",job="kube-state-metrics"} 
```

**备注**:由于 kube_pod_owner 的value一定是1, 所以使用乘除二元运算符并不影响结果。


### 知识扩展:

#### 1. 为什么会使用kube_pod_owner?

在Kubernetes的资源对象的ObjectMeta.Owner 有这样的定义

```
// ObjectMeta is metadata that all persisted resources must have, which includes all objects
// users must create.
type ObjectMeta struct 
	// List of objects depended by this object. If ALL objects in the list have
	// been deleted, this object will be garbage collected. If this object is managed by a controller,
	// then an entry in this list will point to this controller, with the controller field set to true.
	// There cannot be more than one managing controller.
	// +optional
	// +patchMergeKey=uid
	// +patchStrategy=merge

    // 中文
    // 该对象所依赖的的对象列表, 如果列表中的所有对象都已经被删除,那么这个对象也会被回收掉. 
	OwnerReferences []OwnerReference `json:"ownerReferences,omitempty" patchStrategy:"merge" patchMergeKey:"uid" protobuf:"bytes,13,rep,name=ownerReferences"`
...
}
```

所以, 通过kubernetes 对Owner的定义, 我们可以确保了资源对象与Pod的级联关系, 不存在Pod列表异常/错乱情况, 可以放心使用。

#### 示例PromQL中的 内置func的使用规则和限制? 

##### on 和 ingoring

1. on: 两个向量在做标签匹配的时候, 使用on(label1,label2) 指明参与匹配的标签。(在一对一时, 可以单独使用)
2. ignoring: 两个向量在做标签匹配时, 使用ignoring(label1,label2) 指明匹配时需要忽略的标签。 (在一对一时, 可以单独使用)

##### group_left 和 group_right

1. group_left和group_right的使用是: 那边向量数据量多,标识那边, 如果左边向量的数据多,使用group_let; 如果右边向量的数据多则使用group_right.
2. group_* 通常和on或者ingoring 一起使用。
3. group_*() 和 on() 或者ingoring() 一起使用时, 不能同时指定同样/包含关系的label,否则会报错, 使用时,只需要在 on 或者 group_*() 其中一个操作符指定即可。
4. group_left 对应的是向量之间的 many-to-one 的匹配关系
5. group_right 对应的是向量之前的 one-to-many 的匹配关系
6. prometheus 不支持向量 many-to-many 的匹配关系

eg:
```
container_memory_working_set_bytes{container!="POD",container!=""} / on(namespace,pod) group_left(namespace,pod)  kube_pod_owner{owner_kind="DaemonSet",owner_name="prometheus-node-exporter",job="kube-state-metrics"}  

在这条 promQL中, on 和 group 同时指定了label(namespace,pod).报错:

Error executing query: parse error at char 112: label "namespace" must not occur in ON and GROUP clause at once
```


### 语法总结

```
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

### bin-op: 二元操作符

#### 二元运算操作符
```
+ 加法
- 减法
* 乘法
/ 除法
% 模
^ 幂等
```

####  二元比较操作符
```
 == 等于
 != 不等于
 > 大于
 < 小于
 >= 大于等于
 <= 小于等于
```

