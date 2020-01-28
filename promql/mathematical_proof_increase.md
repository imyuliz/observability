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

1. increase 返回的是区间向量在指定时间范围内(start ~ end),指定时间范围内的增长数。
2. increase 只能和计数器类型的指标一起使用.


### 数学证明

3m, 5m http_requests_total增长数 PromQL:
```
3m 增长数:
increase(http_requests_total{job="kubernetes-apiservers"}[3m])

5m 增长数:
increase(http_requests_total{job="kubernetes-apiservers"}[5m])
```

![prometheus 结果与数学计算对比](/promql/increase.png)

通过上面的数学推断和prometheus 指标对比发现:

1. 通过数学计算得出的1m,3m,5m 增长数与prometheus 计算出来的结果表现一致。
2. increase() 计算的是指定时间范围内的增长数, 单位:增长数/指定时间范围,

总结公式:

 $$ increase = \frac{\Delta r}{\Delta t}  * t_{指定的时间范围}$$

## 源码分析:

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

	for _, samples := range matrix {
		// No sense in trying to compute a rate without at least two points. Drop
		// this Vector element.
		if len(samples.Points) < 2 {
			continue
		}
		var (
			counterCorrection float64
			lastValue         float64
		)
		for _, sample := range samples.Points {
			if isCounter && sample.V < lastValue {
				counterCorrection += lastValue
			}
			lastValue = sample.V
		}
		resultValue := lastValue - samples.Points[0].V + counterCorrection
		// Duration between first/last samples and boundary of range.
		durationToStart := float64(samples.Points[0].T-rangeStart) / 1000
		durationToEnd := float64(rangeEnd-samples.Points[len(samples.Points)-1].T) / 1000

		sampledInterval := float64(samples.Points[len(samples.Points)-1].T-samples.Points[0].T) / 1000
		averageDurationBetweenSamples := sampledInterval / float64(len(samples.Points)-1)
		if isCounter && resultValue > 0 && samples.Points[0].V >= 0 {
			// Counters cannot be negative. If we have any slope at
			// all (i.e. resultValue went up), we can extrapolate
			// the zero point of the counter. If the duration to the
			// zero point is shorter than the durationToStart, we
			// take the zero point as the start of the series,
			// thereby avoiding extrapolation to negative counter
			// values.
			durationToZero := sampledInterval * (samples.Points[0].V / resultValue)
			if durationToZero < durationToStart {
				durationToStart = durationToZero
			}
		}

		// If the first/last samples are close to the boundaries of the range,
		// extrapolate the result. This is as we expect that another sample
		// will exist given the spacing between samples we've seen thus far,
		// with an allowance for noise.
		extrapolationThreshold := averageDurationBetweenSamples * 1.1
		extrapolateToInterval := sampledInterval

		if durationToStart < extrapolationThreshold {
			extrapolateToInterval += durationToStart
		} else {
			extrapolateToInterval += averageDurationBetweenSamples / 2
		}
		if durationToEnd < extrapolationThreshold {
			extrapolateToInterval += durationToEnd
		} else {
			extrapolateToInterval += averageDurationBetweenSamples / 2
		}
		resultValue = resultValue * (extrapolateToInterval / sampledInterval)
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