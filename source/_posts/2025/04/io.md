---
title: IO
date: 2025-04-13 17:30:35
tags:
---

IO, 全称Input/Output，是信息处理系统与外部世界进行通信的过程[^1]。

## 概念

IO中经常会出现两对概念，阻塞/非阻塞与同步/异步，他们既有关联，又不完全一样。

+ 阻塞/非阻塞：阻塞指的是在资源（数据、锁等）未就绪时，当前执行单元（进程、线程）进入休眠状态（Sleeping State）[^2]，直到资源就绪后，操作系统会唤醒当前执行单元继续往下执行。非阻塞是即使在资源未就绪的时候，执行单元也不会进入休眠状态，因此可以执行其他任务。

思考：Java的线程在等待IO时，是什么状态？

+ 同步/异步：同步/异步是和控制权相关的概念，同步就是面向过程，按代码顺序执行，控制权一直在当前函数中。异步则是面向事件，函数内部分代码在事件发生之后才执行，执行这部分代码的时候，控制权在外边容器中（如Event Loop、 操作系统等），通常由回调函数实现。

所以，同步的代码可以是阻塞的也可以是非阻塞的，但是异步的代码一定是非阻塞的。


## IO的分类

IO主要被分为两个阶段：

1. 数据准备：等待数据从外部设备进入内核缓冲区。
2. 数据复制：从内核缓冲区复制数据到用户空间。

根据上面两个阶段是否阻塞，IO可以分为5种模型。首先根据阶段2是否阻塞可以将IO模型分为同步IO和异步IO，然后同步IO又可以根据阶段1是否阻塞和如何阻塞分为阻塞IO、非阻塞IO、IO多路复用、信号驱动IO等4种模型。

![](image.png)

### 同步IO

+ 阻塞IO：应用程序针对**一个**IO发起操作（read/recv）后，执行单元进入休眠状态，无法执行其他任务，操作系统在完成阶段1和阶段2后唤醒当前执行单元。
+ 非阻塞IO：应用程序针对**一个**IO发起操作（read/recv）后，操作系统根据阶段1的状态返回错误或者进入阶段2，在阶段2中执行单元还是会进入休眠状态。
+ IO多路复用：应用程序针对**多个**IO发起请求（select）后，如果所有被请求的IO都没有就绪，和阻塞IO一样，执行单元进入休眠状态；当其中任意一个IO就绪后，执行单元被唤醒，由应用程序再次发起针对就绪IO的操作（read/recv）。
  + 根据IO被设置为阻塞或者非阻塞模式，就绪的定义不一样，在阻塞模式下，只要有一个字节的数据可以读取，执行单元就被唤醒，后续的read/recv仍就可能被阻塞（阶段1），而在非阻塞（Non-blocking）模式下，select会确保IO完成阶段1后才唤醒执行单元[^3][^4]。
+ 信号驱动IO：应用程序针对**一个**IO向操作系统注册一个信号处理函数，其后可以执行其他任务，不会进入休眠状态。当操作系统完成阶段1后，会给应用程序发送SIGIO信号[^5]，由应用程序再次发起对这个IO的操作（read/recv）。
  + 虽然信号驱动IO被划分为同步IO，但是在信号驱动IO模型中，阶段1的操作显然是一个异步操作。

虽然在同步IO中阶段2是阻塞的，但是从内核缓冲区复制数据到用户空间的速度是很快的。

### 异步IO
+ 异步IO：应用程序针对**一个**IO发起操作（aio_read）后，其后可以执行其他任务，不会进入休眠状态。当操作系统完成阶段1和阶段2后，会给应用程序发送信号[^6]。与信号驱动IO不同的是，这时候操作系统已经完成阶段2了。

## IO多路复用

上述的这些模型中，实际应用最多的是IO多路复用（I/O Multiplexing）。

### Reactor

我们前面介绍了IO多路复用有2个步骤，第一个步骤是请求多个IO，第二个步骤是针对其中就绪的IO进行操作。基于这个模型，Douglas C. Schmidt提出了Reactor范式[^7]，把IO多路复用的2个步骤抽象为了Reactor（Dispatcher）和Handler两个组件。

+ Reactor负责监听和分发IO事件，比如就绪
+ Handler负责IO事件的处理，比如读写

Reactor和Handler分别可以有一个或者多个，在不同语言中，可以使用进程或者线程来实现相应的组件。总共有以下四种组合：

+ 单Reactor单Handler

  ![](srsh.png)

  Reactor通过系统指令select监听多个IO的事件，收到事件后根据事件类型分发给Acceptor和Handler。Acceptor是Handler的一个特例，负责接受新的连接。

  + 优点：
    + 实现简单，可维护性高
    + 无上下文切换开销，无资源竞争带来的开销
  + 缺点：
    + 无法利用多核CPU
    + 如果某个Handler耗时较高，则会影响整体性能
  + 案例：Redis、Node.js

+ 单Reactor多Handler

  ![](srmh.png)

  针对单Reactor单Handler存在的问题，可以把Handler设计为用多进程/线程来处理IO事件。

  优缺点正好和单Reactor单Handler相反。

+ 多Reactor单Handler（没有实际应用）
+ 多Reactor多Handler

  ![](mrmh.png)

  在单Reactor多Handler的实现中，由多进程/线程实现的Handler可以充分利用多核CPU的性能。同样地Reactor也可以被设计为使用多进程/线程来提升性能。主Reactor负责监听IO连接事件，建立连接后这个IO被交给子Reactor监听操作事件。

  + 优点：
    + 相较于单Reactor多Handler可以进一步利用多核CPU提升性能
  + 案例：Netty（线程）、Nginx（进程）

## References

[^1]: <https://en.wikipedia.org/wiki/Input/output>
[^2]: <https://www.baeldung.com/linux/process-states>
[^3]: <https://docs.oracle.com/javase/8/docs/api/java/nio/channels/SocketChannel.html>
[^4]: <https://stackoverflow.com/questions/18401428/how-to-check-whether-read-system-call-has-read-entire-data-or-not>
[^5]: <https://pdai.tech/md/java/io/java-io-model.html>
[^6]: <https://twdev.blog/2024/12/asyncio/>
[^7]: <https://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf>
[^8]: <https://xiaolincoding.com/os/8_network_system/reactor.html>
