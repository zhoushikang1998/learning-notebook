AOP 原理：看给容器中注册了什么组件，这些组件什么时候工作，这个组件工作时候的功能

1. @EnableAspectJAutoProxy 是什么？

```
@Import(AspectJAutoProxyRegistrar.class)：给容器导入 AspectJAutoProxyRegistrar 组件
	利用 AspectJAutoProxyRegistrar 自定义给容器中注册 bean ：BeanDefinetion

	internalAutoProxyCreator = AnnotationAwareAspectJAutoProxyCreator
	
给容器中注册一个 AnnotationAwareAspectJAutoProxyCreator
```

2. AnnotationAwareAspectJAutoProxyCreator

![](E:\学习笔记\Spring\images\AnnotationAwareAspectJAutoProxyCreator 的父类和实现的接口.PNG)

关注后置处理器（在 bean 初始化完成前后做事情），自动装配 BeanFactory



 debug 打断点：

AbstractAutoProxyCreator.setBeanFactory();

AbstractAutoProxyCreator. 有后置处理器的逻辑的方法



AbstractAdvisorAutoProxyCreator.setBeanFactory --> initBeanFactory



AnnotationAwareAspectJAutoProxyCreator.initBeanFactory()





流程：

```
1.传入配置类，创建 IOC 容器
2.注册配置类，调用 refresh(); 方法刷新容器
3.registerBeanPostProcessors(beanFactory); 注册 bean 的后置处理器，来方便拦截 bean 的创建
	1.先获取 IOC 容器已经定义了的需要创建对象的所有 BeanPostProcessor
	2.给容器中加别的 BeanPostProcessor
	3.优先注册实现了 PriorityOrdered 接口的 BeanPostProcessor
	4.再给容器中注册实现了 Ordered 接口的 BeanPostProcessor
	5.注册没实现优先级接口的 BeanPostProcessor
	6.注册 BeanPostProcessor，实际上就是创建 BeanPostProcessor 对象， 保存在容器中：
		创建 
		1.创建 Bean 的实例
		2.populateBean ：给 Bean  的各种属性赋值
		3.initializeBean ：  初始化 bean
			1.invokeAwareMethods：处理 Aware 接口的方法回调
			2.applyBeanPostProcessorsBeforeInitialization：应用后置处理器的postProcessBeforeInitialization
			3.invokeInitMethods：执行自定义的初始化方法
			4.applyBeanPostProcessorsAfterInitialization：执行后置处理器的postProcessAfterInitialization
		4.BeanPostProcessor(AnnotationAwareAspectJAutoProxyCreator)创建成功；
	7.把 BeanPostProcessor 注册到 BeanFactory 中：
		beanFactory.addBeanPostProcessor(postProcessor)
```

