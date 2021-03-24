---
title: 'Linux C++ 网络编程(一): socket实现简单echo服务'
date: 2020-08-02 00:10:19
tags: 
- C++
- Linux
- 网络编程
- Socket
---

> 本文的前置知识是socket的基本概念，建议不了解socket的话，先去网络上搜索，网上有很多博客或者百科都有介绍socket。

## 1. API介绍

在正式编写代码之前，需要见介绍几个Linux socket编程中常用的几个API，分别是：

- socket
- bind
- listen
- connect
- accept
- send, recv等读写API

### socket

socket函数位于<sys/socket.h>头文件中，其作用是创建一个用于通行的socket

函数原型如下：

```c++
/*
@domain 通信域，其值可以是某个协议族，例如IPV4，IPV6等等
@type socket类型，例如TCP使用SOCK_STREAM
@protocol 协议，指定某种特定的协议。通常情况下设置为0即可
@return 如果创建成功则返回一个大于0的fd，返回-1表示error，并将全局的errorno置位
*/
int socket(int domain, int type, int protocol);
```



### bind

bind函数位于<sys/socket.h>头文件中，其作用是给创建好的socket的一个“名字”，可以指定端口, 地址等。

函数原型：

```c++
/*
@sockfd socket函数返回的fd
@sockaddr socket address, 这是个结构体，通过这个结构体可以指定绑定的端口，协议族等等
@addrlen sockaddr的size
@return 绑定成功则返回0，错误则返回1
*/
int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);
```



### listen

listen函数位于<sys/socket.h>头文件中，其作用是监听socket连接请求，完成这个调用之后，socket就可以开始接受连接了，另外这个函数调用并不会阻塞。

函数原型：

```c++
/*
@sockfd socket函数返回的fd
@backlog 连接队列的长度
@return 成功则返回0，错误则返回1
*/
int listen(int sockfd, int backlog);
```

### connect

connect函数位于<sys/socket.h>头文件中，客户端调用该函数发起于服务端的连接。

```c++
/*
@sockfd socket函数返回的fd
@addr 类型是和bind函数一样的结构体，这里需要指定服务端的地址，端口，以及使用的协议族等等
@addrlen 同bind
@return 成功则返回0，错误则返回1
*/
int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
```

### accept

accept函数位于<sys/socket.h>头文件中，服务端调用该函数等待接受客户端发起的连接，该函数会阻塞当前线程直到有连接进来。

```c++
/*
@sockfd socket函数返回的fd
@addr 这里addr是函数从阻塞状态返回的时候，函数会填入客户端的地址信息
@addrlen 同上
@return 成功则返回一个大于0的值，表示客户端连接的fd,错误则返回-1
*/
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

### send, recv等

send和recv都位于<sys/socket.h>头文件，他们是成对的写读操作，send用于发送数据，recv用于接收数据，两者都会发生阻塞，当阻塞的条件不同，具体可以网上自行搜索学习。

函数原型：

```c++
/*
@sockfd socket函数返回的fd
@buf 待发送数据的buffer
@len buffer len
@flags 标志位，指定一些特殊的标志
*/
ssize_t send(int sockfd, const void *buf, size_t len, int flags);

/*
@sockfd socket函数返回的fd
@buf 接收数据使用的buffer
@len buffer len
@flags 同上
*/
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

还有其他的读写操作，例如sendmsg, recvmsg等就不在这里列出了，本文没有使用到。

上面只是简单介绍的几个API的作用，其实这几个函数远没有那么简单，还存在很多细节，例如sock_addr结构体的定义是怎样的？send函数的flags参数都可以哪些可选项？各自代表的是什么意义？....这些信息可以从linux api文档中找到，在linux控制台中使用man命令即可（例如man connect)。

## 2. 编写服务端

编写socket服务端是有“章法”的，需要如下几个步骤：

1. 调用socket函数创建socket实例。
2. 调用bind绑定刚刚创建的socket实例，指定socket端口地址等信息。
3. 调用listen监听socket连接，调用成功之后该socket才可以接受客户端连接。
4. 调用accept接受客户端连接，该函数会阻塞当前线程直到有新的连接进来。
5. 执行读写操作（业务逻辑）

代码如下（解释在注释中）：

```c++
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <unistd.h>
#include <arpa/inet.h>

#include <iostream>
#include <cstring>
#include <string>


int main() {
	//1. 创建socket
    int listenFd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenFd == -1) {
        std::cerr << "Error socket!!!" << std::endl;
        exit(0);
    }

    //构造sockaddr_in结构体，该结构体的定义在<netinet/in.h>中
    struct sockaddr_in addr;
    //指定协议组，这里指定的是IPV4
    addr.sin_family = AF_INET;
    //指定端口
    addr.sin_port = 8080;
    //INADDR_ANY表示接受任何远程地址的请求
    addr.sin_addr.s_addr = INADDR_ANY;

    //2. 绑定socket
    if (bind(listenFd, (const struct sockaddr*)&addr, sizeof(addr)) == -1) {
        std::cerr << "bind server error!!!" << std::endl;
        exit(0);
    }

    //3. 监听socket，之后可以接受客户端请求
    if (listen(listenFd, 5) == -1) {
        std::cerr << "listen error!!!" << std::endl;
        exit(0);
    }

  	//loop保证服务端一直可用
    while (true) {
        struct sockaddr_in clientAddr;
        socklen_t addrLen = sizeof(clientAddr);
        //接受客户端请求，会阻塞线程直到有客户端请求，返回一个用于表示客户端连接的fd
        int connFd = accept(listenFd, (struct sockaddr*)&clientAddr, &addrLen);
        if (connFd == -1) {
            std::cerr << "accept error!!!" << std::endl;
            exit(0);
        }

        //这里是从clientAddr(accpet函数会填入)中获取client ip，和通用服务端socket无关，业务相关
        char clientIP[INET_ADDRSTRLEN];
        std::memset(clientIP, 0, sizeof(clientIP));

        inet_ntop(AF_INET, &clientAddr.sin_addr, clientIP, INET_ADDRSTRLEN);
        std::cout << "accept client ip : " << clientIP << std::endl;

       
        char buf[4 * 1024];
        while (true) {
            std::memset(buf, 0, sizeof(buf));
            //从客户端接受数据，同样会阻塞直到接收缓冲区中有数据可读，返回读到的数据长度
            int len = recv(connFd, buf, sizeof(buf), 0);
            //如果数据长度为0，通常表示客户端关闭了连接，为了防止服务端crash,需要作一些处理
            if (len == 0) {
                std::cout << "client force close connection!!!" << std::endl;
                break;
            }
            //业务相关，用于友好的关闭连接
            if (std::strcmp(buf, "exit") == 0) {
                std::string bye = "bye!";
                send(connFd, bye.c_str(), sizeof(bye.c_str()), 0);
                break;
            }
            //将数据回传给客户端，也是业务相关的，因为我们这里主要实现的功能是echo
            send(connFd, buf, sizeof(buf), 0);
        }
		//关闭客户端fd,防止资源泄漏
        close(connFd);
    }
    //防止资源泄漏
    close(listenFd);
}
```

## 3. 编写客户端

客户端的编写较服务端容易一些，主要是如下步骤：

1. 创建socket，用于通行。
2. 调用connect发起和服务端的连接，之后和服务端的信息传递就可以通过刚刚创建的socket了。
3. 读写消息。

代码如下：

```c++
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <unistd.h>
#include <arpa/inet.h>

#include <iostream>
#include <cstring>

int main() {
	//1. 创建socket
    int socketFd = socket(AF_INET, SOCK_STREAM, 0);
    if (socketFd == -1) {
        std::cerr << "Error socket!!!" << std::endl;
        exit(0);
    }

    //这里的sockaddr_in需要指定服务端的端口和地址（本例是在本地，所以地址是127.0.0.1)
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = 8080;
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
    //2. 调用connect发起连接
    if (connect(socketFd, (const struct sockaddr*)&addr, sizeof(addr)) == -1) {
        std::cerr << "connect server error!!!" << std::endl;
        exit(0);
    }

    char data[4 * 1024];
    char buf[4 * 1024];
    while (true) {
        memset(buf, 0, sizeof(buf));
        memset(data, 0, sizeof(data));
        //从标准输入读取数据，业务相关（完全可以从其他地方输入，例如文件）
        std::cin >> data;
        //发送数据给服务端
        send(socketFd, data, sizeof(data), 0);

        //接收服务段发送的数据，业务相关，因为本例是一个echo客户端
        int len = recv(socketFd, buf, sizeof(buf), 0);
        //用于优雅的退出
        if (std::strcmp("bye!", buf) == 0) {

            std::cout << buf << std::endl;
            break;
        }

        std::cout << buf << std::endl;
    }
    //关闭fd，防止fd泄漏
    close(socketFd);

}
```



## 小结

本文介绍了socket编程中常用的几个API，并写了一个简单的echo服务，包括服务端和客户端，理论上该服务段是可以适用任何echo客户端的，所以也可以使用一些网络上开源的echo客户端来验证这个小玩具。

之后的文章会进一步拓展这个小程序，例如实现IO多路服务，线程池，事件分发等等，从而进一步提高该程序的功能和性能。