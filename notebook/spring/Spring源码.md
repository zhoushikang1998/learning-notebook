### Spring概述

![Spring模块关系](.\images\Spring模块关系.png)

对Spring源码有深入研究----->能够对Spring进行二次扩展

### Spring循环依赖

Spring当中的循环依赖怎么解决？

​	Spring默认支持单例循环依赖（怎么证明，关闭）

Spring解决循环依赖的细节------源码



依赖注入的功能——什么时候完成？

​		单例是在初始化的时候完成，原型懒加载（什么时候去拿就什么时候完成）

​		1. 初始化 bean



Spring bean 的生命周期在哪个步骤完成的依赖注入？看源码



- Spring初始化一个 bean

  Spring bean 的产生过程-----------bean是怎么产生的

  Class --------------BeanDefinition-------------Object（bean）



普通类的实例化过程：User.java ----javac----->xxx.class--------new----->Object

spring bean 的实例化过程：包扫描下的各种.java如Order.java ----classLoader----->Order.class---------scan parse----->BeanDefinition的子类（有很多属性来存储Order类的所有信息）-----put map----->map（bean放到map中）----->遍历map，new 出对象放到单例池中（）

扫描——》解析——》调用扩展（实现BeanFactoryPostProcessor接口）——》



最终创建的Spring bean 和.java中的类没有直接联系，可以通过实现 BeanFactoryPostProcessor接口更改



**getSingleton()**

1. 为什么需要从Map中拿？————正在实例化
2. 创建 bean 的时候为什么需要get？



**isPrototypeCurrentlyInCreation(beanName)**