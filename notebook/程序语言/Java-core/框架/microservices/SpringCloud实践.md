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