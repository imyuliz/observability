# PromQL 动态标签处理

在PromQL中,允许对label进行操作的只有`label_join()`和`label_replace()`, 下文对这两个内置函数进行了详细解释。

### `label_join()`

#### 官方定义
For each timeseries in `v`, `label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)` joins all the values of all the `src_labels`
using `separator` and returns the timeseries with the label `dst_label` containing the joined value.
There can be any number of `src_labels` in this function.

This example will return a vector with each time series having a `foo` label with the value `a,b,c` added to it:

```
label_join(up{job="api-server",src1="a",src2="b",src3="c"}, "foo", ",", "src1", "src2", "src3")
```

#### 俗语定义

`label_join()` 专用于为 PromQL 结果集添加新的一个标签。新标签的值是指定的一个或多个源标签的值,多个值以`separator`(大多数情况分割符设置为`,`)拼接而成。使用的格式:
```
label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)
label_join(瞬时向量 instant-vector, 目标标签key string, 分隔符 string , 源标签_1 string, 源标签而_2 string, ...)
```

#### 使用示例

样本:
```
up{instance="localhost:9090",job="prometheus"} 1
```

现在需要为`up`指标添加一个新的label:`host`,`host`的值是`instance`和`job`的值,根据需求得出PromQL:
```
label_join(up,"host",",","instance","job")
```

结果:
```
up{host="localhost:9090,prometheus",instance="localhost:9090",job="prometheus"} 1
```

### `label_replace()`

#### 官方定义
For each timeseries in `v`, `label_replace(v instant-vector, dst_label string,
replacement string, src_label string, regex string)` matches the regular
expression `regex` against the label `src_label`.  If it matches, then the
timeseries is returned with the label `dst_label` replaced by the expansion of
`replacement`. `$1` is replaced with the first matching subgroup, `$2` with the
second etc. If the regular expression doesn't match then the timeseries is
returned unchanged.

This example will return a vector with each time series having a `foo`
label with the value `a` added to it:

```
label_replace(up{job="api-server",service="a:c"}, "foo", "$1", "service", "(.*):.*")
```

#### 俗语定义

`label_replace()`专用于替换 PromQL 结果集中某一个标签的值。使用的格式:
```
label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)
label_replace(瞬时向量 instant-vector, 目标标签key string, 替换值 string, 源标签 string, 值的正则表达式 string)
```
对于v中的每个时间序列, 将正则表达式与标签src_label的值相匹配。 如果匹配，则返回时间序列，标签dst_label替换为目标标签。 `$1`替换为第一个匹配的子组，`$2`替换为第二个,等等。如果正则表达式不匹配，则返回时间序列不变。

#### 使用示例

样本:
```
up{instance="localhost:9090",job="prometheus"} 1
```
现在需要为`up` 指标添加一个`host`标签,且`host`标签的值为`instance`标签分号(`:`)之前的值。根据需求得出PromQL:
```
label_replace(up, "host", "$1", "instance",  "(.*):.*")
```

结果:
```
up{host="localhost",instance="localhost:9090",job="prometheus"} 1
```
