# Redis中一个基础的事件驱动库

## 事件驱动库的基础定义
首先在*Redis*中注册了四种时间类型:
|事件|描述|
|---|----|
|`#define AE_NONE 0`|没有事件被注册|
|`#define AE_READABLE 1`|当描述符可读时触发|
|`#define AE_WRITABLE 2`|当描述符可写时触发|
|`#define AE_BARRIER 4`|如果在同一个事件循环迭代中，如果有读事件触发，那么即使可写也不触发该时间|

在*Redis*中，定了两种事件类型，分别是文件事件与时间事件：
```c
/*文件事件结构体*/
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
    aeFileProc *rfileProc;    //读方法
    aeFileProc *wfileProc;    //写方法
    void *clientData;         //客户端数据
} aeFileEvent;
/*时间事件结构体*/
typedef struct aeTimeEvent {
    long long id;                         /*时间事件ID*/
    long when_sec;                        /*时间秒数*/
    long when_ms;                         /*时间毫秒*/
    aeTimeProc *timeProc;                 /*时间事件中的处理函数*/
    aeEventFinalizerProc *finalizerProc;  /*事件最终被删除时的处理函数*/
    void *clientData;                     /*客户端数据*/
    struct aeTimeEvent *prev;             /*双向链表中下一个时间事件*/
    struct aeTimeEvent *next;             /*双向链表中上一个时间事件*/
} aeTimeEvent;
/*触发事件结构体，用于表示将要被处理的文件事件*/
typedef struct aeFiredEvent {
    int fd;        /*文件描述符*/
    int mask;      /*事件掩码*/
} aeFiredEvent;
```

对于事件循环的基本数据类型
```c
typedef struct aeEventLoop {
    int maxfd;                   /*目前创建的最高的文件描述符*/
    int setsize;                 /*用于定义最多可以被追踪的描述符*/
    long long timeEventNextId;   /*下一个时间事件ID*/
    time_t lastTime;             /*用于探测系统时钟偏移*/
    aeFileEvent *events;         /*已被注册的事件*/
    aeFiredEvent *fired;         /*已经触发的事件*/
    aeTimeEvent *timeEventHead;
    int stop;                    /*事件停止标志符*/
    void *apidata;               /*用于Poll的API调用的指定数据*/
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```

基础操作接口
|函数定义|函数含义|
|-------|------|
|`aeEventLoop *aeCreateEventLoop(int setsize)`|创建一个事件循环`aeEventLoop`|
|`void aeDeleteEventLoop(aeEventLoop *eventLoop)`|删除事件循环，释放对应时间所占的空间|
|`void aeStop(aeEventLoop *eventLoop)`|设置`aeEventLoop.stop`的停止标记为1|
|`int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,...)`|在给定的`aeEventLoop`找那个创建对应的文件事件|
|`void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)`||
|`int aeGetFileEvents(aeEventLoop *eventLoop, int fd)`||
|`long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,...)`||
|`int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)`||
|`int aeProcessEvents(aeEventLoop *eventLoop, int flags)`||
|`int aeWait(int fd, int mask, long long milliseconds)`||
|`void aeMain(aeEventLoop *eventLoop)`||
|`char *aeGetApiName(void)`||
|`void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep)`||
|`void aeSetAfterSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *aftersleep)`||
|`int aeGetSetSize(aeEventLoop *eventLoop)`||
|`int aeResizeSetSize(aeEventLoop *eventLoop, int setsize)`||