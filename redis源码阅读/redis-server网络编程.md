# redis-server中的网络编程

## 一、理解封装

先来看一个使用epoll的基本Linux网络编程示例：

```C
#include "sys/epoll.h"
#include "sys/socket.h"
#include "sys/types.h"
#include <arpa/inet.h>
#include <assert.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <string.h>
#include <strings.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include "stdio.h"
#include "errno.h"

int setnoblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option|O_NONBLOCK;
    fcntl(fd, F_SETFL,new_option);
    return old_option;
}

void add_fd(int epfd,int fd)
{
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
    setnoblocking(fd);
}

void recv_data(int connfd)
{
    char buf[1024];
    memset(buf, 0, 1024);
    int ret = recv(connfd,buf,1024-1,0);
    if(ret<=0)
    {
        if(EAGAIN == errno || errno==EWOULDBLOCK)
            printf("not blocking, next time read\n");
        else
        {
            close(connfd);
            printf("client socket has been close\n");
        }
    }
    else
        printf("recv data: %s\n",buf);
}

int main(int argc,char* argv[])
{
    if(argc<=2)
    {
        printf("Usage:%s ip_address port_number\n",basename(argv[0]));
        return 1;
    }
    u_int8_t ret;
    const char* ip = argv[1];
    uint16_t port = atoi(argv[2]);

    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(PF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);

    int listenfd = socket(PF_INET,SOCK_STREAM,0);
    ret = bind(listenfd,(struct sockaddr*)&address,sizeof(address));
    assert(ret!=1);
    ret = listen(listenfd, 5);
    assert(ret!=1);
    
    int epfd = epoll_create(1024);
    add_fd(epfd,listenfd);

    while(1)
    {
        struct epoll_event ev[1024];
        int n = epoll_wait(epfd, ev, 1024, -1);
        for(int i=0;i<n;i++)
        {
            if(ev[i].data.fd == listenfd)
            {
                struct sockaddr_in client;
                socklen_t client_len = sizeof(client);
                int connfd = accept(listenfd, (struct sockaddr*)&client, &client_len);
                add_fd(epfd,connfd);
            }
            else if(ev[i].events & EPOLLIN)
            {
                recv_data(ev[i].data.fd);
            }
            else
            {
                printf("something wrong\n");
            }
        }
    }

    close(listenfd);
    return 0;
}
```

---

再来看Redis的server部分初始化：

全局变量：

```c
struct redisServer server;  //Server global state 
```

server可以看作是redis服务端的一个实例，在initServer()函数中对其进行初始化。几个重点看的部分：

```C
server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR); //创建EventLoop，相当于实例化一个Reactor(IO多路复用的封装)
listenToPort(server.port,&server.ipfd)								//创建tcp监听socket fd
createSocketAcceptHandler(&server.ipfd, acceptTcpHandler) 		    //创建监听socket的连接事件处理器（如果是epoll，本质上是调用epoll_ctl）
调用-->aeCreateFileEvent(server.el, sfd->fd[j], AE_READABLE, accept_handler,NULL) 
    调用-->(aeApiAddEvent(eventLoop, fd, mask) 调用-->epoll_ctl(state->epfd,op,fd,&ee)
    	//并且，redis以fd直接作为eventLoop->events[fd]的数组下标，这样fd发生事件时可以快速找到相应的数据区，找到回调函数。
```

初始化后，调用aeMain(server.el);不停的进行循环

```C
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```

```C
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */ //如Linux中的epoll
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```

---

对比Redis的server和使用epoll的基本Linux网络编程示例：

示例中的main函数做的事情就是创建监听socket，创建epollfd，将监听socket加入epollfd，当监听socket有事件到来时做相应处理，不停循环epoll_wait。

那么redis中的server全局变量可以看作是示例中main中内容的封装。

而aeEventLoop可以看作是示例中对epoll操作（包括创建epoll，epoll_ctl加入监听事件，以及epoll_wait等有事件发生时做相应处理）的封装。

---

## 二、其他细节

