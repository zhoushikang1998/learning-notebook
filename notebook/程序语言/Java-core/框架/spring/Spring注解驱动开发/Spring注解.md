### @Scope：调整作用域

```java
// prototype : 多实例的，IOC 容器不会调用方法创建对象，而是每次获取的时候才会调用方法创建对象
// singleton : 单实例的（默认值），IOC 容器启动会调用方法创建对象放到 IOC 容器中。以后每次获取直接从容器中拿。饿汉式加载
// request : 同一个请求中的实例相同（不常用）
// session : 同一个 session 中的实例相同（不常用）
```

懒加载：**@Lazy**



### @Conditional : 按照一定的条件进行判断，满足条件给容器中注册 bean



## 给容器中注册 bean

### 1、@Controller/@Service/@Repository/@Component

### 2、@bean : 导入的第三包里面的组件

### 3、@Import：在类开头快速导入组件，id 默认是组件的全类名

1. @Import(要导入到容器中的组件)：容器就会自动注册这个组件
2. ImportSelector（接口）*：通过实现这个接口，返回需要导入的组件的全类名数组。在@Import中，会将 全类名数组中的类导入到容器中
3. ImportBeanDefinitionRegistrar（接口）：手动注册 bean 到容器中

### 4、使用 Spring 提供的 FactoryBean（接口）

1. 重写getObject方法：返回工厂 bean 要创建的对象
2. 重写getObjectType 方法：返回工厂 bean 创建的对象的类型
3. getBean : 直接获取工厂 id，获取的是工厂bean 调用 getObject 创建的对象
4. getBean : 在工厂 id 前面加上 ‘&’前缀获取的是工厂类本身



## bean 的生命周期

创建对象—— 初始化方法（对象创建完成并赋值好） —— 销毁方法（容器关闭的时候）

1. 构造（对象创建）：单实例：在容器启动时创建对象；多实例：在每次获取的时候创建对象
2. BeanPostProcessor.postProcessorBeforeInitialization
3. 初始化：对象创建完成，并赋值好，调用初始化方法。
4. BeanPostProcessor.postProcessorAfterInitialization
5. 销毁：单实例：容器关闭的时候；多实例：容器不会管理这个 bean，容器不会调用销毁方法。



### 1、指定初始化和销毁方法

销毁方法：单实例：容器关闭的时候；多实例：不会自动调用销毁方法

在 @Bean 注解上指定initMethod 和 destroyMethod

### 2、通过让 bean 实现 InitializingBean（定义初始化逻辑），DisposableBean（定义销毁逻辑）

### 3、使用 JSR250规定的两个注解

1. @PostConstruct：bean 对象创建完成并赋值好；来执行初始化方法
2. @PreDestroy：在容器销毁 bean 之前通知我们进行清理工作

###  4、BeanPostProcessor（接口）：bean 的后置处理器

在bean 初始化前后进行一些处理工作；

1. postProcessBeforeInitialization：在对象创建后，初始化之前调用
2. postProcessAfterInitialization：在初始化之后调用



**BeanPostProcessor 原理：**

```java
populateBean(beanName, mbd, instanceWrapper);	// 给 bean 进行属性赋值
initializeBean {
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    invokeInitMethods(beanName, wrappedBean, mbd);
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
}
```

**Spring 底层对 BeanPostProcessor 的使用**



## 属性赋值

### @PropertySource

使用 @PropertySource 读取外部配置文件中的 k/v 保存到运行的环境变量中；

可重复标注：使用@PropertySources

### @Value

使用 @Value 赋值
       1. 基本数值
    2. 可以写SpEL：#{}
    3. 可以写${}:取出配置文件中的值（在运行环境里面的值）



## 自动装配

Spring 利用依赖注入（DI），完成对 IOC 容器中各个组件的依赖关系赋值

### 1、对象的自动装配

#### 1.1 @Autowired（Spring 定义的）

```java
可标注的位置：构造器，参数，方法，属性；都是从容器中获取的
1、[标注在方法上]：@Bean + 方法参数；参数从容器中获取；默认不写 @Autowired的效果是一样的
2、[标在构造器上]：如果组件只有一个有参构造器，这个有参构造器的 @Autowired 可以省略，参数位置的组件会从容器中自动获取
@Bean标注的方法创建对象的时候，方法参数的值从容器中获取
```



1. 自动注入，默认优先按照类型在容器中找对应的组件；
2. 如果找到多个相同类型的组件，再将属性的名称作为组件的 id 去容器中查找
3.  
4. 自动装配默认一定要将属性赋值好，如果容器中没有对应的 bean，就会报错。
5. 可以使用 @Autowired(required=false)，这样没有对应的 bean时也不会报错。

##### 2、@Qualifier

3. @Qualifier("bookDao")：使用@Qualifier 指定需要装配的组件的 id，而不是使用属性名

#### 3、@Primary

6. 让 Spring 进行自动装配的时候，默认使用首选的 bean。没用 @Qualifier 明确指定装配的 bean时，在对应 bean 的前面添加上 @Primary 注解，表示首选的 bean；如果使用@Qualifier 指定，则使用@Qualifier 指定的bean。

#### 4、Spring 还支持使用 @Resource（JSR250）和 @Inject（JSR330）[Java 规范的注解]

1. @Resource：可以和 @Autowired 一样实现自动装配功能；默认是按照组件名称进行装配的；没有能支持 @Primary 的功能也没有  @Autowired(required=false)
2. @Inject：需要导入 javax.inject 包；和 @Autowired 功能一样。但是没有 required=false 的功能

#### 5、原理

AutowiredAnnotationBeanPostProcessor：解析完成自动装配功能



#### 6、自定义组件

自定义组件想要使用 Spring 容器底层的一些组件（ApplicationContext，BeanFactory，xxx）；

自定义组件实现 xxxAware：在创建对象的时候，会调用接口规定的方法注入相关组件

把 Spring 底层的一些组件注入到自定义的 bean 中



xxxAware 的功能是使用 xxxAwareProcessor实现的

​		ApplicationContextAware ==> ApplicationContextAwareProcessor



#### 7、@Profile

Spring 提供的可以根据当前环境，动态的激活和切换一系列组件的功能；

开发环境、测试环境、生产环境；

 数据源：（/A）（/B）（/C） ==》不同的环境中使用不同的数据源



1. 加了环境标识的 bean，只有这个环境被激活的时候才能注册到容器中。默认是 default 环境
2. 激活方式：
   1. 使用命令行动态参数：在虚拟机参数位置加上-Dspring.profiles.active=test
   2. 使用 Java 代码的方式
      1. 首先，不能使用有参构造器创建 IOC 容器，因为有参构造器会自动刷新容器
      2. 使用无参构造器创建一个 ApplicationContext
      3. 设置需要激活的环境 applicationContext.getEnviroment().setActiveProfiles("test", "dev");
      4. 注册主配置类：applicationContext.register(MainConfigOfProfile.class);
      5. 启动刷新容器：applicationContext.refresh();
3. 写在配置类上，只有是指定的环境的时候，整个配置类里面的所有配置才能生效
4. 没有标注环境标识的 bean，在任何环境中都能被加载