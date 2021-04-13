## 1  简单的 IOC

### 1.1 简单的 IOC 容器实现的步骤

1. 加载 xml 配置文件，遍历其中的标签
2. 获取标签中的 id 和 class 属性，加载 class 属性对应的类，并创建 bean
3. 遍历标签 标签，获取属性值，并将属性值填充到 bean中
4. 将 bean 注册到 bean 容器中

### 1.2 代码结构

```java
SimpleIOC		// IOC 的实现类，实现了上面说的四个步骤
SimpleIOCTest	// IOC 的测试类
Car				// IOC 测试使用的bean
Wheel			// 同上
ioc.xml(放到源代码目录下)	// bean 配置文件
```

### 1.3 使用到的技术

- 加载xml配置文件，使用org.w3c.dom包下的Document等类来遍历其中的标签
- beanClass = Class.forName(className)   ==> beanClass.newInstance();
- 反射：利用反射将 bean 相关字段访问权限设为可访问，将属性值填充到相关字段中
- 使用HashMap 模拟 bean 容器

### 1.4 代码

**容器实现类 SimpleIOC 的代码：**

```java
package ioc;

public class SimpleIOC {
    private Map<String, Object> beanMap = new HashMap<String, Object>();
    
    public SimpleIOC(String location) throws Exception {
        loadBeans(location);
    }
    
    public Object getBean(String name) {
        Object bean = beanMap.get(name);
        if (bean == null) {
            throw new IllegalArgumentException("there is no  bean with name: " + name);
        }
        return bean;
    }
    
    private void loadBeans(String location) throws Exception {
        // 加载 xml 配置文件
        System.out.println(location);
        InputStream inputStream = new FileInputStream(location);
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        DocumentBuilder docBuilder = factory.newDocumentBuilder();
        Document document = docBuilder.parse(inputStream);
        Element root = document.getDocumentElement();
        NodeList nodes = root.getChildNodes();
        
        // 遍历 <bean> 标签
        for (int i = 0; i < nodes.getLength(); i++) {
            Node node = nodes.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                String id = ele.getAttribute("id");
                String className = ele.getAttribute("class");
                
                // 加载 beanClass
                Class beanClass = null;
                try {
                    beanClass = Class.forName(className);
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                    return;
                }
                
                // 创建 bean
                Object bean = beanClass.newInstance();
                
                // 遍历 <property> 标签
                NodeList propertyNodes = ele.getElementsByTagName("property");
                for (int j = 0; j < propertyNodes.getLength(); j++) {
                    Node propertyNode = propertyNodes.item(j);
                    if (propertyNode instanceof Element) {
                        Element propertyElement = (Element) propertyNode;
                        String name = propertyElement.getAttribute("name");
                        String value = propertyElement.getAttribute("value");
                        
                        // 利用反射将 bean 相关字段访问权限设为可访问
                        Field declaredField = bean.getClass().getDeclaredField(name);
                        declaredField.setAccessible(true);
                        
                        if (value != null && value.length() > 0) {
                            // 将属性值填充到相关字段中
                            declaredField.set(bean, value);
                        } else {
                            String ref = propertyElement.getAttribute("ref");
                            if (ref == null || ref.length() == 0) {
                                throw new IllegalArgumentException("ref config error");
                            }
                            
                            // 将引用填充到相关字段中
                            declaredField.set(bean, getBean(ref));
                            
                        }
                        // 将 bean 注册到 bean 容器中
                        registerBean(id, bean);
                    }
                }
            }
        }
    }


    private void registerBean(String id, Object bean) {
        beanMap.put(id, bean);
    }
}
```

**容器测试类使用的 bean 代码**：

```java
package bean;

public class Car {
    private String name;
    private String length;
    private String width;
    private String height;
    private Wheel wheel;
    
    // getter 和 setter 方法
    
    @Override
    public String toString() {
        return "Car [name=" + name + ", length=" + length + ", width=" + width + ", height=" + height + ", wheel="
                + wheel + "]";
    }
    
}

package bean;

public class Wheel {
    private String brand;
    private String specification;
    
    // getter 和 setter 方法
    @Override
    public String toString() {
        return "Wheel [brand=" + brand + ", specification=" + specification + "]";
    }
}
```

**bean 配置文件 bean.xml 内容:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean id="wheel" class="bean.Wheel">
        <property name="brand" value="Michelin"/>
        <property name="specification" value="265/60 R18"/>
    </bean>
    
    <bean id="car" class="bean.Car">
        <property name="name" value="Mercedes Benz G 500"/>
        <property name="length" value="4717mm"/>
        <property name="width" value="1855mm"/>
        <property name="height" value="1949mm"/>
        <property name="wheel" ref="wheel"/>
    </bean>
</beans>
```

**IOC 测试类 SimpleIOCTest：**

```java
class SimpleIOCTest {
    public static void main(String[] args) throws Exception {
        // 注意项目名或工作目录不能带有空格
        System.out.println(SimpleIOC.class.getClassLoader().getResource("bean.xml").getFile());
        String location = SimpleIOC.class.getClassLoader().getResource("bean.xml").getFile();
        SimpleIOC bf = new SimpleIOC(location);
        Wheel wheel = (Wheel) bf.getBean("wheel");
        System.out.println(wheel);
        Car car = (Car) bf.getBean("car");
        System.out.println(car);
    }
}
```

**测试结果**：

![](images/ioc测试结果.png)



## 2 简单的AOP

### 2.1 Spring AOP 的基本概念

**通知（Advice）**：通知定义了要织入对象的逻辑，以及执行时机。Spring中对应了 5 种 不同类型的通知。

- 前置通知（Before）：在目标方法执行前，执行通知
- 后置通知（After）：在目标方法执行后，执行通知，此时不关系目标方法会返回什么
- 返回通知（After-returning）：在目标方法执行返回后，执行通知
- 异常通知（After-throwing）：在目标方法抛出异常后执行通知
- 环绕通知（Around）：目标方法被通知包裹，通知在目标方法执行前和执行后都会被调用

**切点（ PointCut）**：定义了在何处执行通知。作用：通过匹配规则查找合适的连接点（JoinPoint），AOP 会在这些连接点上织入通知。

**切面（Aspect）**：切面包含了通知和切点，通知和切点共同定义了切面是什么，在何时何处执行切面逻辑。

### 2.2 简单 AOP 实现的步骤

基于 JDK 动态代理实现，需要以下三步：

1. 定义一个包含切面逻辑的对象
2. 定义一个Advice 对象（实现了 InvocationHandler 接口），并将上面定义的对象和目标对象传入
3. 将上面的 Advice 对象和目标对象传给 JDK 动态代理方法，为目标对象生成代理。



### 2.3 Java SDK 动态代理原理

Java SDK代理面向的是一组接口，只能为接口创建代理。实现原理如下：

1. 通过Proxy.newProxyInstance 创建代理类对象，需要三个参数：一个是ClassLoader，一个是接口数组，还有一个是InvocationHandler 对象
2. ClassLoader是类加载器，使用本类即SimpleAOP 的类加载器即可
3. 接口数组是我们需要为其创建代理的接口，即该数组中包含 HelloService
4.  InvocationHandler：对代理接口所有方法的调用都会转给该对象。可以由我们自己实现，在本例中即是 BeforeAdvice
5. BeforeAdvice 实现了 InvocationHandler ，它的 **invoke** 方法处理所有的接口调用。（主要逻辑在 invoke 方法中）

### 2.4 简单 AOP 的代码结构

- MethodInvocation 接口：实现类包含了切面逻辑
- Advice 接口：继承了 InvocationHandler 接口
- BeforeAdvice 类：实现了 Advice 接口，是一个前置通知
- SimpleAOP 类：生成代理类
- SimpleAOPTest 类：测试类
- HelloService 接口：目标对象接口
- HelloServiceImpl类：目标对象

### 2.5 代码

**MethodInvocation 接口代码：**

```java
public interface MethodInvocation {
    void invoke();
}
```

**Advice 接口代码：**

```java
public interface Advice extends InvocationHandler {
}
```

**BeforeAdvice 实现代码：**

```java
public class BeforeAdvice implements Advice {
    private Object bean;
    private MethodInvocation methodInvocation;
    
    public BeforeAdvice(Object bean, MethodInvocation methodInvocation) {
        this.bean = bean;
        this.methodInvocation = methodInvocation;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 在目标方法执行前调用通知
        methodInvocation.invoke();
        return method.invoke(bean, args);
    }
}
```

**SimpleAOP 实现代码：**

```java
public class SimpleAOP {
    public static Object getProxy(Object bean, Advice advice) {
        return Proxy.newProxyInstance(SimpleAOP.class.getClassLoader(),
                bean.getClass().getInterfaces(), advice);
    }
}
```

**HelloService 接口，及其实现类代码：**

```java
public interface HelloService {
    public void sayHello();
}

public class HelloServiceImpl implements HelloService {

    @Override
    public void sayHello() {
        System.out.println("hello");
    }
}
```

**SimpleAOPTest 代码:**

```java
public class SimpleAOPTest {
    public static void main(String[] args) {
        // 1.创建一个 MethodInvocation 实现类
        MethodInvocation logTask = () -> System.out.println("log task start");
        HelloServiceImpl helloService = new HelloServiceImpl();
        
        // 2.创建一个 Advice
        Advice advice = new BeforeAdvice(helloService, logTask);
        
        // 3.为目标对象生成代理
        HelloService proxy = (HelloService) SimpleAOP.getProxy(helloService, advice);
        // 4.使用代理对象运行目标对象的方法
        proxy.sayHello();
    }
}
```

