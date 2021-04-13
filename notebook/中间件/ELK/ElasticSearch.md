### ELK

- 不仅适用于日志分析，还可以支持任何数据收集和分析的场景，日志收集和分析只是更具有代表性。



### 安装可视化界面 ES head 插件

- 依赖 node.js
- 地址：https://github.com/mobz/elasticsearch-head



## kibana

### kibana 安装

- 选择和 ES 对应的版本。

- 概述：针对 ES 的开源分析及可视化平台，用来搜索、查看、交互存储在 ES 索引中的数据。
- 在日常工作中使用 Kibana 来操作 ES 数据。
- 安装及启动过程：
  - 安装解压：https://www.elastic.co/cn/downloads/kibana
  - bin 目录下启动
  - 访问测试：http://localhost:5601/app/home#/
  - 开发工具：（postman、curl、head、谷歌浏览器插件）
    - 建议直接使用 kibana，工作中方便使用。
  - 汉化：
    - 目录：D:\ELK\kibana-7.9.3-windows-x86_64\x-pack\plugins\translations
    - 修改 kibana.yml 文件

### kibana 使用

- 检查 kibana 状态

```bash
# 可视化界面
http://localhost:5601/status
# json 格式
http://localhost:5601/api/status
```

- 

## ElasticSearch 

### 核心概念

- ElasticSearch 是面向文档的。

- 索引（Index）
- 全文搜索（Full-text Search）
- 倒排索引（Inverted Index）
- 节点&集群（Node & Cluster）
- 文档（Document）
- 文档元数据（Document metadata）
  - 文档元数据为 `_index`, `_type`,`_id`, 这三者可以唯一表示一个文档，`_index` 表示文档在哪存放，`_type` 表示文档的对象类别，`_id` 为文档的唯一标识。
- 字段（Fields）
- 

### 配置文件

- config 目录下：
  - elasticsearch.yml：配置 ES
  - jvm.options：配置 ES 的 JVM设置
  - log4j2.properties：配置 ES 的日志

- 配置文件默认放在 `$ES_HOME/config` 目录下，可通过配置 `ES_PATH_CONF` 环境变量修改。

- 可配置的类型：
  - 集群和节点：
    - 配置文件：elasticsearch.yml
    - 通过 cluster update settings API 配置
    - 重启后生效
    - 配置生效优先级：
      - Transient setting
      - Persistent setting
      - `elasticsearch.yml` setting
      - Default setting value
  - JVM 配置
    - 修改方式：
      - jvm.options 文件
      - ES_JAVA_OPTS 环境变量
    - 常用修改：
      - 修改堆大小
  - 安全配置
    - ES 提供了密钥
    - 只有密钥支持的设置才可以修改，配置不支持的设置会导致 ES 不能启动
    - 重启后生效。
    - 所有节点的配置必须一样。
    - **reloadable** 
  - HTTP 配置
  - Circuit breaker 设置
  - 分片分配和恢复设置
  - 集群备份设置

#### 重要的 ES 配置

- path.data 和 path.logs
  - 修改数据和日志的存放地址
- cluster.name
  - 节点通过共享 cluster.name 加入一个集群。
  - 确保不在不同的环境中使用相同的集群名。
- node.name
  - 通过 elasticsearch.yml 指定节点名称。
- network.host
  - 设置网络 ip，用来建立集群。
  - 开发模式和生产模式不同。
- 两个重要的 discovery 和 cluster formation 设置
  - discovery.seed_hosts
    - 设置集群的节点 ip:port 列表 (ip 或域名)
    - 默认 9300 端口号，可通过配置文件修改
    - ipv6 要用 [] 包含起来。
  - cluster.initial_master_nodes
    - 在生产环境中，必须指定第一个主节点
    - 重启集群或向集群中添加一个新节点是不能使用此配置。
    - 使用 node.name 来配置
- 设置 jvm 堆大小
  - 默认 Xmx 和 Xms 为 1 GB（至少）
  - 不能超过 50% 物理 RAM
  - 不能超过压缩指针（Compressed oops）的阈值（< 32GB)
  - 不能超过零基压缩优化（zero-based compressed oops)的阈值（26GB最佳）
  - 通过 jvm.options 文件或 ES_JAVA_OPTS 环境变量设置。
- 设置 jvm heap dump 路径
  - 在 jvm.options 文件中设置：-XX:HeapDumpPath=...
  - 指定目录时，jvm 会自动根据 pid 创建文件；指定文件时，该文件必须不存在，否则 dump 会失败。
- 设置 gc logging
  - 在 jvm.options 文件中配置，默认输出在 ES logs 配置的地址中。
  - 先配置 -Xlog:disable，再自定义 gc logging 配置。
- 设置临时目录
  - 启动 ES 前，通过 $ES_TMPDIR 环境变量设置临时目录
- 设置 JVM fetal error logs
  - 在 jvm.options 中通过 -XX:ErrorFile=... 设置

#### 重要的系统配置

- 默认情况下，ES 的配置都是开发环境的配置。当修改了 `network.host` ，ES 假定为生产环境，会通过警告和异常来禁止 ES 启动。重要的安全策略，防止丢失数据。

- 设置系统配置：（Linux 下）

  - ulimit：临时限制资源数量。（root 权限修改）

  - /etc/security/limits.conf：通过修改 limits.conf 文件持久设置 ES 的最大文件打开数。

    - ```sh
      elasticsearch  -  nofile  65535
      ```

  - systemd 设置

- 禁止 swaping

  - swaping 可能出现的问题：

    - 导致 out of disk
    - 导致 GC 持续数分钟
    - 导致节点响应缓慢甚至从集群中断开连接，在弹性分布式系统中，可能会导致操作系统 kill 该节点。

  - 最好的做法是完全禁止 swap

  - Linux 下：

    - sudo swapoff -a ： 临时禁止 swap（不需要重启 ES）

    - 永久禁止 swap：在 /etc/fstab 文件下注释掉所有包含 swap 的行

    - 配置 sysctl 的 vm.swappiness 值为 1。可以减少内核 swap 的趋势，但会允许在紧急情况下 swap

    - 开启 bootstrap.memory_lock：

      - elasticsearch.yml 文件下设置：bootstrap.memory_lock: true

      - 查看是否开启 memory_lock

        ```console
        GET _nodes?filter_path=**.mlockall
        ```

- FIle Descriptors （文件描述符）====> 同 ulimit 配置

  - 在 Windows 上可以忽略。
  - ES 使用了大量的文件描述符或文件句柄。文件描述符耗尽会导致丢失数据、服务停止...

- Virtual memory

  - root 设置：sysctl -w vm.max_map_count=262144
  - 持久化设置：在 /etc/sysctl.conf 文件下，设置 vm.max_map_count 的值

- 线程数量：至少 4096

  - ulimit -u 4096
  - 设置 /etc/security/limits.conf 中的 nproc  为 4096

- 设置 DNS 缓存



### 发现节点、建立集群



### 插件



### Index 模块



### Mapping 



### 文本分析



### Index 模板



### 数据流



### 搜索数据



### 查询 DSL



### 聚合分析



### 监控集群



### 搭建高可用集群



### 集群安全



### 术语表



### REST APIs



