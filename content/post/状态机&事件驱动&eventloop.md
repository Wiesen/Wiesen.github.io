+++
date = "2018-02-18T21:31:20+08:00"
draft = false
title = "状态机&事件驱动&eventloop"
tags = ["network", "Design Pattern"]
+++

### 状态机编程思想：“致人 而不致于人”

#### 1. 有限状态机

Difination: 有限状态机（finite-state machine）又称有限状态自动机，简称状态机，是表示有限个状态以及在这些状态之间的转移和动作等行为的数学模型。

Application:

1. 自然语言处理：okenizer、parser、regexp
2. 网络协议
3. 游戏设计
4. 自动客服

#### 2. 事件驱动与状态模式

状态模式：

1. motivation： 一个业务如果包含多个阶段，而各个阶段之间又并非顺序执行，可能存在复杂的条件跳转。此时，运用状态模式可以帮助我们理清业务逻辑，并发现一些逻辑缺陷。
1. 定义：状态模式（State Pattern）中，类的行为是基于它的状态改变的。这种类型的设计模式属于行为型模式。
2. 意图：允许对象在内部状态发生改变时改变它的行为，对象看起来好像修改了它的类。
3. 主要解决：对象的行为依赖于它的状态（属性），并且可以根据它的状态改变而改变它的相关行为。
4. 何时使用：代码中包含大量与对象状态有关的条件语句。
5. 如何解决：将各种具体的状态类抽象出来。

状态模式包含三个角色：

1. Context：环境类又称为上下文类，它是拥有状态的对象，在环境类中维护一个抽象状态类State的实例，这个实例定义当前状态，在具体实现时，它是一个State子类的对象，可以定义初始状态；
2. State：抽象状态类，用于定义一个接口以封装与环境类的一个特定状态相关的行为；
3. ConcreteState：具体状态类是抽象状态类的子类，每一个子类实现一个与环境类的一个状态相关的行为，每一个具体状态类对应环境的一个具体状态，不同的具体状态类其行为有所不同。

![state-pattern-uml-diagram](http://7vij5d.com1.z0.glb.clouddn.com/state_pattern_uml_diagram2.jpg)

[事件驱动下的状态模式](http://chng.github.io/blog/2016/05/13/事件驱动下的状态模式/):

1. 基于事件驱动的模型，可以使得系统按照预先定义的方式，在状态机中扭转（进入某个状态时，触发什么事件）。
2. 一个完整的业务处理过程包含多个状态迁移。这就需要一个驱动器来管理每一次的状态迁移。
3. 由于状态的迁移是事件驱动的，于是很容易想到运用观察者模式来设计这样一个驱动器。于是，状态是Observable，由许多个Observer订阅，而每个Observer的任务是，当状态机因为某个事件而跳转到某个状态时，向状态机发送下一个事件的通知，直到状态机到达一个终止状态（在这一模型中，终止状态就是没有任何Observer订阅的状态）。

### eventloop

difination:

1. the event loop, message dispatcher, message loop, message pump, or run loop is a programming construct that waits for and dispatches events or messages in a program.
1. It works by making a request to some internal or external "event provider" (that generally blocks the request until an event has arrived), and then it calls the relevant event handler ("dispatches the event").
2. The event-loop may be used in conjunction with a reactor, if the event provider follows the file interface, which can be selected or 'polled' (the Unix system call, not actual polling).
3. The event loop almost always operates asynchronously with the message originator.

Handling signals:

1. One of the few things in Unix that does not conform to the file interface are asynchronous events (signals). Signals are received in signal handlers, small, limited pieces of code that run while the rest of the task is **suspended**.
2. To avoid heavy handler, ann obvious way to handle signals is for signal handlers to set a global flag and have the event loop check for the flag immediately before and after the select() call; if it is set, handle the signal in the same manner as with events on file descriptors: gives rise to a race condition.
2. The solution arrived at by POSIX is the pselect() call, which is similar to select() but takes an additional sigmask parameter, which describes a signal mask. This allows an application to mask signals in the main task, then remove the mask for the duration of the select() call such that signal handlers are only called while the application is I/O bound: implementations of pselect() have only recently become reliable
3. An alternative, more portable solution, is to convert asynchronous events to file-based events using the self-pipe trick, where "a signal handler writes a byte to a pipe whose other end is monitored by select() in the main program". In Linux kernel version 2.6.22, a new system call signalfd() was added, which allows receiving signals via a special file descriptor.

[everything is a file (descriptor), but](https://www.zhihu.com/question/21040222/answer/96976318):

1. 在 Linux 引入 signalfd 之前，signal 不是文件，你没办法用 poll 来等待 signal，只能用危险的 signal handler。
2. 在 FreeBSD 引入 pdfork 之前，子进程不是文件，你没办法用 poll 来等待子进程退出，只能 wait。也没有办法安全（race-free）地 kill 进程。Linux 不支持 pdfork。
3. 直到现在，线程也不是文件，你没办法用 poll 来等待线程退出，只能 join。Linux 的 clone_fd 似乎能以文件的方式管理子进程和线程，但还没有进入 main stream。
4. Poll 也不能用于磁盘IO，Linux 不支持非阻塞的 file IO, 就算这个文件位于 NFS 上，也只能阻塞地读。
5. 算起来，只有 Sockets 才是真的可 poll 的 “好文件”啊。其实 select 本身也是跟 Sockets API 同时发明的。

### asyncore (Deprecated since version 3.6, develop asyncio instead)

introduction: provides the basic infrastructure for writing asynchronous socket service clients and servers.

[overview: The basic idea behind the asyncore module is eventloop](https://parijatmishra.wordpress.com/2008/01/04/writing-a-server-with-pythons-asyncore-module/)

1. there is a function, asyncore.loop() that does select() on a bunch of ‘channels’. Channels are thin wrappers around sockets, and is maintained in a global map.
2. when select returns, it reports which sockets have data waiting to be read, which ones are now free to send more data, and which ones have errors; loop() examines the event and the socket’s state to create a higher level event;
3. it then calls a method on the channel corresponding to the higher level event.

```
+-------------+           +--------+
| driver code |---------> | loop() |
+-------------+           +--------+
      |                      |
      |                      | loop-dispatcher API (a)
      |                      |
      |                  +--------------+
      |                  | dispatcher   |
      +----------------->| subclass     |
                         +--------------+
                             |
                             | dispatcher-logic API (b)
                             |
                         +--------------+
                         | server logic |
                         +--------------+
```

The asyncore module’s API consists of:

1. the loop() method, to be called by a driver program written by you;
2. the dispatcher class, to be subclassed by you to do useful stuff. The dispatcher class is what is called ‘channel’ elsewhere

The loop-dispatcher API: `loop( [timeout[, use_poll[, map[,count]]]])`

1. When we create a new dispatcher object, it automatically gets added to a
global list of sockets.
2. We can over-ride the list that loop looks at, by providing an explicit map:
    1. we would need to add/remove dispatchers we create to/from this map ourselves
    2. we will be able to launch multiple threads, each calling loop on different maps (thread safe)

Methods a dispatcher subclass should implement:

1. `readable()`: should return True, if you want the fd to be observed for read events;
2. `writable()`: should return True, if you want the fd to be observed for write events;
3. `handle_read`: socket is readable; dispatcher.recv() can be used to actually get the data
4. `handle_write`: socket is writable; dispatcher.send(data) can be used to actually send the data
5. `handle_error`: socket encountered an error
6. `handle_expt`: socket received OOB data (not really used in practice)
7. `handle_close`: socket was closed remotely or locally
8. (For server) `handle_accept:` a new incoming connection can be accept()ed. Call the accept() method really accept the connection. To create a server socket, call the bind() and listen() methods on it first.
9. (For client) `handle_connect`: connection to remote endpoint has been made. To initiate the connection, first call the connect() method on it.

使用经验：

1. client 链接建立问题：`handle_connect()` 在第一次读或写事件到来时被调用，而非建立链接成功马上调用
2. 重连问题：我在使用asyncore时需要实现链接重连，分以下三种情况：
    1. 链接未成功建立 || server socket close || client socket close unexpectly：在 `handle_close` 中 `close` socket，再 `create_socket` 并进行 `connect`（默认即为非阻塞）
    2. 应用层心跳失败令主动重连：接收到应用层发送的重连事件后，调用上述实现的 `handle_close` 进行复用即可。
3. 端口复用问题：`set_resue_addr` tell the OS to set the SO_REUSEADDR flag on the server socket

### 非阻塞 socket 基本原理

#### 1. [非阻塞connect](http://dongxicheng.org/network/non-block-connect-implemention/)

background:

1. TCP连接的建立涉及到一个三次握手的过程，且SOCKET中connect函数需要一直等到客户接收到对于自己的SYN的ACK为止才返回
2. 这意味着每个connect函数总会阻塞其调用进程至少一个到服务器的RTT时间，而RTT波动范围很大，从局域网的几个毫秒到几百个毫秒甚至广域网上的几秒
3. 这段时间内，我们可以执行其他处理工作，以便做到并行。在此，需要用到非阻塞connect

fcntl函数：

1. fcntl函数可执行各种描述符的控制操作，对于socket描述符，常用应用是将其设置为阻塞式IO
2. socket 一旦设置了 timeout, 就进入了 non-blocking 工作模式，原来的 send() 和 recv() 等的用法就完全不同了，可能会只发送或者接收了部分数据，需要检查返回值并多次重试
3. makefile() 是完全不允许使用的，它已经在 socket 模块的文档中明确声明

```c++
int flags;
if((flags = fcntl(fd, F_GETFL)) < 0) //获取当前的flags标志
    err_sys(“F_GETFL error!”); 
flags |= O_NONBLOCK; //修改非阻塞标志位
if(fcntl(fd, F_SETFL, flags) < 0) 
    err_sys(“F_SETFL error!”);
```

connect函数：对于非阻塞式套接字，如果调用connect函数会之间返回-1（表示出错），且错误为EINPROGRESS，表示连接建立启动但是尚未完成；如果返回0，则表示连接已经建立，这通常是在服务器和客户在同一台主机上时发生

```c++
if (connect(fd, (struct sockaddr*)&sa, sizeof(sa)) == -1)
if (errno != EINPROGRESS) {
    return -1;
    }
if(n == 0)
    goto done;
```

select函数：IO多路复用机制

1. connect本身并不具有设置超时功能，如果想对套接字的IO操作设置超时，可使用select函数
2. 对于select和非阻塞connect，注意两点：当连接成功建立时，描述符变成可写；当连接建立遇到错误时，描述符变为即可读，也可写，遇到这种情况，可调用getsockopt函数

实现步骤：

1. 创建socket，并利用fcntl将其设置为非阻塞
2. 调用connect函数，如果返回0，则连接建立(unlikely) ；如果返回-1，检查errno ，如果值为 EINPROGRESS，则连接正在建立
3. 为了控制连接建立时间，将该socket描述符加入到select的可写集合中，采用select函数设定超时
4. 如果规定时间内成功建立，则描述符变为可写；否则，采用getsockopt函数捕获错误信息
5. 恢复套接字的文件状态并返回

#### 2. socket断开检测、socket关闭、socket端口复用

socket断开检测：

1. TCP协议有一个定时器来决定连接是否被异常关闭
2. 但是该超时时间值缺省的情况下会非常长，如果你希望尽快的检查出这种状态改进性能，最好的方法就是在应用程序协议设计的时候引入keepalive（保持连接）机制。

[socket关闭](http://lexuslee.me/2017/09/06/close-socket/)：

1. Python Socket 的 terminate() 只是发送 socket.SHUT_WR 和 socket.SHUT_RD 来关闭通道的读写权限而并没有释放连接句柄；导致了连接已经无法使用，但仍然处于 ESTABLISHED 状态
2. 如果是 DDoS 攻击的话，可能会阻塞住 socket.close() ，导致后续连接未关闭，大量流量进入服务器
3. 比较好的方式是在 socket.close() 之前先调用 socket.terminate() 关闭通道的读写权限，再调用 socket.close()

socket端口复用：

1. 端口还处于于fin_wait_2状态，solaris要4分钟才能关闭，若等不及有以下2种解决方案
2. `s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)`
3. 修改系统fin_wait,time_wait的时间设置

#### 3. getsockopt/setsockopt

`int getsockopt(int s, int level, int optname, void *optval, socklen_t *optlen)`

1. 可获取影响套接字的选项，比如SOCKET的出错信息
2. 当函数成功时返回0。当发生错误时会返回-1，而错误原因会存放在外部变量errno中
3. 协议层参数指明了我们希望访问一个选项所在的协议栈。通常我们需要使用下面中的一个
    1. SOL_SOCKET来访问套接口层选项
    2. SOL_TCP来访问TCP层选项
4. 参数optname为一个整数值。在这里所使用的值首先是由所选用的level参数来确定的

```c++
err = 0;
errlen = sizeof(err);
if (getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &errlen) == -1) {
    sprintf("getsockopt(SO_ERROR): %s", strerror(errno)));
    close(fd);
    return ERR;
    }
if (err) {
    errno = err;
    close(fd);
    return ERR;
}

    协议层       选项名字
SOL_SOCKET   SO_REUSEADDR
SOL_SOCKET   SO_KKEPALIVE
SOL_SOCKET   SO_LINGER
SOL_SOCKET   SO_BROADCAST
SOL_SOCKET   SO_OOBINLINE
SOL_SOCKET   SO_SNDBUF
SOL_SOCKET   SO_RCVBUF
SOL_SOCKET   SO_TYPE
SOL_SOCKET   SO_ERROR
SOL_TCP       SO_NODELAY
```

`int setsockopt(int s, int level, int optname, const void *optval, socklen_t optlen)`

1. 选项改变所要影响的套接字s
2. 选项的套接口层次level
3. 要设置的选项名optname
4. 指向要为新选项所设置的值的指针optval
5. 选项值长度optlen
