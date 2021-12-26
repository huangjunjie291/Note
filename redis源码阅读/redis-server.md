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

涉及全局变量：

```c
struct redisServer server;  //Server global state 
static uint8_t dict_hash_function_seed[16]; 
server.el  //aeEventLoop *el;
```

调用

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

