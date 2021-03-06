## 一、自动内存管理



### 1.1 Java 内存区域



#### 1.1.1 运行时数据区域

![](images/Java内存区域.jfif)

- 程序计数器
  - **分支、循环、跳转、异常处理、线程恢复等基础功能**都需要依赖这个计数器来完成
- Java 虚拟机栈
  - 每个方法被执行的时候，Java 虚拟机都会同步创建一个栈帧（Stack Frame）用于**存储局部变量表、操作数栈、动态连接、方法出口等信息**。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。
- 本地方法栈
  - 同虚拟机栈
  - 为虚拟机使用到的本地（Native）方法服务
- Java 堆
  - 存储 Java 对象的实例。
- 方法区
  - 用于**存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据**。
  - 运行时常量池：用于**存放编译期生成的各种字面量与符号引用**，这部分内容**将在类加载后存放到方法区的运行时常量池中**。
- 直接内存
  - 本机直接内存的分配不会受到 Java 堆大小的限制，但是会受到**本机总内存（包括物理内存、SWAP分区或者分页文件）大小以及处理器寻址空间**的限制

#### 1.1.2 HotSpot 虚拟机对象探秘



### 1.2 垃圾收集器和内存分配策略



### 1.3 虚拟机性能监控、故障处理工具