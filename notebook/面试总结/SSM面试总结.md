## Spring



### 1.Spring IoC 和 AOP

- IoC（Inverse of Control:控制反转）是一种**设计思想**，就是 **将原本在程序中手动创建对象的控制权，交由Spring框架来管理**。
  -  **IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个Map（key，value）,Map 中存放的是各种对象。**
  - 将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。 **IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。**
- AOP：将那些与业务无关，**却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来**，便于**减少系统的重复代码**，**降低模块间的耦合度**，并**有利于未来的可拓展性和可维护性**。
  - **基于动态代理实现**
    - 要代理的对象实现了某个接口，使用 JDK Proxy 创建代理对象
    - 没有实现接口的对象，使用 Cglib 生成一个被代理对象的子类来作为代理

### 2.IoC 和 DI

- IoC：控制翻转，是一种设计思想。在 Spring 中，将设计好的对象交给容器控制。

  ```java
  /**
  	1.谁控制谁？
  		- IoC 容器控制了对象
      2.控制什么？
      	- 控制了外部资源获取和生命周期
     	3.
  */
  ```

  

- DI：依赖注入，由容器动态的将某个依赖关系注入到组件之中。

  ```java
  /**
  	1.作用：提升组件重用的频率
  	2.谁依赖于谁？
  		- 应用程序依赖于 IOC 容器
  	3.为什么需要依赖？
  		- 应用程序需要IOC容器来提供对象需要的外部资源；
  	4.谁注入谁？
  		- IOC 容器注入应用程序某个对象，应用程序依赖的对象
  	5.注入了什么？
  		- 注入某个对象所需要的外部资源（包括对象、资源、常量数据）。
  	6.IOC 和 DI 有什么关系呢？
  		- 同一个概念的不同角度描述。DI 更具体，指获得依赖对象的方式反转了。
  */
  ```



### 3.Spring 循环依赖



### 4.Spring bean

- Spring bean 的作用域
  - singleton：单例（默认），唯一 bean 实例。
  - prototype：原型，每次请求都会创建一个新的 bean 实例。
  - request : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
  - session : 每一次 HTTP 请求都会产生一个新的 bean，该bean仅在当前 HTTP session 内有效。
- Spring 中单例 bean 的线程安全问题
  - 当多个线程操作同一个对象的时候，对这个对象的成员变量的写操作会存在线程安全问题。
  - 通常情况：`Controller`、`Service`、`Dao` 这些 Bean 是无状态的。无状态的 Bean 不能保存数据，因此是线程安全的。
  - 常用解决方法：
    1. 使用 ThreadLocal（推荐）
    2. 改变 Bean 的作用域为 “prototype”

### 5.Spring Bean 的生产过程（生命周期）

- 

![Spring Bean 生命周期](images\Spring-Bean生命周期.jfif)









### 6.使用的设计模式

- **工厂设计模式** : Spring使用工厂模式通过 `BeanFactory`、`ApplicationContext` 创建 bean 对象。
- **代理设计模式** : Spring AOP 功能的实现。
- **单例设计模式** : Spring 中的 Bean 默认都是单例的。
- **模板方法模式** : Spring 中 `jdbcTemplate`、`hibernateTemplate` 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。
- **观察者模式:** Spring 事件驱动模型就是观察者模式很经典的一个应用。
  - 定义一个事件: 实现一个继承自 `ApplicationEvent`，并且写相应的构造函数；
  - 定义一个事件监听者：实现 `ApplicationListener` 接口，重写 `onApplicationEvent()` 方法；
  - 使用事件发布者发布消息: 可以通过 `ApplicationEventPublisher` 的 `publishEvent()` 方法发布消息。
- **适配器模式**：Spring AOP 的增强或通知（Advice）使用了适配器模式；SpringMVC 适配 Controller 使用了适配器模式。





### 7.Spring 事务

#### @Transactional(rollbackFor = Exception.class)注解

- 如果类或者方法加了这个注解，那么这个类里面的方法抛出异常，就会回滚，数据库里面的数据也会回滚。
- 在`@Transactional`注解中如果不配置`rollbackFor`属性，那么**事务只会在遇到`RuntimeException`的时候才会回滚**，加上`rollbackFor=Exception.class`，可以让事务在遇到非运行时异常时也回滚。



### 8.Spring 注解



### 9.Spring 源码

## SpringMVC

### 1.SpringMVC 工作原理

- DispatcherServlet：接收用户请求和响应结果
- HandlerMapping：请求查找，返回执行链
- HandlerAdapter：请求适配器执行 Handler，返回 ModelAndView
- Handler 处理器（Controller）：执行 Handler，返回 ModelAndView
- View Resolver(视图解析器)：视图解析
- 前端进行视图渲染

![](images/SpringMVC工作原理.jfif)





## Mybatis



### 1.#{} 和 ${} 的区别

- `${}`是 Properties 文件中的变量占位符，它可以用于标签属性值和 sql 内部，属于静态文本替换，比如${driver}会被静态替换为`com.mysql.jdbc.Driver`。
- `#{}`是 sql 的参数占位符，MyBatis 会将 sql 中的`#{}`替换为?号，在 sql 执行前会使用 PreparedStatement 的参数设置方法，按序给 sql 的?号占位符设置参数值，比如 ps.setInt(0, parameterValue)，`#{item.name}` 的取值方式为使用反射从参数对象中获取 item 对象的 name 属性值，相当于 `param.getItem().getName()`。