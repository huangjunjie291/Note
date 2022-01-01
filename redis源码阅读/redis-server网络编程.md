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

## 二、信号处理

上面说到，全局变量struct redisServer server; 可以看做整个服务器的封装，服务器启动后需要初始化，在initServer();函数中对其初始化，此函数中一进入便对几个信号做了相关处理。

```c
void initServer(void) {
    int j;

    signal(SIGHUP, SIG_IGN);
    signal(SIGPIPE, SIG_IGN);
    setupSignalHandlers();
    ....
}
```

这里面对**SIGHUP**和**SIGPIPE**做了忽略处理，为什么要忽略这两个信号呢？

首先，先了解**进程组**和**会话**。

- 进程组：

  进程组就是一组相互关联的进程集合，系统中每一个进程都从属于某个进程组。每个进程组有唯一的PGID（Process group ID），PGID一般等于创建进程组的进程PID（Process ID）。每个进程组有一个首领进程（process group leader），除了首领进程外其余的都是子进程。

- 会话：

  会话就是一些进程组的集合，系统中的每个进程组都从属于某个会话。

  一个会话**最多**可以有一个**控制终端**（比如linux中的terminal），该终端为会话中所有的进程组共用。

  一个会话中**只会有**一个**前台进程组**，除此之外都是后台进程组，只有前台进程组才能和终端交互。

  类似于进程组，**会话中也有首领（session leader），也可以叫做控制进程（注意这是个进程，不是组），表示连接终端的进程。**一般就是登入系统的shell进程。

![会话](/home/hjj/Note/redis源码阅读/images/会话.jpg)

> 示例：
>
>  hjj@HuanG>    sleep 50 &
> 	[1] 3280
> hjj@HuanG>     ps j 3280 $$
>    PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
>    1639    1676    1676    1676 pts/0       3288 Ss    1000   0:00 zsh
>    1676    3280    3280    1676 pts/0       3288 SN    1000   0:00 sleep 50

PPID是父进程PID，PID是进程ID，PGID是进程组ID，SID是会话ID。TTY是会话控制终端设备。TPGID是前台进程组ID。

---

**2.1 SIGHUP信号**

以上对进程组和会话有了解后，再来看SIGHUP信号。

SIGHUP信号在终端连接结束（无论正常结束还是异常结束）时发出，通知同一个会话内的各个作业。

- 终端关闭时，SIGHUP信号发送给session首领进程（一般是shell进程）和使用&提交的后台运行进程（如上面示例中的sleep 50 &）。
- session首领进程关闭时，会发送SIGHUP信号给前台进程组的每一个进程。
- 若一个父进程退出了，导致进程组成为孤儿进程组，且该进程组中有进程处于停止状态（收到SIGSTOP或SIGTSTP信号），SIGHUP信号会被发送到该进程组中的每一个进程。

**比如，我们在登录Linux系统后，系统会给我们用户分配一个终端，会话中的首领进程表示连接终端的进程（如shell），我们在这个终端里运行各种程序，有前台进程组，后台进程组，他们都属于这个session。当我们用户登出linux后，前台进程组和后台有对终端输出的进程将会收到SIGHUP信号。而SIGHUP信号的默认操作是终止进程。**

可以用上面的示例再试一下，开启一个终端，输入sleep 50 &，然后把这个终端关闭，这时候再输入ps j 【对应的进程PID】 $$是找不到sleep进程的，它被终止了。所以，我们编写服务器程序，通过shell启用服务程序，肯定不想在shell关闭时将程序关闭而使服务停掉，所以一般使用signal(SIGHUP, SIG_IGN);将SIGHUP信号忽略。

**2.2 SIGPIPE信号**

当往一个写端关闭的管道(pipe)或socket连接中连续写入数据时会引发SIGPIPE信号，并且写操作将设置errno为EPIPE。

在使用TCP的通信中，当通信双方中的一方close时，如果另一方接着发数据，根据TCP协议的规定，先会收到一个RST响应报文，之后如果还向这个socket发送数据时，系统会发出一个SIGPIPE信号给进程，告诉进程这个连接已经断开了，不能再写入数据。

SIGPIPE的默认操作是终止进程，而对于一个服务器程序，我们不想当客户端关闭时，我们仍然向客户端写数据而引发SIGPIPE就是进程关闭停止服务。所以一般使用signal(SIGPIPE,SIG_IGN);将SIGPIPE信号忽略。

---

后面继续看setupSignalHandlers()函数对其他信号的处理。

```C
void setupSignalHandlers(void) {
    struct sigaction act;

    /* When the SA_SIGINFO flag is set in sa_flags then sa_sigaction is used.
     * Otherwise, sa_handler is used. */
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    act.sa_handler = sigShutdownHandler;
    sigaction(SIGTERM, &act, NULL);
    sigaction(SIGINT, &act, NULL);

    sigemptyset(&act.sa_mask);
    act.sa_flags = SA_NODEFER | SA_RESETHAND | SA_SIGINFO;
    act.sa_sigaction = sigsegvHandler;
    if(server.crashlog_enabled) {
        sigaction(SIGSEGV, &act, NULL);
        sigaction(SIGBUS, &act, NULL);
        sigaction(SIGFPE, &act, NULL);
        sigaction(SIGILL, &act, NULL);
        sigaction(SIGABRT, &act, NULL);
    }
    return;
}
```

捕获信号设置处理函数可以直接使用signal()系统调用，如signal(SIGHUP, SIG_IGN);但是sigaction()函数对比与signal()函数有一些新的功能：

```C
#include <signal.h>
int sigaction(int signum, const struct sigaction *act,struct sigaction *oldact);

struct sigaction {
	void (*sa_handler)(int);
	void (*sa_sigaction)(int, siginfo_t *, void *); //sa_sigaction和sa_handler不能同时用
	sigset_t sa_mask;
	int sa_flags;
	void (*sa_restorer)(void);
};
```

- signal只能捕获信号，对信号进行处理，但是不能获取信号的其它信息.
- sigaction可以使用sigaction结构体的sa_handler函数对信号进行处理（此处等同于signal函数），也可以使用sa_sigaction函数查看信号的各种详细信息（第一个参数如同sa_handler函数是个整形入参，第二个参数是指向siginfo结构的指针，第三个参数是指向进程上下文标识符的指针。但是sa_handler只能接收一个函数，这就看出来了使用sa_sigaction可以接收更多信息。）
- sigaction函数还可以通过sa_mask、sa_flags对信号处理时进行很多其他操作

**struct sigaction结构**中的sa_mask是阻塞信号集，如果不填默认是阻塞自己这个信号，但是这里的阻塞指的是，当信号被触发，**正在进行信号处理函数**时，阻塞sa_mask中的信号。注意，这里晋档正在进行信号处理函数时才能阻塞sa_mask中的信号。另外，及时不填写sa_mask也需要调用sigemptyset对其进行初始化。

struct sigaction结构中的sa_flags指定了对信号的特殊处理，如上面代码中的SA_NODEFER代表当信号处理函数执行时，不会自动阻塞（sa_mask中没有这个信号设置阻塞）。SA_RESETHAND表示信号处理函数执行完之后会对该信号恢复默认行为SIG_DFL。SA_SIGINFO表示需要可以对信号处理函数提供额外的两个信息：一个指向siginfo结构的指针以及一个指向进程上下文标识符的指针。所以，上面代码中才会写到SA_SIGINFO使用时需要使用sa_sigaction而不是sa_handler，因为sa_handler只能接收一个入参，这个标志开了能提供更多的信息，只有sa_sigaction()才能接收。



## 三、定时器

redis将定时器以事件的方式进行管理，同样会加入到aeEventLoop中，这样**统一事件源**。

```C
typedef struct aeEventLoop {
	....
    long long timeEventNextId;
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead; //定时事件链表
	....
} aeEventLoop;

```

定时事件结构：

```C
/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    monotime when;  //微秒计数的单调时间
    aeTimeProc *timeProc; //时间对应的处理函数
    aeEventFinalizerProc *finalizerProc; //定时器回收时的处理函数
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
    int refcount; /* refcount引用计数 to prevent timer events from being
  		   * freed in recursive time event calls. */
} aeTimeEvent;
```

创建一个定时事件，将这个定时事件加入到eventLoop中**定时事件双向链表**中：

```C
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    long long id = eventLoop->timeEventNextId++;
    aeTimeEvent *te;

    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
    te->id = id;
    te->when = getMonotonicUs() + milliseconds * 1000; //设置到期时间
    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    te->clientData = clientData;
    te->prev = NULL;
    te->next = eventLoop->timeEventHead;
    te->refcount = 0;
    if (te->next)
        te->next->prev = te;
    eventLoop->timeEventHead = te;
    return id;
}

getMonotonicUs()的实质是：
static monotime getMonotonicUs_posix() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts); //系统运行时间
    return ((uint64_t)ts.tv_sec) * 1000000 + ts.tv_nsec / 1000;
}
```

定时时间何时触发？在eventLoop总循环中，会在处理IO事件后，处理定时事件：

```C
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}

int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    ...
    if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            usUntilTimer = usUntilEarliestTimer(eventLoop); //扫一遍定时事件链表，找出最快要触发的定时事件，目的是设置epoll_wait的最大等待事件，以便定时器																到时间时能够及时从epoll_wait中出来去尽快执行定时任务。
    ...
    numevents = aeApiPoll(eventLoop, tvp);  //IO多路复用找出触发IO事件
    //执行IO事件对应的处理函数
    ...
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop); //执行定时器事件
    return processed; /* return the number of processed file/time events */
}
```

processTimeEvents就是扫描定时事件双向链表，对标记了AE_DELETED_EVENT_ID的定时事件进行删除，对到期事件执行设置的回调函数timeProc，根据执行的回调函数判断是重新设置时间（如循环定时器）等待下次再次触发，还是只执行这一次就够了，标记AE_DELETED_EVENT_ID等待被删除。

```C
/* Process time events */
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;

    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    monotime now = getMonotonicUs();
    while(te) {
        long long id;
        /* Remove events scheduled for deletion. */
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            if (te->refcount) {
                te = next;
                continue;
            }
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;
            if (te->finalizerProc) {
                te->finalizerProc(eventLoop, te->clientData);
                now = getMonotonicUs();
            }
            zfree(te);
            te = next;
            continue;
        }

        if (te->id > maxId) {
            te = te->next;
            continue;
        }

        if (te->when <= now) {
            int retval;
            id = te->id;
            te->refcount++;
            retval = te->timeProc(eventLoop, id, te->clientData); //执行到时的定时事件，执行回调
            te->refcount--;
            processed++;
            now = getMonotonicUs();
            if (retval != AE_NOMORE) {
                te->when = now + retval * 1000;
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
    return processed;
}
```

!一个定时事件示例，在初始化阶段ServerInit中设置了一个定时事件，回调函数是serverCron，回调返回下次触发需要的间隔ms数：

```C
aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL)
```

根据回调函数serverCron()我们可以看到，**在这一个定时事件中其实处理了很多事情，但是每件事情触发的间隔可能并不相同，**这是怎么做的呢？

它对其中的不同任务做了快慢任务的区分，使用了一个宏定义，用来判断是快任务还是慢任务，对于快任务，在定时事件触发时直接执行，对于慢任务，需要等待server.cronloops累积到想要的值才会执行。另外server.hz是一秒钟执行的次数，1000/server.hz即触发间隔(ms)

```C
#define run_with_period(_ms_) if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))

int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ...
    // 快任务，时间间隔 <= 时间事件触发时间间隔，执行。
    run_with_period(100) {
        ...
    }
    ...
    // 慢任务，定时时间间隔比较长的，需要通过 server.cronloops 累加达到长时间间隔，才会执行慢任务。
    run_with_period(5000) {
        ...
    }

    // 每次触发定时事件，都会执行。
    /* We need to do a few operations on clients asynchronously. */
    clientsCron();

    /* Handle background operations on Redis databases. */
    databasesCron();
    ...
    server.cronloops++;
    return 1000/server.hz;  //下一次触发事件间隔,这就相当于了一个循环触发定时器。
}
```

其实这个定时器也可以看做一个基准定时器，我们在自己写代码的时候也可以借助这样一个思想，注册一个定时器事件，假设每10ms触发一次，每次定时到时，对于快任务（如10ms的）直接执行，对于慢任务（如50ms的）让计数累加5次时再执行。

----

网络编程要处理：IO，信号，定时三大类事件。上面就是redis处理三个事件的一个简单概述，与处理框架。
