##GO语言消息队列调研

本文档主要是go mq方案的调研，这里主要调研了go语言版本的nsq和kafka的go语言驱动sarama以及对sarama驱动的扩展wvanbergen。很遗憾的是两种方案都无法支持完全有序。

### NSQ

NSQ 是实时的分布式消息处理平台，其设计的目的是用来大规模地处理每天数以十亿计级别的消息。它具有分布式和去中心化拓扑结构，该结构具有无单点故障、故障容错、高可用性以及能够保证消息的可靠传递的特征。

**NSQ** 是实时的分布式消息处理平台，其设计的目的是用来大规模地处理每天数以十亿计级别的消息。

NSQ 具有**分布式**和**去中心化**拓扑结构，该结构具有无单点故障、故障容错、高可用性以及能够保证消息的可靠传递的特征。

#### 架构和消费模型

NSQ 由 3 个守护进程和一些工具组成：

- **nsqd** 是接收、队列和传送消息到客户端的守护进程。

- **nsqlookupd** 是管理的拓扑信息，并提供了最终一致发现服务的守护进程。

  `nsqlookupd`，它提供了一个目录服务，消费者可以查找到提供他们感兴趣订阅话题的`nsqd` 地址 。在配置方面，把消费者与生产者解耦开（它们都分别只需要知道哪里去连接 `nsqlookupd` 的共同实例，而不是对方），降低复杂性和维护。

  在更底的层面，每个 `nsqd` 有一个与 `nsqlookupd` 的长期 TCP 连接，定期推动其状态。这个数据被 `nsqlookupd` 用于给消费者通知 `nsqd` 地址。对于消费者来说，一个暴露的 HTTP `/lookup` 接口用于轮询。

  为话题引入一个新的消费者，只需启动一个配置了 nsqlookup 实例地址的 **NSQ** 客户端。无需为添加任何新的消费者或生产者更改配置，大大降低了开销和复杂性。

- **nsqadmin** 是一个 Web UI 来实时监控集群（和执行各种管理任务）。

  它提供了一个 Web UI 来浏览 topics/channels/consumers 和深度检查每一层的关键统计数据。此外，它还支持几个管理命令例如，移除通道和清空通道（这是一个有用的工具，当在一个通道中的信息可以被安全地扔掉，以使深度返回到 0）。

- **utilities** 常见基础功能、数据流处理工具，如nsq_stat、nsq_tail、nsq_to_file、nsq_to_http、nsq_to_nsq、to_nsq。

![消费模型]({{site.github.url}}/assets/go-mq/design.gif)

NSQ的消费模型与其他的消息队列的消费模型没有太大的区别，订阅的基本单位还是topic，只是这里使用了channels的概念替换了group。同一个topic的数据会被每个channels独立消费，即每个channels都可以获取到该topic的所有数据，一个channel内部的多个consumer再根据策略分配这些数据；

#### 安装及简单试用

NSQ官方提供了多个平台的安装包，可以直接下载使用，这里下载了linux下64位的nsq进行安装测试。[下载地址](http://nsq.io/deployment/installing.html)

1. 在一个终端中运行;

   ```shell
   ./nsqlookupd
   ```

2. 在第二个终端中运行;

   ```shell
   ./nsqd --lookupd-tcp-address=127.0.0.1:4160
   ```

3. 在第三个终端中运行;

   ```shell
   ./nsqadmin --lookupd-http-address=127.0.0.1:4161
   ```

4. 在第四个shell中推送一条初始化数据(并且在集群中创建一个 topic):

   ```shell
   curl -d 'hello world 1' 'http://127.0.0.1:4151/put?topic=test'
   ```

5. 在第五个终端中启动一个消费者;

   ```shell
   nsq_to_file --topic=test --output-dir=../data --lookupd-http-address=127.0.0.1:4161
   ```

6. 在第六个终端中推送更多的消息;

   ```shell
   curl -d 'hello world 2' 'http://127.0.0.1:4151/put?topic=test'
   curl -d 'hello world 3' 'http://127.0.0.1:4151/put?topic=test'
   ```

7. 可以下消费者目录中看到消费到的数据;

#### 内部原理

##### topic

每个话题维护着 3 个主要的 goroutines。

第一个被称为 `router`，它负责用来从 incoming go-chan 读取最近发布的消息，并把消息保存到队列中（内存或硬盘）。

第二个，称为 `messagePump`，是负责复制和推送消息到如上所述的通道。

第三个是负责 DiskQueue IO 和将在后面讨论。

![topic]({{site.github.url}}/assets/go-mq/topic.png)

##### 高可用

NSQ被设计以分布的方式被使用。`nsqd` 客户端（通过 TCP ）连接到指定话题的所有生产者实例。没有中间人，没有消息代理，也没有单点故障：

![nsq clients]({{site.github.url}}/assets/go-mq/单点.png)

这种拓扑结构消除单链，聚合，反馈。相反，你的消费者直接访问所有生产者。*从技术上讲*，哪个客户端连接到哪个**NSQ** 不重要，只要有足够的消费者连接到所有生产者，以满足大量的消息，保证所有东西最终将被处理。

对于 `nsqlookupd`，高可用性是通过运行多个实例来实现。他们不直接相互通信和数据被认为是最终一致。消费者轮询所有的配置的 `nsqlookupd` 实例和合并 response。失败的，无法访问的，或以其他方式故障的节点不会让系统陷于停顿。

##### 消息传递担保

**NSQ** 保证消息将交付至少一次，虽然消息可能是重复的。消费者应该关注到这一点，删除重复数据或执行幂等操作。这个担保是作为协议和工作流的一部分，工作原理如下（假设客户端成功连接并订阅一个话题）：

1. 客户表示他们已经准备好接收消息
2. **NSQ** 发送一条消息，并暂时将数据存储在本地（在 re-queue 或 timeout）
3. 客户端回复 FIN（结束）或 REQ（重新排队）分别指示成功或失败。如果客户端没有回复, **NSQ** 会在设定的时间超时，自动重新排队消息

这确保了消息丢失唯一可能的情况是不正常结束 `nsqd` 进程。在这种情况下，这是在内存中的任何信息（或任何缓冲未刷新到磁盘）都将丢失。

如何防止消息丢失是最重要的，即使是这个意外情况可以得到缓解。一种解决方案是构成冗余 `nsqd`对（在不同的主机上）接收消息的相同部分的副本。因为你实现的消费者是幂等的，以两倍时间处理这些消息不会对下游造成影响，并使得系统能够承受任何单一节点故障而不会丢失信息。

##### 内存限制

`nsqd` 提供一个 `--mem-queue-size` 配置选项，这将决定一个队列保存在内存中的消息数量。如果队列深度超过此阈值，消息将透明地写入磁盘。`nsqd` 进程的内存占用被限定于 `--mem-queue-size * #of_channels_and_topics`：

![message overflow]({{site.github.url}}/assets/go-mq/内存限制.png)

此外，一个精明的观察者可能会发现，这是一个方便的方式来获得更高的传递保证：把这个值设置的比较低（如 1 或甚至是 0）。磁盘支持的队列被设计为在不重启的情况下存在（虽然消息可能被传递两次）。

此外，涉及到信息传递保证，干净关机（通过给 `nsqd` 进程发送 TERM 信号）坚持安全地把消息保存在内存中，传输中，延迟，以及内部的各种缓冲区。

请注意，一个以 `#ephemeral` 结束的通道名称不会在超过 `mem-queue-size` 之后刷新到硬盘。这使得消费者并不需要订阅频道的消息担保。这些临时通道将在最后一个客户端断开连接后消失。

##### 效率

**NSQ** 被设计成一个使用简单 size-prefixed 为前缀的，与“memcached-like”类似的命令协议。所有的消息数据被保持在核心中，包括像尝试次数、时间截等元数据类。这消除了数据从服务器到客户端来回拷贝，当重新排队消息时先前工具链的固有属性。这也简化了客户端，因为他们不再需要负责维护消息的状态。

此外，通过降低配置的复杂性，安装和开发的时间大大缩短（尤其是在有超过 > 1 消费者的话题）。

对于数据的协议，我们做了一个重要的设计决策，通过推送数据到客户端最大限度地提高性能和吞吐量的，而不是等待客户端拉数据。这个概念，我们称之为 `RDY` 状态，基本上是客户端流量控制的一种形式。

当客户端连接到 `nsqd` 和并订阅到一个通道时，它被放置在一个 `RDY` 为 0 状态。这意味着，还没有信息被发送到客户端。当客户端已准备好接收消息发送，更新它的命令 RDY 状态到它准备处理的数量，比如 100。无需任何额外的指令，当 100 条消息可用时，将被传递到客户端（服务器端为那个客户端每次递减 RDY 计数）。

客户端库的被设计成在 `RDY` 数达到配置 `max-in-flight` 的 25% 发送一个命令来更新 RDY 计数（并适当考虑连接到多个 `nsqd` 情况下，适当地分配）。

![nsq protocol]({{site.github.url}}/assets/go-mq/效率.png)

这是一个重要的性能控制，使一些下游系统能够更轻松地批量处理信息，并从更高的 `max-in-flight` 中受益。

值得注意的是，因为它既是基于缓冲和推送来满足需要(通道)流的独立副本的能力，我们已经提供了行为像`simplequeue` 和 pubsub 相结合的守护进程。这是简化我们的系统拓扑结构的强大工具，如上述讨论那样我们会维护传统的 toolchain。

##### 部署结构

![]({{site.github.url}}/assets/go-mq/推荐部署结构.png)

#### 配置和工具说明

##### nsqd

`nsqd` 是一个守护进程，负责接收，排队，投递消息给客户端。

它可以独立运行，不过通常它是由 `nsqlookupd` 实例所在集群配置的（它在这能声明 topics 和 channels，以便大家能找到）。

它在 2 个 TCP 端口监听，一个给客户端，另一个是 HTTP API。同时，它也能在第三个端口监听 HTTPS。

###### 参数说明

```
-auth-http-address=: <addr>:<port> 查询授权服务器 (可能会给多次)
-broadcast-address="": 通过 lookupd  注册的地址（默认名是 OS）
-config="": 配置文件路径
-data-path="": 缓存消息的磁盘路径
-deflate=true: 运行协商压缩特性（客户端压缩）
-e2e-processing-latency-percentile=: 消息处理时间的百分比（通过逗号可以多次指定，默认为 none）
-e2e-processing-latency-window-time=10m0s: 计算这段时间里，点对点时间延迟（例如，60s 仅计算过去 60 秒）
-http-address="0.0.0.0:4151": 为 HTTP 客户端监听 <addr>:<port>
-https-address="": 为 HTTPS 客户端 监听 <addr>:<port>
-lookupd-tcp-address=: 解析 TCP 地址名字 (可能会给多次)
-max-body-size=5123840: 单个命令体的最大尺寸
-max-bytes-per-file=104857600: 每个磁盘队列文件的字节数
-max-deflate-level=6: 最大的压缩比率等级（> values == > nsqd CPU usage)
-max-heartbeat-interval=1m0s: 在客户端心跳间，最大的客户端配置时间间隔
-max-message-size=1024768: (弃用 --max-msg-size) 单个消息体的最大字节数
-max-msg-size=1024768: 单个消息体的最大字节数
-max-msg-timeout=15m0s: 消息超时的最大时间间隔
-max-output-buffer-size=65536: 最大客户端输出缓存可配置大小(字节）
-max-output-buffer-timeout=1s: 在 flushing 到客户端前，最长的配置时间间隔。
-max-rdy-count=2500: 客户端最大的 RDY 数量
-max-req-timeout=1h0m0s: 消息重新排队的超时时间
-mem-queue-size=10000: 内存里的消息数(per topic/channel)
-msg-timeout="60s": 自动重新队列消息前需要等待的时间
-snappy=true: 打开快速选项 (客户端压缩)
-statsd-address="": 统计进程的 UDP <addr>:<port>
-statsd-interval="60s": 从推送到统计的时间间隔
-statsd-mem-stats=true: 切换发送内存和 GC 统计数据
-statsd-prefix="nsq.%s": 发送给统计keys 的前缀(%s for host replacement)
-sync-every=2500: 磁盘队列 fsync 的消息数
-sync-timeout=2s: 每个磁盘队列 fsync 平均耗时
-tcp-address="0.0.0.0:4150": TCP 客户端 监听的 <addr>:<port>
-tls-cert="": 证书文件路径
-tls-client-auth-policy="": 客户端证书授权策略 ('require' or 'require-verify')
-tls-key="": 私钥路径文件
-tls-required=false: 客户端连接需求 TLS
-tls-root-ca-file="": 私钥证书授权 PEM 路径
-verbose=false: 打开日志
-version=false: 打印版本
-worker-id=0: 进程的唯一码(默认是主机名的哈希值)
```

###### API

- `/ping` - 活跃度
- `/info` - 版本
- `/stats` - 检查综合运行
- `/pub` - 发布消息到话题（topic)
- `/mpub- 发布多个消息到话题（topic)
- `/debug/pprof` - pprof 调试入口
- `/debug/pprof/profile` - 生成 pprof CPU 配置文件
- `/debug/pprof/goroutine` - 生成 pprof 计算配置文件
- `/debug/pprof/heap`- 生成 pprof 堆配置文件
- `/debug/pprof/block`- 生成 pprof 块配置文件
- `/debug/pprof/threadcreate` - 生成 pprof OS 线程配置文件

`v1` 命名空间 (as of `nsqd` `v0.2.29+`):

- `/topic/create` - 创建一个新的话题（topic)
- `/topic/delete`- 删除一个话题（topic)
- `/topic/empty` - 清空话题（topic)
- `/topic/pause` - 暂停话题（topic)的消息流
- `/topic/unpause` - 恢复话题（topic)的消息流
- `/channel/create` - 创建一个新的通道（channel)
- `/channel/delete` - 删除一个通道（channel)
- `/channel/empty` - 清空一个通道（channel)
- `/channel/pause` - 暂停通道（channel)的消息流
- `/channel/unpause` - 恢复通道（channel)的消息流

[详细说明](http://wiki.jikexueyuan.com/project/nsq-guide/nsqd.html)

##### nsqlookupd

`nsqlookupd` 是守护进程负责管理拓扑信息。客户端通过查询 `nsqlookupd` 来发现指定话题（topic）的生产者，并且`nsqd` 节点广播话题（topic）和通道（channel）信息。

有两个接口：TCP 接口，`nsqd` 用它来广播。HTTP 接口，客户端用它来发现和管理。

###### 参数说明

```
-http-address="0.0.0.0:4161": <addr>:<port> 监听 HTTP 客户端
-inactive-producer-timeout=5m0s: 从上次 ping 之后，生产者驻留在活跃列表中的时长
-tcp-address="0.0.0.0:4160": TCP 客户端监听的 <addr>:<port> 
-broadcast-address: 这个 lookupd 节点的外部地址, (默认是 OS 主机名)
-tombstone-lifetime=45s: 生产者保持 tombstoned  的时长
-verbose=false: 允许输出日志
-version=false: 打印版本信息
```

###### API

* `/lookup` - 返回某个话题（topic）的生产者列表。
* `/topics` - 返回所有已知的话题（topic）。
* `/channels` - 返回所有的channel。
* `/nsqd` - 返回所有已知的 `nsqd` 列表。
* `/delete_topic` - 删除一个已有的topic。
* `/delete_channel` - 删除一个已有的channel。
* `/tombstone_topic_producer` - 逻辑删除（Tombstones）某个话题（topic）的生产者。
* `/ping` - 健康检查。
* `/info` - 版本信息。

[详细说明](http://wiki.jikexueyuan.com/project/nsq-guide/nsqlookupd.html)

##### nsqadmin

`nsqadmin` 是一套 WEB UI，用来汇集集群的实时统计，并执行不同的管理任务。

###### 参数说明

```
-graphite-url="": URL to graphite HTTP 地址
-http-address="0.0.0.0:4171": <addr>:<port> HTTP clients 监听的地址和端口
-lookupd-http-address=[]: lookupd HTTP 地址 (可能会提供多次)
-notification-http-endpoint="": HTTP 端点 (完全限定) ，管理动作将会发送到
-nsqd-http-address=[]: nsqd HTTP 地址 (可能会提供多次)
-proxy-graphite=false: Proxy HTTP requests to graphite
-template-dir="": 临时目录路径
-use-statsd-prefixes=true: expect statsd prefixed keys in graphite (ie: 'stats_counts.')
-version=false: 打印版本信息
```

##### utilities

###### nsq_stat

为所有的话题（topic）和通道（channel）的生产者轮询 `/stats`.

*参数说明*

```
-channel="": NSQ 通道（channel）
-lookupd-http-address=: lookupd HTTP 地址 (可能会给多次)
-nsqd-http-address=: nsqd HTTP 地址 (可能会给多次)
-status-every=2s: 轮询/打印输出见的时间间隔
-topic="": NSQ 话题（topic）
-version=false: 打印版本
```

###### nsq_tail

消费指定的话题（topic）/通道（channel），并写到 stdout (和 tail(1) 类似)。

*参数说明*

```
-channel="": NSQ 通道（channel）
-consumer-opt=: 传递给 nsq.Consumer (可能会给多次, http://godoc.org/github.com/bitly/go-nsq#Config)
-lookupd-http-address=: lookupd HTTP 地址 (可能会给多次)
-max-in-flight=200: 最大的消息数 to allow in flight
-n=0: total messages to show (will wait if starved)
-nsqd-tcp-address=: nsqd TCP 地址 (可能会给多次)
-reader-opt=: (已经抛弃) 使用 --consumer-opt
-topic="": NSQ 话题（topic）
-version=false: 打印版本信息
```

###### nsq_to_file

消费指定的话题（topic）/通道（channel），并写到文件中，有选择的滚动和/或压缩文件。

*参数说明*

```
-channel="nsq_to_file": nsq 通道（channel）
-consumer-opt=: 传递给 nsq.Consumer 的参数 (可能会给多次, http://godoc.org/github.com/bitly/go-nsq#Config)
-datetime-format="%Y-%m-%d_%H": strftime，和 filename 里 <DATETIME> 格式兼容
-filename-format="<TOPIC>.<HOST><GZIPREV>.<DATETIME>.log": output 文件名格式 (<TOPIC>, <HOST>, <DATETIME>, <GZIPREV> 重新生成. <GZIPREV> 是当已经存在的 gzip 文件的前缀)
-gzip=false: gzip 输出文件
-gzip-compression=3: (已经抛弃) 使用 --gzip-level, gzip 压缩级别(1 = 速度最佳, 2 = 最近压缩, 3 = 默认压缩)
-gzip-level=6: gzip 压缩级别 (1-9, 1=BestSpeed, 9=BestCompression)
-host-identifier="": 输出到 log 文件，提供主机名。 <SHORT_HOST> 和 <HOSTNAME> 是有效的替换者
-lookupd-http-address=: lookupd HTTP 地址 (可能会给多次)
-max-in-flight=200: 最大的消息数 to allow in flight
-nsqd-tcp-address=: nsqd TCP 地址 (可能会给多次)
-output-dir="/tmp": 输出文件所在的文件夹
-reader-opt=: (已经抛弃) 使用 --consumer-opt
-skip-empty-files=false: 忽略写空文件
-topic=: nsq 话题（topic） (可能会给多次)
-topic-refresh=1m0s: 话题（topic）列表刷新的频率是多少？
-version=false: 打印版本信息
```

###### nsq_to_http

消费指定的话题（topic）/通道（channel）和执行 HTTP requests (GET/POST) 到指定的端点。

*参数说明*

```
-channel="nsq_to_http": nsq 通道（channel）
-consumer-opt=: 参数，通过 nsq.Consumer (可能会给多次, http://godoc.org/github.com/bitly/go-nsq#Config)
-content-type="application/octet-stream": the Content-Type 使用d for POST requests
-get=: HTTP 地址 to make a GET request to. '%s' will be printf replaced with data (可能会给多次)
-http-timeout=20s: timeout for HTTP connect/read/write (each)
-http-timeout-ms=20000: (已经抛弃) 使用 --http-timeout=X, timeout for HTTP connect/read/write (each)
-lookupd-http-address=: lookupd HTTP 地址 (可能会给多次)
-max-backoff-duration=2m0s: (已经抛弃) 使用 --consumer-opt=max_backoff_duration,X
-max-in-flight=200: 最大的消息数 to allow in flight
-mode="round-robin": the upstream request mode options: multicast, round-robin, hostpool
-n=100: number of concurrent publishers
-nsqd-tcp-address=: nsqd TCP 地址 (可能会给多次)
-post=: HTTP 地址 to make a POST request to.  data will be in the body (可能会给多次)
-reader-opt=: (已经抛弃) 使用 --consumer-opt
-round-robin=false: (已经抛弃) 使用 --mode=round-robin, enable round robin mode
-sample=1: % of messages to publish (float b/w 0 -> 1)
-status-every=250: the # of requests between logging status (per handler), 0 disables
-throttle-fraction=1: (已经抛弃) 使用 --sample=X, publish only a fraction of messages
-topic="": nsq 话题（topic）
-version=false: 打印版本信息
```

###### nsq_to_nsq

消费者指定的话题/通道和重发布消息到目的地 `nsqd` 通过 TCP。

*参数说明*

```
-channel="nsq_to_nsq": nsq 通道（channel）
-consumer-opt=: 参数，通过 nsq.Consumer (可能会给多次, see http://godoc.org/github.com/bitly/go-nsq#Config)
-destination-nsqd-tcp-address=: destination nsqd TCP 地址 (可能会给多次)
-destination-topic="": destination nsq 话题（topic）
-lookupd-http-address=: lookupd HTTP 地址 (可能会给多次)
-max-backoff-duration=2m0s: (已经抛弃) 使用 --consumer-opt=max_backoff_duration,X
-max-in-flight=200: 允许 flight 最大的消息数
-mode="round-robin": 上行请求的参数: round-robin (默认), hostpool
-nsqd-tcp-address=: nsqd TCP 地址 (可能会给多次)
-producer-opt=: 传递到 nsq.Producer (可能会给多次, 参见 http://godoc.org/github.com/bitly/go-nsq#Config)
-reader-opt=: (已经抛弃) 使用 --consumer-opt
-require-json-field="": JSON 消息: 仅传递消息，包含这个参数
-require-json-value="": JSON 消息: 仅传递消息要求参数有这个值 
-status-every=250: # 请求日志的状态(每个目的地), 0 不可用
-topic="": nsq 话题（topic）
-version=false: 打印版本信息
-whitelist-json-field=: JSON 消息: 传递这个字段 (可能会给多次)
```

###### to_nsq

采用 stdin 流，并分解到新行（默认），通过 TCP 重新发布到目的地 `nsqd`。

*参数说明*

```
-delimiter="\n": 分割字符串(默认'\n')
-nsqd-tcp-address=: 目的地 nsqd TCP 地址 (可能会给多次)
-producer-opt=: 参数，通过 nsq.Producer (可能会给多次, http://godoc.org/github.com/bitly/go-nsq#Config)
-topic="": 发布到的 NSQ 话题（topic）
```

#### 示例代码

```shell
go get github.com/nsqio/go-nsq
```

##### 生成者

```go
package main

import (
	"log"
	"github.com/nsqio/go-nsq"
)

func main() {
	config := nsq.NewConfig()
	w, err := nsq.NewProducer("192.168.45.170:4150", config)
	if err != nil {
		log.Panic("Could not create producer.")
	}
	defer w.Stop()
	for i := 0; i < 100; i++ {
		err := w.Publish("write_test", []byte("test"))
		if err != nil {
			log.Panic("Could not connect.")
		}
	}
	w.Stop()
}
```

##### 消费者

```go
package main

import (
	"log"
	"github.com/nsqio/go-nsq"
	"time"
)

func main() {

	config := nsq.NewConfig()
	q, err := nsq.NewConsumer("write_test", "ch", config)
	if err != nil {
		log.Panic("Could not create consumer.")
	}
	defer q.Stop()

	q.AddHandler(nsq.HandlerFunc(func(message *nsq.Message) error {
		log.Printf("Got a message: %v", string(message.Body))
		return nil
	}))
	err = q.ConnectToNSQD("192.168.45.170:4150")
	if err != nil {
		log.Panic("Could not connect")
	}

	time.Sleep(3600 * time.Second)

}
```

##### 运行截图

![]({{site.github.url}}/assets/go-mq/client.png)

#### 性能测试

GOMAXPROCS=1 (单个生产者，单个消费者)

```
$ ./bench.sh 
results...
PUB: 2014/01/12 22:09:08 duration: 2.311925588s - 82.500mb/s - 432539.873ops/s - 2.312us/op
SUB: 2014/01/12 22:09:19 duration: 6.009749983s - 31.738mb/s - 166396.273ops/s - 6.010us/op
```

GOMAXPROCS=4 (4 个生产者, 4 个消费者)

```
$ ./bench.sh 
results...
PUB: 2014/01/13 16:58:05 duration: 1.411492441s - 135.130mb/s - 708469.965ops/s - 1.411us/op
SUB: 2014/01/13 16:58:16 duration: 5.251380583s - 36.321mb/s - 190426.114ops/s - 5.251us/op
```

### sarama

sarama是一家加拿大的电子商务网站开源的kafka驱动。由于kafka系统我们使用的比较多，这里不再介绍，直接开始使用示例。

```shell
go get gopkg.in/Shopify/sarama.v1
```

#### 生产者

```go
package main

import (
	"gopkg.in/Shopify/sarama.v1"
	"strings"
)

func main() {
	config := sarama.NewConfig()
	config.Producer.RequiredAcks = sarama.WaitForLocal       // Only wait leader to ack
	config.Producer.Compression = sarama.CompressionSnappy   // Compress messages
	brokerList := strings.Split("192.168.45.168:9093,192.168.45.168:9094", ",")
	producer, err := sarama.NewSyncProducer(brokerList, config)

	if err != nil {
		defer producer.Close();
	}

	for i := 0; i < 10; i++ {
		msg := &sarama.ProducerMessage{}
		msg.Topic = "gotest"
		msg.Value = sarama.StringEncoder("hello sarama")
		producer.SendMessage(msg)
	}
}
```

#### 消费者

```go
package main

import (
	"gopkg.in/Shopify/sarama.v1"
	"fmt"
)

func main() {
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true

	// Specify brokers address. This is default one
	brokers := []string{"192.168.45.168:9093", "192.168.45.168:9094"}

	// Create new consumer
	master, err := sarama.NewConsumer(brokers, config)
	if err != nil {
		panic(err)
	}

	defer func() {
		if err := master.Close(); err != nil {
			panic(err)
		}
	}()

	topic := "gotest"

	consumer, err := master.ConsumePartition(topic, 0, sarama.OffsetOldest)
	if err != nil {
		panic(err)
	}

	for ; true; {
		msg := <-consumer.Messages()
		fmt.Println(string(msg.Value))
	}
}
```

### wvanbergen

wvanbergen是在sarama集成上封装了一层，支持zk的消费

```shell
go get github.com/wvanbergen/kafka/consumergroup
go get github.com/wvanbergen/kazoo-go
```

#### 消费者

```go
package main

import (
	"log"
	"os"
	"os/signal"
	"strings"
	"time"

	"github.com/Shopify/sarama"
	"github.com/wvanbergen/kafka/consumergroup"
	"github.com/wvanbergen/kazoo-go"
)

var (
	consumerGroup = "gotest"
	kafkaTopicsCSV = "gotest"
	zookeeper = "192.168.45.168:4201/kafka"
	zookeeperNodes []string
)

func init() {
	sarama.Logger = log.New(os.Stdout, "[Sarama] ", log.LstdFlags)
}

func main() {

	config := consumergroup.NewConfig()
	config.Offsets.Initial = sarama.OffsetNewest
	config.Offsets.ProcessingTimeout = 10 * time.Second

	zookeeperNodes, config.Zookeeper.Chroot = kazoo.ParseConnectionString(zookeeper)

	kafkaTopics := strings.Split(kafkaTopicsCSV, ",")

	consumer, consumerErr := consumergroup.JoinConsumerGroup(consumerGroup, kafkaTopics, zookeeperNodes, config)
	if consumerErr != nil {
		log.Fatalln(consumerErr)
	}

	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt)
	go func() {
		<-c
		if err := consumer.Close(); err != nil {
			sarama.Logger.Println("Error closing the consumer", err)
		}
	}()

	go func() {
		for err := range consumer.Errors() {
			log.Println(err)
		}
	}()

	eventCount := 0
	offsets := make(map[string]map[int32]int64)

	for message := range consumer.Messages() {
		if offsets[message.Topic] == nil {
			offsets[message.Topic] = make(map[int32]int64)
		}

		eventCount += 1
		if offsets[message.Topic][message.Partition] != 0 && offsets[message.Topic][message.Partition] != message.Offset - 1 {
			log.Printf("Unexpected offset on %s:%d. Expected %d, found %d, diff %d.\n",
                message.Topic,
				message.Partition, offsets[message.Topic][message.Partition] + 1,
				message.Offset, message.Offset - offsets[message.Topic][message.Partition] + 1)
		}

		log.Println(string(message.Value))

		time.Sleep(10 * time.Millisecond)

		offsets[message.Topic][message.Partition] = message.Offset
		consumer.CommitUpto(message)
	}

	log.Printf("Processed %d events.", eventCount)
	log.Printf("%+v", offsets)
}
```

### 对比

1. nsq有着类似kafka的无中心设计，比kafak更进一步的生产者不依赖nsqd。
2. nsq在小消息情况下性能非常优异。
3. sarama提供了kafka的go语言驱动，但是直接暴露的比较裸露的驱动，需要订阅者自己关注partition和offset。如果partition发生变动重新消费什么的很麻烦。很不方便使用，万不得已需要使用自己记录partition和offset并同步起来。
4. 虽然别人开源了基于sarama的zk解决方案，但是开发方不是很知名，中间可能有各种小问题，如果使用需要完备的测试；
