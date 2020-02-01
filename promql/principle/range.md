# 深入理解Step

在使用 PromQL 时, 都需要指定查询的时间范围, 而 `Range` 就包含了时间返回的入参。

```go
// Range represents a sliced time range.
type Range struct {
	// The boundaries of the time range.
	Start, End time.Time
	// The maximum time between two slices within the boundaries.
	Step time.Duration
}
```
Start: 查询的开始时间
End: 查询的结束时间
Step: 样本数据之间的时间间隔

对于`Start`,`End`是非常好理解, 但`Step` 可能不太好理解, 下面通过使用示例和数学计算的方式带你理解`Step`.

#### Step 作用和证明:

在Prometheus 界面的使用:

![Prometheus 界面使用](/promql/principle/step.jpg)

Prometheus API接口使用示例:

PromQL: `http_requests_total{code="200",handler="prometheus",job="kubernetes-apiservers",method="get"}`
start: 1580525543.021
end: 1580529143.021
step: **14**

这里的`start`,`end` 使用的是时间戳, `step` 单位是:秒(s)

prometheus 接口返回的数据:
```
            t             v
"values":[
        [1580525543.021,"220244"],   // 1
        [1580525557.021,"220244"],   // 2
        [1580525571.021,"220244"],   // 3
        [1580525585.021,"220244"],   // 4
        [1580525599.021,"220248"],   // 5
        [1580525613.021,"220248"],   // 6
        [1580525627.021,"220248"]    // 7
```

$\Delta t = t_n - t_{n-1} = 14 $  

通过数学计算发现 `Step`= $\Delta t$,也通过数据计算证明了`Step`的作用和使用意义。

#### Step 注意事项
1. 当Start ~ End 的区间非常大时, Step 不能太小, 因为查询区间范围大, 数据粒度细会有大量的数据返回, 会增大服务器的压力; 前端数据渲染慢

#### 相关文档

1. [详解Prometheus range query中的step参数](https://yq.aliyun.com/articles/683127)