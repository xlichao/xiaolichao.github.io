# Introduction

`mtail` is very simple and thus limits what is possible with metric
manipulation, but is very good for getting values into the metrics.  This page
describes some common patterns for writing useful `mtail` programs.

mtail 非常简单，虽然处理指标的能力不太够，但是能很方便地给指标填值。

## Changing the exported variable name
## 更改输出的指标名
`mtail` only lets you use "C"-style identifier names in the program text, but
you can rename the exported variable as it gets presented to the collection
system if you don't like that.

在程序中，只允许使用C风格的命名，不过你可以在输出给指标收集系统时重命名。


```
counter connection_time_total as "connection-time_total"
```


## Reusing pattern pieces
## 重用模式

If the same pattern gets used over and over, then define a constant and avoid
having to check the spelling of every occurrence.

如果有一个模式到处都要用，可以定义一个常量，这样就能避免每次用的时候都要检查一遍拼写。

```
# Define some pattern constants for reuse in the patterns below.
# 定义以下使用的模式常量
const IP /\d+(\.\d+){3}/
const MATCH_IP /(?P<ip>/ + IP + /)/

...

    # Duplicate lease
    /uid lease / + MATCH_IP + / for client .* is duplicate on / {
        duplicate_lease++
    }
```

## Parse the log line timestamp
## 处理日志中每行的时间戳

`mtail` attributes a timestamp to each event.

`matail` 对每个事件都分配一个时间戳。

If no timestamp exists in the log and none explicitly parsed by the mtail program, then mtail will use the current system time as the time of the event.

如果日志中不存在时间戳，或者时间戳没有显式告诉mtail如何解析，那么mtail会用当前系统事件作为该事件的时间戳，

Many log files include the timestamp of the event as reported by the logging program.  To parse the timestamp, use the  `strptime` function with
a [Go time.Parse layout string](https://golang.org/pkg/time/#Parse).

很多日志文件都对每个事件写了时间戳，如果想使用的话，需要使用 `strptime` 函数以 Go 语言的形式处理时间。

```
/^(?P<date>\w+\s+\d+\s+\d+:\d+:\d+)\s+[\w\.-]+\s+sftp-server/ {
    strptime($date, "Jan _2 15:04:05")
```

Don't try to disassemble timestamps into component parts (e.g. year, month, day) separately.  Keep them in the same format as the log file presents them and change the strptime format string to match it.

不要尝试分开处理一个时间戳的多个部分，如年、月、日之类的。修改 `strptime` 的模式匹配字符串去统一处理。
```
/^/ +
/(?P<date>\d{4}\/\d{2}\/\d{2} \d{2}:\d{2}:\d{2}) / +
/.*/ +
/$/ {
    strptime($date, "2006/01/02 15:04:05")
```

N.B.  If no timestamp parsing is done, then the reported timestamp of the event
may add some latency to the measurement of when the event really occurred.
Between your program logging the event, and mtail reading it, there are many
moving parts: the log writer, some system calls perhaps, some disk IO, some
more system calls, some more disk IO, and then mtail's virtual machine
execution.  While normally negligible, it is worth stating in case users notice
offsets in time between what mtail reports and the event really occurring.  For
this reason, it's recommended to always use the log file's timestamp if one is
available.

特别注意：如果没有处理时间戳，那么为事件分配的时间戳可能会引入一定的延迟。在应用程序记录事件
到mtail读到它并分配时间戳可能已经经过了一段时间：日志记录、系统调用、磁盘IO、更多的系统调用
更多的磁盘IO，以及mtail的虚拟机调用。虽然通常可以忽略不计，但是有时候也可能产生用户能够注意
到的延迟。因此，推荐尽可能处理日志文件中提供的时间戳。

## Repeating common timestamp parsing
## 重复的时间戳处理

The decorator syntax was designed with common timestamp parsing in mind.  It
allows the code for getting the timestamp out of the log line to be reused and
make the rest of the program text more readable and thus maintainable.

装饰器语法在设计时考虑了常见的时间戳解析。允许代码重用提取时间戳的逻辑。

```
# The `syslog' decorator defines a procedure.  When a block of mtail code is
# `syslog` 装饰器定义了一个程序，当一个mtail程序块被装饰了，那么这个程序会在该程序块
# "decorated", it is called before entering the block.  The block is entered
# 之前执行。在关键词 `next` 处执行被装饰的程序块。
# when the keyword `next' is reached.
def syslog {
    /(?P<date>(?P<legacy_date>\w+\s+\d+\s+\d+:\d+:\d+)|(?P<rfc3339_date>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}.\d+[+-]\d{2}:\d{2}))/ +
        /\s+(?:\w+@)?(?P<hostname>[\w\.-]+)\s+(?P<application>[\w\.-]+)(?:\[(?P<pid>\d+)\])?:\s+(?P<message>.*)/ {
        # If the legacy_date regexp matched, try this format.
        # 如果 legacy_date 表达式被匹配上了，则尝试这个模式
        len($legacy_date) > 0 {
            strptime($legacy_date, "Jan _2 15:04:05")
        }
        # If the RFC3339 style matched, parse it this way.
        # 或者这个
        len($rfc3339_date) > 0 {
            strptime($rfc3339_date, "2006-01-02T15:04:05-07:00")
        }
        # Call into the decorated block
        next
    }
}
```

This can be used around any blocks later in the program.

然后就可以在程序之后的任意程序块使用

```
@syslog {
/foo/ {
  ...
}

/bar/ {
}
} # end @syslog decorator
```

Both the foo and bar pattern actions will have the syslog timestamp parsed from
them before being called.

这些程序块在都能拿到调用前就解析好的时间戳。

## Conditional structures
## 条件结构

The `/pattern/ { action }` idiom is the normal conditional control flow structure in `mtail` programs.

`/模式/ { 操作 }` 表达式是常见的条件控制流。

If the pattern matches, then the actions in the block are executed.  If the
pattern does not match, the block is skipped.

如果这个模式匹配，那么大括号中的操作就会被执行，反之跳过。

The `else` keyword allows the program to perform action if the pattern does not match.

也可以加上`else`关键词允许执行反向操作。

```
/pattern/ {
  action
} else {
  alternative
}
```

The example above would execute the "alternative" block if the pattern did not
match the current line.

上面的代码会在模式不配时执行 `alternative` 代码块。

The `otherwise` keyword can be used to create control flow structure
reminiscent of the C `switch` statement.  In a containing block, the
`otherwise` keyword indicates that this block should be executed only if no
other pattern in the same scope has matched.

`otherwise` 关键词可以用来创建类似C语言怀旧风格的 `switch` 分支流。

```
{
/pattern1/ { _action1_ }
/pattern2/ { _action2_ }
otherwise { _action3_ }
}
```

In this example, "action3" would execute if both pattern1 and pattern2 did not
match the current line.

在这里，如果前面两个模式没有匹配上，那么最后一个 "action3" 就会被执行。

## Storing intermediate state
## 存储中间状态

Hidden metrics are metrics that can be used for internal state and are never
exported outside of `mtail`.  For example if the time between pairs of log
lines needs to be computed, then a hidden metric can be used to record the
timestamp of the start of the pair.

隐藏的指标用来统计内部状态但是不会由 `mtail` 输出。例如想计算两行日志之间经过的时间，那么
可以用一个隐藏的指标来记录起始的时间。

**Note** that the `timestamp` builtin _requires_ that the program has set a log
line timestamp with `strptime` or `settime` before it is called.

**注意** 内置的 `timestamp` _要求_ 程序在调用之前就已经使用 `strptime` 以及 `settime` 
对该行日志设置了时间戳。

```
hidden gauge connection_time by pid
使用pid测量的隐变量connection_time
...

  # Connection starts
  # 连接开始的日志模板
  /connect from \S+ \(\d+\.\d+\.\d+\.\d+\)/ {
    connections_total++

    # Record the start time of the connection, using the log timestamp.
    # 使用时间戳记下这个连接的时间
    connection_time[$pid] = timestamp()
  }

...

  # Connection summary when session closed
  # 在连接结束时统计连接
  /sent (?P<sent>\d+) bytes  received (?P<received>\d+) bytes  total size \d+/ {
    # Sum total bytes across all sessions for this process
    # 统计该进程在所有连接发送的总字节数
    bytes_total["sent"] += $sent
    bytes_total["received"] += $received
    
    # Count total time spent with connections open, according to the log timestamp.
    # 统计该连接总得耗时，通过计算时间戳实现
    connection_time_total += timestamp() - connection_time[$pid]

    # Delete the datum referenced in this dimensional metric.  We assume that
    # this will never happen again, and hint to the VM that we can garbage
    # collect the memory used.
    # 删除该指标维度的数据引用。我们假设后面不会再有这个连接，因此通知VM可以回收垃圾。
    del connection_time[$pid]
  }
```

In this example, the connection timestamp is recorded in the hidden variable
`connection_time` keyed by the "pid" of the connection.  Later when the
connection end is logged, the delta between the current log timestamp and the
start timestamp is computed and added to the total connection time.

在这个例子中，连接的时间戳存放在隐变量 `connection_time` 中，由 pid 索引。在记录到
连接结束时，当前时间戳和起始的时间戳可以求差再加到总连接耗时中。

In this example, the average connection time can be computed in a collection
system by taking the ratio of the number of connections (`connections_total`)
over the time spent (`connection_time_total`).  For example
in [Prometheus](http://prometheus.io) one might write:

在这个例子中，平均连接时间可以在一个收集系统中，用连接数 (`connections_total`) 和
总连接时间 (`connection_time_total`) 的比值求得。
```
connection_time_10s_moving_avg = 
  rate(connections_total[10s])
    / on job
  rate(connection_time_total[10s])
```

Note also that the `del` keyword is used to signal to `mtail` that the
connection_time value is no longer needed.  This will cause `mtail` to delete
the datum referenced by that label from this metric, keeping `mtail`'s memory
usage under control and speeding up labelset search time (by reducing the
search space!)

注意 `del` 关键词用于通知 `mtail` 变量不再使用，会导致指标中根据该变量标签引用的原始数据
不再有效，能够控制内存用量，并加速标签的搜索速度。

Alternatively, the statement `del connection_time[$pid] after 72h` would do the
same, but only if `connection_time[$pid]` is not changed for 72 hours.  This
form is more convenient when the connection close event is lossy or difficult
to determine.

或者，`del connection_time[$pid] after 72h` 能够产生相同的效果，不过只会在72小时内
没有使用才会删掉。这个形式在连接关闭事件可能丢失，或者难以判断的时候更加好用。

See [state](state.md) for more information.

## Computing moving averages
## 计算滑动平均

`mtail` deliberately does not implement complex mathematical functions.  It
wants to process a log line as fast as it can.  Many other products on the
market already do complex mathematical functions on timeseries data,
like [Prometheus](http://prometheus.io) and [Riemann](http://riemann.io), so
`mtail` defers that responsibility to them.  (Do One Thing, and Do It Pretty
Good.)

`mtail` 故意不支持复杂的数学操作，因为更重要的是处理每一行日志的速度。很多其他产品都能够在
时序数据上执行复杂的数学函数，如 Prometheus 和 Riemann。

But say you still want to do a moving average in `mtail`.  First note that
`mtail` has no history available, only point in time data.  You can update an
average with a weighting to make it an exponential moving average (EMA).

假如说还是想算滑动平均的话，首先要留意到， `mtail` 不会存放历史数据，只有每个时间点的数据。你可以
用指数平均方式去更新一个变量。

```
gauge average

/some (\d+) match/ {
  # Use a smoothing constant 2/(N + 1) to make the average over the last N observations
  average = 0.9 * $1 + 0.1 * average
}
```

However this doesn't take into account the likely situation that the matches arrive irregularly (the time interval between them is not constant.)  Unfortunately the formula for this requires the exp() function (`e^N`) as described here: http://stackoverflow.com/questions/1023860/exponential-moving-average-sampled-at-varying-times .  I recommend you defer this computation to the collection system

注意这种方法不能处理指标到达时间不均匀的情况。

## Histograms
## 直方图

Histograms are preferred over averages in many monitoring howtos, blogs, talks,
and rants, in order to give the operators better visibility into the behaviour
of a system.

相对于平均值，直方图在很多场景都更加实用，能够提供更好的可视化结果。

`mtail` supports histograms as a first class metric kind, and should be created with a list of bucket boundaries:

`mtail` 对于直方图作为第一类指标支持，可以使用一系列边界去构建。

```
histogram foo buckets 1, 2, 4, 8
直方图的构建： 桶为 1 2 4 8 
```
creates a new histogram `foo` with buckets for ranges [0-1), [1-2), [2-4), [4-8), and from 8 to positive infinity.

构建了一个新的 `foo` 直方图，使用的桶为 [0-1), [1-2), [2-4), [4-8) 以及 [8,正无穷)。

> *NOTE: The 0-n and m-+Inf buckets are created automatically.*
> *注意： 前后两个桶是自动构建的

You can put labels on a histogram as well:

还可以给直方图加上标签：

```
histogram apache_http_request_time_seconds buckets 0.005, 0.01, 0.025, 0.05 by server_port, handler, request_method, request_status, request_protocol
```

At the moment all bucket boundaries (excepting 0 and positive infinity) need to be explicitly named (there is no shorthand form to create geometric progressions).

此时直方图除了0和正无穷的桶边界之外，其他的边界都需要显式一一对应地命名。

Assignment to the histogram records the observation:

将直方图和观察记录对应起来：

```
  ###
  # HTTP Requests with histogram buckets.
  #
  apache_http_request_time_seconds[$server_port][$handler][$request_method][$request_status][$request_protocol] = $time_us / 1000000
```

In tools like [Prometheus](http://prometheus.io) these can be manipulated in
aggregate for computing percentiles of response latency.

在类似 Prometheus 的工具中，这个直方图可以用于聚合计算响应延迟的占比。

```
apache_http_request_time:rate10s = rate(apache_http_request_time_seconds_bucket[10s])
apache_http_request_time_count:rate10s = rate(apache_http_request_time_seconds_count[10s])


apache_http_request_time:percentiles = 
  apache_http_request_time:rate10s
    / on (job, port, handler, request_method, request_status, request_protocol)
  apache_http_request_time_seconds_count:rate10s
```

This new timeseries can be plotted to see the percentile bands of each bucket,
for example to visualise the distribution of requests moving between buckets as
the performance of the server changes.

新的时间序列会可视化展现每个桶的百分比边界，可以作为可视化请求分布在服务性能变化时移动的情况。

Further, these timeseries can be used
for
[Service Level](https://landing.google.com/sre/book/chapters/service-level-objectives.html)-based
alerting (a technique for declaring what a defensible service level is based on
the relative costs of engineering more reliability versus incident response,
maintenance costs, and other factors), as we can now see what percentage of
responses fall within and without a predefined service level:

这些时间序列可以用于基于服务水平的告警，即一种根据平衡可维护性和可靠性之间的花费来决定可达到
服务水平的技术。通过直方图我们可以看到落在一个预定义的服务水平中请求的占比。

```
apache_http_request_time:latency_sli = 
  apache_http_request_time:rate10s{le="200"}
    / on (job, port, handler, request_method, request_status, request_protocol)
  apache_http_request_time_seconds_count:rate10s

ALERT LatencyTooHigh
IF apache_http_request_time:latency_sli < 0.555555555
LABELS { severity="page" }
ANNOTATIONS {
  summary = "Latency is missing the service level objective"
  description = "Latency service level indicator is {{ $value }}, which is below nine fives SLO."
}
```

In this example, prometheus computes a service level indicator of the ratio of
requests at or below the target of 200ms against the total count, and then
fires an alert if the indicator drops below nine fives.

在这个例子中，prometheus 利用 200ms 以上请求的占比构建了一个服务水平指示器。如果指标掉到 5/9 
则开始告警。


## Parsing number fields that are sometimes not numbers
## 处理有时候不是数字的数字字段

Some logs, for example Varnish and Apache access logs, use a hyphen rather than a zero.

有一些日志，如 Varnish 和 Apache 的接入日志，用一个连字符而不是填充0.

You may be tempted to use a programme like 你可能忍不住用：

```
counter total

/^[a-z]+ ((?P<response_size>\d+)|-)$/ {
  $response_size > 0 {
    total = $response_size
  }
}
```

to parse a log like 来处理类似这样的日志：

```
a 99
b -
```

except that `mtail` will issue a runtime error on the second line like `Runtime error: strconv.ParseInt: parsing "": invalid syntax`.

在第二行日志，`mtail` 会抛出一个运行异常，如 `Runtime error: strconv.ParseInt: parsing "": invalid syntax`. （应该是 $response_size 没有值的原因？）

This is because in this programme the capture group is only matching on a set of digits, and is not defined when the alternate group matches (i.e. the hyphen).

这是因为在这个程序中，捕获组仅仅匹配一系列的数字，并且没有定义备用的匹配模式，如连字符。

Instead one can test the value of the surrounding capture group and do nothing if the value matches a hyphen:

因此可以判断一下捕获到的是啥，如果是连字符就啥也不干。

```
counter total

/^[a-z]+ ((?P<response_size>\d+)|-)$/ {
  $1 != "-" {
    total = $response_size
  }
}
```

`mtail` does not presently have a way to test if a capture group is defined or not.

`mtail` 不会显式去测试一个捕获组有没有定义。

# Avoiding unnecessary work
# 避免没用操作

You can stop the program if it's fed data from a log file you know you want to ignore:

你可以在输入是一个你想忽略的文件名时，把程序停掉。

```
getfilename() !~ /apache.access.?log/ {
  stop
}
```

This will check to see if the input filename looks like
`/var/log/apache/accesslog` and not attempt any further pattern matching on the
log line if it doesn't.

这样会去看输入的用户名是不是看起来像 `/var/log/apache/accesslog` 然后不会去解析字段。

