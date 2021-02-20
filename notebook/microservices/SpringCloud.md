## 一、微服务架构开发

- 微服务核心思路就是分而治之。
- 服务：服务是一个可以独立运行、提供范围有限的功能（可以是业务功能，也有可能是非业务功能）的组件。





## 二、服务治理与负载均衡



### 服务治理——Eureka



### 客户端负载均衡——Ribbon



### 简化微服务调用——Feign



### 微服务健康监控——Actuator



### 异构服务解决方案——Sidecar



## 三、微服务容错保护——Hystrix



- 快速失败，而不是在队列中积压服务请求。
- 提供服务降级（fallback）处理机制。
- 使用隔离技术（如舱壁隔离、泳道和断路器等模式）来隔离服务依赖之间的影响。



### 服务隔离

- 实现服务隔离的思路：
  - 使用命令模式（HystrixCommand/HystrixObservableCommand）对服务调用进行封装，使每个命令在单独线程中/信号授权下执行。
  - 为每一个命令的执行提供一个小的线程池/信号量，当线程池/信号已满时，立即拒绝执行该命令，直接转入服务降级处理。
  - 为每一个命令的执行提供超时处理，当调用超时时，直接转入服务降级处理。
  - 提供断路器组件，通过设置相关配置及实时的命令执行数据统计，完成服务健康数据分析，使得在命令执行过程中可以快速判断是否可以执行，还是执行服务降级处理。
- 服务隔离策略：
  - 线程池隔离
  - 信号量隔离

### 服务降级模式



- 快速失败：在服务降级处理逻辑中不提供任何处理，直接抛出一个异常。
- 静默失败：当进行服务降级处理时返回空的结果，针对返回值类型，返回的可能是 null、空 List 或者空 Map 等。
- 返回默认值：返回静态的在代码中固定的值
- 返回组装的值
  - cookie
  - param/header
- 返回远程缓存：在服务处理失败的情况下再发起一次远程请求，不过这次请求的是一个缓存，比如读取 Redis 中的缓存结果。
- 主/从降级模式：使用门面模式构建新的命令
  - 由于主/从命令都是采用线程池隔离方式执行的，那么所构建的门面命令则可以使用信号量隔离方式，避免了系统多余的开销。



### 请求缓存

- Hystrix 的请求缓存处理是在 construct() 或 run() 方法调用之前，这样可以有效地减少业务服务请求数，降低了服务的并发。

- 同一请求中后续所有相同的 Hystrix 调用都将直接返回该缓存中的值，从而保证了数据的一致性。

- 常用方法：

  ```java
  /**
  	清除缓存：HystrixRequestCache.clear()
  	
  	判断是否是从缓存返回：isResponseFromCache()
  */
  ```

- 缓存注解：

  ```java
  /**
  	@CacheResult：标记返回结果需要进行缓存，该注解要与 @HystrixCommand 注解一起使用。
  	
  	@CacheKey：用来标记如果构建缓存的值，其功能类似于 getCacheKey() 方法。
  	
  	@CacheRemove：用来标记在方法执行完毕后清除指定的缓存。
  */
  ```

  



### 请求合并

- 多个请求能自动合并的前提：执行的间隔时长要足够小（默认为 ms，可通过 `hystrix.collapser.default.timerDelayInMilliseconds` 进行设置），即执行间隔超过 10ms 的请求不会合并执行。
- 实现批量处理



### Hystrix 监控

- Actuator 监控
- Hystrix 仪表盘监控





## 四、API 服务网关——Zuul

- API 服务网关是微服务访问的统一入口，负责服务请求路由、组合及协议转换等处理。

- API 服务网关的核心是：为所有客户端请求或其他消费者提供统一的网关，通过该网关接入不同的微服务，并隐藏架构实现的细节。

- 考虑因素：

  - 当使用 API 服务网关后，API 服务网关极有可能会成为系统的一个瓶颈，所以**高可用是构建 API 服务网关首要考虑的**。
  - API 服务网关的更新/注册过程要尽可能地简单。
  - API 服务网关需要知道与之通信的每个微服务服务器地址和端口信息，特别是在 API 服务网关中启用了负载均衡时。

- Zuul 组件可实现的功能：

  - **动态路由**：Zuul 路由服务器支持与 Eureka 服务器的整合，可以动态对注册到 Eureka 服务器中的微服务进行路由映射。
  - **监控与审查**：通过对一些特定的接口设置访问白名单、访问次数、访问频率等各类设置，可以在不影响微服务实现的情况下，对访问实施监控和审查处理。
  - **身份认证与安全**：通过 Zuul 可以将认证的部分单独抽取出来，让微服务系统无须关注认证的逻辑，只需要关注业务本身即可。可以统一在服务网关层增加一个额外的保护层来防止恶意攻击。
  - **压力测试**：通过 Zuul 所提供的过滤器功能可以逐渐增加对某一服务集群的流量，以了解服务性能，从而及早对服务运维架构做出调优。
  - **金丝雀（灰度）**、A/B 测试：新版本、新功能可能都需要测试用户对其的反应，通过 API 服务网关，可以轻松控制部分用户访问服务实例，并且可以对用户行为进行记录和分析，以便对新版本及新功能进行评价，获取应用的最优方案。
  - **服务迁移**：通过Zuul代理可以处理来自旧端点的客户端上的所有流量，将一些请求重定向到新的端点，从而慢慢地用不同的实现来替换旧端点。
  - **负载剪裁/限流**：为每一个负载类型分配对应的容量，对超过限定值的请求弃用，这样可以防止站点不被未知的大流量冲跨。

- 规范 IP 地址和端口号的分配：

  > 我们将服务治理服务器的端口号统一定为 8260，各业务服务器端口号从 2100 开始，各业务微服务的服务器端口按 100 进行递增，一个业务微服务的服务器端口按照 10 进行递增

- KISS 原则和 stateless 原则

  - KISS 原则：Keep it Simple and Stupid，即要**保持 API 服务网关的简单和轻量**。服务网关只不过是服务调用过程中的一个检查点，不应该用来处理业务的复杂性，也不应将其用于解决架构的难点上，这些应该是微服务需要处理的事情。
  - stateless 原则：是指在 Zuul 服务网关中不应该、也不可以保存有关服务调用过程中的状态数据。

- Zuul 默认整合：Hystrix、负载均衡功能



### 路由配置规则

- 服务路由默认规则：

  ```bash
  # http://[zuul路由服务器地址]/[serviceId]/[具体服务的端点]
  http://localhost:8280/userservice/users/3
  
  # 查询所有的微服务访问路径
  http://localhost:8280/routes
  
  # 当微服务部署架构中包含了 Eureka 服务时，在增加或移除一个服务时无须对 Zuul 进行任何修改，Zuul 可以自动根据 Eureka 服务器中所注册的服务自动完成路由映射、负载均衡等处理。
  ```

- 自定义微服务访问路径：

  ```bash
  # zuul.routes.微服务Id=指定路径
  zuul.routes.userservice = /user/**
  # /user/* : 只能匹配一级路径，即 /user/users
  # /user/** : 能匹配所有以 /user/ 开头的访问路径
  ```

- 忽略指定微服务

  ```bash
  # 指定在默认映射中所要忽略的微服务，指定后 Zuul 的路由服务将不再代理该路径下的访问。
  zuul.ignored-services=userservice
  # 忽略所有，全部采用自定义方式
  zuul.ignored-services=*
  ```

- 设置路由前缀

  ```bash
  # 默认情况下，Zuul 代理会在转发到具体服务实例时自动剥离这个前缀。如果需要在转发时带上该前缀，可以将 zuul.stripPrefix 属性的值设置为 false 来关闭这个默认行为。
  zuul.prefix=/api
  
  # zuul.stripPrefix 只会对 zuul.prefix 的前缀起作用，而对于 path 指定的前缀不会起作用。
  ```

- 通过静态 URL 路径配置路由映射

  ```bash
  zuul.routes.python-service.path=/pythonservice/**
  
  zuul.routes.python-service.url=http://pythonserver:8686
  
  # 负载均衡问题：禁用 Ribbon 与 Eureka 自动集成功能
  ribbon.eureka.enabled=false
  # 通过手工方式配置服务地址列表
  python-service.ribbon.listOfService=http://pythonserver:8686,http://pythonserver:8687,http://pythonserver:8688
  ```

- 路由配置顺序

  - 在配置时需要按照配置的顺序进行路由规则控制，使用 yaml 格式的配置文件
  - 使用 properties 文件格式，会丢失配置顺序



### Zuul 路由其他设置

- Header 设置

  - 敏感 Header 设置：在路由配置中设置一个忽略 Header 的清单。

    ```properties
    # 对指定路由开启自定义敏感头
    zuul.routes.[route].customSensitiveHeaders=true
    zuul.routes.[route].sensititiveHeaders=[要过滤的敏感头]
    
    # 全局设置
    zuul.sensititiveHeaders=[要过滤的敏感头]
    ```

  - 忽略 Header 设置

    ```properties
    # 默认没有这个配置
    zuul.ignoredHeaders=[要忽略的 Header]
    
    # 如果项目中引入了 Spring Security，那么 Spring Security 会自动加上这个配置，默认值为 Pragma、Cache-Control、X-Frame-Options、X-Content-Type-Options、X-XSS-Protection 和 Expries。
    ```

- HttpClient 配置

  ```properties
  # 启用 Ribbon 的 RestClient
  ribbon.restclient.enabled=true
  
  # 启用 OkHttpClient，需要确保项目中已经包含 com.squareup.okhttp3 相关包的引用
  ribbot.okhttp.enabled=true
  ```

- 路由配置的动态加载

  - 使用 Spring Cloud Config
    - 提供了配置文件的统一管理，我们需要将原来存放在 src/main/resources 目录下的配置文件 application.properties 或 yml 文件抽出来统一存放在版本管理服务器上，比如Git中
    - 然后将 Zuul 路由服务器的配置从统一配置服务器中进行加载。
    - 当需要修改路由映射规则时，就需要将修改后的配置文件提交到 Git 仓库中，然后在Zuul 路由服务器中使用 /refresh 端点重新加载配置。



### Zuul 容错与回退

- Zuul 路由服务在与 Hystrix 整合的时候其监控的**颗粒度是微服务**，而不是微服务中具体的某个方法。
  - 把一个微服务关闭时，在方法中的服务降级处理是不起作用的。
- Zuul 提供了一个 ZuulFallbackProvider 接口，通过实现该接口就可以为 Zuul 实现回退功能。
  - getRoute 方法：返回的是要为哪个微服务提供回退功能。这里需要注意其**返回的值是 route 的名称**，而不是微服务的名称，所以不能写为USERSERVICE，否则该回退将不起作用。
  - fallbackResponse() 方法：返回 ClientHttpResponse 对象，作为我们的回退响应。这里实现非常简单，仅仅是返回一个假的用户对象。
- 服务超时：
  - 当 Zuul 路由服务将客户端请求转发到具体服务时，**Zuul 会使用 HystrixCommand 来包装这些执行过程**，Hystrix 中的配置及服务容错机制，对于 Zuul的请求执行都是适用的，也会影响到 API 服务网关的行为。



### Zuul 过滤器

- 滤器的功能：负责对请求的处理过程进行干预，是实现请求校验。Zuul 路由映射和请求转发这些功能都是由几个不同的过滤器组合完成的，所以说 Zuul 的过滤器才是核心所在。

- Zuul 的过滤器则是**在微服务之间实现切面的处理**。

- 自定义过滤器需要实现的方法：

  - filterType()方法：返回过滤器的类型；
  - filterOrder() 方法：返回过滤器的执行顺序；
  - shouldFilter() 方法：判断是否需要执行该过滤器；
  - run() 方法：是该过滤器所要执行的具体过滤动作。

- Zuul 的四种标准过滤器类型（对应服务请求的典型生命周期）：

  - PRE 过滤器：**在请求被路由之前调用**，可用来实现身份验证、在集群中选择请求的微服务、记录调试信息等。
  - ROUTING 过滤器：**在调用目标服务之前被调用**，通常可以用来处理一些动态路由。
  - POST 过滤器：**在目标微服务执行以后，所返回的结果在送回给客户端时被调用**，我们可以利用该过滤器实现为响应添加标准的 HTTP Header、数据采集、统计信息和指标、审计日志处理等。
  - ERROR 过滤器：**该过滤器在处理请求过程中发生错误时被调用**，可以使用该过滤器实现对异常、错误的统一处理，从而为客户端调用显示更加友好的界面。

- 全局异常处理：

  > 新增一个类型为 Error 的过滤器，在该过滤器中将错误信息写入 RequestContext 中，这样 SendErrorFilter 就可以获取该错误信息，并转发到 Spring Boot 中进行通用的错误处理。



### @EnableZuulServer 与 @EnableZuulProxy 比较

|                  | @EnableZuulProxy                                             | @EnableZuulServer                                          |
| ---------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| PRE 类型过滤器   | PreDecorationFilter                                          | ServletDetectionFilter、FormBodyWrapperFilter、DebugFilter |
| ROUTE 类型过滤器 | RibbonRoutingFilter、SimpleHostRoutingFilter                 | SendForwardFilter                                          |
| POST 类型过滤器  |                                                              | SendResponseFilter                                         |
| Error 类型过滤器 |                                                              | SendErrorFilter                                            |
|                  | 不会自动加载任何代理过滤器                                   | 开发者添加的任何 ZuulFilter 类型实体类都会被自动加载       |
|                  | 包含 @EnableZuulServer 的所有功能，并且还加入了 @EnableCircuitBreaker 和 @EnableDiscoveryClient |                                                            |



## 五、统一配置中心——Config

- 通过一个服务端和多个客户端实现配置服务。

- 快速启动：

  - 依赖：`spring-cloud-config-server`

  - 启动配置服务：在 Application 类的注解中增加 `@EnableConfigServer` 注解

  - 编写配置文件(bootstrap.properties)

    ```properties
    server.port=8888
    
    spring.application.name=config-server
    
    spring.cloud.config.server.git.uri=https://github.com/xxxx/xxxxx
    
    spring.cloud.config.server.git.username=git username
    spring.cloud.config.server.git.password=git password
    ```

  - 创建应用配置文件

    - 在 Git 仓库中创建应用的配置文件。

  - 启动配置服务器

  - 读取配置文件：

    ```bash
    http://localhost:8888/productservice/default
    
    http://localhost:8888/productservice/dev
    ```

- 改造微服务

  1. 在项目中引入 `spring-cloud-starter-config` 依赖

  2. 修改配置文件 `bootstrap.properties`

     ```properties
     # productservice 微服务默认端口
     server.port=2200
     
     # productservice 微服务的服务名称，配置服务器后续会根据该名称查找相应的配置文件
     spring.application.name=productservice
     
     # 配置服务器的地址及所启用的 profile
     spring.cloud.config.profile=dev
     spring.cloud.config.uri=http://localhost:8888/
     ```



### 配置资源库

- 配置资源规则

  - Spring Cloud Config 是通过 EnvironmentRepository 来获得 Environment 对象，该对象是对 Spring 的 Environment（包括做为主要配置属性的 propertySources）对象的浅拷贝。在加载 Environment 相应资源时参数化成了下面3个变量。
    - {application}：对应客户端配置中的 spring.application.name
    - {profile}：对应客户端配置中的 spring.profiles.active（多个 profile 使用逗号分开）
    - {label}：对应配置服务器端所配置的 spring.cloud.config.label，如 Git 中的 master

- 配置客户端从配置服务器获取配置数据的整个流程

  1. 当配置客户端启动时，根据 bootstrap.properties 中配置的应用名称（spring. application.name）、环境名（spring.profiles.active），向配置服务器请求获取配置数据。
  2. 配置服务器根据客户端的请求参数，以及配置文件中所配置的标签（spring.cloud. config.label，如果没有配置，对于 Git 来说默认为 master 分支），从Git 仓库中按照上述规则去**查找符合的配置文件**。
  3. 配置服务器将匹配到的 Git 仓库**拉取到本地，并建立本地缓存**。
  4. 配置服务器根据所拉取到的配置文件**创建 Spring 的 ApplicationContext 实例，然后将该配置信息返回给客户端。**
  5. 客户端获取到配置服务器返回的数据后，将**这些配置数据加载到自己的上下文中**。同时，因为这些配置数据的优先级高于本地JAR包中的配置，因此将**不再加载本地的配置**。

- 本地缓存

  - 配置服务器检查到无法访问 Git 仓库时，则会读取之前存储在本地文件系统中的配置，然后将这些配置信息返回给配置客户端。

  ```properties
  # Git 仓库本地临时目录
  spring.cloud.config.server.git.basedir=tmp/
  ```

  

### 配置的加密与解密

- 安装 JCE（Java Cryptography Extension）

- 加密/解密端点

  - /encrypt：加密端点。使用该端点可以对所提供的字符串进行加密，这样就可以通过该端点来获取配置项加密后的字符串了。
  - /decrypt：解密端点。使用该端点可以对所要解密的字符串进行解密。

- 服务端：在 bootstrap.properties 中配置 encrypt.key 。如果是在 application.properties（或 yml）中进行配置，那么在调用加/解密端点时会报错误。

- 一个比较好的建议就是将密码存放在配置服务器的系统环境变量中。

- 客户端解密

  - 禁用掉配置服务端的解密处理：bootstrap.properties

    ```properties
    spring.cloud.config.server.encrypt.enabled=false
    ```

  - 在客户端工程增加和服务器同样的密钥：encrypt.key=xxx_key

  - 为客户端工程增加对 `spring-security-rsa` 的依赖

  - 重新启动配置服务器和商品服务

### 配置服务器访问安全

- 配置服务器的用户名和密码

  - 在配置服务器中增加 Spring Security 依赖

  - 在配置文件（bootstrap.properties）中配置访问用户的用户名和登录密码

    ```properties
    security.user.name=username
    security.user.password=password
    ```

  - 重新启动配置服务器

- 在客户端对应的配置文件中增加配置

  ```properties
  spring.cloud.config.username=username
  spring.cloud.config.password=password
  ```



### 配置服务器的高可用



- 整合 Eureka

  - 改造配置服务端

    1. 在 pom.xml 文件中增加对 Eureka 的依赖。

    2. 配置文件（bootstrap.properties/yml）中配置服务名称，以及 Eureka 服务器的地址。

       ```properties
       spring.application.name=configserver
       eureka.client.service-url.defaultZone=http://localhost:8260/eureka
       ```

    3. 再对启动类进行修改，增加 @EnableDiscoveryClient 注解，该注解是用来启动对 Eureka服务 的支持。

  - 改造配置客户端

    - 修改配置文件

      ```properties
      # 原来的配置 spring.cloud.config.uri=http://localhost:8888
      spring.cloud.config.uri=http://configserver
      ```

  - 依次启动 Eureka 服务器、配置服务器、客户端服务，日志输出如下，说明启动成功。

    > Fetching config from server at : http://xxxxxxx:8888

- 快速失败与响应

  - 在默认情况下，只有当客户端向配置服务发起请求时，配置服务器才会从配置仓库（如 Git 仓库）中加载配置文件。
  - 可以通过设置 clone-on-start，让配置服务器在启动时就加载配置文件。
    - 一方面在启动时就执行加载可以及时告知运维人员配置仓库是否可用；
    - 另一方面当客户端第一次发起请求时可以立即返回配置数据。
  - 当无法连接到配置服务器时希望其能够快速返回失败：设置 fail-fast 属性

- 为配置客户端开启重试机制

  - 在配置客户端的依赖中增加 `spring-retry` 和 `spring-boot-starter-aop` 依赖

- 动态刷新配置

  - 

## 六、分布式服务跟踪——Sleuth

- 功能：

  - 耗时分析：通过 Sleuth 可以很方便地了解到每个采样请求的耗时，从而分析出哪些微服务调用比较耗时。
  - 可视化错误：对于程序未捕捉的异常，可以在集成 Zipkin 服务界面上看到
  - 链路优化：通过Sleuth可以轻松识别出调用比较频繁的微服务，开发者可以针对这些微服务实施相应的优化措施。

- 快速启用 Sleuth

  - 微服务修改配置文件 bootstrap.properties

    ```properties
    server.port=2200
    
    # 服务名称，sleuth 会使用到
    spring.application.name=productservice
    ```

  - 修改 pom 文件：添加 `spring-cloud-starter-sleuth` 的依赖。自动启用 Sleuth 监控追踪机制

  - 启动测试，微服务控制台会有日志输出

    ```bash
    # Sleuth所生成的追踪数据，它们的格式为[ApplicationName, TraceId, SpanId,Exportable]
    [productservice,826bfe 5c0116e8f3,826bfe5c0116e8f3, false]
    ```

- Sleuth 追踪数据的格式：[ApplicationName, TraceId, SpanId, Exportable]

- 抽样率

  - `spring.sleuth.sampler.percentage`



### 整合 ELK

- 直接在 Logback 配置中增加 Logstash 的 Appender

### 整合 Zipkin 服务

- 构建 Zipkin 服务器
- 整合微服务



## 七、消息驱动——Stream



### 基本概念

- **消息发送通道接口 Source**：当使用该通道接口发送一个消息时，Spring Cloud Stream 会将所要发送的消息进行序列化，然后通过该接口所提供的 MessageChannel 将所要发送的消息发送到相应的消息中间件中。
- **消息通道 Channel**：对消息队列的一种抽象，用来存放消息发布者发布的消息或者消费者所要消费的消息。
- **消息绑定器 Binder**：实现了应用程序与具体消息中间件细节之间的隔离，向应用程序暴露统一的消息通道，使应用程序不需要考虑与各种不同的消息中间件的对接。
  - 默认提供了对 RabbitMQ 和 Kafka 的绑定器
- **消息监听通道接口 Sink**：应用程序监听通道消息的抽象处理接口。当从消息中间件中接收到一个待处理消息时，该接口将负责把消息数据反序列化为 Java 对象，然后交由业务所定义的具体业务处理方法进行处理。



### 使用消息对应用重构

- 为商品服务增加缓存功能：**必须是分布式缓存**。选择 Redis。
- 改造商品微服务增加缓存处理功能，能够**将远程请求所得到的用户数据缓存到 Redis 数据库中**，下次当其他服务调用这些数据时可以直接从 Redis 数据中获取，而不是进行远程请求。
  - 在项目中添加 `jedis` 和 `spring-data-redis` 依赖
  - 当加载一个用户信息时，首先尝试从Redis缓存中加载，如果加载不到（即不存在），再从远程获取，即通过访问用户微服务来获取，获取后再将用户的信息缓存到 Redis 中。
- 改造用户微服务，当一个用户信息被更新、删除时，可以**通过 Kafka 发送一条消息给商品微服务。**
  - 首先需要**构建一个用户变更消息对象**，该对象至少需要包含如下内容：变更用户的 ID、变更事件类型（更新还是删除）等数据。
  - **构建消息发送处理器**，能够将上一步所构建的用户变更消息发送给消息中间件 Kafka。
  - **修改用户管理服务中的保存、删除等功能**，当用户信息更新或删除时就构建一个用户信息变更消息，并通过上一步所提供的消息发送处理器发送该消息。
- 改造商品微服务，**增加用户信息变更消息监听功能**，当接收到 Kafka 发送来的用户更新、删除等消息时，可以对缓存数据进行相应的更新处理。
  - 添加对 `spring-cloud-starter-stream-kafka` 的依赖
  - 商品微服务在启动的时候去绑定 Spring Cloud Stream 的消息代理（在应用引导类中添加 @EnableBinding 注解）





### Spring Cloud Stream 高级主题

- **单元测试**：TestSupportBinder
  - 消息发送：注册一个类型为 MessageCollector 的 Bean，通过该 Bean 可以获取到所发送的消息，这样就可以判断消息是否发送成功。
  - 消息监听：通过直接向入站通道发送消息进行模拟。
- **错误处理**：Spring Cloud Stream 提供了一个全局错误消息处理通道，当出现异常时，SpringCloud Stream 就会将该异常包装成 ErrorMessage，然后发送到该消息通道中。默认该消息通道的名称为 errorChannel，可以通过项目配置文件中的 `spring.cloud.stream.bindings.error.destination` 属性来指定通道的名称。
- **消息处理分发**：持将同一个消息通道中的消息，根据条件分发给不同的方法进行处理。
  - 方法满足的条件：
    1. @StreamListener 注解
    2. 该方法没有返回值
    3. 该方法只能处理独立的消息，不能是响应式消息处理器。
  - 消息分发的条件可以通过 @StreamListener 注解中的 condition 属性设定，条件可以使用 **SpEL 表达式**。
- 消费组与消息分区
  - 默认情况下，如果没有为应用指定消费者组，Spring Cloud Stream 会为该应用创建一个匿名组，并且该组中只有其一个应用。
  - 可以在应用的配置文件中设置 `spring.cloud. stream.bindings.input.group` 属性来指定所属消费者组的 ID。
- 消息绑定器
  - 消息发送：需要调用绑定器的 bindProducer() 方法，并根据所要绑定的具体消息代理，创建一个消息通道。
  - 消息监听：需要调用 bindConsumer() 方法，创建一个消息监听通道。

### 消息总线——Spring Cloud Bus

- 配置自动刷新配置

  - 修改配置服务：引入 Spring Cloud Bus 依赖，更改服务器配置

  - 修改商品微服务：引入 Spring Cloud Bus 依赖

  - 刷新配置：

    ```bash
    http://localhost:8888/bus/refresh
    ```

- 发布自定义事件





## 八、微服务应用安全——Security

- 业务应用安全：
  - 保障只有认证的用户才可以访问应用，也就是**用户认证**。
  - 保障访问者只有拥有足够的权限才可以访问某个资源，也就是**用户鉴权**。