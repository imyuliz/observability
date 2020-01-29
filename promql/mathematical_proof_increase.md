# 以数学证明的方式深入理解increase()
<div align=right> 2020.1.29 </div>
## `increase()`

### 官方定义
`increase(v range-vector)` calculates the increase in the
time series in the range vector. 

increase(v range-vector) 计算的是区间向量在一段时间内增长数。

Breaks in monotonicity (such as counter
resets due to target restarts) are automatically adjusted for.

自适应单调性中断(比如target重启导致的计数器重置).

 The increase is extrapolated to cover the full time range as specified
in the range vector selector, so that it is possible to get a
non-integer result even if a counter increases only by integer
increments.

根据区间向量选择器的指定,可以推断出在整个时间范围内的增长量, 另外,因此，即使计数器仅以整数增量增加，也可能会获得非整数结果。

The following example expression returns the number of HTTP requests as measured
over the last 5 minutes, per time series in the range vector:

以下示例表达式返回区间向量中每个时间序列在最近5分钟内测得的HTTP请求数:
```
increase(http_requests_total{job="api-server"}[5m])
```

`increase` should only be used with counters. 

**increase 仅应与计数器一起使用.**

### 俗语定义

1. increase 返回的是区间向量在指定时间范围内(start ~ end)指定时间范围内的增长数。
2. increase 只能和计数器类型的指标一起使用.


### 数学证明

 **样本数据:**

![样本数据](/promql/basedata.png)


**数学计算与对比:**

3m, 5m http_requests_total增长数 PromQL:
```
3m 增长数:
increase(http_requests_total{job="kubernetes-apiservers"}[3m])

5m 增长数:
increase(http_requests_total{job="kubernetes-apiservers"}[5m])
```

![prometheus 结果与数学计算对比](/promql/increase.png)

通过上面的数学推断和prometheus 指标对比发现:

1. 通过数学计算得出的 1m(4次), 3m(12次), 5m(20次) 增长数与prometheus 计算出来的结果表现一致。
2. increase() 计算的是指定时间范围内的增长数, 单位:增长数/指定时间范围,

总结公式:

 $$ increase() = \frac{\Delta r}{\Delta t}  * t_{指定的时间范围}$$

### 源码分析:

```golang

// 源码位置:github.com/prometheus/prometheus/promql/functions.go

// === increase(node ValueTypeMatrix) Vector ===
func funcIncrease(vals []Value, args Expressions, enh *EvalNodeHelper) Vector {
	return extrapolatedRate(vals, args, enh, true, false)
}

// extrapolatedRate is a utility function for rate/increase/delta.
// It calculates the rate (allowing for counter resets if isCounter is true),
// extrapolates if the first/last sample is close to the boundary, and returns
// the result as either per-second (if isRate is true) or overall.
func extrapolatedRate(vals []Value, args Expressions, enh *EvalNodeHelper, isCounter bool, isRate bool) Vector {
    ms := args[0].(*MatrixSelector)

    var (
        matrix     = vals[0].(Matrix)
        rangeStart = enh.ts - durationMilliseconds(ms.Range+ms.Offset)
        rangeEnd   = enh.ts - durationMilliseconds(ms.Offset)
    )
    // 遍历多个vector，分别求解delta/increase/rate。
    for _, samples := range matrix {
        // 忽略少于两个点的vector。
        if len(samples.Points) < 2 {
            continue
        }
        var (
            counterCorrection float64
            lastValue         float64
        )
        // 由于counter存在reset的可能性，因此可能会出现0, 10, 5, ...这样的序列，
        // Prometheus认为从0到5实际的增值为10 + 5 = 15，而非5。
        // 这里的代码逻辑相当于将10累计到了couterCorrection中，最后补偿到总增值中。
        for _, sample := range samples.Points {
            if isCounter && sample.V < lastValue {
                counterCorrection += lastValue
            }
            lastValue = sample.V
        }
        resultValue := lastValue - samples.Points[0].V + counterCorrection

        // 采样序列与用户请求的区间边界的距离。
        // durationToStart表示第一个采样点到区间头部的距离。
        // durationToEnd表示最后一个采样点到区间尾部的距离。
        durationToStart := float64(samples.Points[0].T-rangeStart) / 1000
        durationToEnd := float64(rangeEnd-samples.Points[len(samples.Points)-1].T) / 1000
        // 采样序列的总时长。
        sampledInterval := float64(samples.Points[len(samples.Points)-1].T-samples.Points[0].T) / 1000
        // 采样序列的平均采样间隔，一般等于scrape interval。
        averageDurationBetweenSamples := sampledInterval / float64(len(samples.Points)-1)

        if isCounter && resultValue > 0 && samples.Points[0].V >= 0 {
            // 由于counter不能为负数，这里对零点位置作一个线性估计，
            // 确保durationToStart不会超过durationToZero。
            durationToZero := sampledInterval * (samples.Points[0].V / resultValue)
            if durationToZero < durationToStart {
                durationToStart = durationToZero
            }
        }

        // *************** extrapolation核心部分 *****************
        // 将平均sample间隔乘以1.1作为extrapolation的判断间隔。
        extrapolationThreshold := averageDurationBetweenSamples * 1.1
        extrapolateToInterval := sampledInterval
        // 如果采样序列与用户请求的区间在头部的距离不超过阈值的话，直接补齐；
        // 如果超过阈值的话，只补齐一般的平均采样间隔。这里解决了上述的速率爆炸问题。
        if durationToStart < extrapolationThreshold {
            // 在scrape interval不发生变化、数据不缺失的情况下，
            // 基本都进入这个条件。
            extrapolateToInterval += durationToStart
        } else {
            // 基本不会出现，除非scrape interval突然变很大，或者数据缺失。
            extrapolateToInterval += averageDurationBetweenSamples / 2
        }
        // 同理，参上。
        if durationToEnd < extrapolationThreshold {
            extrapolateToInterval += durationToEnd
        } else {
            extrapolateToInterval += averageDurationBetweenSamples / 2
        }
        // 对增值进行等比放大。
        resultValue = resultValue * (extrapolateToInterval / sampledInterval)
        // 如果是求解rate，除以总的时长。
        if isRate {
            resultValue = resultValue / ms.Range.Seconds()
        }

        enh.out = append(enh.out, Sample{
            Point: Point{V: resultValue},
        })
    }
    return enh.out
}
```

### 相关链接
[为什么 Prometheus increase 不返回整数?](https://lotabout.me/2019/QQA-Why-Prometheus-increase-return-float/)
[Prometheus Extrapolation原理解析](https://ihac.xyz/2018/12/11/Prometheus-Extrapolation%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)