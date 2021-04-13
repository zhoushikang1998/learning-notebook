# 一、Spring Boot 入门

## 1、Spring Boot 简介



## 2、微服务

## 3、环境准备

## 4、Spring Boot HelloWorld

一个功能：浏览器发送hello请求，服务器接收请求并处理，相应 hello world字符串

### 1、创建一个maven工程（jar）

### 2、导入springboot相关的依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.21.RELEASE</version>
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### 3、编写一个主程序；启动spring boot 应用

```java
@SpringBootApplication
public class HelloWorldMainApplication {
	public static void main(String[] args) throws Exception {
		
		// 启动springboot 应用
		SpringApplication.run(HelloWorldMainApplication.class, args);
	}
}
```

### 4、编写相关的controller、service

```java
@Controller
public class HelloController {
	
	@ResponseBody
	@RequestMapping("/hello")
	public String hello() {
		return "Hello World";
	}
}
```

### 5、运行主程序测试

### 6、简化部署

```xml
<!-- 导入插件，可以将应用打包成一个可执行的jar包 -->
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
```

将这个应用打成jar包，直接使用java -jar命令进行执行

## 5、Hello World探究

### 1、POM文件

#### 1、父项目

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.21.RELEASE</version>
</parent>

他的父项目：真正管理Spring Boot应用里的所有依赖版本
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>1.5.21.RELEASE</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
</parent>

```

Spring Boot的版本仲裁中心；

我们导入依赖默认是不需要写版本；（没有在dependencies里面管理的依赖自然需要声明版本号）

#### 2、启动器

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

**spring-boot-starter**-==web==：

​	spring-boot-starter：spring-boot场景启动器；帮我们导入了web模块正常运行所依赖的组件

Spring Boot将所有的功能场景都抽取出来，做成一个个的starters（启动器），只需要在项目里面引入这些starter，相关场景的所有依赖都会导入进来。要用什么功能就导入什么场景的启动器。

### 2、主程序类，主入口类

```java
@SpringBootApplication
public class HelloWorldMainApplication {
	public static void main(String[] args) throws Exception {
		
		// 启动springboot 应用
		SpringApplication.run(HelloWorldMainApplication.class, args);
	}
}
```

**@SpringBootApplication**：Spring Boot应用标注在某个类上说明这个类是springboot的主配置类，SpringBoot就应该运行这个类的main方法来启动SpringBoot应用



```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM,
				classes = AutoConfigurationExcludeFilter.class) })
```



**@SpringBootConfiguration**：Spring Boot 的配置类；

​			标注在某个类上，表示这是一个 Spring Boot 的配置类

​			**@Configuration**：配置类上来标注这个注解；

​					配置类 ------ 配置文件；配置类也是容器中的一个组件；@Component

 **@EnableAutoConfiguration**：开启自动配置功能

​			以前需要配置的东西，Spring Boot 帮我们自动配置；**@EnableAutoConfiguration **告诉 SpringBoot 开启自动配置功能；这样自动配置才能生效；

```java
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

​		**@AutoConfigurationPackage**：自动配置包

​				**@Import(**AutoConfigurationPackages.Registrar.class)：

​				Spring的底层注解**@Import**，给容器中导入一个组件；导入的组件由AutoConfigurationPackages.Registrar.class

==将主配置类（@SpringBootApplication标注的类）的所在包及下面所有子包里面的所有组件扫描到Spring容器；==

​		**@Import**(EnableAutoConfigurationImportSelector.class)；

​				给容器中导入组件：

​				**EnableAutoConfigurationImportSelector**：导入哪些组件的选择器

​				将所有需要导入的组件以全类名的方式返回；这些组件会被添加到容器中；	

​				会给容器中导入非常多的自动配置类（xxxAutoConfiguration）；就是给容器中导入这个场景需要的所有组件，并配置好这些组件。

​				![自动配置类](images\xxxAutoConfiguration.PNG)

有了自动配置类，免去了我们手动编写配置注入功能组件等的工作；

​						SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class，ClassLoader);

==SpringBoot在启动的时候，从类路径下的META-INF/spring.factories中获取EnableAutoConfiguration指定的值，将这些值作为自动配置类导入到容器中，自动配置类就生效，帮我们进行自动配置工作；==以前需要我们自己配置的东西，自动配置类都帮我们做了；

J2EE的整体整合解决方案和自动配置都在：spring-boot-autoconfigure-1.5.21.RELEASE.jar；



## 6、使用Spring Initializer 快速创建Spring Boot项目

IDE都支持使用Spring的项目创建向导快速创建一个Spring Boot项目；

选择我们需要的模块；向导会**联网**创建Spring Boot项目；

默认生成的Spring Boot项目；

- 主程序已经生成好了，我们只需要我们自己的逻辑
- resources文件夹目录结构
  - static：保存所有的静态资源；js、css、images；
  - templates：保存所有的模板页面（SpringBoot默认jar包使用嵌入式的Tomcat，默认不支持jsp页面）；可以使用模板引擎（freemaker、thymeleaf）；
  - application.properties：SpringBoot应用的配置文件；可以修改一些默认配置（如端口号）



# 二、配置文件

## 1、配置文件

SpringBoot使用一个全局的配置文件，配置文件名是固定的；

- application.properties
- application.yml



配置文件的作用：修改SpringBoot自动配置的默认值；SpringBoot在底层都给我们自动配置好；



YAML（YAML Ain't Makeup Language）

标记语言：

​			以前的配置文件；大多都是使用  xxxx.xml文件

​			YAML：以数据为中心，比json、xml等更适合做配置文件；

​			YAML配置例子：

```yml
server:
  port: 8082
```

​			XML：

```xml
<server>
	<port>8082</port>
</server>
```





## 2、YAML语法

### 1、基本语法

k:（空格）v ：表示一对键值对（空格必须有）；

以**空格**的缩进来控制层级关系；只要是左对齐的一列数据，都是同一个层级的

```yml
server: 
	port: 8081
	path: /hello
```

属性和值也是大小写敏感；

### 2、值的写法

**字面量：普通的值（数字、字符串、布尔）**

​		k: v : 字面直接来写；

​				字符串默认不用加上单引号或者双引号；

​				"" : 双引号；会转义字符串里面的特殊字符；特殊字符会作为本身想表示的意思

​								name: "zhangsan \n lisi" ：输出 zhangsan 换行 lisi

​				'' ：单引号；不转义特殊字符，特殊字符最终只是一个普通的字符串数据

​								name: 'zhangsan \n lisi' ：输出 zhangsan \n lisi

**对象、map（属性和值）（键值对）：**

​		k: v : 在下一行来写对象的属性和值的关系；注意缩进

​				对象还是 k: v 的方式

```yaml
friends: 
	lastName: zhangsan
	age: 20
```

行内写法：

```yam
friends: {lastName: zhangsan,age: 18}
```



**数组（List、Set）：**

用- 值表示数组中的一个元素

```yaml 
pets: 
 - cat
 - dog
 - pig
```

行内写法：

```yam
pets: [cat,dog,pig]
```

## 3、配置文件值注入

配置文件

```yam
person: 
    lastName: zhangsan
    age: 18
    is-boss: false
    birth: 2016/12/12
    maps: {k1: v1,k2: 18}
    dog: 
      name: 小狗
      age: 2
    lists: 
      - zhaoliu
      - lisi
```

javaBean：

```java
/**
 * 将配置文件中配置的每一个属性的值，映射到这个组件中
 * @author acer
 *@ConfigurationProperties：告诉SpringBoot将本类中的所有属性和配置文件中相关的配置进行绑定
 *prefix="person"：配置文件中哪个下面的所有属性进行一一映射
 *@Component：只有这个组件是容器中的组件，才能使用容器提供的@ConfigurationProperties功能；
 */
@Component
@ConfigurationProperties(prefix="person")
public class Person {
	
	private String lastName;
	private Integer age;
	private Boolean isBoss;
	private Date birth;
	
	private Map<String, Object> maps;
	private Dog dog;
	private List<Object> lists;
```



可以导入配置文件处理器，以后编写配置就有提示：

```xml
<!-- 导入配置文件处理器，配置文件进行绑定就会有提示 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

### 1、@Value获取值和@ConfigurationProperties获取值比较

|                      | @ConfigurationProperties | @Value     |
| -------------------- | ------------------------ | ---------- |
| 功能                 | 批量注入配置文件中的属性 | 一个个指定 |
| 松散绑定（松散语法） | 支持                     | 不支持     |
| SpEL                 | 不支持                   | 支持       |
| JSR303数据校验       | 支持                     | 不支持     |
| **复杂类型封装**     | 支持                     | 不支持     |

配置文件yml还是properties他们都能获取到值；

​	如果只是在某一个业务逻辑中需要获取配置文件中的某项值，使用@Value；

​	如果专门编写了一个JavaBean来和配置文件进行映射，直接使用@ConfigurationProperties;		

### 2、配置文件注入值数据校验

@Validated

### 3、@PropertySource  && @ImportResource

**@PropertySource：**加载指定的配置文件

```java
@PorpertySource(value = {"classpath:person.properties"})
```

**@ImportResource**：导入Spring的配置文件，让配置文件里面的内容生效

SpringBoot里面没有Spring的配置文件，手动编写的spring配置文件不能自动识别；

想让Spring的配置文件生效，加载进来，使用**@ImportResource**标注在一个配置类上

```java
@ImportResource(locations = {"classpath:beans.xml"})
@SpringBootApplication
```

### 4、SpringBoot推荐的配置方式

不推荐编写Spring的配置文件

```xml
<bean id="helloService" class="com.atguigu.service.HelloService">
</bean>
```

SpringBoot推荐使用全注解的方式

- 配置类   ====Spring配置文件

- 使用@Bean给容器中添加组件

  ```java
  /**
   * @Configuration：指明当前类是一个配置类，就是来替代之前的spring配置文件
   * 在配置文件中使用<bean><bean/>标签添加组件
   *
   */
  @Configuration
  public class MyAppConfig {
  
  	// 将方法的返回值添加到容器中，容器中这个组件默认的id就是方法名
  	@Bean
  	public HelloService helloService02() {
  		System.out.println("配置类@Bean给容器中添加组件了。。。。");
  		return new HelloService();
  	}
  }
  ```

## 4、配置文件占位符

### 1、随机数

```properties
${random.value}
${random.int}
${random.long}
${random.int(10)}
${random.int[1024,65536]}
```

### 2、占位符获取之前配置的值，如果没有可以使用 : 指定默认值

## 5、Profile

Profile是Spring对不同环境提供不同配置功能的支持，可以通过激活、指定参数等方式快速切换环境

### 1、多Profile文件形式

- 格式：application-{profile}.properties
  - application-dev.properties
  - application-prod.properties
- 默认使用application.properties的配置

### 2、yml支持多文档块方式

```yaml
server:
  port: 8082
spring:
  profiles:
    active:
    - dev
---

server:
  port: 8083
spring: 
  profiles: dev


---

server:
  port: 8084
spring:
  profiles: prod

```



### 3、激活指定Profile

- 在配置文件中指定	spring.profile.active=dev

- 命令行：

  - --spring.profiles.active=dev
  - java -jar xxxxx.jar  --spring.profiles.active=dev

  ![eclipse中指定命令行参数](images/命令行激活指定profile.PNG)

- 虚拟机参数

  - -Dspring.profiles.active=dev

## 6、配置文件加载位置（application.yml）

SpringBoot启动会扫描以下位置的application.properties或者application.yml文件作为SpringBoot的默认配置文件

- -file:./config/
- -file:./
- -classpath:/config/
- -classpath:/

优先级由高到低，高优先级的配置会覆盖低优先级的配置

SpringBoot会从这四个位置==全部==加载主配置文件；**互补配置**



==可以通过spring.config.location来改变默认的配置文件位置==

项目打包好以后，可以使用命令行参数的形式，启动项目的时候来指定配置文件的新位置；指定配置文件和默认加载的这些配置文件共同起作用形成互补配置；（运维方便）





## 7、外部配置加载顺序

SpringBoot也可以从以下位置加载配置；优先级从高到低；高优先级的配置会覆盖低优先级的配置，所有的配置形成互补配置；

1. 命令行参数

   java -jar xxxxxxx.jar --server.port=9005 --server.context-path=/abc（多个配置用空格分开）

2. 来自java:comp/env的JNDI属性

3. java系统属性（System.getProperties()）

4. 操作系统环境变量

5. RandomValuePropertySource配置的random.*属性值

   

   **==优先加载profile, 由jar包外到jar包内==**

6. **jar包外部的application-{profile}.properties或application.yml(带Spring.profile)配置文件**

7. **jar包内部的application-{profile}.properties或application.yml(带Spring.profile)配置文件**

8. **jar包外部的application.properties或application.yml(带Spring.profile)配置文件**

9. **jar包内部的application.properties或application.yml(不带spring.profile)配置文件**

   

10. @Configuration注解类的@PropertySource

11. 通过SpringApplication.setDefaultProperties指定的默认属性

[参考官方文档](https://docs.spring.io/spring-boot/docs/1.5.21.RELEASE/reference/html/boot-features-external-config.html)



## 8、自动配置原理

配置文件能配置的属性参照 



### 1、自动配置原理：

1) SpringBoot启动的时候加载主配置类，开启了自动配置功能==@EnableAutoConfiguration==

2) @EnableAutoConfiguration的作用：

- 利用EnableAutoConfigurationImportSelector给容器中导入一些组件

- 可以查看selectImports()方法的内容

- 获取候选的配置

- 将类路径下 MATE-INF/spring.factories里面配置的所有的EnableAutoConfiguration的值加入到了容器中；

```java
List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
```



3)  每一个自动配置类进行自动配置功能；

4) 以 **HttpEncodingAutoConfiguration** 为例

```java
@Configuration //表示是一个配置类，以前编写的配置文件一样，也可以给容器中添加组件
@EnableConfigurationProperties({HttpEncodingProperties.class})//启动指定类的Configurationproperties功能；将配置文件中的值和HttpEncodingProperties绑定起来了；并把HttpEncodingProperties加入ioc容器中
@ConditionalOnWebApplication//根据不同的条件，进行判断，如果满足条件，整个配置类里面的配置就会生效，判断是否为web应用；
(
    type = Type.SERVLET
)
@ConditionalOnClass({CharacterEncodingFilter.class})//判断当前项目有没有这个类，解决乱码的过滤器
@ConditionalOnProperty(
    prefix = "spring.http.encoding",
    value = {"enabled"},
    matchIfMissing = true
)//判断配置文件是否存在某个配置 spring.http.encoding，matchIfMissing = true如果不存在也是成立，即使不配置也生效
public class HttpEncodingAutoConfiguration {
   //给容器添加组件，这个组件的值需要从properties属性中获取
    private final HttpEncodingProperties properties;
	//只有一个有参数构造器情况下，参数的值就会从容器中拿
    public HttpEncodingAutoConfiguration(HttpEncodingProperties properties) {
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean
    public CharacterEncodingFilter characterEncodingFilter() {
        CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
        filter.setEncoding(this.properties.getCharset().name());
        filter.setForceRequestEncoding(this.properties.shouldForce(org.springframework.boot.autoconfigure.http.HttpEncodingProperties.Type.REQUEST));
        filter.setForceResponseEncoding(this.properties.shouldForce(org.springframework.boot.autoconfigure.http.HttpEncodingProperties.Type.RESPONSE));
        return filter;
    }
```

5） 所有在配置文件中能配置的属性都是在xxxProperties类中封装着；配置文件能配置什么就可以参照某个功能对应的这个属性类

```java
@ConfigurationProperties(prefix = "spring.http.encoding")//从配置文件中的值进行绑定和bean属性进行绑定
public class HttpEncodingProperties {
```

根据当前不同条件判断，决定这个配置类是否生效？

一旦这个配置类生效；这个配置类会给容器添加各种组件；这些组件的属性是从对应的properties中获取的，这些类里面的每个属性又是和配置文件绑定的

能配置的属性都是来源于这个功能的properties类

### 2、精髓：

**1）、SpringBoot启动会加载大量的自动配置类**

**2）、我们看我们需要的功能有没有SpringBoot默认写好的默认配置类；**

**3）、如果有再看这个自动配置类中配置了哪些组件；（只要我们要用的组件有，我们不需要再来配置）**

**4）、给容器中自动配置添加组件的时候，会从 properties 类中获取属性。我们就可以在配置文件中指定这些属性的值**

xxxAutoConfiguration:自动配置类；

====》给容器中添加组件

====》xxxProperties:封装配置文件中的属性；

====》跟之前的Person类一样，配置文件中值加入bean中

### 3、细节

#### 1、@Conditional派生注解

>  利用Spring注解版原生的@Conditional作用

作用：必须是@Conditional指定的条件成立，才给容器中添加组件，配置类里面的所有内容才生效

| @Conditional派生注解            | 作用（判断是否满足当前指定条件）                |
| ------------------------------- | ----------------------------------------------- |
| @ConditionalOnJava              | 系统的java版本是否符合要求                      |
| @ConditionalOnBean              | 容器中存在指定Bean                              |
| @ConditionalOnMissingBean       | 容器中不存在指定Bean                            |
| @ConditionalOnExpression        | 满足spEL表达式                                  |
| @ConditionalOnClass             | 系统中有指定的类                                |
| @ConditionalOnMissingClass      | 系统中没有指定的类                              |
| @ConditionalOnSingleCandidate   | 容器中只有一个指定的Bean,或者这个Bean是首选Bean |
| @ConditionalOnProperty          | 系统中指定的属性是否有指定的值                  |
| @ConditionalOnResource          | 类路径下是否存在指定的资源文件                  |
| @ConditionalOnWebApplication    | 当前是web环境                                   |
| @ConditionalOnNotWebApplication | 当前不是web环境                                 |
| @ConditionalOnJndi              | JNDI存在指定项                                  |

#### 2、自动配置报告

**自动配置类必须在一定的条件下才能生效；**

开启SpringBoot的debug模式

```properties
debug=true
```

通过启用 debug=true 属性；来让控制台打印自动配置报告

- positive matched
- negative matched

# 三、日志

> SpringBoot2对日志有更改

## 1、日志框架

小张：开发一个大型系统内：

1、System.out.println("");将关键数据打印在控制台上；去掉？写在文件中？

2、写一个框架记录系统的一些运行信息；日志框架zhanglogging.jar

3、高大上的功能；异步模式？自动归档？。。。？zhanglogging-good.jar

4、将以前的框架卸下来？换上新的框架，重新修改之前的相关API；zhanglogging-perfect.jar

5、JDBC--数据库驱动：写了一个统一的接口层；日志门面（日志的一个抽象层）；logging-abstract.jar；给项目中导入具体的日志实现就行了；



市面上的日志框架

| 日志门面（日志的抽象层）                                     | 日志实现                                          |
| ------------------------------------------------------------ | ------------------------------------------------- |
| ~~JCL(Jakarta Commons Logging)~~ SLF4j(Simple Logging Facade for Java) ~~jboss-logging~~ | Log4j  ~~JUL(java.util.logging)~~  Log4j2 Logback |

SLF4J -- Logback

Spring Boot:底层是Spring框架，Spring默认框架是JCL；

​	**SpringBoot选用SLF4J和logback**

## 2、SLF4J使用

### 1、如何在系统中使用SLF4J

开发的时候，日志记录方法的调用，不应该直接调用日志的实现类，而是调用日志抽象层里面的方法

向系统中导入SLF4J的jar和locback的实现jar

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HelloWorld {
  public static void main(String[] args) {
    Logger logger = LoggerFactory.getLogger(HelloWorld.class);
    logger.info("Hello World");
  }
}
```

图示：

![SLF4J](images\SLF4J-concrete-bindings.png)

==每一个日志的实现框架都有自己的配置文件。使用SLF4J以后，配置文件还是使用日志实现框架本身的配置文件。==

### 2、遗留问题

统一日志记录，统一使用SLF4J进行输出：

![](images/legacy.png)

**如何让系统中所有的日志都统一到SLF4J；**

==1、将系统中其他日志框架先排除出去；==

==2、用中间包来替换原有的日志框架；==

==3、导入SLF4J的其他实现；==



### 3、SpringBoot日志关系

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```



SpringBoot使用它来做日志功能

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-logging</artifactId>
		</dependency>
```





底层依赖关系：

​		1）、SpringBoot底层也是使用SLF4J+logback的方式进行日志记录；

​		2）、SpringBoot也把其他的日志都替换成了SLF4J

​		3）、中间替换包

```java 
@SuppressWarnings("rawtypes")
public abstract class LogFactory {

    static String UNSUPPORTED_OPERATION_IN_JCL_OVER_SLF4J = "http://www.slf4j.org/codes.html#unsupported_operation_in_jcl_over_slf4j";

    static LogFactory logFactory = new SLF4JLogFactory();
```

![](images/中间转换包.png)

4）、如果要引入其他框架？一定要把这个框架的默认日志依赖移除掉；

​			Spring框架用的是commons-logging；

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <exclusions>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

==SpringBoot能自动适配所有的日志，而且底层使用SLF4J+Logback的方式记录日志，引入其他框架的时候，只需要把这个框架依赖的日志框架排除掉；==



### 4、日志使用

#### 1、默认配置

```java
// 记录器
	Logger logger = LoggerFactory.getLogger(getClass());
	@Test
	public void contextLoads() {
		// System.out.println();
		/*
		 * 日志的级别；
		 * 由低到高：trace<debug<info<warn<error
		 * 可以调整输出的日志级别；日志就只会在这个级别及以后的高级别生效
		 */
		logger.trace("这是trace日志。。。。");
		logger.debug("这是debug日志。。。。");
		// SpringBoot默认使用的是info级别的
		logger.info("这是info日志....");
		logger.warn("这是warn日志。。。。");
		logger.error("这是error日志。。。。");
	}
```

日志输出格式

```properties
#控制台输出的日志格式 
#%d：日期
#%thread：线程号 
#%-5level：靠左 级别 
#%logger{50}：全类名50字符限制,否则按照句号分割
#%msg：消息+换行
#%n：换行
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n
```

SpringBoot修改日志的默认配置

```properties
logging.level.com.wdjr=trace
#不指定path就是当前目录下生成springboot.log
#logging.file=springboot.log
#当前磁盘下根路径创建spring文件中log文件夹，使用spring.log作为默认
logging.path=/spring/log
#控制台输出的日志格式 日期 + 线程号 + 靠左 级别 +全类名50字符限制+消息+换行
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n
#指定文件中日志输出的格式
logging.pattern.file=xxx
```

#### 2、指定配置

给类路径下放上每个日志框架自己的配置文件即可；SpringBoot就不使用默认配置

| logging System         | Customization                                                |
| ---------------------- | ------------------------------------------------------------ |
| Logback                | logback-spring.xml ,logback-spring.groovy,logback.xml or logback.groovy |
| Log4J2                 | log4j2-spring.xml or log4j2.xml                              |
| JDK(Java Util Logging) | logging.properties                                           |

logback.xml：直接就被日志框架识别了；

**logback-spring.xml**：日志框架不直接加载日志的配置项，由SpringBoot解析日志配置，可以使用SpringBoot的高级Profile功能

```xml
<springProfile name="staging">
	可以指定某段配置只在某个环境下生效
</springProfile>
```



否则

```java
no applicable action for [springProfile]
```

### 5、切换日志框架

可以按照SLF4J的日志适配图，进行相关的切换；

SLF4J+log4j的方式：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <artifactId>logback-classic</artifactId>
            <groupId>ch.qos.logback</groupId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
</dependency>
```

不推荐使用log4j，仅作为演示。



切换为log4j2：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <artifactId>spring-boot-starter-logging</artifactId>
            <groupId>org.springframework.boot</groupId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```





# 四、SpringBoot与Web开发

## 1、简介

使用SpringBoot**：

1）、创建SpringBoot应用，选中需要的模块；

2）、SpringBoot已经默认将这些场景配置好了，只需要在配置文件中指定少量配置就可以运行起来；

3）、自己编写业务代码





**自动配置原理？**

这个场景SpringBoot帮我们配置了什么？我们能不能修改？能修改哪些配置？能不能扩展？xxxx

```java
xxxxAutoConfiguration：给容器中自动配置组件
xxxxProperties：配置类，来封装配置文件的内容
```

## 2、SpringBoot对静态资源的映射规则

```java
@ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
public class ResourceProperties implements ResourceLoaderAware, InitializingBean {

  // 可以设置和静态资源相关的参数，如缓存时间等  
```



```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
        return;
    }
    Integer cachePeriod = this.resourceProperties.getCachePeriod();
    if (!registry.hasMappingForPattern("/webjars/**")) {
        customizeResourceHandlerRegistration(registry
                                             .addResourceHandler("/webjars/**")
                                             .addResourceLocations("classpath:/META-INF/resources/webjars/")
                                             .setCachePeriod(cachePeriod));
    }
    String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    if (!registry.hasMappingForPattern(staticPathPattern)) {
        customizeResourceHandlerRegistration(
            registry.addResourceHandler(staticPathPattern)
            .addResourceLocations(
                this.resourceProperties.getStaticLocations())
            .setCachePeriod(cachePeriod));
    }
}
```

1）、所有/webjars/**，都去classpath:/META-INF/resources/webjars/找资源；

​						webjars：以jar包的形式 引入静态资源；

​	https://www.webjars.org/

![](images/jquery-webjar.png)

localhost:8080/webjars/jquery/3.4.1/jquery.js

```xml
<!-- 引入jQuery-webjar -->  在访问的时候只需要写webjars下面资源的名称即可
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>jquery</artifactId>
    <version>3.4.1</version>
</dependency>
```

2）、本地资源

```java
private String staticPathPattern = "/**";
```

"/**"访问当前项目的任何资源，（静态资源的文件夹）;会在这几个文件夹下找静态资源

```java
"classpath:/META-INF/resources/", 
"classpath:/resources/",
"classpath:/static/", 
"classpath:/public/",
"/";	//当前项目的根路径
```

localhost:8080/abc ==>去静态资源文件夹中找abc



3）、欢迎页；静态资源文件夹下的所有index.html页面；被"/**"映射

localhost:8080/		找index页面

```java
// 配置欢迎页映射
@Bean
public WelcomePageHandlerMapping welcomePageHandlerMapping(
    ResourceProperties resourceProperties) {
    return new WelcomePageHandlerMapping(resourceProperties.getWelcomePage(),
                                         this.mvcProperties.getStaticPathPattern());
}
```

4）、网站title的图标favicon；==所有的**/favicon.ico 都是在静态资源文件下找；==

```java
// 配置喜欢的图标
@Configuration
@ConditionalOnProperty(value = "spring.mvc.favicon.enabled",
                       matchIfMissing = true)
public static class FaviconConfiguration {

    private final ResourceProperties resourceProperties;

    public FaviconConfiguration(ResourceProperties resourceProperties) {
        this.resourceProperties = resourceProperties;
    }

    @Bean
    public SimpleUrlHandlerMapping faviconHandlerMapping() {
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setOrder(Ordered.HIGHEST_PRECEDENCE + 1);
        mapping.setUrlMap(Collections.singletonMap("**/favicon.ico",
                                                   faviconRequestHandler()));
        return mapping;
    }

    @Bean
    public ResourceHttpRequestHandler faviconRequestHandler() {
        ResourceHttpRequestHandler requestHandler = new ResourceHttpRequestHandler();
        requestHandler
            .setLocations(this.resourceProperties.getFaviconLocations());
        return requestHandler;
    }
}
```

5）、可以在配置文件配置静态资源文件夹

```properties
spring.resources.static-locations=classpath:xxxx
```

## 3、模板引擎

![](images/template-engine.png)

SpringBoot推荐使用thymeleaf；

语法更简单，功能更强大；

### 1、引入thymeleaf

```xml
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
    		2.1.6版本
		</dependency>
切换thymeleaf版本
<properties>
    <java.version>1.8</java.version>
    <thymeleaf.version>3.0.10.RELEASE</thymeleaf.version>
    <!-- 布局功能的支持程序  thymeleaf3主程序  layout2以上版本 -->
    <!-- thymeleaf2 layout1 -->
    <thymeleaf-layout-dialect.version>2.3.0</thymeleaf-layout-dialect.version>
</properties>
```

### 2、thymeleaf使用&&语法

**使用**

```java
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {

	private static final Charset DEFAULT_ENCODING = Charset.forName("UTF-8");

	private static final MimeType DEFAULT_CONTENT_TYPE = MimeType.valueOf("text/html");

	public static final String DEFAULT_PREFIX = "classpath:/templates/";

	public static final String DEFAULT_SUFFIX = ".html";
    
    // 只要把HTML页面放在classpath:/templates/，thymeleaf就能自动渲染
```

只要把HTML页面放在classpath:/templates/，thymeleaf就能自动渲染

1、 导入thymeleaf的名称空间

```html
<html xmlns:th="http://www.thymeleaf.org">
```

2、 使用thymeleaf的语法

```html
<!DOCTYPE html>
<html  xmlns:th="http://www.thymeleaf.org">
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
<h1>成功</h1>
<!-- th:text 设置div里面的文本内容 -->
<div th:text="${hello}"></div>
</body>
</html>
```



**语法规则**

1、 th:text；改变当前元素里面的文本内容

​			th:任意html属性；来替换原生属性的值

![](images/thymeleaf中属性的优先级.PNG)

2、 表达式

```properties
Simple expressions:
    Variable Expressions: ${...}：获取变量值；OGNL
    			1）、获取对象的属性、调用方法
    			2）、使用内置的基本对象
    			3）、内置的一些工具对象
    Selection Variable Expressions: *{...}：选择表达式：和${}在功能上是一样的
    			补充：配合 th:object="${session.user}"使用
        <div th:object="${session.user}">
			<p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
			<p>Surname: <span th:text="${session.user.lastName}">Pepper</span>.</p>
			<p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
		</div>

    Message Expressions: #{...}：获取国际化内容
    Link URL Expressions: @{...}：定义URL；
    	
    Fragment Expressions: ~{...}：片段引用表达式
Literals（字面量）
    Text literals: 'one text' , 'Another one!' ,…
    Number literals: 0 , 34 , 3.0 , 12.3 ,…
    Boolean literals: true , false
    Null literal: null
    Literal tokens: one , sometext , main ,…
Text operations:（文本操作）
    String concatenation: +
    Literal substitutions: |The name is ${name}|
Arithmetic operations:（数学运算）
    Binary operators: + , - , * , / , %
    Minus sign (unary operator): -
Boolean operations:（布尔运算）
    Binary operators: and , or
    Boolean negation (unary operator): ! , not
Comparisons and equality:（比较运算）
    Comparators: > , < , >= , <= ( gt , lt , ge , le )
    Equality operators: == , != ( eq , ne )
Conditional operators:（条件运算，包括三元运算符）
    If-then: (if) ? (then)
    If-then-else: (if) ? (then) : (else)
    Default: (value) ?: (defaultvalue)
Special tokens:（特殊字符，无操作）
	No-Operation: _

```

## 4、SpringMVC自动配置

### 1、SpringMVC自动配置

[官方文档](https://docs.spring.io/spring-boot/docs/1.5.21.RELEASE/reference/html/boot-features-developing-web-applications.html)

SpringBoot自动配置好了SpringMVC

以下是SpringBoot对SpringMVC的默认配置：

- Inclusion of `ContentNegotiatingViewResolver` and `BeanNameViewResolver` beans.

  - 自动配置了ViewResolver（视图解析器：根据方法的返回值得到视图对象（View），视图对象决定如何渲染（转发？重定向？））
  - ContentNegotiatingViewResolver：组合所有的视图解析器
  - ==如何定制：可以给容器中添加一个视图解析器；ContentNegotiatingViewResolver会自动将其组合进来==

- Support for serving static resources, including support for WebJars (see below).静态资源文件夹路径，webjars

- 自动注册了 `Converter`, `GenericConverter`, `Formatter` beans.

  - Converter：转换器；类型转换使用
  - Formatter：格式化器

  ```java
  @Bean
  // 在文件中配置日期格式化的规则
  @ConditionalOnProperty(prefix = "spring.mvc", name = "date-format")
  public Formatter<Date> dateFormatter() {
      return new DateFormatter(this.mvcProperties.getDateFormat());// 日期格式化组件
  }
  ```

  ==自己添加的 格式化转换器，只需要放在容器中即可==

- Support for `HttpMessageConverters` (see below).

  - HttpMessageConverter：SpringMVC用了转换HTTP请求和响应的；User----Json；
  - HttpMessageConverters是从容器中确定的；获取所有的HttpMessageConverter；==给容器中添加HttpMessageConverter，只需要将组件注册到容器中（@Bean，@ Component）==

- Automatic registration of `MessageCodesResolver` (see below).定义错误代码生成规则

- Static `index.html` support.静态首页访问

- Custom `Favicon` support (see below).favicon.ico

- Automatic use of a `ConfigurableWebBindingInitializer` bean (see below).

  ==可以配置一个ConfigurableWebBindingInitializer来替换默认的；（添加到容器中）

  ```java
  初始化WebDataBinder;
  请求数据=====JavaBean;
  ```

If you want to keep Spring Boot MVC features, and you just want to add additional [MVC configuration](https://docs.spring.io/spring/docs/4.3.24.RELEASE/spring-framework-reference/htmlsingle#mvc) (interceptors, formatters, view controllers etc.) you can add your own `@Configuration` class of type `WebMvcConfigurerAdapter`, but **without** `@EnableWebMvc`. If you wish to provide custom instances of `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter` or `ExceptionHandlerExceptionResolver` you can declare a `WebMvcRegistrationsAdapter` instance providing such components.

If you want to take complete control of Spring MVC, you can add your own `@Configuration` annotated with `@EnableWebMvc`.

### 2、扩展SpringMVC

**==编写一个配置类（@Configuration），是WebMvcConfigurerAdapter类型的；不能标注@EnableWebMvc==**

既保留了所有的自动配置，也能用自己扩展的配置；

```java
// 使用WebMvcConfigurerAdapter可以来扩展SpringMVC的功能
@Configuration
public class MyMVCConfig extends WebMvcConfigurerAdapter {
	
	@Override
	public void addViewControllers(ViewControllerRegistry registry) {
		// TODO Auto-generated method stub
//		super.addViewControllers(registry);
		// 浏览器发送 /atguigu 请求来到 success
		registry.addViewController("/atguigu").setViewName("success");
	}
}
```

原理：

​		1、 WebMvcAutoConfiguration 是SpringMVC的自动配置类

​		2、 在做其他自动配置时会导入；**@Import(EnableWebMvcConfiguration.class)**

```java
@Configuration
	public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {

        // 从容器中获取所有的WebMvcConfigurer
        @Autowired(required = false)
        public void setConfigurers(List<WebMvcConfigurer> configurers) {
            if (!CollectionUtils.isEmpty(configurers)) {
                this.configurers.addWebMvcConfigurers(configurers);
                // 一个参考实现：将所有的WebMvcConfigurer相关配置都一起调用
            /*@Override
            public void addViewControllers(ViewControllerRegistry registry) {
                for (WebMvcConfigurer delegate : this.delegates) {
                    delegate.addViewControllers(registry);
                }
            }
            */
            }
        }
```

​		3、 容器中所有的WebMvcConfigurer都会一起起作用；

​		4、 自己定义的配置类也会被调用

​		**效果：SpringMVC的自动配置和我们的扩展配置都会起作用**

### 3、全面接管SpringMVC

SpringBoot对SpringMVC的自动配置不需要了，所有的配置都是自己配置的；所有的SpringMVC的自动配置都失效了

**只需要在自己定义的配置类中添加@EnableWebMvc即可；（不推荐）**



原理：为什么@EnableWebMvc自动配置就失效了？

1、@EnableWebMvc的核心

```java
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

2、 DelegatingWebMvcConfiguration是WebMvcConfigurationSupport的子类

```java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

```

3、容器中没有WebMvcConfigurationSupport这个组件的时候，自动配置类才生效

```java
@Configuration
@ConditionalOnWebApplication
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class,
		WebMvcConfigurerAdapter.class })
// 容器中没有这个组件的时候，自动配置类才生效
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
```

4、 @EnableWebMvc将WebMvcConfigurationSupport组件导入了进来；

5、导入的WebMvcConfigurationSupport只是SpringMVC最基本的功能；

## 5、如何修改SpringBoot的默认配置

### ==模式==

​		1、 SpringBoot在自动配置很多组件的时候，先看容器中有没有用户自己配置的（@Bean、@Component），如果有就用用户配置的，没有，才自动配置；如果有些组件可以有多个（如ViewResolver），将用户配置的和默认的组合起来；

​		2、 在SpringBoot中会有很多的xxxConfigurer帮助我们进行扩展配置

​		3、在SpringBoot中会有很多的xxxxCustomizer帮助我们进行定制配置

## 6、RestfulCRUD

### 0、实验要求

1、RestfulCRUD：CRUD满足Rest风格

URI：/资源名称/资源标识			HTTP请求方式区分对资源CRUD操作

  

|      | 普通CURD（URI来区分操作） | RestfulCRUD         |
| ---- | ------------------------- | ------------------- |
| 查询 | getEmp                    | emp---GET           |
| 添加 | addEmp?xxx                | emp----POST         |
| 修改 | updateEmp?id=xxx&xxx=xx   | emp/{id}----PUT     |
| 删除 | deleteEmp?id=xxx          | emp/{id}-----DELETE |

2、实验的请求架构

|                                      | 请求URI  | 请求方式 |
| ------------------------------------ | -------- | -------- |
| 查询所有员工                         | emps     | GET      |
| 查询某个员工（来到修改页面）         | emp/{id} | GET      |
| 来到添加页面                         | emp      | GET      |
| 添加员工                             | emp      | POST     |
| 来到修改页面（查询员工信息进行回显） | emp/{id} | GET      |
| 修改员工                             | emp      | PUT      |
| 删除员工                             | emp/{id} | DELETE   |



### 1、默认访问页面

配置WebMvcConfigurerAdapter，将/和/index.html映射到login页面

```java
// 所有的WebMvcConfigurerAdapter组件都会一起起作用
// 将组件注册在容器中
@Bean
public WebMvcConfigurerAdapter webMvcConfigurerAdapter() {
    WebMvcConfigurerAdapter adapter = new WebMvcConfigurerAdapter() {
        @Override
        public void addViewControllers(ViewControllerRegistry registry) {
            registry.addViewController("/").setViewName("login");
            registry.addViewController("/index.html").setViewName("login");
        }
    };
    return adapter;
}
```



修改bootstrap引入模板

```html
<!-- Bootstrap core CSS -->
		<link href="asserts/css/bootstrap.min.css" th:href="@{/webjars/bootstrap/4.3.1/css/bootstrap.css}" rel="stylesheet">
		<!-- Custom styles for this template -->
		<link href="asserts/css/signin.css" th:href="@{/asserts/css/signin.css}" rel="stylesheet">

```

### 2、国际化

SpringMVC中实现国际化

​		**1、编写国际化配置文件；**

​		2、使用ResourceBundleMessageSource管理国际化资源文件

​		3、在页面使用fmt:message取出国际化内容

SpringBoot实现国际化步骤：

​		1、编写国际化配置文件，抽取页面需要显示的国际化消息

​		2、SpringBoot自动配置好了管理国际化资源文件的组件

```java
@ConfigurationProperties(prefix = "spring.messages")
public class MessageSourceAutoConfiguration {
    
    /**
	 * Comma-separated list of basenames (essentially a fully-qualified classpath
	 * location), each following the ResourceBundle convention with relaxed support for
	 * slash based locations. If it doesn't contain a package qualifier (such as
	 * "org.mypackage"), it will be resolved from the classpath root.
	 */
	private String basename = "messages";	// 配置文件可以直接放在类路径下叫做messages.properties
    
    @Bean
	public MessageSource messageSource() {
		ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
		if (StringUtils.hasText(this.basename)) {
            // 设置国际化资源文件的基础名（去掉语言国家代码的）
			messageSource.setBasenames(StringUtils.commaDelimitedListToStringArray(
					StringUtils.trimAllWhitespace(this.basename)));
		}
		if (this.encoding != null) {
			messageSource.setDefaultEncoding(this.encoding.name());
		}
		messageSource.setFallbackToSystemLocale(this.fallbackToSystemLocale);
		messageSource.setCacheSeconds(this.cacheSeconds);
		messageSource.setAlwaysUseMessageFormat(this.alwaysUseMessageFormat);
		return messageSource;
	}
```



3、去页面获取国际化的值



```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
	<head>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
		<meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
		<meta name="description" content="">
		<meta name="author" content="">
		<title>Signin Template for Bootstrap</title>
		<!-- Bootstrap core CSS -->
		<link href="asserts/css/bootstrap.min.css" th:href="@{/webjars/bootstrap/4.3.1/css/bootstrap.css}" rel="stylesheet">
		<!-- Custom styles for this template -->
		<link href="asserts/css/signin.css" th:href="@{/asserts/css/signin.css}" rel="stylesheet">
	</head>

	<body class="text-center">
		<form class="form-signin" action="dashboard.html">
			<img class="mb-4" th:src="@{/asserts/img/bootstrap-solid.svg}" src="asserts/img/bootstrap-solid.svg" alt="" width="72" height="72">
			<h1 class="h3 mb-3 font-weight-normal" th:text="#{login.tip}">Please sign in</h1>
			<label class="sr-only" th:text="#{login.username}">Username</label>
			<input type="text" class="form-control" placeholder="Username" th:placeholder="#{login.username}" required="" autofocus="">
			<label class="sr-only" th:text="#{login.password}">Password</label>
			<input type="password" class="form-control" placeholder="Password" th:placeholder="#{login.password}" required="">
			<div class="checkbox mb-3">
				<label>
          <input type="checkbox" value="remember-me"> [[#{login.remember}]]
        </label>
			</div>
			<button class="btn btn-lg btn-primary btn-block" type="submit" th:text="#{login.btn}">Sign in</button>
			<p class="mt-5 mb-3 text-muted">© 2017-2019</p>
			<a class="btn btn-sm">中文</a>
			<a class="btn btn-sm">English</a>
		</form>

	</body>

</html>
```

效果：根据浏览器语言的信息切换国际化

原理：

​			国际化Locale（区域信息对象）：LocaleResolver（获取区域信息对象）

```java
@Bean
		@ConditionalOnMissingBean
		@ConditionalOnProperty(prefix = "spring.mvc", name = "locale")
		public LocaleResolver localeResolver() {
			if (this.mvcProperties
					.getLocaleResolver() == WebMvcProperties.LocaleResolver.FIXED) {
				return new FixedLocaleResolver(this.mvcProperties.getLocale());
			}
            // 从请求头中获取区域信息
			AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
			localeResolver.setDefaultLocale(this.mvcProperties.getLocale());
			return localeResolver;
		}
// 默认的解析器就是根据请求头带来的区域信息获取Locale进行国际化
```

4、点击链接切换国际化

```html
<a class="btn btn-sm" th:href="@{/index.html(l='zh_CN')}">中文</a>
<a class="btn btn-sm" th:href="@{/index.html(l='en_US')}">English</a>		
```



```java
/**
 * 可以在链接上携带区域信息
 * @author acer
 *
 */
public class MyLocaleResolver implements LocaleResolver {

	@Override
	public Locale resolveLocale(HttpServletRequest request) {
		String l = request.getParameter("l");
		// 默认的是中文的，不管浏览器怎么设置都是中文的
		Locale locale = Locale.getDefault();
		if (!StringUtils.isEmpty(l)) {
			String[] split = l.split("_");
			locale = new Locale(split[0],split[1]);
		}
		return locale;
	}

	@Override
	public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
		// TODO Auto-generated method stub
		
	}

}


// MyMVCConfig下添加自定义的LocaleResolver
	@Bean
	public LocaleResolver localeResolver() {
		return new MyLocaleResolver();
	}
```



### 3、登录

开发期间，模板引擎页面修改后要实时生效：

1）、禁用模板引擎的缓存（eclipse只要这一步）

2）、页面修改完成后重新编译（IDEA需要两步）Ctrl+F9



登录错误消息的显示

```html
<!-- 判断的优先级大于text -->
<p style="color: red" th:text="${msg}" th:if="${not #strings.isEmpty(msg)}"></p>
```

使用重定向，防止表单重复提交

```java
if (!StringUtils.isEmpty("username") && "123456".equals(password)) {
			// 登录成功，为了防止表单重复提交，可以重定向到主页
			return "redirect:/main.html";
		} else {
			// 登录失败
			map.put("msg","用户名或密码错误！");
			return "login";
		}
```



### 4、拦截器进行登录检查

自定义拦截器

```java
/**
 * 登录检查
 * @author acer
 *
 */
public class LoginHandlerInterceptor implements HandlerInterceptor {

	// 目标方法执行之前
	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		Object user = request.getSession().getAttribute("loginUser");
		if (user == null) {
			// 未登录，返回登录页面
			request.setAttribute("msg", "没有权限，请先登录！");
			request.getRequestDispatcher("/index.html").forward(request, response);
			return false;
		} else {
			// 已登录，放行请求
			return true;
		}
	}

	@Override
	public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			ModelAndView modelAndView) throws Exception {
		// TODO Auto-generated method stub
		
	}

	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
		// TODO Auto-generated method stub
		
	}

}

```

将自定义的拦截器配置到MyMVCConfig配置类中

```java
// 注册拦截器
			@Override
			public void addInterceptors(InterceptorRegistry registry) {
//				// TODO Auto-generated method stub
//				super.addInterceptors(registry);
				// 静态资源   *.css  *.js
				// SpringBoot已经做好了静态资源映射
				registry.addInterceptor(new LoginHandlerInterceptor()).addPathPatterns("/**")
						.excludePathPatterns("/index.html", "/", "/user/login");
			}
```



将登录用户的用户名显示出来

```html
<a class="navbar-brand col-sm-3 col-md-2 mr-0" href="http://getbootstrap.com/docs/4.0/examples/dashboard/#">[[${session.loginUser}]]</a>
```



### 5、CRUD--员工列表

#### 5.1 Thymeleaf公共页面元素抽取：

```html
1、抽取公共片段
<div th:fragment="copy">
&copy; 2011 The Good Thymes Virtual Grocery
</div>
2、引入公共片段 
<div th:replace="~{footer :: copy}"></div>
<div th:replace="footer :: copy"></div>
~{templatename::selector} ： 模板名::选择器
~{templatename::fragmentname}： 模板名::片段名

3、默认效果
insert的公共片段在div标签中
如果使用th:insert等属性进行引入，可以不用写~{}
行内写法可以加上:[[~{}]];[(~{})]
```

三种引入公共片段的th属性

**th:insert**：将公共片段整个插入到声明引入的元素中

**th:replace**：将声明引入的元素替换为公共片段

**th:include**：将被引入的片段的内容包含进标签中

```html
<footer th:fragment="copy">
&copy; 2011 The Good Thymes Virtual Grocery
</footer>

引入方式
<div th:insert="footer :: copy"></div>
<div th:replace="footer :: copy"></div>
<div th:include="footer :: copy"></div>

效果
<div>
<footer>
&copy; 2011 The Good Thymes Virtual Grocery
</footer>
</div>
<footer>
&copy; 2011 The Good Thymes Virtual Grocery
</footer>
<div>
&copy; 2011 The Good Thymes Virtual Grocery
</div>
```

#### 5.2 链接高亮&&列表完成

链表高亮

```html
<div th:replace="commons/bar::#sidebar(activeUri='main')"></div>
```

遍历取出员工数据

#### 5.3 添加员工

```HTML
<form th:action="@{/emp}" method="post">
    <!--发送put请求修改员工数据-->
    <!--
1、SpringMVC中配置HiddenHttpMethodFilter;（SpringBoot自动配置好的）
2、页面创建一个post表单
3、创建一个input项，name="_method";值就是我们指定的请求方式
-->
    <input type="hidden" name="_method" value="put" th:if="${emp!=null}"/>
    <input type="hidden" name="id" th:if="${emp!=null}" th:value="${emp.id}">
    <div class="form-group">
        <label>LastName</label>
        <input name="lastName" type="text" class="form-control" placeholder="zhangsan" th:value="${emp!=null}?${emp.lastName}">
    </div>
    <div class="form-group">
        <label>Email</label>
        <input name="email" type="email" class="form-control" placeholder="zhangsan@atguigu.com" th:value="${emp!=null}?${emp.email}">
    </div>
    <div class="form-group">
        <label>Gender</label><br/>
        <div class="form-check form-check-inline">
            <input class="form-check-input" type="radio" name="gender" value="1" th:checked="${emp!=null}?${emp.gender==1}">
            <label class="form-check-label">男</label>
        </div>
        <div class="form-check form-check-inline">
            <input class="form-check-input" type="radio" name="gender" value="0" th:checked="${emp!=null}?${emp.gender==0}">
            <label class="form-check-label">女</label>
        </div>
    </div>
    <div class="form-group">
        <label>department</label>
        <!--提交的是部门的id-->
        <select class="form-control" name="department.id">
            <option th:value="${dept.id}" th:each="dept:${depts}" th:text="${dept.departmentName}"></option>
            <!-- 							<option th:selected="${emp!=null}?${dept.id == emp.department.id}" th:value="${dept.id}" th:each="dept:${depts}" th:text="${dept.departmentName}">1</option> -->
        </select>
    </div>
    <div class="form-group">
        <label>Birth</label>
        <input name="birth" type="text" class="form-control" placeholder="zhangsan" th:value="${emp!=null}?${#dates.format(emp.birth, 'yyyy-MM-dd HH:mm')}">
    </div>
    <button type="submit" class="btn btn-primary" th:text="${emp!=null}?'修改':'添加'">添加</button>
</form>
```

提交的数据格式不对：生日：日期；

2017/12/12	2017-12-12 	2017.12.12

日期的格式化：SpringMVC将页面提交的值需要转换成指定的类型

默认日期是按照/的方式

修改配置：spring.mvc.date-format参数



#### 5.4 修改员工

#### 5.5 删除员工



## 7、错误处理机制

### 7.1 SpringBoot默认的错误处理机制

默认效果：

​			1、浏览器，返回一个默认的错误页面

​			2、其他的客户端，返回一个json数据

原理：可以参照ErrorMvcAutoConfiguration；错误处理的自动配置

​	给容器添加了以下组件：

​			1、DefaultErrorAttributes:

```java
在页面共享信息；
```



​			2、BasicErrorController：处理默认 /error 请求

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
    
    // 两种请求处理方式，第一种对应浏览器，第二种对应json数据，根据发送请求的请求头区分
    @RequestMapping(produces = "text/html")	// 产生HTML类型的数据
	public ModelAndView errorHtml(HttpServletRequest request,
			HttpServletResponse response) {
		HttpStatus status = getStatus(request);
		Map<String, Object> model = Collections.unmodifiableMap(getErrorAttributes(
				request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
		response.setStatus(status.value());
        // 去哪个页面做为错误页面；包含页面地址和页面内容
		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
		return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
	}

	@RequestMapping
	@ResponseBody	// 产生json数据
	public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
		Map<String, Object> body = getErrorAttributes(request,
				isIncludeStackTrace(request, MediaType.ALL));
		HttpStatus status = getStatus(request);
		return new ResponseEntity<Map<String, Object>>(body, status);
	}
```



​			3、ErrorPageCustomizer

```java
@Value("${error.path:/error}")
	private String path = "/error";	// 系统出现错误后，来到error请求进行处理；（web.xml注册的错误页面规则）
```



​			4、DefaultErrorViewResolver

```java
@Override
	public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status,
			Map<String, Object> model) {
		ModelAndView modelAndView = resolve(String.valueOf(status), model);
		if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
			modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
		}
		return modelAndView;
	}

	private ModelAndView resolve(String viewName, Map<String, Object> model) {
        // 默认SpringBoot可以找到一个页面？	error/404
		String errorViewName = "error/" + viewName;
        
        // 模板引擎可以解析这个页面地址就用模板引擎解析
		TemplateAvailabilityProvider provider = this.templateAvailabilityProviders
				.getProvider(errorViewName, this.applicationContext);
		if (provider != null) {
            // 模板引擎可用的情况下返回到 errorViewName指定的视图地址
			return new ModelAndView(errorViewName, model);
		}
        // 模板引擎不可用，就在静态资源文件夹下找errorViewName对应的页面	error/404.html
		return resolveResource(errorViewName, model);
	}
```



​	步骤：

​			一旦系统出现4xx或5xx之类的错误，**ErrorPageCustomizer**就会生效（定制错误的响应规则）；就会来到/error请求； 就会被**BasicErrorController**处理；

​			1）、响应HTML页面：去哪个页面是由**DefaultErrorViewResolver**解析得到的

```java
protected ModelAndView resolveErrorView(HttpServletRequest request,
			HttpServletResponse response, HttpStatus status, Map<String, Object> model) {
	// 所有的ErrorViewResolver 得到ModelAndView
    for (ErrorViewResolver resolver : this.errorViewResolvers) {
			ModelAndView modelAndView = resolver.resolveErrorView(request, status, model);
			if (modelAndView != null) {
				return modelAndView;
			}
		}
		return null;
	}
```



### 7.2 如何定制错误响应：

#### 		1、如何定制错误的页面

​			1）、有模板引擎的情况下：error/状态码；【将错误页面命名为 ：错误状态码.html 放在模板引擎文件夹里面的  error 文件夹下】，发生此状态码的错误就会来到 对应的页面；

​					可以使用4xx和5xx做为错误页面的文件名来匹配这种类型的所有错误，精确优先（优先寻找精确的状态码.html）

​					页面能获取的信息：

​							timestamp：时间戳

​							status：状态码

​							error：错误提示

​							exception：异常对象

​							message：异常消息

​							errors：JSR303校验的错误都在这里

​							trace

​							path

​			2）、没有模板引擎（模板引擎找不到这个错误页面），静态资源文件夹下找；

​			3）、以上都没有错误页面，就是默认来到 SpringBoot 默认的错误提示页面 

#### 		2、如何定制错误的json数据

​				1）、自定义异常处理&&返回定制json数据

```java
@ControllerAdvice
public class MyExceptionHandler {
	
	@ResponseBody
	@ExceptionHandler(UserNotExistException.class)
	public Map<String, Object> handleException(Exception e) {
		
		Map<String, Object> map = new HashMap<String, Object>();
		map.put("code", "user.notexist");
		map.put("message", e.getMessage());
		return map;
	}
}
// 没有自适应效果
```

​				2）、转发到/error, BasicErrorController进行自适应处理

```java
@ExceptionHandler(UserNotExistException.class)
	public String handleException(Exception e, HttpServletRequest request) {
		
		Map<String, Object> map = new HashMap<String, Object>();
		// 传入我们自己的错误状态码 4xx 5xx，否则就不会进入自定义的错误页面的解析流程
		/*
		 * Integer statusCode = (Integer) request
				.getAttribute("javax.servlet.error.status_code");
		 */
		request.setAttribute("javax.servlet.error.status_code", 404);
		map.put("code", "user.notexist");
		map.put("message", e.getMessage());
		// 转发到/error, BasicErrorController进行自适应处理
		return "forward:/error";
	}
```

#### 3、将定制的数据携带出去

出现错误后，会来到/error请求，会被BasicErrorController处理，响应出去可以获取的数据是由getErrorAttributes得到的（是AbstractErrorController规定的方法（ErrorController））；

1、完全来编写一个ErrorController的实现类【或者是编写AbstractErrorController的子类】，放在容器中；					**2、**页面上能用的数据，或者是json返回能用的数据都是通过errorAttributes.getErrorAttributes得到

​		容器中DefaultErrorAttributes.getErrorAttributes()；默认进行数据处理

自定义ErrorAttributes

```java
@Component
public class MyErrorAttributes extends DefaultErrorAttributes {
	@Override
	public Map<String, Object> getErrorAttributes(RequestAttributes requestAttributes, boolean includeStackTrace) {
		// TODO Auto-generated method stub
		Map<String, Object> map = super.getErrorAttributes(requestAttributes, includeStackTrace);
		map.put("company", "atguigu");
		return map;
	}
}
```

最终的效果：响应是自适应的，通过定制ErrorAttributes改变需要返回的内容



## 8、配置嵌入式Servlet容器

SpringBoot默认使用Tomcat做为嵌入式的Servlet容器（Tomcat）

问题？

### 1、如何定制和修改Servlet容器的相关配置

​		1）、修改和server有关的配置( ServerProperties类 【也是EmbeddedServletContainerCustomizer】)

```properties
server.port=8081
server.context-path=/crud

server.tomcat.uri-encoding=UTF-8

# 通用的Servlet容器设置
server.xxx
# Tomcat的设置
server.tomcat.xxx
```

​		2）、编写一个**EmbeddedServletContainerCustomizer**：嵌入式的 Servlet 容器的定制器；来修改Servlet容器的配置

```java
@Bean
	public EmbeddedServletContainerCustomizer embeddedServletContainerCustomizer() {
		return new EmbeddedServletContainerCustomizer() {
			
			// 定制嵌入式的 Servlet 容器相关的规则
			@Override
			public void customize(ConfigurableEmbeddedServletContainer container) {
				container.setPort(8083);
				
			}
		};
	}
```

### 2、注册Servlet三大组件【Servlet、FIlter、Listener】

由于SpringBoot默认是以jar包的形式启动嵌入式的Servlet容器来启动SpringBoot的web应用，没有web.xml文件。

注册三大组件用以下方式：

​		ServletRegistrationBean

```java
@Bean
public ServletRegistrationBean myServlet() {
    ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean(new MyServlet(), "/myServlet");
    return servletRegistrationBean;
}
```



​		FIlterRegistrationBean

```java
@Bean
public FilterRegistrationBean myFilter() {
    FilterRegistrationBean registrationBean = new FilterRegistrationBean();
    registrationBean.setFilter(new MyFilter());
    registrationBean.setUrlPatterns(Arrays.asList("/hello", "/myServlet"));
    return registrationBean;
}
```



​		ServletListenerRegistrationBean

```java
@Bean
public ServletListenerRegistrationBean myListener() {
    ServletListenerRegistrationBean<MyListener> listenerRegistrationBean = new ServletListenerRegistrationBean<MyListener>(new MyListener());
    return listenerRegistrationBean;
}
```



SpringBoot 自动配置SpringMVC的时候，自动注册SpringMVC的前端控制器；DispatcherServlet；

DispatcherServlet自动配置了ServletRegistrationBean

```java
@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
@ConditionalOnBean(value = DispatcherServlet.class,
                   name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
public ServletRegistrationBean dispatcherServletRegistration(
    DispatcherServlet dispatcherServlet) {
    ServletRegistrationBean registration = new ServletRegistrationBean(
        dispatcherServlet, this.serverProperties.getServletMapping());
    // 默认拦截： / 所有请求；包括静态资源，但是不拦截jsp请求	/* 会拦截jsp请求
    // 可以通过server.servletPath 来修改SpringMVC前端控制器默认拦截的请求路径
    registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
    registration.setLoadOnStartup(
        this.webMvcProperties.getServlet().getLoadOnStartup());
    if (this.multipartConfig != null) {
        registration.setMultipartConfig(this.multipartConfig);
    }
    return registration;
}
```



### 3、SpringBoot使用其他的Servlet容器

在ServerProperties中：

```java
private final Tomcat tomcat = new Tomcat();

private final Jetty jetty = new Jetty();

private final Undertow undertow = new Undertow();
```



Tomcat

```xml
<!-- 引入web模块 -->
<!-- 引入web模块默认使用嵌入式的Tomcat做为Servlet容器 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```



Jetty（长连接）

```xml
<!-- 引入web模块 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 引入其他的Servlet容器 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```



Undertow（不支持JSP）（并发性能好）

```xml
<!-- 引入web模块 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 引入其他的Servlet容器 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

### 4、嵌入式Servlet容器自动配置原理

EmbeddedServletContainerAutoConfiguration：嵌入式的Servlet容器自动配置

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@ConditionalOnWebApplication
// 给容器导入EmbeddedServletContainerCustomizerBeanPostProcessor组件
// 后置处理器 在Bean初始化前后执行前置后置的逻辑 (创建完对象还没属性赋值进行初始化工作)
@Import(BeanPostProcessorsRegistrar.class)
public class EmbeddedServletContainerAutoConfiguration {
    
    @Configuration
    // 判断当前是否引入了Tomcat依赖
	@ConditionalOnClass({ Servlet.class, Tomcat.class })
    // 判断当前容器没有用户自己定义的EmbeddedServletContainerFactory：嵌入式的Servlet容器工厂；作用：创建嵌入式的Servlet容器
	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class,
			search = SearchStrategy.CURRENT)
	public static class EmbeddedTomcat {

		@Bean
		public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
			return new TomcatEmbeddedServletContainerFactory();
		}
	}
    
    /**
	 * Nested configuration if Jetty is being used.
	 */
	@Configuration
	@ConditionalOnClass({ Servlet.class, Server.class, Loader.class,
			WebAppContext.class })
	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class,
			search = SearchStrategy.CURRENT)
	public static class EmbeddedJetty {

		@Bean
		public JettyEmbeddedServletContainerFactory jettyEmbeddedServletContainerFactory() {
			return new JettyEmbeddedServletContainerFactory();
		}

	}

	/**
	 * Nested configuration if Undertow is being used.
	 */
	@Configuration
	@ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class,
			search = SearchStrategy.CURRENT)
	public static class EmbeddedUndertow {

		@Bean
		public UndertowEmbeddedServletContainerFactory undertowEmbeddedServletContainerFactory() {
			return new UndertowEmbeddedServletContainerFactory();
		}

	}
```

1）、EmbeddedServletContainerFactory（嵌入式Servlet容器工厂）



``` java
public interface EmbeddedServletContainerFactory {

    //获取嵌入式的Servlet容器
	EmbeddedServletContainer getEmbeddedServletContainer(
		ServletContextInitializer... initializers);
｝
```

2）、EmbeddedServletContainer（嵌入式的Servlet容器）



3）、TomcatEmbeddedServletContainerFactory为例

```java
@Override
public EmbeddedServletContainer getEmbeddedServletContainer(
    ServletContextInitializer... initializers) {
    // 创建一个Tomcat
    Tomcat tomcat = new Tomcat();
    
    // 配置Tomcat的基本环境
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory
        : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    configureEngine(tomcat.getEngine());
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    // 将配置好的tomcat传入进去；返回一个EmbeddedServletContainer；并且启动Tomcat服务器
    return getTomcatEmbeddedServletContainer(tomcat);
}
```



4）、嵌入书容器的配置修改是怎样生效的

```
ServerProperties、EmbeddedServletContainerCustomizer
```

**EmbeddedServletContainerCustomizer**:定制器帮我们修改了Servlet容器配置

5）、容器中导入了：EmbeddedServletContainerCustomizerBeanPostProcessor

```java
// 初始化之前
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName)
    throws BeansException {
    // 如果当前初始化的是一个ConfigurableEmbeddedServletContainer类型的组件
    if (bean instanceof ConfigurableEmbeddedServletContainer) {
        // 
        postProcessBeforeInitialization((ConfigurableEmbeddedServletContainer) bean);
    }
    return bean;
}

private void postProcessBeforeInitialization(
			ConfigurableEmbeddedServletContainer bean) {
    	// 获取所有的定制器，调用每个定制器的customer方法给Servlet容器进行属性赋值
		for (EmbeddedServletContainerCustomizer customizer : getCustomizers()) {
			customizer.customize(bean);
		}
	}

private Collection<EmbeddedServletContainerCustomizer> getCustomizers() {
		if (this.customizers == null) {
			// Look up does not include the parent context
			this.customizers = new ArrayList<EmbeddedServletContainerCustomizer>(
					this.beanFactory
                // 从容器中获取所有这个类型的组件：EmbeddedServletContainerCustomizer
                // 定制Servlet容器，可以给容器中添加一个EmbeddedServletContainerCustomizer类型组件
							.getBeansOfType(EmbeddedServletContainerCustomizer.class,
									false, false)
							.values());
			Collections.sort(this.customizers, AnnotationAwareOrderComparator.INSTANCE);
			this.customizers = Collections.unmodifiableList(this.customizers);
		}
		return this.customizers;
	}


// ServerProperties也是定制器
```



步骤：

1）、SpringBoot根据导入的依赖情况，给容器中添加相应的容器工厂 例：TomcatEmbeddedServletContainerFactory

EmbeddedServletContainerFactory【TomcatEmbeddedServletContainerFactory】

2）、容器中某个组件要创建对象就要通过后置处理器；

```
EmbeddedServletContainerCustomizerBeanPostProcessor
```

只要是嵌入式的Servlet容器工厂，后置处理器就工作；

3）、后置处理器，从容器中获取的所有的EmbeddedServletContainerCustomizer，调用定制器的定制方法



### 5、嵌入式Servlet容器启动原理

什么时候创建嵌入式的Servlet的容器工厂？什么时候获取嵌入式的Servlet容器并启动Tomcat;

获取嵌入式的Servlet容器工厂：

1）、SpringBoot应用启动Run方法

2）、refreshContext(context); SpringBoot刷新IOC容器对象【创建IOC容器对象，并初始化容器，创建容器的每一个组件】；如果是web环境**AnnotationConfigEmbeddedWebApplicationContext**, 如果不是**AnnotationConfigApplicationContext**

```
if (contextClass == null) {
   try {
      contextClass = Class.forName(this.webEnvironment
            ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
   }
```

3）、refresh(context);刷新创建好的IOC容器

```java
try {
   // Allows post-processing of the bean factory in context subclasses.
   postProcessBeanFactory(beanFactory);

   // Invoke factory processors registered as beans in the context.
   invokeBeanFactoryPostProcessors(beanFactory);

   // Register bean processors that intercept bean creation.
   registerBeanPostProcessors(beanFactory);

   // Initialize message source for this context.
   initMessageSource();

   // Initialize event multicaster for this context.
   initApplicationEventMulticaster();

   // Initialize other special beans in specific context subclasses.
   onRefresh();

   // Check for listener beans and register them.
   registerListeners();

   // Instantiate all remaining (non-lazy-init) singletons.
   finishBeanFactoryInitialization(beanFactory);

   // Last step: publish corresponding event.
   finishRefresh();
}
```

4）、 **onRefresh**();web的ioc容器重写了onRefresh方法

5）、web ioc会创建嵌入式的Servlet容器；**createEmbeddedServletContainer**

**6）、获取嵌入式的Servlet容器工厂；**

```
EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
```

从ioc容器中获取EmbeddedServletContainerFactory组件；

```
@Bean
public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
return new TomcatEmbeddedServletContainerFactory();
}
```

**TomcatEmbeddedServletContainerFactory**创建对象，后置处理器看这个对象，就来获取所有的定制器来定制Servlet容器的相关配置；

7）、**使用容器工厂获取嵌入式的Servlet容器**

8）、嵌入式的Servlet容器创建对象并启动Servlet容器；

先启动嵌入式的Servlet容器，再将ioc容器中剩下的没有创建出的对象获取出来

IOC启动创建Servlet容器



## 9、使用外置的Servlet容器

### 1、嵌入式Servlet容器和外置的Servlet容器

嵌入式Servlet容器：应用达成可执行的jar包

​				优点：简单、便携；

​				缺点：默认不支持JSP、优化定制比较复杂（使用定制器【ServerProperties、自定义EmbeddedServletContainerCustomizer】，自己编写嵌入式Servlet容器的创建工程【EmbeddedServletContainerFactory】）

外置的Servlet容器：外面安装Tomcat----应用以war包的形式打包





### 2、步骤：

1、必须创建一个war项目；（创建好目录结构）

2、将嵌入式的Tomcat指定为provided

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

3、必须编写一个SpringBootServletInitializer的子类，并调用configure方法里面的固定写法

```java
public class ServletInitializer extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        //传入SpringBoot的主程序，
        return application.sources(SpringBoot04WebJspApplication.class);
    }

}
```

4、启动服务器就可以；

### 3、原理

jar包：执行SpringBoot主类的main方法，启动ioc容器，创建嵌入式的Servlet的容器；

war包：启动服务器，服务器启动SpringBoot应用，【SpringBootServletInitializer】启动ioc容器

**servlet3.0规范**

8.2.4 共享库和运行时插件

**规则**：

1、服务器启动（web应用启动），会创建当前的web应用里面每一个jar包里面ServletContrainerInitializer的实现类的实例

2、SpringBootServletInitializer这个类的实现需要放在jar包下的META-INF/services文件夹下，有一个命名为javax.servlet.ServletContainerInitalizer的文件，内容就是ServletContainerInitializer的实现类全类名

3、还可以使用@HandlerTypes注解，在应用启动的时候可以启动我们感兴趣的类

流程：

1、启动Tomcat服务器

2、spring web模块里有一个文件，文件内容：（应用启动的时候会创建它的实例对象）

```java
org.springframework.web.SpringServletContainerInitializer
```

3、SpringServletContainerInitializer将handlerTypes（WebApplicationInitializer.class ）标注的所有这个类型的类传入到onStartip方法的Set<Class<?>>;为这些WebApplicationInitializer类型的类创建实例；

4、每个创建好的WebApplicationInitializer调用自己的onStratup

5、相当于我们的SpringBootServletInitializer的类会被创建对象，并执行onStartup方法

6、SpringBootServletInitializer实例执行onStartup方法的时候会创建createRootApplicationContext；创建容器

```java
protected WebApplicationContext createRootApplicationContext(ServletContext servletContext) {
     // 1、创建环境构建器 SpringApplicationBuilder
    SpringApplicationBuilder builder = this.createSpringApplicationBuilder();
    StandardServletEnvironment environment = new StandardServletEnvironment();
    environment.initPropertySources(servletContext, (ServletConfig)null);
    builder.environment(environment);
    builder.main(this.getClass());
    ApplicationContext parent = this.getExistingRootWebApplicationContext(servletContext);
    if (parent != null) {
        this.logger.info("Root context already created (using as parent).");
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, (Object)null);
        builder.initializers(new ApplicationContextInitializer[]{new ParentContextApplicationContextInitializer(parent)});
    }
	
    builder.initializers(new ApplicationContextInitializer[]{new ServletContextApplicationContextInitializer(servletContext)});
    builder.contextClass(AnnotationConfigEmbeddedWebApplicationContext.class);
    // 调用Configure,子类重写了这个方法，将SpringBoot的主程序类传入进来
    builder = this.configure(builder);
    // 使用builder创建一个spring应用
    SpringApplication application = builder.build();
    if (application.getSources().isEmpty() && AnnotationUtils.findAnnotation(this.getClass(), Configuration.class) != null) {
        application.getSources().add(this.getClass());
    }

    Assert.state(!application.getSources().isEmpty(), "No SpringApplication sources have been defined. Either override the configure method or add an @Configuration annotation");
    if (this.registerErrorPageFilter) {
        application.getSources().add(ErrorPageFilterConfiguration.class);
    }
	//最后启动Spring容器
    return this.run(application);
}
```

7、Spring的应用就启动完了并且创建IOC容器；（同嵌入式Servlet容器）

```java
public ConfigurableApplicationContext run(String... args) {
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   FailureAnalyzers analyzers = null;
   configureHeadlessProperty();
   SpringApplicationRunListeners listeners = getRunListeners(args);
   listeners.starting();
   try {
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
      Banner printedBanner = printBanner(environment);
      context = createApplicationContext();
      analyzers = new FailureAnalyzers(context);
      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      listeners.finished(context, null);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
      return context;
   }
   catch (Throwable ex) {
      handleRunFailure(context, listeners, analyzers, ex);
      throw new IllegalStateException(ex);
   }
}
```

启动Servlet容器，再启动SpringBoot应用