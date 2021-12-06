# Keeping state in mtail programs
# 在mtail中保存状态

The program is run on each log line from start to finish, with no loops.  The only state emitted by the program is the content of the exported metrics.  Metrics can be read by the program, though, so exported metrics are the place to keep state between lines of input.

程序在每行日志中从头到尾运行，并且没有循环。程序使用的唯一状态变量是对外指标的内容。指标可以由
程序读取，所以对外指标可以用于在不同行的输入之间保存状态。

It's often the case that a log line is printed by an application at the start of some session-like interaction, and another at the end.  Often these sessions have a session identifier, and every intermediate event in the same session is tagged with that identifier.  Using map-valued exported metrics is the way to store session information keyed by session identifier.

通常会有这种情况，程序在一些类似会话之类的互动事件开始时输出了一条日志，然后又在结尾的时候输出
另一条日志。一般情况下，这种会话有一个可以区分的会话id，然后在相关的日志中，都会标记上该id。使用
映射(map-valued)的对外指标，能够以id为键，存放每个会话的数据。

The example program [`rsyncd.mtail`](../examples/rsyncd.mtail) shows how to use a session tracking metric for measuring the total user session time.

以下示例展示如何使用状态变量追踪会话的指标。

    counter connection_time_total
    hidden gauge connection_time by pid

    /connect from \S+/ {
      connection_time[$pid] = timestamp()

      del connection_time[$pid] after 72h
    }

    /sent .* bytes  received .* bytes  total size \d+/ {
      connection_time_total += timestamp() - connection_time[$pid]
      del connection_time[$pid]
    }

`rsyncd` uses a child process for each session so the `pid` field of the log format contains the session identifier in this example.

`rsyncd` 使用子进程处理每个会话，所以 pid 就可以用来区分会话。

## hidden metrics
## 隐藏指标

A hidden metric is only visible to the mtail program, it is hidden from export.   Internal state can be kept out of the metric collection system to avoid unnecessary memory and network costs.

隐藏指标只能被mtail程序访问，不能被暴露到外界。内部状态可以在指标收集系统之外保存，减少不必要的内存
使用和网络耗费。


Hidden metrics are declared by prepending the word `hidden` to the declaration:

在声明前方放置 `hidden` 能够声明隐藏指标

    hidden gauge connection_time by pid

## Removing session information at the end of the session
## 在会话结尾删除会话信息

The maps can grow unbounded with a key for every session identifier created as the logs are read.  If you see `mtail` consuming a lot of memory, it is likely that there's one or more of these maps consuming memory.

如果每个会话的id都存到映射中，这个映射会没有上限地变大，如果你发现mtail吃掉了一大堆内存，很可能
一个或者多个映射吃掉了内存。

(You can remove the `hidden` keyword from the declaration, and let `mtail` reload the program without restarting and the contents of the session information metric will appear on the exported metrics page.  Be warned, that if it's very big, even loading this page may take a long time and cause mtail to crash.)

你也可以删掉 `hidden` 这样让 `mtail` 在不重启的情况下重新加载程序，然后会话信息指标的内存就会
在暴露的指标页面上出现。事先提醒一下，如果这个映射很大的话，每次加载页面可能会花很长的时间，然后
mtail还可能会崩。

`mtail` can't know when a map value is ready to be garbage collected, so you need to tell it.  One way is to defer deletion of the key and its value if it is not updated for some duration of time.  The other way is to immediately delete it when the key and value are no longer needed.

`mtail` 无法感知映射里面的值是否能够被垃圾回收，所以你需要显式声明。另一个方案是设置过期时间，如果
一段时间没有读写就干掉。

   ```
   del connection_time[$pid] after 72h
   ```

Upon creation of a connection time entry, the `rsyncd.mtail` program instructs mtail to remove it 72 hours after it's no longer updated.  This means that the programmer expects, in this case, that sessions typically do not last longer than 72 hours because `mtail` does not track the timestamps for all accesses of metrics, only writes to them.

在每次记录下连接建立时间后，该程序会告诉mtail在该变量72小时后没有更新，就能够删除该变量。这意味着
程序员预设了一个连接不可能持续72小时这个前提，因为mtail其实不会追踪对变量的读操作，只会追踪写操作。


   ```
   del connection_time[$pid]
   ```

The other form indicates that when the session is closed, the key and value can be removed.  The caveat here is that logs can be lossy due to problems with the application restarting, mtail restarting, or the log delivery system (e.g. syslog) losing the messages too.  Thus it is recommended to use both forms in programs.

另一个声明该连接已经关闭的形式是直接删除。但是如果应用重启的话，或者应用少输出了一行日志，之前保留的时间戳可能就会丢失或者没走到删除这一步而导致内存泄露。因此推荐结合使用这两种形式。

   1. `del ... after` form when the metric is created, giving it an expiration time longer than the expected lifespan of the session. 从变量创建开始计算存活时间，超时就干掉
   1. `del` form when the session is ended, explicitly removing it before the expiration time is up. 能够在还没超时但是会话已经结束的时候显式干掉变量。

It is not an error to delete a nonexistent key from a map.

如果试图在映射中删除一个不存在的键，不会引发错误。

Expiry is only processed once ever hour, so durations shorter than 1h won't take effect until the next hour has passed.

过期时间一小时才算一次，所以如果定时小于一小时的话还是一小时才会生效。