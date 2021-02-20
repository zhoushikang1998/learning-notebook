## kafka 简介

- 三种角色：
  - 消息中间件
  - 存储系统 ===> 持久化到磁盘中
  - 流式处理平台 ===> 流式处理类库
- 基本概念
  - Producer：生产者
  - Broker：服务代理节点。
  - Consumer ===> Pull 模式拉取数据
  - ZK 集群
  - Topic：消息以主题为单位进行归类
  - Partition （Topic-Partition）
    - 存储层面：可追加的日志文件
    - 消息：分配特定的 Offset
    - offset 是分区的唯一标识，但不可跨越分区。所以，Kafka 保证的是分区有序而不是主题有序。
    - 多副本（Replica）机制：一主多从模式，Leader 负责读写，Follower 只负责和 Leader 同步消息。
    - AR = ISR + OSR（正常情况：AR = ISR，ISR 和 OSR 可相互转化）
      - HW：高水位，标识一个特定的 offset，消费者只能拉取这个 offset 之前的消息（不包含该 offset）。
      - LEO：下一条待写入消息的 offset（最后一条消息的 offset + 1）
    - Kafka 使用 ISR 的方式，使得 kafka 的复制机制介于同步和异步之间。

## 安装和配置

> 环境：Linux Centos 7

### 1.jdk 安装

- 删除自带的 openjdk

```bash
# 查看自带的openJDK
rpm -qa | grep java
# 逐一将显示的openJDK删除
rpm -e --nodeps xxx-openjdk-xxx
```

- 下载 jdk 1.8 以上版本，复制到 /opt 目录下
- 配置 JDK 环境变量，修改 /etc/profile，然后执行 source /etc/profile 让配置生效。

```properties
# java config
export JAVA_HOME=/opt/jdk1.8.0_144
export JRE_HOME=$JAVA_HOME/jre
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=./://$JAVA_HOME/lib:$JRE_HOME/lib
```

- 通过 java -version 命令查看是否安装配置成功。

### 2.zookeeper 安装

- 从 Apache 官网中下载安装包（名称带有 bin 后缀），复制到 /opt 目录下，然后解压缩

```bash
tar zxcf apache-zookeeper-3.6.1-bin.tar.gz
```

- 向 /etc/profile 添加配置，执行 source /etc/profile 让配置生效。

```properties
# zookeeper config
export ZOOKEEPER_HOME=/opt/apache-zookeeper-3.6.1-bin
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

- 修改 zookeeper 的配置文件。将 zoo_sample.cfg 文件复制为 zoo.cfg，修改 zoo.cfg 文件

```properties
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/tmp/zookeeper/data
# 日志目录
dataLogDir=/tmp/zookeeper/log
# the port at which the clients will connect
clientPort=2181

```

- 创建 /tmp/zookeeper/data 和 /tmp/zookeeper/log 目录
- 在 /tmp/zookeeper/data 目录下创建 myid 文件，并写入一个数值，比如 0（服务器的编号）。
- 启动 zookeeper 服务

```bash
zkServer.sh start
```

### 3.kafka 安装

- 官网中下载 kafka 安装包，复制到 /opt 目录下，解压缩。
- 修改 broker 的配置文件 config/server.properties，主要的几个参数：

```properties
# broker 的编号
broker.id=0
# broker 对外提供的服务入口地址
# listeners=PLAINTEXT://localhost:9092
advertised.listeners=PLAINTEXT://192.168.234.128:9092
# 存放消息日志文件的地址
log.dirs=/tmp/kafka-logs
# zookeeper 集群地址
zookeeper.connect=localhost:2181/kafka
```

- 确保 zookeeper 服务已经启动，再启动 kafka 服务。

```bash
# 前台启动
bin/kafka-server-start.sh config/server.properties
# 后台启动
bin/kafka-server-start.sh -daemon config/server.properties
```

- 使用 jps 命令查看 kafka 服务是否已经启动进程。

### 4.java 连接

- 关闭 Linux 防火墙

```bash
# 查看防火墙状态
systemctl status firewalld.service
# 关闭防火墙
systemctl stop firewalld.service
# 禁止开机自启动
systemctl disable firewalld.service
```

- 设置 kafka 配置文件 config/server.properties 中 broker 对外提供的服务入口地址：

```properties
# broker 对外提供的服务入口地址
# listeners=PLAINTEXT://localhost:9092
advertised.listeners=PLAINTEXT://192.168.234.128:9092
```



## 生产者

### 1.客户端开发

- 正常的生产逻辑步骤：
  1. 配置生产者客户端参数及创建相应的生产者实例。（Properties、KafkaProducer）
  2. 构建待发送的消息。（ProducerRecord）
  3. 发送消息。（send()）
  4. 关闭生产者实例。（close()）
- 发送消息的三种模式：
  - 发后即忘（fire-and-forget)
  - 同步（sync）：通过重试机制保证可靠性。
  - 异步（async）：回调函数的调用也可以保证分区有序。
- 序列化：生产者需要用序列化器（Serializer）把对象转换成字节数组才能通过网络发送给Kafka。
  - 生产者使用的序列化器和消费者使用的反序列化器是需要一一对应的
- 分区器：为消息分配分区，可通过 partition 字段指定分区。
- 生产者拦截器：在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的消息、修改消息的内容等，也可以用来在发送回调逻辑前做一些定制化的需求，比如统计类工作。
  - onSend()
  - onAcknowledgement()
  - close()
  - 可以指定拦截器链

### 2.原理分析

- 整体结构
  - RecordAccumulator 主要用来缓存消息以便 Sender 线程可以批量发送，进而减少网络传输的资源消耗以提升性能。
  - ProducerBatch是指一个消息批次，ProducerRecord会被包含在ProducerBatch中，这样可以使字节的使用更加紧凑。
  - 在 RecordAccumulator 的内部还有一个 BufferPool，它主要用来实现 ByteBuffer 的复用，以实现缓存的高效利用。
  - InFlightRequests：主要作用是缓存了已经发出去但还没有收到响应的请求
    - 获得 leastLoadedNode，选择 leastLoadedNode 发送请求可以使它能够尽快发出，避免因网络拥塞等异常而影响整体的进度。

![](images/生产者客户端整体架构.jfif)

## 消费者

### 1.消费者和消费组

- 每个消费者都有一个对应的消费组。当消息发布到主题后，只会被投递给订阅它的每个消费组中的一个消费者。
- 每一个分区只能被一个消费组中的一个消费者所消费。
- 通过消费组的特性实现点对点、发布/订阅模式。
  - 点对点：所有消费者属于同一个消费组。
  - 发布/订阅：所有消费者都属于不同的消费组。
- 通过 group.id 设置消费组，默认为空字符串。

### 2.客户端开发

- 正常的消费逻辑步骤：
  1. 配置消费者客户端参数及创建相应的消费者实例。
  2. 订阅主题。
  3. 拉取消息并消费。
  4. 提交消费位移。
  5. 关闭消费者实例。
- 订阅主题与分区
  - 一个消费者可以订阅一个或多个主题
  - 三种不同的订阅状态：AUTO_TOPICS、AUTO_PATTERN 和 USER_ASSIGNED（如果没有订阅，那么订阅状态为NONE）
- 位移提交
  - 消费位移存储在 Kafka 内部的主题 __consumer_offsets 中
  - 自动位移提交的方式在正常情况下不会发生消息丢失或重复消费的现象，但是在编程的世界里异常无可避免，与此同时，自动位移提交也无法做到精确的位移管理。
- 指定位移消费
  -  KafkaConsumer 中的 seek（）方法可以让我们从特定的位移处开始拉取消息，让我们得以追前消费或回溯消费。
- 再均衡
  - **再均衡是指分区的所属权从一个消费者转移到另一消费者的行为**，它为消费组具备高可用性和伸缩性提供保障，使我们可以既方便又安全地删除消费组内的消费者或往消费组内添加消费者。
  - 应尽量避免不必要的再均衡的发生。
- 消费者拦截器
  - 主要在消费到消息或在提交消费位移时进行一些定制化的操作。
- 多线程实现：
  - KafkaProducer是线程安全的，然而KafkaConsumer却是非线程安全的。
  - 最常见的方式：线程封闭，即为每个线程实例化一个KafkaConsumer对象
  - 

## 主题与分区

### 1.主题管理

- 主要操作：创建主题、查看主题信息、修改主题和删除主题
- 操作方式：
  1. kafka-topics.sh 脚本：实质上是调用了kafka.admin.TopicCommand类来执行主题管理的操作。
     - 命令：create、list、describe、alter、delete
     - 参数：replica-assignment、partitions、replication-factor、config
  2. Java：调用 kafka.admin.TopicCommand 类
  3. KafkaAdminClient
- 查看集群中各个 broker 的分区副本的分配情况
  - 通过日志文件的根目录
  - 通过 ZooKeeper 客户端来获取
- 分区副本的分配：
  - 生产者的分区分配是指为每条消息指定其所要发往的分区；消费者中的分区分配是指为消费者指定其可以消费消息的分区；而这里的分区分配是指为集群制定创建主题时的分区副本分配方案，即在哪个 broker 中创建哪些分区的副本。
  - 使用 kafka-topics.sh 脚本创建主题时的内部分配逻辑按照机架信息划分成两种策略：
    - 未指定机架信息
    - 指定机架信息。
  - startIndex选择随机产生，是因为这样可以在多个主题的情况下尽可能地均匀分布分区副本。
- 目前 Kafka **只支持增加分区数**而不支持减少分区数。
- kafka-configs.sh 脚本是专门用来对配置进行操作的，这里的操作是指在运行状态下修改原有的配置，如此可以达到**动态变更**的目的。
- **主题端参数**：
  - 与主题相关的所有配置参数在 broker 层面都有对应参数
  - 如果没有修改过主题的任何配置参数，那么就会使用 broker 端的对应参数作为其默认值。
- Kafka的内部一共包含2个主题，分别为 __consumer_offsets 和 __transaction_state。
- **主题合法性验证**
  - kafka-topics.sh 脚本创建的方式一般由运维人员操作，普通用户无权过问。
  - 自定义实现  CreateTopicPolicy 接口



### 2.分区管理

- 优先副本的选举
  - 优先副本是指在 AR 集合列表中的第一个副本。
  - 通过一定的方式促使优先副本选举为 leader 副本，以此来促进集群的负载均衡，这一行为也可以称为“分区平衡”。
  - 在实际生产环境中，一般使用 path-to-json-file 参数来分批、手动地执行优先副本的选举操作。
- 分区重分配
  - Kafka 提供了 kafka-reassign-partitions.sh 脚本来执行分区重分配的工作，它可以在**集群扩容、broker 节点失效的场景**下对分区进行迁移。
  - 3 个步骤：
    - 首先创建需要一个包含主题清单的 JSON 文件
    - 其次根据主题清单和 broker 节点清单生成一份重分配方案
    - 最后根据这份方案执行具体的重分配动作。
  - 分区重分配的基本原理是：先**通过控制器为每个分区添加新副本**（增加副本因子），新的副本将**从分区的 leader 副本那里复制所有的数据**。根据分区的大小不同，复制过程可能需要花一些时间，因为数据是通过网络复制到新副本上的。在复制完成之后，**控制器将旧副本从副本清单里移除**（恢复为原先的副本因子数）。注意在重分配的过程中要确保有足够的空间。
  - 分区重分配本质在于**数据复制**，先增加新的副本，然后进行数据同步，最后删除旧的副本来达到最终的目的。
  - 分区重分配对集群的性能有很大的影响，需要占用额外的资源，比如网络和磁盘。在实际操作中，我们将降低重分配的粒度，分成多个小批次来执行，以此来将负面的影响降到最低。
- 复制限流
  - 可以对副本间的复制流量加以限制来保证重分配期间整体服务不会受太大的影响。
  - 两种方式：
    - kafka-config.sh 脚本
      - 主要以动态配置的方式来达到限流的目的
    - **kafka-reassign-partitions.sh 脚本**
      - 配合 verify 参数，对临时设置的一些限制性的配置在使用完后要及时删除
- 修改副本因子
  - kafka-reassign-partition.sh 脚本
  - 副本数可以增加也可以减少

### 3.选择合适的分区数

- 性能测试工具
  - 生产者性能测试：kafka-producer-perf-test.sh
  - 消费者性能测试：kafka-consumer-perf-test.sh
- 分区数和吞吐量的关系
  - 分区是 Kafka 中最小的并行操作单元，**对生产者而言，每一个分区的数据写入是完全可以并行化的**；对消费者而言，Kafka 只允许单个分区中的消息被一个消费者线程消费，**一个消费组的消费并行度完全依赖于所消费的分区数**。
  - 实际应用中可以通过类似的测试案例（比如复制生产流量以便进行测试回放）来找到一个**合理的临界值区间**。
- 分区数的上限
  - 最关键的异常：“Too many open files” 文件描述符不足，它一般发生在创建线程、创建 Socket、打开文件这些场景下
  - 可以使用 **ulimit-n 65535 命令将上限提高到 65535**，这样足以应对大多数的应用情况，再高也完全没有必要了。



## 日志存储

- Kafka 消息 ===存储===> 磁盘

### 1.文件目录布局

- Log ===> 切分成多个 LogSegment（日志分段）

  - Log 在物理上只以**文件夹的形式**存储，而每个LogSegment 对应于磁盘上的一个**日志文件和两个索引文件**，以及可能的其他文件

    <img src="images/日志关系.jfif" style="zoom:100%;" />

  - 向Log 中追加消息时是顺序写入的，**只有最后一个 LogSegment 才能执行写入操作**，在此之前所有的 LogSegment 都不能写入数据。

  - 消费者提交的位移是保存在 Kafka 内部的主题  __consumer_offsets中的，初始情况下这个主题并不存在，当第一次有消费者消费消息时会自动创建这个主题。

### 2.日志格式

- v0 ===> v1 ===> v2
- 消息压缩：
  - Kafka实现的压缩方式是**将多条消息一起进行压缩**，这样可以保证较好的压缩效果。
  - 生产者发送的压缩数据在 broker 中也是保持压缩状态进行存储的，消费者从服务端获取的也是压缩的消息，消费者在处理消息之前才会解压消息，这样**保持了端到端的压缩**。
  - 当消息压缩时是将整个消息集进行压缩作为内层消息（inner message），内层消息整体作为外层（wrapper message）的value。
  - 外层消息保存了内层消息中最后一条消息的绝对位移（absoluteoffset），绝对位移是相对于整个分区而言的。
- 变长字段（Varints）：Base 128 Varints 编码（压缩算法）
  - Varints 是使用一个或多个字节来序列化整数的一种方法。数值越小，其占用的字节数就越少。
    - msb：最高位，除最后一个字节外，其余 msb 位都设置为1，最后一个字节的 msb 位为0。
- 使用 kafka-dump-log.sh 脚本查看日志内容

### 3.日志索引

- 提高查找消息的效率。
- Kafka 中的索引文件以**稀疏索引**（sparse index）的方式构造消息的索引。
- 偏移量索引文件用来建立消息偏移量（offset）到物理地址之间的映射关系，方便快速定位消息所在的物理文件位置；时间戳索引文件则根据指定的时间戳（timestamp）来查找对应的偏移量信息。
- 在索引文件切分的时候，Kafka 会关闭当前正在写入的索引文件并置为只读模式，同时以可读写的模式创建新的索引文件。
- 偏移量索引（跳跃表结构）
  - relativeOffset：相对偏移量，表示消息相对于baseOffset 的偏移量，占用4个字节，当前索引文件的文件名即为baseOffset的值。
  - position：物理地址，也就是消息在日志分段文件中对应的物理位置，占用4个字节。
- 时间戳索引（二分查找，顺序递增）
  - timestamp：当前日志分段最大的时间戳。（8B)
  - relativeOffset：时间戳所对应的消息的相对偏移量。（4B)

### 4.日志清理

- 清除策略：
  - 日志删除（Log Retention）：按照一定的**保留策略**直接删除不符合条件的**日志分段**。
  - 日志压缩（Log Compaction）：针对每个消息的 key 进行整合，**对于有相同 key 的不同 value 值，只保留最后一个版本。**
- broker 端：通过 log.cleanup.policy 参数设置日志清理策略。（delete、compact）可同时支持两种策略。
- 日志删除（Log Retention）===> 默认
  - 专门的日志删除任务来**周期性地检测和删除**不符合保留条件的日志分段文件。
  - 删除日志分段
    - 首先会从 Log 对象中所维护日志分段的跳跃表中移除待删除的日志分段，以保证没有线程对这些日志分段进行读取操作。
    - 然后将日志分段所对应的所有文件添加上“.deleted”的后缀（当然也包括对应的索引文件）。
    - 最后交由一个以“delete-file”命名的延迟任务来删除这些以“.deleted”为后缀的文件，这个任务的延迟执行时间可以通过 file.delete.delay.ms 参数来调配，此参数的默认值为 60000，即1分钟。
  - 保留策略：
    - 基于时间
      - 默认情况下只配置了log.retention.hours参数，其值为168，故默认情况下日志分段文件的保留时间为7天。
      - 根据日志分段中最大的时间戳 largestTimeStamp 来计算。
    - 基于日志大小
      - 检查当前日志的大小是否超过设定的阈值
    - 基于日志起始偏移量
      - 基于日志起始偏移量的保留策略的判断依据是某日志分段的下一个日志分段的起始偏移量baseOffset 是否小于等于logStartOffset，若是，则可以删除此日志分段。
- 日志压缩（Log Compaction）
  - 每一个日志目录下都有一个名为“cleaner-offset-checkpoint”的文件，这个文件就是清理检查点文件，用来记录每个主题的每个分区中已清理的偏移量。通过检查点 cleaner checkpoint 来划分出一个已经清理过的 clean 部分和一个还未清理过的 dirty 部分。
  - Kafka中用于保存消费者消费位移的主题 __consumer_offsets 使用的就是 Log Compaction 策略。
  - 筛选操作：
    - Kafka 中的每个日志清理线程会使用一个名为“SkimpyOffsetMap”的对象来构建 key 与 offset 的映射关系的哈希表。
    - 日志清理需要遍历两次日志文件
      - 第一次遍历把每个 key 的哈希值和最后出现的 offset 都保存在 SkimpyOffsetMap 中。
      - 第二次遍历会检查每个消息是否符合保留条件，如果符合就保留下来，否则就会被清理。
  - 墓碑消息：如果一条消息的 key 不为 null，但是其 value 为 null，那么此消息就是墓碑消息。
  - 为了防止出现太多的小文件，Kafka 在实际清理过程中并不对单个的日志分段进行单独清理，而是将日志文件中 offset 从 0 至 firstUncleanableOffset 的所有日志分段进行分组，每个日志分段只属于一组。**同一个组的多个日志分段清理过后，只会生成一个新的日志分段。**

### 5.磁盘存储

- Kafka 依赖于文件系统（更底层地来说就是磁盘）来存储和缓存消息。
- 顺序内存 >> 顺序磁盘 > 随机内存 >> 随机磁盘
- Kafka 采用了**文件追加的方式**来写入消息，即只能在日志文件的尾部追加新的消息，并且也不允许修改已写入的消息，这种方式属于**典型的顺序写盘**的操作。
- 页缓存：把磁盘中的数据缓存到内存中，把对磁盘的访问变为对内存的访问。
  - 被修改过后的页也就变成了脏页，操作系统会在合适的时间把脏页中的数据写入磁盘，以保持数据的一致性。
  - 对脏页进行处理的过程中，新的 I/O 请求会被阻挡直至所有脏页被冲刷到磁盘中。
  - 文件系统 + 依赖于页缓存 > 维护一个进程内缓存或其他结构（至少省去了一份进程内部的缓存消耗，还可以通过结构紧凑的字节码来替代使用对象的方式以节省更多的空间，即使Kafka服务重启，页缓存还是会保持有效，然而进程内的缓存却需要重建。）
- Linux 系统中的 I/O 调度策略：
  - NOOP
  - CFQ
  - DEADLINE
  - ANTICIPATORY
- 零拷贝
  - 零拷贝是指将数据直接从磁盘文件复制到网卡设备中，而不需要经由应用程序之手。
  - 减少了内核和用户模式之间的上下文切换。
  - 对 Linux操作系统而言，零拷贝技术依赖于底层的 **sendfile（）**方法实现。对应于 Java 语言，FileChannal.transferTo（）方法的底层实现就是sendfile（）方法。
  - 零拷贝技术通过 DMA（Direct Memory Access）技术将文件内容复制到内核模式下的 Read Buffer 中
  - 零拷贝是针对内核模式而言的，数据在内核模式下实现了零拷贝。

## 深入服务端

### 1.协议设计

- 每种协议类型都有对应的请求（Request）和响应（Response），它们都遵守特定的协议模式。
  - Request：
    - 相同结构的协议请求头（RequestHeader）：api_key、api_version、correlation_id 和 client_id
    - 不同结构的协议请求体（RequestBody）
  - Response：
    - 相同结构的协议响应头（ResponseHeader）
    - 不同结构的响应体（ResponseBody）
- 消息发送的协议类型，即 ProduceRequest/ProduceResponse，对应的 **api_key=0，表示 PRODUCE**。
- 拉取消息的协议类型，即 FetchRequest/FetchResponse，对应的 **api_key=1，表示 FETCH**。



### 2.时间轮

- kafka 基于时间轮的概念自定义实现了一个用于延时功能的定时器（SystemTimer）。

- 时间轮结构

  ![](images/kafka时间轮结构.jfif)

- 层级时间轮：当任务的到期时间超过了当前时间轮所表示的时间范围时，就会尝试添加到上层时间轮中。

  - 我们常见的钟表就是一种具有三层结构的时间轮：
    - 第一层时间轮 tickMs = 1s、wheelSize = 60、interval = 1min，此为秒钟；
    - 第二层 tickMs = 1min、wheelSize = 60、interval = 1hour，此为分钟；
    - 第三层 tickMs = 1hour、wheelSize = 12、interval = 12hours，此为时钟。

- 时间轮降级操作：

- 细节：

  - TimingWheel 在创建的时候以当前系统时间为第一层时间轮的起始时间，调用了 Time.SYSTEM.hiResClockMs（采用了System.nanoTime（）/1_000_000来将精度调整到毫秒级）。
  - TimingWheel 中的每个双向环形链表 TimerTaskList 都会有一个哨兵节点（sentinel），引入哨兵节点可以简化边界条件。（也叫哑元节点：dummy node）
  - Kafka 中的定时器只需持有 TimingWheel 的第一层时间轮的引用。

- Kafka 中的定时器借了 JDK 中的 DelayQueue 来协助推进时间轮。采用DelayQueue来辅助以少量空间换时间，从而做到了“精准推进”。

- **Kafka 中的 TimingWheel 专门用来执行插入和删除 TimerTaskEntry 的操作，而 DelayQueue 专门负责时间推进的任务。**



### 3.延时操作

- 延时生产、延时拉取（DelayedFetch）、延时数据删除（DelayedDeleteRecords）
- 延时操作触发条件：
  - 超时：必须有一个超时时间（delayMs）
  - 外部事件：
    - follower副本的延时拉取，它的外部事件就是消息追加到了 leader 副本的本地日志文件中
    - 消费者客户端的延时拉取，它的外部事件可以简单地理解为 HW 的增长。
- 定时器（SystemTimer） + “收割机”线程（ExpiredOperationReaper） + 延时操作管理器（DelayedOperationPurgatory） + 监听池（监听外部事件）

### 4.控制器

- 选举一个 broker 控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态。
  - 职责：
    - 选举新的 leader 副本
    - 通知所有 broker 更新其元数据信息
    - 负责分区的重新分配
    - 监听分区、主题、broker 相关信息的变化及管理。
  - 在任意时刻，集群中有且仅有一个控制器。
  - 每个 broker 都会在内存中保存当前控制器的 brokerid 值，这个值可以标识为 activeControllerId。
  - 选举工作依赖于 zookeeper
    - 成功竞选为控制器的 broker 会在 ZooKeeper 中创建 /controller 这个临时（EPHEMERAL）节点
    - 与控制器有关的 /controller_epoch 节点，这个节点是持久（PERSISTENT）节点，节点中存放的是一个整型的controller_epoch值。controller_epoch 用于记录控制器发生变更的次数。——**Kafka 通过 controller_epoch 来保证控制器的唯一性，进而保证相关操作的一致性。**
  - 初始化 + 管理 上下文信息（Controller context）====> 多线程间同步问题
    - 解决方案：Kafka 的控制器使用单线程基于事件队列的模型，将每个事件都做一层封装，然后按照事件发生的先后顺序暂存到 LinkedBlockingQueue 中，最后使用一个专用的线程（ControllerEventThread）按照FIFO（First Input First Output，先入先出）的原则顺序处理各个事件，这样**不需要锁机制**就可以在多线程间维护线程安全。
- 优雅关闭 kafka：
  - kafka-server-stop.sh 脚本：通过 ps ax 的方式找出正在运行 Kafka 的进程号 PIDS，然后使用 kill-s TERM $PIDS的方式来关闭。
    - ps 命令的限制：ps命令限制输出的字符数不得超过页大小 PAGE_SIZE（4096）
    - 与 Kafka 进程有关的输出信息太长，所以 kafka-server-stop.sh 脚本在很多情况下并不会奏效。
  - 优雅关闭的两个步骤：
    - 获取 Kafka 的服务进程号 PIDS。可以使用 Java 中的 jps 命令或使用 Linux 系统中的 ps 命令来查看。
    - 使用 kill-s TERM $PIDS 或 kill-15 $PIDS 的方式来关闭进程，注意千万不要使用 kill -9 的方式。
  - 为什么是优雅的？除了正常关闭一些必要的资源，还会执行一个控制关闭（ControlledShutdown）的动作。
    - 使用 ControlledShutdown 的方式关闭 Kafka 有两个优点：
      - 一是可以让消息完全同步到磁盘上，在服务下次重新上线时不需要进行日志的恢复操作；
      - 二是 ControllerShutdown 在关闭服务之前，会对其上的 leader 副本进行迁移，这样就可以减少分区的不可用时间。
- 分区 Leader 的选举：（控制器负责具体实施）
  - 创建分区/分区上线时执行：选举策略为 OfflinePartitionLeaderElectionStrategy
    - 按照 AR 集合中副本的顺序查找第一个存活的副本，并且这个副本在 ISR 集合中
  - 分区进行重分配时执行：选举策略为 ReassignPartitionLeaderElectionStrategy
    - 从重分配的AR列表中找到第一个存活的副本，且这个副本在目前的ISR列表中。
  - 发生优先副本的选举时执行：选举策略为 PreferredReplicaPartitionLeaderElectionStrategy
    - 直接将优先副本设置为 leader 即可，AR集合中的第一个副本即为优先副本。
  - 当某节点被优雅地关闭，位于这个节点上的leader副本都会下线，与此对应的分区需要执行leader的选举：选举策略为 ControlledShutdownPartitionLeaderElectionStrategy
    - 从 AR 列表中找到第一个存活的副本，且这个副本在目前的ISR列表中，与此同时还要确保这个副本不处于正在被关闭的节点上。

### 5.服务器参数

- broker.id（配置不一致会抛出异常）
  - config/server.properties
  - meta.properties
  - 自动生成



## 深入客户端

### 1.分区分配策略

- Kafka 提供了消费者客户端参数 partition.assignment.strategy 来设置**消费者与订阅主题之间的分区分配策略。**
- RangeAssignor 策略：RangeAssignor 分配策略的原理是按照消费者总数和分区总数进行整除运算来获得一个跨度，然后**将分区按照跨度进行平均分配**，以保证分区尽可能均匀地分配给所有的消费者。
  - 分配不均匀，部分消费者过载
- RoundRobinAssignor 分配策略：
  - 消费组内的所有消费者的订阅信息相同时，分配均匀。
  - 如果同一个消费组内的消费者订阅的信息是不相同的，那么在执行分区分配的时候就不是完全的轮询分配，有可能导致分区分配得不均匀。
- StickyAssignor 分配策略
  - 目标：
    1. 分区的分配要尽可能均匀。（优先于第二个目标）
    2. 分区的分配尽可能与上次分配的保持相同。
  - 尽可能地让前后两次分配相同，进而减少系统资源的损耗及其他异常情况的发生。
    - 可以使分区重分配具备“黏性”，减少不必要的分区移动（即一个分区剥离之前的消费者，转而分配给另一个新的消费者）。
- 自定义分区策略：
  - 实现 PartitionAssignor 接口

### 2.消费者协调器和组协调器

- 将全部消费组分成多个子集，每个消费组的子集在服务端对应一个 GroupCoordinator 对其进行管理，GroupCoordinator 是 Kafka 服务端中用于管理消费组的组件。而**消费者客户端中的 ConsumerCoordinator 组件负责与 GroupCoordinator 进行交互。**
- 最重要的职责：**执行消费者再均衡的操作**。
- 触发再均衡操作的情形：
  - 有新的消费者加入消费组。
  - 有消费者宕机下线。
  - 有消费者主动退出消费组
  - 消费组所对应的 GroupCoorinator 节点发生了变更。
  - 消费组内所订阅的任一主题或者主题的分区数量发生变化。
- 消费者加入消费组的再均衡操作的四个阶段：
  - FIND_COORDINATOR：消费者需要确定它**所属的消费组对应的 GroupCoordinator 所在的broker**，并创建与该 broker 相互通信的网络连接。
    - 消费者已经保存了与消费组对应的 GroupCoordinator 节点的信息，并且与它之间的网络连接是正常的，进入第二阶段。
    - 否则，向集群中的 leastLoadedNode（最小负载节点）发送 FindCoordinatorRequest 请求来查找对应的 GroupCoordinator。
      - 根据消费组 groupId 的哈希值计算 **__consumer_offsets 中的分区编号**，找到对应的 __consumer_offsets 中的分区之后，再寻找此分区 leader 副本所在的 broker 节点，该 broker 节点即为这个 groupId 所对应的 GroupCoordinator 节点。
  - JOIN_GROUP：在此阶段的消费者会向 GroupCoordinator **发送 JoinGroupRequest 请求**，并处理响应。
    - 选举消费组的 leader：第一个或随机
    - 选举分区分配策略：投票数最多且都支持的策略。如果有消费者并不支持选出的分配策略，就会抛出异常。
    - JoinGroupRequest/JoinGroupResponse：
      - Kafka 发送给普通消费者的 JoinGroupResponse 中的 members 内容为空，而只有 leader 消费者的 JoinGroupResponse 中的 members 包含有效数据。
  - SYN_GROUP：各个消费者会向 GroupCoordinator 发送 SyncGroupRequest 请求来同步分配方案。
    - 只有 **leader 消费者发送的 SyncGroupRequest 请求中才包含具体的分区分配方案**，这个分配方案保存在 group_assignment 中，而其余消费者发送的 SyncGroupRequest 请求中的 group_assignment 为空。
    - GroupCoordinator 同样会先对 SyncGroupRequest 请求做合法性校验，在此之后会将从 leader 消费者发送过来的**分配方案提取出来，连同整个消费组的元数据信息一起存入Kafka的 __consumer_offsets主题中**，最后发送响应给各个消费者以提供给各个消费者各自所属的分配方案。
  - HEARTBEAT：心跳线程是一个独立的线程，可以在轮询消息的空档发送心跳。



### 3.__consumer_offsets 剖析

- 位移提交的内容最终会保存到 Kafka 的内部主题 __consumer_offsets 中
- 客户端提交消费位移是使用 OffsetCommitRequest 请求实现的
  - 保留时长：offsets.retention.minutes（默认为 7 天）
- 可以通过 kafka-console-consumer.sh 脚本来查看 __consumer_offsets 中的内容
  - 格式正常
  - 如果某个 key（version + group + topic + partition 的组合）对应的消费位移过期了，那么对应的 value 就会被设置为 null，也就是墓碑消息。
  - 消费位移过期：定时任务清理（默认 10 分钟）

### 4. 事务

- 消息传输保障

  - at most once：至多一次。消息可能会丢失，但绝对不会重复传输。
  - at least once：最少一次。消息绝不会丢失，但可能会重复传输。
  - exactly once：恰好一次。每条消息肯定会被传输一次且仅传输一次。

- 幂等：对接口的多次调用所产生的结果和调用一次是一致的。

  - kafka 的幂等性功能可以避免**生产者在进行重试的时候有可能会重复写入消息**的情况。
  - 开启幂等性功能：将生产者客户端参数 enable.idempotence 设置为 true
    - retries 参数不能大于 5
    - acks 参数必须为 -1
  - 引入序列号来实现幂等也只是针对每一对＜PID，分区＞而言的，幂等只能保证单个生产者会话（session）中单分区的幂等：
    - producer id（PID）：消息发送到的每一个分区都有对应的序列号，这些序列号从 0 开始单调递增。生产者每发送一条消息就会将＜PID，分区＞对应的序列号的值加1。
    - sequence number（序列号）

- 事务：事务可以保证对多个分区写入操作的原子性。

  - Kafka 中的事务可以使应用程序将**消费消息、生产消息、提交消费位移当作原子操作来处理**，同时成功或失败，即使该生产或消费会跨多个分区。

  - 事务要求生产者**开启幂等特性**，transactionalId 与 PID 一一对应，两者之间所不同的是 transactionalId 由用户显式设置，而 PID 是由 Kafka 内部分配的。

  - **控制消息**（ControlBatch）：控制消息一共有两种类型：COMMIT 和 ABORT，分别用来表征事务已经成功提交或已经被成功中止。

    - KafkaConsumer 可以通过这个控制消息来判断对应的事务是被提交了还是被中止了，然后结合参数 isolation.level 配置的隔离级别来决定是否将相应的消息返回给消费端应用。

  - **事务协调器**（TransactionCoordinator）：每一个生产者都会被指派一个特定的 TransactionCoordinator，所有的事务逻辑包括分派 PID 等都是由 TransactionCoordinator 来负责实施的。

  - consume-transform-produce 流程：

    1. **查找 TransactionCoordinator**：找出对应的 TransactionCoordinator 所在的broker节点（逻辑同查找GroupCoordinator的逻辑）
    2. **获取 PID**：为当前生产者分配一个 PID。===> 通过InitProducerIdRequest请求来实现
       - 保存 PID：＜transaction_Id，PID＞的对应关系以消息的形式保存到 **__transaction_state** 中持久化，保证即使 TransactionCoordinator 宕机该对应关系也不会丢失。
    3. **开启事务**：通过 KafkaProducer 的 beginTransaction（）方法可以开启一个事务
    4. Consume-Transform-Produce
       - 给一个新的分区发送数据：发送 AddPartitionsToTxnRequest
       - 发送消息：ProduceRequest
       - 在一个事务批次里处理消息的消费和发送：AddOffsetsToTxnRequest
       - 将本次事务中包含的消费位移信息 offsets 存储到主题 __consumer_offsets 中：TxnOffsetCommitRequest
    5. 提交或中止事务
       - 向 TransactionCoordinator 发送 **EndTxnRequest 请求**，以此来通知它提交（Commit）事务还是中止（Abort）事务。
       - 由 TransactionCoordinator 向事务中各个分区的leader 节点发送 **WriteTxnMarkersRequest 请求**，当节点收到这个请求之后，会在相应的分区中写入控制消息（ControlBatch）
       - TransactionCoordinator 将最终的 COMPLETE_COMMIT 或 COMPLETE_ABORT 信息写入主题 **__transaction_state**以表明当前事务已经结束，此时可以删除主题 __transaction_state 中所有关于该事务的消息（日志压缩策略，设置为墓碑消息即可）。

    

## 可靠性探究

### 1.副本探究

- 失效副本：
  - 处于同步失效或功能失效的副本统称为失效副本，失效副本对应的分区也就称为同步失效分区，即 under-replicated 分区。
  - 导致副本失效的两种情况：
    - follower 副本进程卡住，在一段时间内根本没有向leader副本发起同步请求，比如频繁的 Full GC。
    - follower 副本进程同步过慢，在一段时间内都无法追赶上 leader 副本，比如 I/O 开销过大。
  - 如果只用一个指标来衡量 Kafka，那么同步失效分区（具有失效副本的分区）的个数必然是首选。
- ISR 的伸缩
  - 两个与 ISR 相关的定时任务
    - isr-expiration 任务会周期性地检测每个分区是否需要缩减其 ISR 集合。
    - isr-change-propagation 任务将ISR集合的变化写入目标节点。
- LEO 与 HW
  - 本地副本和远程副本
    - 本地副本：对应的 Log 分配在当前的 broker 节点上
    - 远程副本：对应的 Log 分配在其他的 broker 节点上
  - leader 副本的 HW 直接影响了**分区数据对消费者的可见性。**
- Leader Epoch：保存 leader 版本号
  - 解决**数据丢失**和 **leader 副本和 follower 副本数据不一致**的问题。
  - **在需要截断数据的时候使用 leader epoch 作为参考依据**而不是原本的 HW。

### 2.日志同步机制

- 日志同步机制的一个基本原则就是：如果告知客户端已经成功提交了某条消息，那么即使 leader宕机，也要保证新选举出来的 leader 中能够包含这条消息。
- tradeoff
  - 少数服从多数：zookeeper
    - 优势：系统的延迟取决于最快的几个节点
    - 劣势：在生产环境下为了保证较高的容错率，必须要有大量的副本，而大量的副本又会在大数据量下导致性能的急剧下降。
  - ISR 模型：处于 ISR 集合内的节点保持与 leader 相同的高水位（HW），只有位列其中的副本才有资格被选为新的 leader。
    - Kafka 的同步方式允许宕机副本重新加入 ISR 集合，但在进入 ISR 之前必须保证自己能够重新同步完 leader 中的所有数据。



### 3.可靠性分析

在只考虑Kafka本身使用方式的前提下如何最大程度地提高可靠性。

- **副本数**：一般为 3 或 5。====> 副本数越多会引起磁盘、网络带宽的浪费,同时会引起性能的下降。
- **acks 参数**：设置为 -1/all。最大程度提高可靠性。===> 等待 ISR 中 follower 全部同步。
- **min.insync.replicas 参数**：指定了 ISR 集合中最小的副本数，如果不满足条件就会抛出异常。**搭配 acks：-1 使用。**
  - 一个典型的配置方案为：副本数配置为 3，min.insync.replicas 参数值配置为 2。
- **同步/异步模式**：发生可重试异常 ===> 客户端内部本身提供了重试机制来应对这种类型的异常，通过 retries 参数即可配置。
- **unclean.leader.election.enable**（默认为 false）：是否可以从非 ISR 集合中选举出新的 leader。（有可能造成数据丢失）
- **调整同步刷盘的策略**：一个组件（尤其是大数据量的组件）的可靠性不应该由同步刷盘这种极其损耗性能的操作来保障，而应该**采用多副本的机制来保障**。
- enable.auto.commit 参数（默认为 true）：可能带来重复消费和消息丢失的问题。
  - 在执行手动位移提交的时候也要遵循一个原则：**如果消息没有被成功消费，那么就不能提交所对应的消费位移。**
- 对于消费端，Kafka 还提供了一个可以兜底的功能，即回溯消费。

## **命令行工具**

| 脚本名称                            | 释义                                          |
| ----------------------------------- | --------------------------------------------- |
| kafka-config.sh                     | 用于配置管理                                  |
| kafka-console-consumer.sh           | 消费消息                                      |
| kafka-console-producer.sh           | 生产消息                                      |
| kafka-consumer-perf-test.sh         | 测试消费性能                                  |
| kafka-producer-perf-test.sh         | 测试生产性能                                  |
| kafka-topics.sh                     | 管理 topic                                    |
| kafka-dump-log.sh                   | 查看日志内容                                  |
| kafka-server-start.sh               | 启动 Kafka 服务，跟上启动配置文件             |
| kafka-server-stop.sh                | 关闭 Kafka 服务                               |
| kafka-preferred-replica-election.sh | 优先副本的选举                                |
| kafka-reassign-partitions.sh        | 分区重分配                                    |
| kafka-consumer-groups.sh            | 消费组管理，消费位移管理                      |
| kafka-delete-records.sh             | 手动删除消息                                  |
| connect-standalone.sh               | 实现以独立的模式运行 KafkaConnect             |
| connect-distributed.sh              | 实现以分布式模式运行 KafkaConnect             |
| kafka-mirror-maker.sh               | 运行 Kafka Mirror Maker，实现集群之间数据同步 |

