### 功能

1. 创建 Topic：createTopics(Collection<NewTopic> newTopics)
2. 删除 Topic：deleteTopics(Collection<String> topics)
3. 罗列所有 Topic：listTopics()
4. 查询 Topic：describeTopics(Collection<String> topicNames)
5. 查询集群信息：describeCluster()
6. 查询 ACL 信息：describeAcls(AclBindingFilter filter)
7. 创建 ACL 信息：createAcls(Collection<AclBinding> acls)
8. 删除 ACL 信息：deleteAcls(Collection<AclBindingFilter> filters)
9. 查询配置信息：describeConfigs(Collection<ConfigResource> resources)
10. 修改配置信息：alterConfigs(Map<ConfigResource, Config> configs)
11. 修改副本的日志目录：alterReplicaLogDirs(Map<TopicPartitionReplica, String> replicaAssignment)
12. 查询节点的日志目录信息：describeLogDirs(Collection<Integer> brokers)
13. 查询副本的日志目录信息：describeReplicaLogDirs(Collection<TopicPartitionReplica> replicas)
14. 增加分区：createPartitions(Map<String, NewPartitions> newPartitions)

### 实现原理

- 其内部原理是使用 Kafka 自定义的一套二进制协议来实现。主要实现步骤：
  1. 客户端根据方法的调用创建相应的协议请求，比如创建 Topic 的 createTopics 方法，其内部就是发送 CreateTopicRequest 请求。
  2. 客户端发送请求至 Kafka Broker。
  3. Kafka Broker 处理相应的请求并回执，比如与 CreateTopicRequest 对应的是 CreateTopicResponse。
  4. 客户端接收相应的回执并进行解析处理。
     和协议有关的请求和回执的类基本都在 `org.apache.kafka.common.requests` 包中，AbstractRequest 和 AbstractResponse 是这些请求和回执类的两个基本父类。