# redis-server启动初始化
redis-server入口：server.c中的main()函数

设置OOM（Out of Memory内存用完）的处理函数；

设置随机种子；

判断是否哨兵模式；

```c
server.sentinel_mode = checkForSentinelMode(argc,argv);
```

初始化配置项；

```c
initServerConfig();
```

....总之，各种初始化

----

重点看全局变量：

```c
struct redisServer server;  //Server global state 
```

server可以看作是redis服务端的一个实例，在initServer()函数中对其进行初始化。几个重点看的部分：

```C
server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR); //创建EventLoop，相当于实例化一个Reactor(IO多路复用的封装)
server.db = zmalloc(sizeof(redisDb)*server.dbnum); 					//创建数据库，根据配置项创建对应数量的数据库
for (j = 0; j < server.dbnum; j++) {
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        server.db[j].expires = dictCreate(&dbExpiresDictType,NULL);
        server.db[j].expires_cursor = 0;
        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].ready_keys = dictCreate(&objectKeyPointerValueDictType,NULL);
        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].id = j;
        server.db[j].avg_ttl = 0;
        server.db[j].defrag_later = listCreate();
        listSetFreeMethod(server.db[j].defrag_later,(void (*)(void*))sdsfree);
    }
listenToPort(server.port,&server.ipfd)								//创建tcp监听socket fd
listenToPort(server.tls_port,&server.tlsfd)							//创建tls监听socket fd
server.sofd = anetUnixServer(server.neterr,server.unixsocket,server.unixsocketperm, server.tcp_backlog);//创建unix监听socket fd

aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL)				//创建定时事件
createSocketAcceptHandler(&server.ipfd, acceptTcpHandler) 		    //创建监听socket的连接事件处理器（如果是epoll，本质上是调用epoll_ctl）
调用-->aeCreateFileEvent(server.el, sfd->fd[j], AE_READABLE, accept_handler,NULL) 
    调用-->(aeApiAddEvent(eventLoop, fd, mask) 调用-->epoll_ctl(state->epfd,op,fd,&ee)
    	//并且，redis以fd直接作为eventLoop->events[fd]的数组下标，这样fd发生事件时可以快速找到相应的数据区，找到回调函数。
aeCreateFileEvent(server.el,server.sofd,AE_READABLE, acceptUnixHandler,NULL) //创建Unix socket的可读事件处理器

//...后续还有一些初始化
```

初始化后，调用

```C
aeMain(server.el);
```

不停的进行循环，这是一个"单Reactor单线程"的模型。

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

程序停止循环后，释放相应资源：

```C
aeDeleteEventLoop(server.el);
```

