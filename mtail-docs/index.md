mtail - extract whitebox monitoring data from application logs for collection into a timeseries database
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

Read more about `mtail` in the [Programming Guide](Programming-Guide.md), [Language](Language.md), [Building from source](Building.md) from source, help for [Interoperability](Interoperability.md) with other monitoring system components, and [Deploying](Deploying.md) and [Troubleshooting](Troubleshooting.md)

Mailing list: https://groups.google.com/forum/#!forum/mtail-users
