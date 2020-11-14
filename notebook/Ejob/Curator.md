## 一、简介

Apache Curator 是一个比较完善的 ZooKeeper 客户端框架，通过封装的一套高级 API 简化了 ZooKeeper 的操作。Curator 主要解决了三类问题：

- 封装 ZooKeeper client 与 ZooKeeper server 之间的连接处理
- 提供了一套 Fluent 风格的操作 API
- 提供 ZooKeeper 各种应用场景(recipe， 比如：**分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等**)的抽象封装

Curator 主要从以下几个方面降低了 zk 使用的复杂性：

- **重试机制**：提供可插拔的重试机制, 它将给捕获所有可恢复的异常配置一个重试策略，并且内部也提供了几种标准的重试策略(比如指数补偿)
- **连接状态监控**： Curator 初始化之后会一直对 zk 连接进行监听，一旦发现连接状态发生变化将会作出相应的处理。
- **zk 客户端实例管理**：Curator 会对 zk 客户端到 server 集群的连接进行管理，并在需要的时候重建 zk 实例，保证与 zk 集群连接的可靠性。
- 各种使用场景支持：Curator 实现了 zk 支持的大部分使用场景（甚至包括 zk 自身不支持的场景），这些实现都遵循了 zk 的最佳实践，并考虑了各种极端情况。

Maven 依赖

```xml
	<!-- 对zookeeper的底层api的一些封装 -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>2.12.0</version>
    </dependency>
    <!-- 封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式Barrier -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>2.12.0</version>
    </dependency>
```



## 二、基于 Curator 的 ZooKeeper 基本用法

### 基本 API

- CuratorFrameworkFactory
  - builder()：构建 CuratorFramework
  - connectString()：Zookeeper 服务地址（ZKHost）
  - connectionTimeoutMs()：连接超时时间
  -  sessionTimeoutMs：会话超时时间
  -  retryPolicy()：重试策略，
  -  namespace()：命名空间
- CuratorFramework
  - start()：开启连接
  - create()：创建节点，默认为永久节点
    - creatingParentContainersIfNeeded()方法：如果指定节点的父节点不存在，则Curator将会自动级联创建父节点
  - forPath()：节点路径
  - withNode()：节点模式
    -  CreateMode.PERSISTENT_SEQUENTIAL：永久有序节点
    -  CreateMode.EPHEMERAL：临时节点
    -  CreateMode.EPHEMERAL_SEQUENTIAL：临时有序节点
  -  checkExists()：测试检查某个节点是否存在
  -  getChildren()：获得某个节点下的所有子节点
  - 节点数据相关：
    - getData()：获得某个节点数据
    - setData()：设置节点数据
    - orSetData()：如果节点存在则 Curator 将会使用给出的数据设置这个节点的值，相当于 setData() 方法。
  -  delete()：删除节点
    - deletingChildrenIfNeeded()：删除级联子节点
    - guaranteed()：如果服务端可能删除成功，但是 client 没有接收到删除成功的提示，Curator 将会在后台持续尝试删除该节点。
    - deletingChildrenIfNeeded()：如果待删除节点存在子节点，则 Curator 将会级联删除该节点的子节点。

```java
public class CuratorBase {
    // 会话超时时间
    private final int SESSION_TIMEOUT = 30 * 1000;

    // 连接超时时间
    private final int CONNECTION_TIMEOUT = 3 * 1000;

    // ZooKeeper 服务地址
    private static final String CONNECT_ADDR = "192.168.1.1:2100,192.168.1.1:2101,192.168.1.:2102";

    // 创建连接实例
    private CuratorFramework client = null;

    public static void main(String[] args) throws Exception {
        // 1.重试策略：初试时间为 1s 重试 10 次
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
        // 2.通过工厂创建连接
        CuratorFramework client = CuratorFrameworkFactory.builder()
                    .connectString(CONNECT_ADDR).connectionTimeoutMs(CONNECTION_TIMEOUT)
                    .sessionTimeoutMs(SESSION_TIMEOUT)
                    .retryPolicy(retryPolicy)
            		// 命名空间
			        .namespace("super")
                    .build();
        // 3.开启连接
        client.start();

        System.out.println(States.CONNECTED);
        System.out.println(cf.getState());

        // 创建永久节点
        client.create().forPath("/curator","/curator data".getBytes());

        // 创建永久有序节点
        client.create().withMode(CreateMode.PERSISTENT_SEQUENTIAL)
                .forPath("/curator_sequential","/curator_sequential data".getBytes());

        // 创建临时节点
        client.create().withMode(CreateMode.EPHEMERAL)
            .forPath("/curator/ephemeral","/curator/ephemeral data".getBytes());

        // 创建临时有序节点
        client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL) .forPath("/curator/ephemeral_path1","/curator/ephemeral_path1 data".getBytes());

        client.create().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
            .forPath("/curator/ephemeral_path2","/curator/ephemeral_path2 data".getBytes());

        // 测试检查某个节点是否存在
        Stat stat1 = client.checkExists().forPath("/curator");
        Stat stat2 = client.checkExists().forPath("/curator2");

        System.out.println("'/curator'是否存在： " + (stat1 != null ? true : false));
        System.out.println("'/curator2'是否存在： " + (stat2 != null ? true : false));

        //获取某个节点的所有子节点
        System.out.println(client.getChildren().forPath("/"));

        //获取某个节点数据
        System.out.println(new String(client.getData().forPath("/curator")));

        //设置某个节点数据
        client.setData().forPath("/curator","/curator modified data".getBytes());

        //创建测试节点
        client.create().orSetData().creatingParentContainersIfNeeded()
            .forPath("/curator/del_key1","/curator/del_key1 data".getBytes());

        client.create().orSetData().creatingParentContainersIfNeeded()
        .forPath("/curator/del_key2","/curator/del_key2 data".getBytes());

        client.create().forPath("/curator/del_key2/test_key","test_key data".getBytes());

        //删除该节点
        client.delete().forPath("/curator/del_key1");

        //级联删除子节点
        client.delete().guaranteed().deletingChildrenIfNeeded().forPath("/curator/del_key2");
    ｝

｝
```

### 事务管理

- CuratorOp
- CuratorTransaction
- CuratorTransactionFinal
- transactionOp()
- forOperations()

```java
	/* 
	 *	事务管理：碰到异常，事务会回滚
     * @throws Exception
     */
    @Test
    public void testTransaction() throws Exception{
        // 定义几个基本操作
        CuratorOp createOp = client.transactionOp().create()
                .forPath("/curator/one_path","some data".getBytes());
        
        CuratorOp setDataOp = client.transactionOp().setData()
                .forPath("/curator","other data".getBytes());
        
        CuratorOp deleteOp = client.transactionOp().delete()
                .forPath("/curator");
        
        // 事务执行结果
        List<CuratorTransactionResult> results = client.transaction()
                .forOperations(createOp,setDataOp,deleteOp);
        
        // 遍历输出结果
        for(CuratorTransactionResult result : results){
            System.out.println("执行结果是： " + result.getForPath() + "--" + result.getType());
        }
    }
// 因为节点“/curator”存在子节点，所以在删除的时候将会报错，事务回滚
```



## 三、监听器

Curator 提供了三种 Watcher(Cache) 来监听结点的变化：

- **Path Cache**：监视一个路径下1）孩子结点的创建、2）删除，3）以及结点数据的更新。产生的事件会传递给注册的 PathChildrenCacheListener。
- **Node Cache**：监视一个结点的创建、更新、删除，并将结点的数据缓存在本地。
- **Tree Cache**：Path Cache 和 Node Cache 的“合体”，监视路径下的创建、更新、删除事件，并缓存路径下所有孩子结点的数据。

/**
         * 在注册监听器的时候，如果传入此参数，当事件触发时，逻辑由线程池处理
                  */
        ​        ExecutorService pool = Executors.newFixedThreadPool(2);

```java
    /**
     * 监听数据节点的变化情况
     */
    final NodeCache nodeCache = new NodeCache(client, "/zk-huey/cnode", false);
    nodeCache.start(true);
    nodeCache.getListenable().addListener(
        new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                System.out.println("Node data is changed, new data: " + 
                    new String(nodeCache.getCurrentData().getData()));
            }
        }, 
        pool
    );
    
    /**
     * 监听子节点的变化情况
     */
    final PathChildrenCache childrenCache = new PathChildrenCache(client, "/zk-huey", true);
    childrenCache.start(StartMode.POST_INITIALIZED_EVENT);
    childrenCache.getListenable().addListener(
        new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event)
                    throws Exception {
                    switch (event.getType()) {
                    case CHILD_ADDED:
                        System.out.println("CHILD_ADDED: " + event.getData().getPath());
                        break;
                    case CHILD_REMOVED:
                        System.out.println("CHILD_REMOVED: " + event.getData().getPath());
                        break;
                    case CHILD_UPDATED:
                        System.out.println("CHILD_UPDATED: " + event.getData().getPath());
                        break;
                    default:
                        break;
                }
            }
        },
        pool
    );
    
    client.setData().forPath("/zk-huey/cnode", "world".getBytes());
    
    Thread.sleep(10 * 1000);
    pool.shutdown();
    client.close();
```


## 四、分布式锁



## 五、Leader 选举

当集群里的某个服务 down 机时，我们可能要从 slave 结点里选出一个作为新的 master，这时就需要一套能在分布式环境中自动协调的 Leader 选举方法。

Curator 提供了 LeaderSelector 监听器实现 Leader 选举功能。同一时刻，只有一个 Listener 会进入 takeLeadership() 方法，说明它是当前的 Leader。

注意：**当 Listener 从 takeLeadership() 退出时就说明它放弃了“Leader 身份”**，这时 Curator 会利用 Zookeeper 再从剩余的 Listener 中选出一个新的 Leader。autoRequeue() 方法使放弃 Leadership 的 Listener 有机会重新获得 Leadership，如果不设置的话放弃了的 Listener 是不会再变成 Leader 的。