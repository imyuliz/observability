# 以数学证明的方式深入理解increase()
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

1. increase 返回的是区间向量从开始时间到结束时间中(start ~ end)指定一段时间范围的增长量。
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
2. increase() 计算的是指定时间范围内的增长量, 单位:增长量/指定时间范围。比如: 5m 增长量, 1d增长量

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

### 额外知识

increase 在某些情况下(比如:查询区间的时间与采样时间不重合,没法得到准确的数值)无法获取准确值,会做一些数据内推或者外推。所以, 我们需要了解更多的数据推断方式。

$$k= \frac{y_1-y_0}{x_1-x_0}$$

#### 外推(插)与线性外推(插)

外推:
外插亦称外推，是插值法的基本类型之一。当自变量 x 不是插值节点，且 x 位于插值区间之外时，用插值函数 P(x) 的值作为被插值函数 f(x) 的近似值，称为外插或外推。

线性外推:

$$y=y_0 +kx (y_0 可以为零, k 为斜率, x 为自变量)$$

通过这个上面这个公式, 即使$x$不在 $x_0$ 到 $ x_1 $ 之间并且 $k$ 也不是介于 0 到 1 之间，这个公式也是成立的。在这种情况下，我们也可以求出 $y$,这种方法叫作线性外插。但是需要注意的时, 由于假定了 $y_0$ 和 $k$ 在 $x_0$ 和 $x_1$ 之外也是固定的, 但现实情况可能和假定情况不一致,所以外推只是一个估计值。


#### 内推

内插法，一般是指数学上的直线内插，利用等比关系，是用一组已知的未知函数的自变量的值和与它对应的函数值来求一种未知函数其它值的近似计算方法,假定情况和外推一致, 线性内推公式如下:

$$ y= k(x-x_0)+y_0(x_0,y_0 可能为零 )$$

### 相关链接

1. [为什么 Prometheus increase 不返回整数?](https://lotabout.me/2019/QQA-Why-Prometheus-increase-return-float/)
2. [Prometheus Extrapolation原理解析](https://ihac.xyz/2018/12/11/Prometheus-Extrapolation%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)