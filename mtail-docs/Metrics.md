# Introduction
# 介绍

A metric is a data type that describes a measurement.

指标是一个描述测量的数据结构。

It has a **name**, and a **value**, and a **time** that the measurement was taken.

它有的要素有 **name** **value** **time**。

It also has **units**, so that measurements can be compared and calculated with.

还有 **units** 因此测量之间可以相互比较和计算。

It has a **class**, so that tools can automatically perform some aggregation operations on collections of measurements.

还有 **class** 所以工具可以自动在一系列测量之间做一些聚合操作。

It has a **type**, describing the sort of data it contains: floating point or integer values.

以及 **type** ，描述了存放数据的种类：是浮点数还是整数。

Finally, it has some **labels**, so that additional information about the measurement can be added to assist queries later.  Labels are key/value pairs, where the value may change for a specific measurement, but the keys remain constant across all measurements in a metric.

最后，它还有一些 **labels** 因此可以附加一些额外信息去帮助使用测量。 标签是k-v对，其中值可能会
为某些特定的测量而变化，不过同一个指标的标签的key对所有测量都是一样的。

## Classes of Metrics
## 指标类别

The class of a Metric can be:指标的类别可以是：

  * a monotonically increasing counter, that allows the calculation of rates of change
  * 单调递增的计数器，可以计算变化率
  * a variable gauge, that records instantaneous values
  * 变量计，记录瞬时值

Counters are very powerful as they are resistant to errors caused by sampling frequency.  Typically used to accumulate events, they can show changes in behaviour through the calculation of rates, and rates of rates.  They can be summed across a group and that sum also derived.  Counter resets can indicate crashes or restarts.

计数器因为它不受采样频率的影响，尤其是用于累计事件，很有用的。通过计算比率，甚至比率的比率，它可以
展示行为的变化。它还可以用于在一组行为之间计数，并且计数器的重启可以用来表示重启或者崩溃事件。

Gauges are less powerful as their ability to report is dependent on the sampling rate -- spikes in the timeseries can be missed.  They record queue lengths, resource usage and quota, and other sized measurements.

变量计则不那么有用，因为它汇报的是独立的事件，受到采样频率的影响，有可能丢掉一部分数据。它可以记录
队列长度，资源消耗之类的指标。

(N.B. Gauges can be simulated with two counters.)

## Types of data
## 数据类型

`mtail` records either integer or floating point values as the value of a metric.  By default, all metrics are integer, unless the compiler can infer a floating point type.

指标的类型支持整数和浮点数，默认来说所有的指标都是整数，除非编译器能够推断出它是浮点类型。

Inference is done through the type checking pass of the compiler.  It uses knowledge of the expressions written in the program as well as heuristics on capturing groups in the regular expressions given.

推断在编译器执行类型检查的时候实现。它利用程序中写的表达式和正则捕获组的信息来推断类型。

For example, in the program:例如，在这个程序中

```
counter a

/(\S+)/ {
  a = $1
}
```

the compiler will assume that `a` is of an integer type.  With more information about the matched text: 此时编译器会认为 `a` 是一个整型。如果对于匹配的文本，有进一步的信息

```
counter a

/(\d+\.\d+)/ {
  a = $1
}
```

the compiler can figure out that the capturing group reference `$1` contains digit and decimal point characters, and is likely then a floating point type.

那么编译器能够发现捕获组指向的 `$1` 包含小数位，因此这个很可能是浮点数。

## Labelling
## 打标签

Labels are added as dimensions on a metric:

标签可以添加成为指标的维度。

```
counter a by x, y, z
```

creates a three dimensional metric called `a`, with each dimension key `x`, `y`, `z`.

创建了一个具有三个维度的指标  `a` ，其中每个维度的 key 是`x`, `y`, `z`。

Setting a measurement by label is done with an indexed expression:
根据标签赋值可以用索引表达式：
```
  a[1, 2, 3]++
```

which has the effect of incrementing the metric a when x = 1, y = 2, and z = 3.

这个的意思就是对于  x = 1, y = 2, and z = 3 的指标，增加它的值。

Dimensions, aka *labels* in the metric name, can be used to export rich data to
the metrics collector, for potential slicing and aggregation by each dimension.

维度，即指标名中的标签，可以用于导出更加复杂的数据给指标收集系统，可以支持后续的切片和聚合操作。