---
title: 'Linux C++ 网络编程(二): io多路复用'
date: 2020-09-20 19:31:18
tags:
- C++
- Linux
- 网络编程
- Socket
- epoll
- poll
- select
---

> 本文所有代码均可以在我的Github上找到，地址：https://github.com/Weiyanyu/budd/tree/master/test

### 1. 理解IO多路复用

> 关于IO多路复用的概念和介绍，网上有很多优秀的资料。本文只是简单谈谈我个人对IO多路复用的理解，更专业，详细的知识建议参考《UNIX 网络编程》。

要理解IO多路复用，私以为首先需要解决3个问题：

- 什么是IO多路复用？
- 为什么需要IO多路复用？
- IO多路复用技术可以带来那些增益？又有哪些不足？

#### 1.1 什么是IO多路复用

在给出IO多路复用的定义之前，我们先提出3个问题：

- 多路是什么意思？
- 复用又是什么意思？复用什么东西？

在Linux下， 多路指的多个文件描述符，在网络编程中，可以从更高的抽象层次理解为多个网络连接，因为Linux中的每一个网络连接都对应一个文件描述符（下文简称fd）。

复用值的是复用某个执行单元，可以是进程（单进程程序），也可以是线程（多线程程序），这取决于程序的架构设计。

将两者结合在一起就是：多个网络连接使用同一个进程或者线程来执行读写操作。

这个定义有些粗糙了，下面是更加详细一些的定义：

IO多路复用是一种同步IO模型，实现一个进程或者线程可以监视多个文件描述符；一旦某个文件描述符就绪，就能够通知应用程序进行相应的读写操作；没有文件描述符就绪时会阻塞进程或者线程，交出cpu。

#### 1.2 为什么需要IO多路复用

同步IO模型有两种：

- 同步阻塞
- 同步非阻塞

同步阻塞模型很容易理解。以read操作为例，当线程调用read时，内核会先从磁盘读取数据到内核空间，再从内核空间拷贝到用户空间返回给用户。在同步阻塞模型下，这整个过程线程都不能干其他事情，即阻塞的含义。（本系列文章第一篇文章就是同步阻塞模型的应用）

而同步非阻塞模型相比于同步阻塞模型来说，则不需要等待上述拷贝数据的整个过程，用户在第一次调用read的时候，如果数据没有准备好（数据可能是准备好的），那么read调用就会返回一个error标志，告知程序此时数据没有准备好，程序可以理解返回去作其他事情，这就是非阻塞。之后程序可能会多次轮询，如果数据准备好了，就会返回对应的数据。使用同步非阻塞模型的一个特点是需要多次轮询，会有一些CPU消耗，但不会阻塞线程，程序的整体吞吐量要由于阻塞模型。

> 关于上述两种模型的更详细介绍可以参考http://www.masterraghu.com/subjects/np/introduction/unix_network_programming_v1.3/ch06lev1sec2.html.

同步阻塞模型下，IO读写操作会阻塞线程，所以程序无法在处理读写操作的时候接受新的读写操作请求。对应到网络编程中，就是线程在处理数据的读写过程中无法接受新的连接。非阻塞模型在这点上会优于阻塞模型，但如上所说，存在CPU轮询的代价。

所以，为了进一步提升性能，我们需要一种非阻塞的，又不需要重复去询问内核数据是否准备好的机制。IO多路复用可以解决这个问题，IO多路复用基于事件，把读完成，可写等IO操作抽象为一个“事件”，当有事件发生的时候，内核会主动通知用户程序来执行相应的IO操作，不需要用户程序主动去询问内核，而没有事件发生的时候，线程可以继续作其他事情。相比于同步非阻塞模型，少了轮询的步骤。

#### 1.3 IO多路复用技术可以带来那些增益？又有哪些不足？

IO多路复用的增益在上面1.2已经有过讨论，其不足主要是：

- IO多路复用依然是同步IO模型，在一些特殊的场景下，吞吐量上还是比异步IO差一些。
- IO多路复用的编程复杂度要高于简单的同步阻塞和同步非阻塞模型。所幸Linux内核提供了select,poll,epoll等系统调用来帮助开发者来运用IO多路复用。

### 2. Linux IO多路复用系统调用select, poll和epoll 

#### 2.1 select

select是最早的IO多路复用实现，下面是select的API描述：

```c++
//1. nfds参数是select需要监听的fd中“值”最大的那个fd+1，和监听的fd数量没有关系，例如现在监听1，18，20三个fd，那么nfds的值就是20 + 1 = 21
//2. readfds即监听读事件的fd列表，类型是fd_set结构体
//3. writefds监听写事件的fd列表
//4. exceptfds即异常fd列表
//5. 超时事件，类型是通用的timeval
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```

select调用需要指定需要监听的读写事件的fd列表，其调用返回的时候（有事件发生）会往readfds，writefds，exceptfds里面写值（把对应的fd置位），用户程序可以通过判断某个fd是否被置位来判断该fd对应的事件是否发生。所以，使用select调用的用户程序还需要在select返回之后遍历fds，而且这个fds的size并不是监听的fd的数量，而是nfds值，这也是select的局限性之一。另一个局限是可监听的fd数量有上限，上限值是FD_SETSIZE，这个可以通过修改宏定义来修改，linux新版本内核中这个值已经很大了。（很多文章说是1024，那是老版本的内核）

下面是select的一个简单使用示例(echo服务):

```c++
const int BUFFER_SIZE = 4 * 1024;

void processConn(int connFd, struct in_addr clientAddr);

int main()
{

    int listenFd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenFd == -1)
    {
        std::cerr << "Error socket!!!" << std::endl;
        exit(0);
    }

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = 8080;
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(listenFd, (const struct sockaddr *)&addr, sizeof(addr)) == -1)
    {
        std::cerr << "bind server error!!!" << std::endl;
        exit(0);
    }

    if (listen(listenFd, 5) == -1)
    {
        std::cerr << "listen error!!!" << std::endl;
        exit(0);
    }

    fd_set readfds;
    fd_set writefds;
    //初始化fd_set
    FD_ZERO(&readfds);
    FD_ZERO(&writefds);
    //置为listenFd
    FD_SET(listenFd, &readfds);

    fd_set tempReadfds;
    fd_set tempWritefds;
    int maxFd = listenFd;
    

    char buffers[1024][BUFFER_SIZE];
    while (true)
    {
		//在循环中使用select必须要重新初始化fd_set
        //reinit
        tempReadfds = readfds;
        tempWritefds = writefds;

        struct sockaddr_in clientAddr;
        socklen_t addrLen = sizeof(clientAddr);

        
        int eventNum = select(maxFd+1, &tempReadfds, &tempWritefds, nullptr, nullptr);
        //判断listenFd在readfds中是否被置位,即判断是否有读事件
        if (FD_ISSET(listenFd, &tempReadfds)) {
            int connFd = accept(listenFd, (struct sockaddr *)&clientAddr, &addrLen);
            if (connFd == -1)
            {
                std::cerr << "accept error!!!" << std::endl;
                exit(0);
            }

            char clientIP[INET_ADDRSTRLEN];
            std::memset(clientIP, 0, sizeof(clientIP));

            inet_ntop(AF_INET, &clientAddr, clientIP, INET_ADDRSTRLEN);
            std::cout << "accept client ip : " << clientIP << std::endl;

            FD_SET(connFd, &readfds);
            maxFd = connFd > maxFd ? connFd : maxFd;
            if (--eventNum == 0) {
                continue; //no need wait other 
            }
        }

        for (int fd = 0; fd <= maxFd; fd++) {
            if (fd == listenFd) continue;
            //判断listenFd在writefds中是否被置位，即判断是否有写事件
            if (FD_ISSET(fd, &tempReadfds)) {
                std::memset(buffers[fd], 0, sizeof(buffers[fd]));
                int len = recv(fd, buffers[fd], sizeof(buffers[fd]), 0);
                //if len equal 0, mean client disconnect, we must add this condition, or else it will cause server crash
                if (len == 0)
                {
                    std::cout << "client force close connection!!!" << std::endl;
                    close(fd);
                    FD_CLR(fd, &readfds);
                    if (maxFd == fd) {
                        maxFd--;
                    }
                    continue;
                }
                FD_SET(fd, &writefds);
            }

            if (FD_ISSET(fd, &tempWritefds)) {
                send(fd, buffers[fd], sizeof(buffers[fd]), 0);
                FD_CLR(fd, &writefds);
            }
        }
    }
}
```

#### 2.2 poll

poll API描述如下：

```c++
//1. fds, 即监听的fd列表，元素是pollfd，一般传入数组，调用结果会设置到pollfd结构体的revents参数里
//2. nfds, 表示监听的fd的数量,注意和select的区别
//3. 超时时间，单位是毫秒
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd
  {
    int fd;			/* File descriptor to poll.  */
    short int events;		/* Types of events poller cares about.  */
    short int revents;		/* Types of events that actually occurred.  */
  };
```

poll与select最大的不同是没有最大fd size的限制，可以监听任意数量的fd（当然是要小于系统的fd数量，且fd过多后性能会下降）。同select一样的地方是依然需要在poll调用返回后遍历fd来检查fd是否被置位。但只需要遍历监听的fd set数量即可。

下面是一个示例程序：

```c++
#include <poll.h>

const int BUFFER_SIZE = 4 * 1024;

void processConn(int connFd, struct in_addr clientAddr);

int main()
{
    int listenFd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenFd == -1)
    {
        std::cerr << "Error socket!!!" << std::endl;
        exit(0);
    }

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = 8080;
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(listenFd, (const struct sockaddr *)&addr, sizeof(addr)) == -1)
    {
        std::cerr << "bind server error!!!" << std::endl;
        exit(0);
    }

    if (listen(listenFd, 5) == -1)
    {
        std::cerr << "listen error!!!" << std::endl;
        exit(0);
    }

    //定义pollfds数组
    struct pollfd pollfds[1024];
    pollfds[0].fd = listenFd;
    //设置监听的事件类型，这里是读
    pollfds[0].events |= (POLLIN | POLLPRI);

    int maxNum = 1;

    for (int i = 1; i < 1024; i++) {
        pollfds[i].fd = -1;
    }


    char buffers[1024][BUFFER_SIZE];
    while (true)
    {

        struct sockaddr_in clientAddr;
        socklen_t addrLen = sizeof(clientAddr);

        int eventNum = poll(pollfds, maxNum, -1);

        for (int i = 0; i < maxNum; i++) {
            //检查是否是读事件
            if (pollfds[i].revents & (POLLIN | POLLPRI)) {
                if (pollfds[i].fd == listenFd) {
                    int connFd = accept(listenFd, (struct sockaddr *)&clientAddr, &addrLen);
                    if (connFd == -1)
                    {
                        std::cerr << "accept error!!!" << std::endl;
                        exit(0);
                    }

                    char clientIP[INET_ADDRSTRLEN];
                    std::memset(clientIP, 0, sizeof(clientIP));

                    inet_ntop(AF_INET, &clientAddr, clientIP, INET_ADDRSTRLEN);
                    std::cout << "accept client ip : " << clientIP << std::endl;

                
                    pollfds[maxNum].fd = connFd;
                    pollfds[maxNum].events |= (POLLIN | POLLPRI);
                    maxNum++;

                    if (--eventNum == 0) {
                        continue; //no need wait other 
                    }
                } else {
                    int fd = pollfds[i].fd;
                    std::memset(buffers[fd], 0, sizeof(buffers[fd]));
                    int len = recv(fd, buffers[fd], sizeof(buffers[fd]), 0);
                    //if len equal 0, mean client disconnect, we must add this condition, or else it will cause server crash
                    if (len == 0)
                    {
                        std::cout << "client force close connection!!!" << std::endl;
                        pollfds[fd].fd = -1;
                        pollfds[fd].events |= 0;
                        close(fd);
                        maxNum--;
                        continue;
                    }
                    pollfds[i].events |= (POLLOUT | POLLWRBAND);
                }
            } else if (pollfds[i].revents & (POLLOUT | POLLWRBAND)) {
                //写事件
                int fd = pollfds[i].fd;
                send(fd, buffers[fd], sizeof(buffers[fd]), 0);
                pollfds[i].events = POLLIN;
            }

        }
    }
    return 0;
}
```

#### 2.3 epoll

epoll的API比较多，还有epoll_create, epoll_ctl等，这里只列出epoll_wait, 其他的API建议使用 man 手册来查看。

```c++
//1. epfd,epoll_create返回的fd
//2. events，事件列表，内含fd
int epoll_wait(int epfd, struct epoll_event *events,
                  int maxevents, int timeout);
   
int epoll_pwait(int epfd, struct epoll_event *events,
                  int maxevents, int timeout,
                  const sigset_t *sigmask);

struct epoll_event
{
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;

typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

epoll比poll更进一步，相比与poll来说，epoll_wait调用返回后不需要轮询fd了，只需要轮询事件数量即可。同时epoll支持两种事件触发模式，边沿触发（ET）和水平触发（LT）。而且epoll使用一个额为的fd(epoll_create创建)来管理多个fd，将用户监听的fd列表存放到内核的一个事件表中，这样的好处是减少用户空间和内核空间拷贝的次数（只需要一次， 其他调用需要拷贝所有的fd）

- 边沿触发。当epoll_wait调用返回的时候，用户程序可以不处理该事件，下次epoll_wait会继续报告此事件，如果一直不处理事件，可能会导致busy loop.
- 水平触发。默认的工作模式，当epoll_wait调用返回的时候，用户程序必须处理该事件，否则会丢失事件。

下面是一个使用epoll的示例程序：

```c++
const int BUFFER_SIZE = 4 * 1024;

void processConn(int connFd, struct in_addr clientAddr);

int main()
{
    int listenFd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenFd == -1)
    {
        std::cerr << "Error socket!!!" << std::endl;
        exit(0);
    }

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = 8000;
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(listenFd, (const struct sockaddr *)&addr, sizeof(addr)) == -1)
    {
        std::cerr << "bind server error!!!" << std::endl;
        exit(0);
    }

    if (listen(listenFd, 5) == -1)
    {
        std::cerr << "listen error!!!" << std::endl;
        exit(0);
    }

    //创建一个epollfd
    int epollfd = epoll_create(1024);

    struct epoll_event event;
    //设置epoll_event
    event.events = EPOLLIN;
    event.data.fd = listenFd;
    //表示epoll新增监听listenFd的epoll_event事件
    epoll_ctl(epollfd, EPOLL_CTL_ADD, listenFd, &event);

    struct epoll_event revents[1024];
    char buffers[1024][BUFFER_SIZE];
    while (true)
    {

        struct sockaddr_in clientAddr;
        socklen_t addrLen = sizeof(clientAddr);

        int eventNum = epoll_wait(epollfd, revents, 1024, -1);
        std::cout << "event num " << eventNum << std::endl;

        for (int i = 0; i < eventNum; i++)
        {
            //返回结果会设置到revents里
            if (revents[i].events & EPOLLIN)
            {
                if (revents[i].data.fd == listenFd)
                {
                    int connFd = accept(listenFd, (struct sockaddr *)&clientAddr, &addrLen);
                    if (connFd == -1)
                    {
                        std::cerr << "accept error!!!" << std::endl;
                        exit(0);
                    }

                    char clientIP[INET_ADDRSTRLEN];
                    std::memset(clientIP, 0, sizeof(clientIP));

                    inet_ntop(AF_INET, &clientAddr, clientIP, INET_ADDRSTRLEN);
                    std::cout << "accept client ip : " << clientIP << std::endl;

                    struct epoll_event newEvent;
                    newEvent.events = EPOLLIN;
                    newEvent.data.fd = connFd;
                    epoll_ctl(epollfd, EPOLL_CTL_ADD, connFd, &newEvent);
                }
                else
                {
                    int fd = revents[i].data.fd;
                    std::memset(buffers[fd], 0, sizeof(buffers[fd]));
                    int len = recv(fd, buffers[fd], sizeof(buffers[fd]), 0);
                    //if len equal 0, mean client disconnect, we must add this condition, or else it will cause server crash
                    if (len == 0)
                    {
                        std::cout << "client force close connection!!!" << std::endl;
                        //删除监听
                        epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, &revents[i]);
                        close(fd);
                        continue;
                    }

                    revents[i].events = EPOLLOUT;
                    //修改监听的事件类型
                    epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &revents[i]);
                }
            }
            else if (revents[i].events & EPOLLOUT)
            {
                int fd = revents[i].data.fd;
                send(fd, buffers[fd], sizeof(buffers[fd]), 0);
                struct epoll_event newEvent;
                newEvent.events = EPOLLIN;
                newEvent.data.fd = fd;
                // epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, &revents[i]);
                epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &newEvent);
            }
        }
    }
    close(listenFd);
    close(epollfd);
    return 0;
}

```

### 3. 小结

本文简单介绍了IO多路复用的概念，需要IO多路复用的理由，使用IO多路复用的好处以及Linux下几个IO多路复用接口的简单使用。

总结以下Linux下select，poll和epoll的特点和区别：

1. select是最古老的实现，有最大监听fd数量的限制，而且需要一些无意义的遍历，因为遍历的size不是fd数量而是最大的fd值加1，导致事件不频繁的时候大量无意义循环。
2. poll虽然看起来没有遍历，实际上内部也是存在遍历。相比于select，最大的优势是没有最大fd的限制，同时编程难度也大于select。
3. epoll使用独立的fd来管理用户fd set，减少内核空间和用户空间的拷贝，不需要遍历所有监听的fd，这是性能上的提升。而且使用支持边沿触发和水平触发两种工作模式，相对更加灵活。但编程复杂度更上一层楼，很容易掉坑里。

碍于篇幅，文中select,poll,epoll的介绍都非常简单（比如没有介绍poll和epoll支持的监听类型等等）， 详细的内容远不至于此，建议参考man手册或者网上一些优秀资料。



