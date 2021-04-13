## Exchange 实践

- **basicPublish**：
  - Exchange：发布到指定 Exchange 的所有队列上
  - RoutingKey：发布到对应的队列上（Fanout 中，该属性失效）
- **Fanout**
  - 需要创建临时队列，才能接受所有消息。====》相当于将消息同时发送到多个临时队列上，否则是在同一个队列上消费
  - 先启动消费者（生成临时队列），再启动生产者，消费者才能消费到消息。
- **Topic**
  - 将 Exchange 和 Queue 绑定，绑定的 RoutingKey 可以使用通配符。
  - 通配符匹配的队列都拥有生产者发送的全部消息。
- **消费者只需要监听相关队列**

- **死信队列**
  - 声明死信队列
  - 声明业务队列的时候绑定死信队列（配置死信 Exchange 和 RoutingKey）
  - Nack 或 Reject 后才由死信队列处理。