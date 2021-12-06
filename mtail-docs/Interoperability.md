# Introduction

mtail is only part of a monitoring ecosystem -- it fills the gap between applications that export no metrics of their own in a [common protocol](Metrics.md) and the timeseries database.

mtail 只是监控系统的一部分，给很多自己没有监控的程序提供一个公共的发送出口。

# Details
# 细节

mtail actively exports (i.e. pushes) to the following timeseries databases:
mtail 支持推送到下列时间序列数据库：

  * [collectd](http://collectd.org/)
  * [graphite](http://graphite.wikidot.com/start)
  * [statsd](https://github.com/etsy/statsd)

mtail also is a passive exporter (i.e. pull, or scrape based) by:
也可以是被动拉取：

  * [Prometheus](http://prometheus.io)
  * Google's Borgmon


# Logs Analysis
# 日志分析

While `mtail` does a form of logs analysis, it does _not_ do any copying,
indexing, or searching of log files for data mining applications.  It is only
intended for real- or near-time monitoring data for the purposes of performance
measurement and alerting.

虽然 `mtail` 可以做一些日志分析，但是不会去做任何的 _复制，索引，搜索_ 等数据挖掘功能。只
实现实时或者接近实时的监控与告警功能。

Instead, see logs ingestion and analysis systems like

一般使用日志移动和分析系统实现高级功能：

  * [Logstash](https://www.elastic.co/products/logstash)
  * [Graylog](https://www.graylog.org/)

if that is what you need.

# Prometheus Exporter Metrics
# 基于 Prometheus Exporter 输出指标

https://prometheus.io/docs/instrumenting/writing_exporters/ describes useful metrics for a Prometheus exporter to export. `mtail` does not follow that guide, for these reasons.

[Prometheus](https://prometheus.io/docs/instrumenting/writing_exporters/) 提供了一些指标类型
，但是mtail不会严格按照该说明执行，因为下列原因：

The exporter model described in that document is for active proxies between an application and Prometheus.  The expectation is that when Prometheus scrapes the proxy (the exporter) that it then performs its own scrape of the target application, and translates the results back into the Prometheus exposition format.  The time taken to query the target application is what is exported as `X_scrape_duration_seconds` and its availability as `X_up`.

该文档中描述的 exporter 模型设计为在应用程序和 Prometheus 之间交换数据的主动代理。主动代理的意思
是，在 Prometheus 抓取 exporter 时，exporter 才去抓取目标应用的信息，并转换为 Prometheus 希望
的格式返回。用于抓取目标应用信息的耗时，一般会标为 `X_scrape_duration_seconds` 以及目标应用的
可用性会标为 `X_up` 。

`mtail` doesn't work like that.  It is reacting to the input log events, not scrapes, and so there is no concept of how long it takes to query the application or if it is available.  There are things that, if you squint, look like applications in `mtail`, the virtual machine programs.  They could be exporting their time to process a single line, and are `up` as long as they are not crashing on input.  This doesn't translate well into the exporter metrics meanings though.

`mtail` 不这样干。它根据输入日志事件构建指标，而不是抓取。因此并没有抓取耗时的概念。粗略点说，
这个过程有点像应用跑在 `mtail` 这台虚拟机里，即仅仅能测量处理每行日志的时间，并且其运行时间指的是
对于输入有多久没有崩溃过了。因此我们可以说 `mtail` 并没有完整利用 exporter 规范的语义。

TODO(jaq): Instead, mtail will export a histogram of the runtime per line of each VM program.

`mtail` doesn't export `mtail_up` or `mtail_scrape_duration_seconds` because they are exactly equivalent* the synthetic metrics that Prometheus creates automatically: https://prometheus.io/docs/concepts/jobs_instances/

`mtail` 不会输出  `mtail_up` 和 `mtail_scrape_duration_seconds` 因为他们和 Prometheus 自动
创建的[合成指标](https://prometheus.io/docs/concepts/jobs_instances/)是等价的。

\* The difference between a scrape duration measured in mtail versus Prometheus would differ in the network round trip time, TCP setup time, and send/receive queue time.  For practical purposes you can ignore them as the usefulness of a scrape duration metric is not in its absolute value, but how it changes over time.

\* mtail 和 Prometheus 测量的抓取时间因为TCP啊网络之类的各种耗时会不太一样。实际上这些差异可以
忽略，因为抓取时间衡量的关键不在于其绝对值，而是在它是否随着时间变化了。