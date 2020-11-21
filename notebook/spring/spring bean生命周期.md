spring bean 生命周期的四个阶段：

1. 实例化 Instantiation
2. 属性赋值 Populate
3. 初始化 Initialization
4. 销毁 Destruction

### 常用扩展点

#### 第一大类：影响多个 Bean 的接口

实现了这些接口的  Bean 会切入到多个 Bean 的生命周期中。

- BeanPostProcessor
- InstantiationAwareBeanPostProcessor

InstantiationAwareBeanPostProcessor 作用于**实例化**阶段的前后，BeanPostProcessor作用于**初始化**阶段的前后。正好和第一、第三个生命周期阶段对应。如图：

![](E:\学习笔记\Spring\images\springbean生命周期.webp)

InstantiationAwareBeanPostProcessor继承了 BeanPostProcessor接口。

```java
InstantiationAwareBeanPostProcessor extends BeanPostProcessor
```

##### InstantiationAwareBeanPostProcessor 源码分析

主要方法：

- postProcessBeforeInstantiation
- postProcessAfterInstantiation
- postProcessPropertyValues

执行时机（调用点）







#### 第二大类：只调用一次的接口

这一大类接口的特点是功能丰富，常用于用户自定义扩展。

第二大类中可以分为两类：

1. Aware 类型的接口
2. 生命周期接口



##### Aware 类型接口

Aware 类型的接口的作用就是让我们能够拿到 spring 容器中的一些资源。基本都能够见名知意，Aware 之前的名字就是可以拿到什么资源，如 BeanNameAware 可以拿到 BeanName，以此类推。

**调用时机注意**：所有的 Aware 方法都是在初始化阶段之前调用的。