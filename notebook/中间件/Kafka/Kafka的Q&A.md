1. 只有 leader 副本对外提供读写服务，而 follower 副本只负责在内部进行消息的同步。当分区的 leader 节点发生故障时，其中一个 follower 节点就会成为新的 leader 节点，这样就会导致集群的负载不均衡；原来的 leader 节点恢复之后重新加入集群时，它只能成为一个新的 follower 节点而不再对外提供服务。
   - 为了能够有效地治理负载失衡的情况，Kafka 引入了优先副本（preferred replica）的概念。
   - 解决方案：**优先副本的选举**。Kafka 要确保所有主题的优先副本在 Kafka 集群中均匀分布，这样就保证了所有分区的 leader 均衡分布。优先副本的选举，即通过一定的方式促使优先副本选举为 leader 副本，也称为**分区平衡**。实际生产中，使用手动执行优先副本的选举。
2. 一个节点上的分区副本都处于功能失效的状态时，Kafka 不会将这些失效的分区副本自动地迁移到集群中剩余的可用 broker 节点上；集群新增 broker 节点时，只有新创建的主题分区才有可能被分配到这个节点上。
   - 解决方案：**分区重分配**。重分配后的负载均衡问题，使用上述的优先副本选举来解决。
   - 分区重分配本质在于数据复制，先增加新的副本，然后进行数据同步，最后删除旧的副本来达到最终的目的。
   - 减小重分配的粒度，以小批次的方式来操作是一种可行的解决思路。
3. listeners 和 advertised.listeners 的区别
   - listeners：是 kafka 真正 bind 的地址
   - advertised.listeners：是暴露给外部的 listeners，如果没有设置，会用 listeners
4. 为什么 Kafka 不支持读写分离（主写从读）？
   - 主写从读有两个明显的缺点：
     1. 数据一致性问题：数据从主节点转到从节点必然会有一个延时的时间窗口，这个时间窗口会导致主从节点之间的数据不一致。
     2. 延时问题：在 Kafka 中，主从同步会比 Redis 更加耗时，它需要经历网络→主节点内存→主节点磁盘→网络→从节点内存→从节点磁盘这几个阶段。比 Redis 多了从内存到磁盘的动作。
   - 主写主读的优点：
     1. 可以简化代码的实现逻辑，减少出错的可能；
     2. 将负载粒度细化均摊，与主写从读相比，不仅负载效能更好，而且对用户可控；（保证分区分配均衡，leader 副本切换均匀）
     3. 没有延时的影响；
     4. 在副本稳定的情况下，不会出现数据不一致的情况。
5. 为什么 Kafka 只支持增加分区数而不支持减少分区数？
   - 此功能完全可以实现，不过会使代码的复杂度急剧增大。
     - 删除的分区中的消息该如何处理？
     - 顺序性问题、事务性问题，以及分区和副本的状态机切换问题
   - 如果真的需要实现此类功能，则完全可以重新创建一个分区数较小的主题，然后将现有主题中的消息按照既定的逻辑复制过去即可。
6. 为什么 kafka 使用**文件系统 + 页缓存**的方式存储，而不使用进程内缓存存储？
   - 对象的内存开销非常大，通常会是真实数据大小的几倍甚至更多，空间使用率低下；Java 的垃圾回收会随着堆内数据的增多而变得越来越慢。（**通过结构紧凑的字节码来替代使用对象的方式以节省更多的空间**）
   - 在进程内部缓存处理所需的数据，然而这些数据有可能还缓存在操作系统的页缓存中，因此同一份数据有可能被缓存了两次。（**省去了一份进程内部的缓存消耗**）
   - 即使 Kafka 服务重启，页缓存还是会保持有效，然而进程内的缓存却需要重建。
7. acks 参数不同取值的意义？
   - -1/all：需要**等待 ISR 集合中的所有副本都确认收到消息**之后才能正确地收到响应的结果，或者捕获超时异常。
     - 搭配 
   - 0：发完即忘（fire-and-forget）
   - 1（默认设置）：**只要 Partition Leader 接收到消息**而且写入本地磁盘了，就认为成功了，不管他其他的 Follower 有没有同步过去这条消息了。
8. 五个检查点文件的用途：
   - cleaner-offset-checkpoint：清理检查点文件，用来记录每个主题的每个分区中已清理的偏移量。
   - recovery-point-offset-checkpoint：对应 LEO，有一个定时任务负责将所有分区的 LEO 刷写到恢复点文件 recovery-point-offset-checkpoint
   - replication-offset-checkpoint：对应 HW，一个定时任务负责将所有分区的HW刷写到复制点文件 replication-offset-checkpoint 中
   - log-start-offset-checkpoint：文件对应 logStartOffset，标识日志的起始偏移量
   - leader-epoch-checkpoint：发生leader epoch变更时，会将对应的矢量对＜LeaderEpoch=＞StartOffset＞追加到这个文件中
9. KafkaProducer 调用 sender.wakeup() 方法 的地方及作用：
   - KafkaProducer 中 `dosend()` 方法调用 `sender.wakeup()` 方法作用就是：当有新的 RecordBatch 创建后，旧的 RecordBatch 就可以发送了（或者此时有 Metadata 请求需要发送），如果线程阻塞在 `select()` 方法中，就将其唤醒，Sender 重新开始运行 `run()` 方法，在这个方法中，旧的 RecordBatch （或相应的 Metadata 请求）将会被选中，进而可以及时将这些请求发送出去。
   -  waitOnMetadata() 方法调用 sender.wakeup() 方法的作用：通过 `sender.wakeup()` 来唤醒 sender 线程，间接唤醒 NetworkClient 线程，NetworkClient.poll() 方法来负责发送 Metadata 请求，并处理 Server 端的响应。
10. KafkaProducer 如果保证单 partition 顺序性？
    - KafkaProducer 的 `max.in.flight.requests.per.connection` 设置为 1
    - 在 Sender 线程中调用 RecordAccumulator 的  `mutePartition()` 与  `unmutePartition()` 方法
      - 将指定的 topic-partition 从 muted 集合中加入或删除
      - 在 RecordAccumulator 的 ready 和 drain 方法中，如果 `muted` 集合包含这个 tp，那么在遍历时将不会处理它对应的 deque
        - 即使它对应的 RecordBatch 可以发送了，也不会触发引起其对应的 leader 被选择出来。
        - 对应 deque 中其他 RecordBatch 即使达到条件也不会被发送，就保证了 tp 在任何时刻只有一个 RecordBatch 在发送。
11. Kafka consumer poll(long) 与 poll(Duration) 的区别
    - 在 poll(0) 中 consumer 会一直阻塞直到它成功获取了所需的元数据信息，之后它才会发起 fetch 请求去获取数据。	 
      - 如果远端的 broker 不可用了， 那么 consumer 程序会被无限阻塞下去。用户指定了超时时间但却被无限阻塞。可能导致的问题在于 Stream Thread无法正常关闭。
    - poll(Duration) 这个版本修改了这样的设计，会把元数据获取也计入整个超时时间。
      - consumer 根本无法在这么短的时间内连接上 coordinator，所以只能赶在超时前返回一个空集合。
12. Kafka offset 索引文件的存储形式和优点：
    - 存储形式： 
      1. 采用 **绝对偏移量+相对偏移量** 的方式进行存储的，每个 segment 最开始绝对偏移量也是其基准偏移量；
      2. 数据文件每隔一定的大小创建一个索引条目，而不是每条消息会创建索引条目，通过 `index.interval.bytes` 来配置，默认是 4096，也就是4KB
    - 优点：
      1. 因为不是每条消息都创建相应的索引条目，所以**索引条目是稀疏的**；
      2. 索引的相对偏移量占据 4 个字节，而绝对偏移量占据 8 个字节，加上物理位置的4个字节，**使用相对索引可以将每条索引条目的大小从 12 字节减少到 8 个字节；**
      3. 因为偏移量有序的，再读取数据时，可以按照**二分查找的方式去快速定位偏移量的位置**；
      4. 稀疏索引是**可以完全放到内存中**，加快偏移量的查找。
