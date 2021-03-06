# 1 Java并发基础

## 1. 什么是线程和进程？

- 进程是程序的一次执行过程，是系统运行程序的基本单位，因此进程是动态的。系统运行一个程序即是一个进程从创建、运行到消亡的过程。

- 线程与进程相似，但线程是比进程更小的执行单位。一个进程执行过程中可以产生多个线程。与进程不同的是同类的多个线程共享进程的**堆**和**方法区**（元空间）资源，但每个线程有自己的程序计数器、虚拟机栈和本地方法栈，所以系统在产生一个线程，或是在多个线程之间切换工作时，负担要比进程小得多。因此，线程也被称为轻量级进程。

## 2.简要描述线程与进程的关系、区别和优缺点

### 2.1 从JVM 角度说进程和线程之间的关系

一个进程可以有多个线程，多个线程共享进程的**堆** 和**方法区**（元空间）资源，但是每个线程有自己的程序计数器、虚拟机栈和本地方法栈。

**总结**：

关系：线程 是 进程 划分成的更小的运行单位。

区别：线程和进程最大的不同在于基本上**各进程是独立的**，而各线程则不一定，因为同一进程中的线程共享堆和元空间，极有可能相互影响。

优缺点：线程执行开销小，但不利于资源的管理和保护；进程则相反。

### 2.2 程序计数器为什么是私有的

程序计数器主要有以下两个功能：

1. 字节码解释器通过改变程序计数器来一次读取指令，从而实现代码的流程控制，如：顺序执行，选择，循环，异常处理。
2. 在多线程的情况下，程序计数器用于**记录当前线程执行的位置**，从而当线程被切换回来的时候能够知道该线程上次运行到哪里。

所以，程序计数器私有主要是为了：**线程切换后能恢复到正确的执行位置**。

### 2.3 虚拟机栈和本地方法栈为什么是私有的？

- **虚拟方法栈**：每个 Java 方法在执行的同时会创建一个栈帧用于存储局部变量表、操作数栈、常量池引用等信息。从方法调用直到执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中入栈和出站的过程。
- **本地方法栈**：和虚拟机栈的作用相似，区别是：**虚拟机栈为虚拟机执行 Java 方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的 native 方法服务。

所以，为了**保证线程中的局部变量不被别的线程访问到**，虚拟机栈和本地方法栈是私有的。

### 2.4 了解堆和方法区

堆和方法区是所有线程共享的资源，其中堆是进程中最大的一块内存，主要用于存放新创建的对象（所有对象都在这里分配内存），方法区主要用于存放已被加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

## 3.为什么要使用多线程？

- 充分利用多 CPU 的计算能力，单线程只能利用一个 CPU，使用多线程可以利用多 CPU 的计算能力。

- 充分利用硬件资源，CPU 和硬盘、网络是可以同时工作的，一个线程在等待网络 IO 的同时，另一个线程完全可以利用CPU，对于多个独立的网络请求，完全可以使用多个线程同时请求。

## 4.使用多线程可能带来的问题

1. 内存泄漏：
2. 上下文切换的成本
3. 死锁
4. 受限于硬件和软件的资源限制问题

## 5.线程的生命周期和状态

Java线程在运行的生命周期中的指定时刻只可能处于下面 6 种状态的其中一个状态。

|   状态名称   |                             说明                             |
| :----------: | :----------------------------------------------------------: |
|     NEW      |      初始状态，线程被构建，但是还没有调用 start（）方法      |
|   RUNNABLE   | 运行状态，Java线程将操作系统中的就绪和运行两种状态笼统地称作“运行中” |
|   BLOCKED    |                  阻塞状态，表示线程阻塞于锁                  |
|   WAITING    | 等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作（通知或中断） |
| TIME_WAITING | 超时等待状态，该状态不同于 WAITING ，它是可以在指定的时间自行返回的 |
|  TERMINATED  |                终止状态，表示当前线程执行完毕                |

Java 线程状态变迁如下图所示：

![](images/线程的生命周期和状态.png)

## 6.什么是上下文切换

CPU 通过时间片分配算法来循环执行任务，当前任务执行一个时间片后会切换到下一个任务。但是，在切换前会保存上一个任务的状态，以便下次切换回这个任务时，可以再加载这个任务的状态。

**任务从保存到再加载的过程就是一次上下文切换。**上下文切换会影响多线程的执行速度。

### 如何减少上下文切换

较少上下文切换的方法有无锁并发编程、CAS算法、使用最少线程和使用协程。

- 无锁并发编程。多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些办法来避免使用锁，如将数据的 ID 按照 Hash 算法取模分段，不用的线程处理不同段的数据。
- CAS算法。Java的 Atomic 包使用 CAS 算法来更新数据，而不需要加锁。
- 使用最少线程。避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这样会造成大量线程都处于等待状态。
- 协程：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换。 

## 7.什么是线程死锁？如何避免死锁？

### 7.1 什么是死锁

**死锁**：多个线程同时被阻塞，它们中的一个或者全部都在等待某个资源被释放。由于线程被无限期地阻塞，因此程序不可能正常终止。

**产生死锁必须具备的四个条件：**

1. 互斥条件：该资源任意一个时刻只由一个线程占用。
2. 请求与保持条件：一个进程因请求资源而阻塞，对已获得的资源保持不放。
3. 不剥夺条件：线程已获得的资源在未使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才释放资源。
4. 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系。

### 7.2如何避免死锁

破坏产生死锁的四个条件中的其中一个即可：

- **破坏互斥条件**：无法破坏（临界资源需要互斥访问）
- **破坏请求与保持条件**：一次性申请所有的资源
- **破坏不剥夺条件**：占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。
- **破坏循环等待条件**：靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。

## 8.sleep() 方法和 wait() 方法的区别和共同点

- **最主要的区别：sleep方法没有释放锁，而 wait 方法释放了锁。**
- wait 通常被用于线程间交互/通信，sleep 通常被用于暂停执行。
- sleep() 方法执行完成后，线程会自动苏醒。wait() 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 notify() 或者 notifyAll() 方法。或者可以使用 wait(long timeout) 超时后线程会自动苏醒。
- 两者都可以暂停线程的执行。

## 9.为什么调用的是start，执行的却是 run方法？ 

start 表示启动该线程，使其成为一条单独的执行流，操作系统会分配线程相关的资源，每个线程会有单独的程序执行计数器和栈，操作系统会把这个线程作为一个独立的个体进行调度，分配时间片让它执行，执行的起点就是 run 方法。

**总结： 调用 start 方法方可启动线程并使线程进入就绪状态，而 run 方法只是 thread 的一个普通方法调用，还是在主线程里执行。**



# 2 Java并发进阶

## 2.1 synchronized关键字

### 2.1.1 对synchronized关键字的理解

synchronized关键字解决的是多个线程之间访问资源的同步性，synchronized关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。

### 2.1.2 synchronized关键字最主要的三种使用方式

### 2.1.3 说说 JDK1.6 之后的 synchronized 关键字底层做了哪些优化，可以详细介绍一下这些优化吗

###  2.1.4 谈谈 synchronized 和 ReentrantLock 的区别



## 2.2 volatile

volatile 是 Java 虚拟机提供的轻量级的同步机制

### 2.2.1 保证内存可见性

JMM 关于同步的规定：（可见性、原子性、有序性）

1. 线程解锁前，必须把共享变量的值刷新回主内存
2. 线程加锁前，必须读取主内存的最新值到自己的工作内存
3. 加锁解锁是同一把锁



### 2.2.2 不保证原子性

原子性：不可分割，完整性，也即某个线程正在做某个业务时，中间不可以被加塞或者被分割，需要整体完整。要么同时成功，要么同时失败。

丢失了数值写入（出现了写覆盖）



如何解决原子性

1. 加 sync
2. 使用 AtomicInteger

### 2.2.3 禁止指令重排