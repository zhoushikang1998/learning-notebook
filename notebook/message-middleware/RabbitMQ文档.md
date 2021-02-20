# 服务端

## 配置

- 环境变量（官方文档）：
  - rabbitmq-env.conf(linux)
  - rabbitmq-env-conf.bat(Windows)
- rabbitmqctl：管理 vhost、用户和权限；管理 **运行参数和策略**
- rabbitmq-queues：管理特定于仲裁队列（quorum queue）的设置的工具
- rabbitmq-plugins：管理 plugin
- rabbitmq-diagnostice：允许检查**节点状态**，包括**有效配置**，以及许多其他指标和运行状况检查。
- 参数与策略（policy）：定义了可以在**运行时更改的集群范围的设置**，以及可以方便地为队列组（交换等）配置的设置，例如包括可选的队列参数。
- Runtime（Erlang VM）Flags：控制系统的底层方面：**内存分配设置、节点间通信缓冲区大小，运行时调度程序设置等**

### 配置文件

- （包括 server(核心服务器) 和 plugin 的设置：）配置文件地址
  - rabbitmq.conf（从 rabbitmq 3.7.0开始，配置格式使用 sysctl format 的格式）
    - 参数配置
    - 语法：
      - One setting uses one line（一个配置一行）
      - Lines are structured Key = Value（key/value 的方式）
      - Any line starting with a # character is a comment（以 # 开头的是注释）
  - advanced.config
    - 不能用 sysctl 格式来配置的参数
    - 类似于 rabbitmq.config
  - rabbitmq.conf 和 advanced.config 的改变在 RabbitMQ 节点重启后生效
  - RabbitMQ 3.7.0 之前的配置文件为：rabbitmq.config
  - **核心服务器参数设置（官网文档）**
  - **部分 plugins 参数设置（官网文档）**
- 配置值加密：遵从一些必要的规范，让些敏感的数据不会出现在文本形式的配置文件中。
  - 在配置文件中将加密之后的值以“{encrypted ，加密的值}”的形式包裹
- **操作系统内核限制**：虚拟内存，堆栈大小，打开的文件句柄等等
  - 修改限制：
    - 使用 systemd
    - 使用 Docker
    - 没有 systemd（旧的Linux发行版）： vim /etc/default/rabbitmq-server 或 rabbitmq-env.conf

### 配置文件位置

| Name                          | Location                                                     |
| :---------------------------- | :----------------------------------------------------------- |
| RABBITMQ_BASE                 | (Not used - Windows only)                                    |
| RABBITMQ_CONFIG_FILE          | ${install_prefix}/etc/rabbitmq/rabbitmq                      |
| RABBITMQ_MNESIA_BASE          | ${install_prefix}/var/lib/rabbitmq/mnesia                    |
| RABBITMQ_MNESIA_DIR           | $RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME                     |
| RABBITMQ_LOG_BASE             | ${install_prefix}/var/log/rabbitmq                           |
| RABBITMQ_LOGS                 | $RABBITMQ_LOG_BASE/$RABBITMQ_NODENAME.log                    |
| RABBITMQ_PLUGINS_DIR          | /usr/lib/rabbitmq/plugins:$RABBITMQ_HOME/pluginsNote that /usr/lib/rabbitmq/plugins is used only when RabbitMQ is [installed](https://www.rabbitmq.com/installing-plugins.html) into the standard (default) location. |
| RABBITMQ_PLUGINS_EXPAND_DIR   | $RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME-plugins-expand      |
| RABBITMQ_ENABLED_PLUGINS_FILE | ${install_prefix}/etc/rabbitmq/enabled_plugins               |
| RABBITMQ_PID_FILE             | $RABBITMQ_MNESIA_DIR.pid                                     |

### 日志

- 3.7.0 之前：使用两个日志文件（常规消息、未解决的异常）；3.7.0 后：使用一个日志文件
  - 老版本: <nodename>.log 和 <nodename>_sasl.log (<nodename> is rabbit@{hostname} by default).
  - 新版本： <nodename>.log 

- 配置日志地址的两种方式：

  - 配置文件
  - RABBITMQ_LOGS 环境变量（优先）

- 日志输出为文件

  - log.file: 日志文件地址（false禁止输出文件）
  - log.file.level: 日志文件输出级别. Default level is info.
  - log.file.rotation.date, log.file.rotation.size, log.file.rotation.count for log file rotation settings.

- log rotation

  - log.file.rotation.date：设置周期循环
  - log.file.rotation.size：

- logging to console

  - **log.console** (boolean): set to true to enable console output. Default is false
  - **log.console.level:** log level for the console output. Default level is info.

- logging to syslog

  - syslog 端点配置：

    ```properties
    log.syslog = true
    log.syslog.transport = tcp
    log.syslog.protocol = rfc5424
    
    log.syslog.ssl_options.cacertfile = /path/to/ca_certificate.pem
    log.syslog.ssl_options.certfile = /path/to/client_certificate.pem
    log.syslog.ssl_options.keyfile = /path/to/client_key.pem
    
    log.syslog = true
    log.syslog.ip = 10.10.10.10
    log.syslog.port = 1514
    
    log.syslog.host = my.syslog-server.local
    log.syslog.port = 1514
    
    log.syslog.identity = my_rabbitmq
    log.syslog.facility = user
    ```

- 日志消息类别

  - **connection**: [connection lifecycle events](https://www.rabbitmq.com/logging.html#connection-lifecycle-events) for AMQP 0-9-1, AMQP 1.0, MQTT and STOMP.

  - **channel**: channel logs. Mostly errors and warnings on AMQP 0-9-1 channels.

  - **queue**: queue logs. Mostly debug messages.

  - **mirroring**: queue mirroring logs. Queue mirrors status changes: starting/stopping/synchronizing.

  - **federation**: federation plugin logs.

  - **upgrade**: verbose upgrade logs. These can be excessive.

  - **default**: all other log entries. You cannot override file location for this category.

  - ```properties
    log.<category>.level
    log.<category>.file 
    
    log.file.level = debug
    log.connection.level = info
    
    log.federation.file = rabbit_federation.log
    
    log.upgrade.level = none
    ```

- 日志级别

  - | Log level | Severity |
    | :-------- | :------- |
    | debug     | 128      |
    | info      | 64       |
    | warning   | 16       |
    | error     | 8        |
    | critical  | 4        |
    | none      | 0        |

  - 修改日志等级（两种方式）

    - 配置文件
    - CLI 工具：rabbitmqctl -n rabbit@target-host set_log_level debug

- 使用 CLI 工具跟踪日志

  - ```bash
    rabbitmq-diagnostics -n rabbit@target-host log_tail -N 300
    
    rabbitmq-diagnostics -n rabbit@target-host log_tail_stream
    ```

- 启用调试日志

  - ```ini
    log.file.level = debug
    
    log.console = true
    log.console.level = debug
    
    rabbitmqctl -n rabbit@target-host set_log_level debug
    
    rabbitmqctl -n rabbit@target-host set_log_level info
    
    log.file.level = debug
    
    log.connection.level = info
    log.channel.level = info
    ```

- Service logs(需要超级用户权限)

  - ```bash
    sudo journalctl --system | grep rabbitmq
    ```

- 记录事件

  - 连接的生命周期事件（

    - connection 发送至少 1 byte 的数据将会被记录下来
    - connection 成功认证、连接到 vhost 也会被记录
    - 关闭 TCP 连接或连接失败，也会被记录

  - 查看内部事件

    - ```bash
      rabbitmq-diagnostics consume_event_stream
      ```

  - 核心服务事件：Queue、Connection、Channel、Consumer、Policy、Parameter、Vhost、User、Permission、Alarm

  - worker 事件：shovel.worker.status、shovel.worker.removed

  - 连接事件：federation.link.status、federation.link.removed

- 高级配置（大多数不需要）

### 持久性配置

- 持久层
  - “持久层”是指用于将两种类型（持久性消息、瞬态消息）的消息存储到磁盘的机制。
  - 具有两个组件
    - 队列索引：负责维护有关给定消息在队列中的位置以及是否已传递和确认消息的知识。（每个队列有一个队列索引。）
    - 消息存储：用于服务器中所有队列之间共享的消息的**键值存储**。
- 内存成本：
  - 在内存压力下，持久层会尝试将尽可能多的内容写入磁盘，并从内存中删除尽可能多的内容。但是，有些东西必须保留在内存中：
    - 每个队列为每个***未确认的*消息维护一些元数据** 。如果消息本身的目的地是消息存储库，则可以将其本身从内存中删除。
    - 消息存储区需要一个索引。默认消息存储索引为存储中的每个消息使用少量内存。
- 消息嵌入队列索引中：
  - 目的：将非常小的消息作为优化存储在队列索引中，并将所有其他消息写入消息存储中。配置项：queue_index_embed_msgs_below 控制存入队列索引的消息大小（默认 4096 字节）。
  - 优点：
    - 消息可以一次完成写入磁盘，而不必两次进行。对于**微小的消息**，这可以是可观的收益。
    - 写入队列索引的消息不需要消息存储索引中的条目，因此在调出页面时不具有内存开销。
  - 缺点：
    - 队列索引在内存中保留固定数量记录的块；如果将非微小的消息写入队列索引，则可能会占用大量内存。
    - 如果将消息通过交换机路由到多个队列，则需要将消息写入多个队列索引。如果将这样的消息写入消息存储，则只需写入一份副本。
    - 目标为队列索引的未确认消息始终保留在内存中。
  - OS 和 Runtime Limits 的影响
    - 文件句柄太少
      - 一个文件句柄太少的繁忙服务器可能每秒执行数百次重新打开操作-在这种情况下，如果提供更多文件句柄，其性能可能会显着提高。
    - I/O 线程池太小
      - Runtime 使用线程池来处理长时间运行的文件 I/O 操作，这些线程池在所有虚拟主机和队列之间共享。每个活动文件 I / O 操作在发生时都使用一个异步线程。因此，异步线程太少会损害性能。
    - 理想情况下，对于正在执行 I / O 操作流的所有队列，应该有足够的文件句柄，并且对于存储层可以合理执行的同时 I / O 操作数量，应该有足够的异步线程。

## 网络

- 所有协议都是基于 TCP 而提供的。
- 网络接口（IPv4、IPv6）
  
  - listeners.tcp.*
- **端口访问**
  - **4369**: [epmd](http://erlang.org/doc/man/epmd.html), a peer discovery service used by RabbitMQ nodes and CLI tools
  - **5672**, 5671: used by AMQP 0-9-1 and 1.0 clients without and with TLS
  - **25672**: used for inter-node and CLI tools communication (Erlang distribution server port) and is allocated from a dynamic range (limited to a single port by default, computed as AMQP port + 20000). Unless external connections on these ports are really necessary (e.g. the cluster uses [federation](https://www.rabbitmq.com/federation.html) or CLI tools are used on machines outside the subnet), these ports should not be publicly exposed. See [networking guide](https://www.rabbitmq.com/networking.html) for details.
  - 35672-35682: used by CLI tools (Erlang distribution client ports) for communication with nodes and is allocated from a dynamic range (computed as server distribution port + 10000 through server distribution port + 10010). See [networking guide](https://www.rabbitmq.com/networking.html) for details.
  - **15672**: [HTTP API](https://www.rabbitmq.com/management.html) clients, [management UI](https://www.rabbitmq.com/management.html) and [rabbitmqadmin](https://www.rabbitmq.com/management-cli.html) (only if the [management plugin](https://www.rabbitmq.com/management.html) is enabled)
  - 61613, 61614: [STOMP clients](https://stomp.github.io/stomp-specification-1.2.html) without and with TLS (only if the [STOMP plugin](https://www.rabbitmq.com/stomp.html) is enabled)
  - 1883, 8883: ([MQTT clients](http://mqtt.org/) without and with TLS, if the [MQTT plugin](https://www.rabbitmq.com/mqtt.html) is enabled
  - 15674: STOMP-over-WebSockets clients (only if the [Web STOMP plugin](https://www.rabbitmq.com/web-stomp.html) is enabled)
  - 15675: MQTT-over-WebSockets clients (only if the [Web MQTT plugin](https://www.rabbitmq.com/web-mqtt.html) is enabled)
  - 15692: Prometheus metrics (only if the [Prometheus plugin](https://www.rabbitmq.com/prometheus.html) is enabled)

- **EPMD 和节点间通信**

  - EPMD：运行在每一个 Rabbitmq 节点上的小的守护进程，运行时使用，以发现特定节点监听的端口

  - 当一个节点或 CLI 工具需要联系到 `rabbit@hostname2`上时，做了以下步骤：

    1. 使用标准的 OS 解析器或在 inetrc 文件中指定的自定义解析器将 hostname2 解析为 IPv4 或 IPv6 地址
    2. 使用上述地址联系运行在 `hostname2` 上的 `epmd`
    3. 向 `epmd` 询问节点 `rabbit` 在其上使用的端口
    4. 使用解析的 IP 地址和发现的端口与节点联系
    5. 继续通信

  - EPMD 接口

    - `epmd` 将在默认情况下监听所有接口。可以使用 `ERL_EPMD_ADDRESS` 将其限制为若干个接口。

      - ```bash
        export ERL_EPMD_ADDRESS="::1"
        ```

    - 当 `ERL_EPMD_ADDRESS` 更改时，主机上的 `RabbitMQ` 节点和 `epmd` 都必须停止。对于 `epmd`,使用

      - ```bash
        epmd -kill
        ```

  - EPMD 端口（默认是 4369）

    - 可以通过使用 `ERL_EPMD_PORT` 环境参数修改

      - ```bash
        export ERL_EPMD_PORT="4369"
        ```

      - 集群中的所有主机必须使用相同的端口。

      - 更改后要停止主机上的 `RabbitMQ` 节点和 `epmd`，对于 `epmd`

      - ```bash
        epmd -kill
        ```

- 节点间通信端口范围

  - RabbitMQ 节点将使用一个**特定范围的端口**称为节点间通信端口范围。CLI 工具在需要联系节点时使用相同的端口。范围可以修改。
  - RabbitMQ 节点使用一个称为分布端口的端口与 CLI 工具和其他节点通信。它是根据一系列值动态分配的。对于 RabbitMQ，默认范围被限制为计算为 `RABBITMQ_NODE_PORT` (5672)+ 20000 的单个值，这导致使用端口 `25672`。这个端口可以使用 `RABBITMQ_DIST_PORT` 环境变量来配置。
  - 35672 - 35682 端口：通过 `RABBITMQ_CTL_DIST_PORT_MIN`  和`RABBITMQ_CTL_DIST_PORT_MAX` 环境变量配置。（推荐范围为 10）
  - 必须打开 `epmd` 端口，才能让 CLI 工具和集群发挥作用。
  - 运行 `epmd -names` 验证节点使用什么端口进行节点间和CLI工具通信。
  - 节点间通信缓冲区大小限制
    - 默认 128 MB，不推荐小于 64 MB。
    - 在节点间流量很大的集群中，增加这个值可能会对吞吐量产生**积极的影响**。

- 中介：Proxies and load balancers

  - 代理和负载平衡器的网络带宽超额配置和吞吐量监视非常重要

  - 启用 Heartbeat：导致周期性的轻型网络流量，具有保护客户端连接的作用，该客户端可能会闲置一段时间，以防止代理和负载均衡器过早关闭。（10 到 30 秒的 Heartbeat，太低会误报）

  - 代理协议：

    - **HAproxy**

    - ```ini
      proxy_protocol = true
      ```

    - 启用此选项后，所有客户端连接都必须通过也支持该协议并配置为发送代理协议标头的代理。

- TLS（SSL）支持

  - 可以通过 RabbitMQ 使用 TLS 加密连接。也可以使用对等证书进行身份验证。
  - 吞吐量调整：以下方式改进（权衡，有利有弊）
    - 增加 TCP 缓冲区大小
    - 确保禁用 Nagle 的算法
    - 启用可选的 TCP 功能和扩展
  - TCP 缓冲区大小
    - 每个 TCP 连接都有为其分配的缓冲区。这些**缓冲区越大，每个连接使用的 RAM 越多**，吞吐量越好。在 Linux 上，操作系统默认情况下会自动调整 TCP 缓冲区大小，通常设置在 80 到 120 KB之间。
    - 为了获得最大吞吐量，可以使用一组配置选项来增加缓冲区大小：
      - AMQP 0-9-1 和AMQP 1.0 的 `tcp_listen_options`  
      - MQTT 的 `mqtt.tcp_listen_options`
      - STOMP 的 `stomp.tcp_listen_options`
  - Erlang VM I / O线程池
    - Erlang 运行时使用线程池异步执行 I / O 操作。池的大小是通过 `RABBITMQ_IO_THREAD_POOL_SIZE` 环境变量配置的。
    - 该变量是设置+ A VM命令行标志（例如+ A 128）的快捷方式。（可以直接设置命令行标志）
  - 调整大量连接（几个因素限制单个节点可以支持多少个并发连接：）
    - [打开文件句柄](https://www.rabbitmq.com/networking.html#open-file-handle-limit)（包括套接字）的最大数量以及其他由内核强制执行的资源限制
    - [每个连接使用](https://www.rabbitmq.com/memory-use.html)的 RAM 量
      - 限制连接上的通道数（通道也会消耗 RAM）
    - 每个连接使用的 CPU 资源量
      - 使用 `collect_statistics_interval` 键增加统计信息收集间隔（默认为 5秒）
      - 将间隔值增加到 30-60s 将减少 CPU 占用空间和峰值内存消耗。这有一个缺点：使用上面示例中的值，所述实体的指标将每60秒刷新一次。
    - VM 配置允许的最大 Erlang 进程数。
    - 
  - Nagle 的算法（"nodelay")
    - 禁用 [Nagle 的算法](http://en.wikipedia.org/wiki/Nagle's_algorithm)主要用于减少延迟，但也可以提高吞吐量
    - 配置 rabbitmq.conf 和 advanced.config
  - 连接积压
    - 当数量达到数万或更多时，确保服务器可以接受入站连接非常重要。不可接受的TCP连接被放入具有限制长度的队列中。

- 处理高连接流失

  - 连接的 TIME_WAIT 设置

- TCP Keepalive

  - 可以起到与心跳相同的作用，并清除陈旧的 TCP 连接

- 

### 使用 Heartbeat 和 TCP keepalive 检测无效的 TCP 连接

- Heartbeat：

  - 功能

    - 确保应用程序层迅速发现中断的连接（以及完全不响应的对等点）
    - 心跳还可以防御某些网络设备，这些设备可能会在一段时间内没有活动时终止“空闲” TCP连接。

  - 心跳超时时间（需要客户端和服务端协商）

    - 0 ：禁用
    - 值不同取较小的值，不推荐太小的值
    - 默认 60s
    - 最佳时间：5 - 20s

  - 心跳帧

    - 每隔 timeout/2 的时间发送一次（心跳间隔）
    - 在两次错过的心跳之后，对等方被视为无法访问。TCP 连接将关闭，需要重新连接。
    - 任何流量都将计入有效心跳。

  - 禁用心跳：

    - 设置超时时间为 0 或设置很高的数（如 1800），因为帧传输太少，无法产生实际的变化。
    - 除非使用 TCP keepalive，否则不建议禁用 Heartbeat
    - 禁用了 Heartbeat，将很难及时发现对等方不可用，这将对数据安全构成重大风险

  - 使用 Java 客户端启用 Heartbeat

    ```java
    ConnectionFactory cf = new ConnectionFactory();
    
    // set the heartbeat timeout to 60 seconds
    cf.setRequestedHeartbeat(60);
    ```

    - 如果服务端设置了一个不为 0 的值，客户端只能低于这个值。

  - Heartbeat 和 TCP 代理

    - 某些网络工具（HAproxy，AWS ELB）和设备（硬件负载平衡器）在一段时间内没有活动时可能会终止“空闲” TCP连接。在大多数情况下，这是不可取的。
    - 在连接上启用心跳后，将导致周期性的轻型网络流量。因此，心跳具有保护客户端连接的副作用，**该客户端连接可能会闲置一段时间**，以防止代理和负载平衡器过早关闭。

  

- TCP keepalives：具有类似用途，但需要内核调整

- 对活动连接和断开连接进行故障排除：

  - RabbitMQ 节点将记录由于缺少心跳而关闭的连接。
  - 检查服务器和客户端日志将提供有价值的信息，并且应该是故障排除的第一步。





# Java 客户端

- 核心类

  - **Channel**: represents an AMQP 0-9-1 channel, and provides most of the operations (protocol methods).
  - **Connection**: represents an AMQP 0-9-1 connection
  - **ConnectionFactory**: constructs `Connection` instances
  - **Consumer**: represents a message consumer
  - **DefaultConsumer**: commonly used base class for consumers
  - **BasicProperties**: message properties (metadata)
  - **BasicProperties.Builder**: builder for `BasicProperties`

- Connection 和 Channel 的生命期限

  - Connection：长连接，
  - Channel：长连接，但短于 Connection
  - 客户端提供的 Connection 名称：作为自定义的标识符

- Exchanges 和 Queues

  - 使用之前必须先声明（Declare）
  - Passive Declaration
    - 被动声明仅检查具有提供名称的实体是否存在。
  - 带有可选响应的操作
    - noWait ：不等待服务器响应
    - “noWait”版本效率更高，但提供的安全保证较低，例如，它们更依赖于[心跳机制](https://www.rabbitmq.com/heartbeats.html)来检测失败的操作。

- Channel 和并发考虑（线程安全）

  - 使用 channel 池，而非在线程之间共用一个 channel

- Consumer 接收消息（订阅方式）

- AddressResolver （服务发现）

  - 适用场景
    - 最适合用于实现自定义服务发现逻辑。
    - Affinity 和 负载均衡器
  - 主要实现类
    - DnsRecordIpAddressResolver
    - DnsSrvRecordAddressResolver

- Java NIO 支持

  - NIO：更容易控制资源

  - 启动 NIO

    - ```java
      ConnectionFactory connectionFactory = new ConnectionFactory();
      connectionFactory.useNio();
      ```

  - 设置 NIO

    - ```java
       connectionFactory.setNioParams(new NioParams().setNbIoThreads(4));
      ```

  - 

## 从网络故障自动恢复

- Connection 恢复
  - 自动恢复过程遵循以下步骤：
    - 重新连接
    - 恢复连接监听器
    - 重新开放通道
    - 恢复通道监听器
    - 恢复通道 `basic.qos` 设置，发布者确认和交易设置
  - 拓扑恢复包括对每个 channel 执行以下操作：
    - 重新声明 Exchange（预定义的交换除外）
    - 重新声明 queue
    - 恢复所有绑定
    - 恢复所有消费者
  - 从 Java 客户端的 4.0.0 版本开始，**默认情况下启用了自动恢复（因此也启用了拓扑恢复）**。
- 什么时候会触发自动恢复
  - 由以下事件触发：（以先发生的为准）
    - 连接的 I / O 循环中引发了 I / O 异常
    - 套接字读取操作超时
    - 检测到丢失的服务器 Heartbeat
    - 连接的 I / O 循环中会引发任何其他意外异常
  - 如果与 RabbitMQ 节点的初始客户端连接失败，则将无法进行自动连接恢复
  - 当应用程序通过 `Connection.Close` 方法关闭连接时，将不会启动连接恢复。
- 恢复监听器
  - 提供了两个具有描述性名称的方法：
    - addRecoveryListener
    - removeRecoveryListener
  - 要使用上述方法，需要将 Connection 和 Channel 强制转换为 `Recoverable`
- 故障检测和恢复的局限性
  - 拓扑恢复以来于实体的每个连接的缓存——使用自动连接恢复的使用者标签在所有通道上必须是唯一的。
  - 配合`发布者确认`使用
  - 由于通道级异常而关闭通道时，连接恢复将不会开始。此类异常通常表示应用程序级别的问题。
- 手动确认和自动恢复
  - 当使用手动确认时，在消息传递和确认之间到 RabbitMQ 节点的网络连接可能会失败。
  - *basic.ack*, *basic.nack*, *basic.reject*  使用旧的传递标签将导致通道异常。为了避免这种情况，RabbitMQ Java客户端跟踪并更新传递标记，使它们在恢复之间单调地增长。
-  Channel 生命周期和拓扑恢复
  - 在应用程序的整个生命周期中保持打开创建通道的状态。
- 未处理的异常
  - 与连接，通道，恢复和使用者生命周期相关的未处理异常将委托给异常处理程序。
  - 异常处理程序是实现 `ExceptionHandler` 接口的任何对象 。
  - 默认情况下，使用 `DefaultExceptionHandler` 的实例。它将异常详细信息打印到标准输出。

- 指标和监控
  - 客户端收集活动连接的运行时度量标准，可选功能，应使用 `setMetricsCollector（metricsCollector）` 方法在 `ConnectionFactory` 级别上进行设置。
  - 收集的指标：
    - 打开的连接数
    - 公开频道数
    - 已发布消息数
    - 消耗的消息数
    - 确认消息数
    - 拒绝的邮件数
  - 启用指标收集的注意事项：
    - 添加依赖项（Maven、Gradle）
    - 指标收集是可扩展的。可自定义 MetricsCollector
    - `MetricsCollector` 设置在 `ConnectionFactory` 级别，可以被不同的实例共享。
    - 指标收集不支持事务。
  - Micrometer 支持 和 Dropwizard指标支持
- 拓扑恢复的注意事项和局限性
  - 为了使拓扑恢复成为可能，RabbitMQ Java 客户端维护了一个声明队列、交换器和绑定的缓存。（在每个连接上）
  - 使缓存项无效的情况：
    - 删除队列时。
    - Exchange 被删除时。
    - 绑定被删除时。
    - 当使用者在自动删除的队列上被取消时。
    - 当队列或交换与自动删除的交换解除绑定时。

- RPC （请求/答复）模式
- TSL 支持
  - 使用 TSL 加密客户端和代理之间的通信 。还支持客户端和服务器身份验证（也称为对等验证）





# RabbitMQ 扩展

## Confirm（发布者确认）

- 确保数据安全。
- 传送标识符：Delivery Tags
  - 范围是每个 channel
  - 最大数字：9223372036854775807，可能会用尽
- 模式：（手动、自动）
  - 手动模式：（与 prefetchCount 使用：限制了通道上未完成(“正在进行”)交付的数量）
    - basic.ack：正确发送，通知队列消息已正确发送，可以删除
    - basic.nack(RabbitMQ 扩展)：忽略 requeue 属性
    - basic.reject：错误发送，通知队列消息发送错误，也可以删除
  - 自动模式：不安全，当消息还未发送完成时，如果断开 connection 或 channel，消息将会丢失。
    - 没有 prefetchCount 限制，可能会内存溢出
- 参数
  - multiple：可同时 Ack 多条消息
  - requeue：消息发送失败后是否重新入队===》入队到原始位置
    - 可能导致循环出错、入队，消耗大量内存
  - redeliver：幂等性
- QoS：
  - basic.PrefetchCount：可配置在特定 channel 或特定 consumer 上
    - 在 basic.get 上不起作用
    - 没有配置将导致 大量消耗 RAM 空间
- Publisher Confirms
  - comfirm.select
  - comfirm.select-ok
  - comfirm 模式 和 事务不可同时使用。
  - 不能保证 消息 的接收顺序
  - 可能重复消费，要消息去重



## Consumer

- Cancel
- Prefetch
- Priority



## RPC 

- Direct reply-to

## Connection

- Blocked Connection
  - Java 通过 connection.addBlockedListener 处理

## Message Routing

- Exchange  to Exchange 绑定
- Alternate Exchange（备用 Exchange）



## Message Lifecycle（生命周期）

- TTL 
  - 队列
  - 队列的所有消息
  - 消息
- Priority Queue
- Dead Letter Queue
- Queue Length Limit

## Authentication and Identity

- User Id
- Authentication Failure Notifications