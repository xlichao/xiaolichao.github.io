mtail - 无侵入地从应用日志提取监控信息写入时序数据库
========================================================================================================

mtail is a tool for extracting metrics from application logs to be exported into a timeseries database or timeseries calculator for alerting and dashboarding.

It aims to fill a niche between applications that do not export their own internal state, and existing monitoring systems, without patching those applications or rewriting the same framework for custom extraction glue code.

The extraction is controlled by `mtail` programs which define patterns and actions:

    # simple line counter
    counter lines_total
    /$/ {
      lines_total++
    }

Metrics are exported for scraping by a collector as JSON or Prometheus format
over HTTP, or can be periodically sent to a collectd, statsd, or Graphite
collector socket.

Read more about `mtail` in the [编程指南](Programming-Guide.md), [编程语言描述](Language.md), [Building from source](Building.md) from source, help for [数据接入](Interoperability.md) with other monitoring system components, and [Deploying](Deploying.md) and [Troubleshooting](Troubleshooting.md)

以下页面亦提供中文：

* [监控指标](Metrics.md) 解释如何定义指标、变量类型，以及如何添加指标的维度标签。
* [状态变量](state.md) 主要描述如何在 mtail 中保存状态，以统计发生在不同日志之间的事件。


Mailing list: https://groups.google.com/forum/#!forum/mtail-users
