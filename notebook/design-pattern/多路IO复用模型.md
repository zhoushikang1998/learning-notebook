**多路复用的本质是同步非阻塞I/O，多路复用的优势并不是单个连接处理的更快，而是在于能处理更多的连接。**



## Linux

目前支持多路复用的系统调用有：select、poll、epoll。

select()，结合了轮询和阻塞两种方式，每次有一个对象事件发生的时候，select() 只是知道有事件发生了，不知道具体是哪个对象发生的，需要从头到尾轮询一遍，复杂度是 O(n)。

poll 函数相对 select 函数变化不大，只是提升了最大的可轮询的对象个数。

epoll 函数把时间复杂度降到 O(1)。

### select

- 缺陷
  - 每次调用`select`，都需要把待监控的 fd 集合从用户态拷贝到内核态，当 fd 很大时，开销很大。
  - 每次调用`select`，都需要轮询一遍所有的 fd，查看就绪状态，当 fd 很大时，开销很大。
  - `select`支持的最大文件描述符数量有限，默认是 1024。

### poll

- 与 select 轮询很像，相对于`select`，`poll`已不存在最大文件描述符限制。

### epoll

- 针对 select 和 poll 的主要缺点进行了改造。主要包括三个主要函数，`epoll_create`, `epoll_ctl`, `epoll_wait`。

```c
// 创建 epoll 句柄，会占用一个fd值，使用完成以后，要关闭。
int epoll_create(int size);

// epoll_ctl：提前注册好要监听的事件类型，监听事件(文件可写，可读，挂断，错误)。不用每次都去轮询一遍注册的 fd，而只是通过 epoll_ctl 把所有 fd 拷贝进内核一次，并为每一个 fd 指定一个回调函数。

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// epoll_wait：监听 epoll_ctl 中注册的文件描述符和事件，在就绪链表中查看有没有就绪的 fd，不用去遍历所有 fd。相当于直接去遍历结果集合，而且百分百命中，不用每次都去重新查找所有的 fd，用户索引文件的事件复杂度为O(1)。
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

### select & poll & epoll比较

- **每次调用 `select` 都需要把所有要监听的文件描述符拷贝到内核空间一次，fd 很大时开销会很大**。`epoll` 会在 epoll_ctl() 中注册，只需要将所有的 fd 拷贝到内核事件表一次，不用再每次 epoll_wait() 时重复拷贝
- 每次 `select` 需要**在内核中遍历所有监听的 fd**，直到设备就绪；`epoll ` 通过 `epoll_ctl ` 注册回调函数，也需要不断调用 `epoll_wait` 轮询就绪链表，当 fd 或者事件就绪时，会调用回调函数，将就绪结果加入到就绪链表。
- `select` 能监听的文件描述符数量有限，默认是 1024；`epoll` 能支持的 fd 数量是最大可以打开文件的数目，具体数目可以在 `/proc/sys/fs/file-max` 查看
- `select `, `poll `在函数返回后需要查看所有监听的 fd，看哪些就绪，而 epoll 只返回就绪的描述符，所以应用程序只需要就绪 fd 的命中率是百分百。

表面上看 epoll 的性能最好，但是**在连接数少并且链接都十分活跃**的情况下，select 和 poll 的性能可能比 epoll 好，毕竟 **epoll 的通知机制需要很多函数回调**。

select 效率低是因为每次都需要轮询，但效率低也是相对的，也可通过良好的设计改善。

### 为什么 select 慢而 epoll 效率高？

- select() 之所以慢的原因: select() 的参数是一个 FD 数组，意味着每次 select 调用，都是一次新的**注册-阻塞-回调**，每次 select 都要把一个数组从用户空间拷贝到内核空间，内核检测到某个对象状态变化并写入后，再从内核空间拷贝回用户空间，select 再把这个数组读取一遍，并返回。这个过程非常低效。

- epoll 的解决方案相当于是一种对 select() 的算法优化：它把 select() 一个函数做的事情分解成了 3 步，首先 epoll_create() 创建一个 epollfd 对象(相当于一个池子)，然后所有被监听的 fd 通过 epoll_ctl() 注册到这个池子，也就是为每个 fd 指定了一个内部的回调函数（这样，就没有了每次调用时的来回拷贝，用户空间的数组到内核空间只有这一次拷贝）。epoll_wait 阻塞等待。在内核态有一个和 epoll_wait 对应的函数调用，把就绪的 fd，填入到一个就绪列表中，而epoll_wait 读取这个就绪列表，做到了快速返回(O(1))。



## 应用搭配

- IO 多路复用 + 非阻塞 IO（select 函数 + 非阻塞 read）