## Eureka

- 启动服务注册中心：在 SpringBoot 启动 Application 类上添加 @EnableEurekaServer 注解

- Eureka-Client：在 SpringBoot 启动 Application 类上添加 @EnableEurekaClient 注解

- 配置文件

  ```yaml
  server:
    port: 8761
  
  
  eureka:
    instance:
      hostname: localhost
    client:
      # 表明是 Eureka server
      register-with-eureka: false
      fetch-registry: false
      service-url:
        default-zone: http://${eureka.instance.hostname}:${server.port}/eureka/
  
  spring:
    application:
      name: eureka-server
  ```

  



## Ribbon

- 建立一个消费者，在启动类上通过 @EnableDiscoveryClient 向服务中心注册
- 开启负载均衡功能：@LoadBalanced





## Open Feign

- 简介：

  - Feign 采用基于接口的注解
  - 整合了 Ribbon，具有负载均衡的能力
  - 整合了 Hystrix，具有熔断的能力

- 定义一个 feign 接口，通过 @FeignClient("服务名") 指定调用哪个服务。自动负载均衡

  ```java
  @FeignClient(value = "service-hi")
  public interface SchedualServiceHi {
      @RequestMapping(value = "/hi", method = RequestMethod.GET)
      String sayHiFronClientOne(@RequestParam(value = "name") String name);
  }
  ```

- 在 Web 层的 Controller 层，对外暴露一个 "/hi" 的 API 接口，通过 Feign 客户端 SchedualServiceHi 来消费服务。



## Hystrix

- Ribbon 使用
  - 启动类上添加 @EnableHystrix 注解开启 Hystrix
  - 在 Service 类的方法上添加 @HystrixCommand 注解，在注解上添加 fallbackMethod，指定熔断方法。
- Feign 使用
  - 开启 Hystrix，feign.hystrix.enabled=true
  - 在 Service 接口的 @FeignClient 注解上添加 fallback = xxxxxHystrix.class
  - 创建 xxxxxHystrix.class 实现 Service 接口，重写熔断方法；注入到 IOC 容器（@Component）





## Zuul

- 路由

  - 在启动类上添加 @EnableZuulProxy 注解

  - 配置路由

    ```yaml
    zuul:
      routes:
        api-a:
          path: /api-a/**
          serviceId: service-ribbon
        api-b:
          path: /api-b/**
          serviceId: service-feign
    ```

- 服务过滤

  - 创建 MyFilter 类，继承 ZuulFilter。注册到 IOC 容器中。
  - filterType：返回一个字符串代表过滤器的类型，四种不同生命周期的过滤器类型
    - pre：路由之前
    - routing：路由时
    - post：路由之后
    - error：发送错误调用
  - filterOrder：过滤的顺序
  - shouldFilter：写逻辑判断，是否要过滤
  - run：过滤器的具体逻辑。可以实现复杂的过滤逻辑。



## Config

- Config Server

  - 在启动类上添加 @EnableConfigServer 注解开启配置服务器的功能

  - 配置文件 application.properties

    ```properties
    spring.application.name=config-server
    server.port=8888
    
    spring.cloud.config.server.git.uri=https://github.com/forezp/SpringcloudConfig/
    spring.cloud.config.server.git.searchPaths=respo
    spring.cloud.config.label=master
    spring.cloud.config.server.git.username=
    spring.cloud.config.server.git.password=
    ```

  - http 请求地址和资源文件映射

- Config Client

  - 配置文件 bootstrap.properties

    ```properties
    spring.application.name=config-client
    server.port=8881
    
    spring.cloud.config.label=master
    spring.cloud.config.profile=dev
    spring.cloud.config.uri= http://localhost:8888/
    ```

  - 可以直接使用配置文件的属性

  -  config-client获取属性配置的过程

    ![](images/config-client获取属性配置的过程.png)

- 高可用的分布式配置中心
  - 将 Config Server 和 Config Client 注册到 Eureka Server 上
  - Config Client 读取配置中心的 ServerId(服务名)，通过部署多份配置服务，起到负载均衡和高可用的效果。