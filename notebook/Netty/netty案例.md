## Netty 优雅退出机制

### Java 优雅退出机制

- 通常**通过注册 JDK 的 ShutdownHook** 来实现，当系统接收到退出指令时，首先标记系统处于退出状态，不再接受新的消息，然后将积压的消息处理完，最后调用资源回收接口将资源销毁，线程退出执行。
- **注意点：**
  - ShutdownHook 在某些情况下并不会被执行，例如JVM崩溃、无法接收信号量和 `kill-9 pid` 等。
  - 当存在多个 ShutdownHook 时，JVM 无法保证它们的执行先后顺序。
  - 在 JVM 关闭期间不能动态添加或者去除 ShutdownHook。
  - 不能在 ShutdownHook 中调用 System.exit()，它会卡住 JVM，导致进程无法退出。

### Netty 优雅退出机制

- 当应用进程优雅退出的时候，作为通信框架的 Netty 也要优雅退出，原因如下：
  - 尽快释放 NIO 线程和句柄等资源。
  - 如果使用 flush 做批量消息发送，需要将积压在发送队列中的待发送消息发送完成。
  - 正在写或者读的消息，需要继续处理。
  - 设置在 NioEventLoop 线程调度器中的定时任务，需要执行或清理。
- 涉及的主要操作和资源对象：
  - 把 NIO线程的状态位设置成 ST_SHUTTING_DOWN，不再处理新的消息（不允许再对外发送消息）。
  - 预处理操作：
    - 通信队列中尚未发送的消息
    - NIO 线程中待处理的定时任务
    - 注册到 NIO 线程中的 ShutdownHook 任务
  - 资源释放：
    - 关闭所有的 channel
    - 从 Selector 上注册
    - 清空所有的队列和定时任务
    - NIO（EventLoop） 线程退出
- 调用的接口和总入口是 EventLoopGroup，调用它的  shutdownGracefully 方法即可。还可以指定退出的超时时间和周期

