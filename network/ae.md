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
    void *apidata;               /*用于Poll的API调用的数据*/
    aeBeforeSleepProc *beforesleep;    /*每次事件循环中都开始执行的函数*/
    aeBeforeSleepProc *aftersleep;     /**/
} aeEventLoop;
```

基础操作接口
|函数定义|函数含义|
|-------|------|
|`aeEventLoop *aeCreateEventLoop(int setsize)`|创建一个事件循环`aeEventLoop`|
|`void aeDeleteEventLoop(aeEventLoop *eventLoop)`|删除事件循环，释放对应时间所占的空间|
|`void aeStop(aeEventLoop *eventLoop)`|设置`aeEventLoop.stop`的停止标记为1|
|`int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,...)`|在给定的`aeEventLoop`找那个创建对应的文件事件|
|`void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)`|从给定的事件循环中删除指定文件描述符的文件事件|
|`int aeGetFileEvents(aeEventLoop *eventLoop, int fd)`|根据文件描述符，找出对应的文件的事件|
|`long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,...)`|在事件循环中添加时间事件|
|`int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)`|从事件循环中删除对应时间事件|
|`int aeProcessEvents(aeEventLoop *eventLoop, int flags)`|处理事件循环中的事件|
|`int aeWait(int fd, int mask, long long milliseconds)`|让某个事件等待|
|`void aeMain(aeEventLoop *eventLoop)`|事件循环执行主程序|
|`char *aeGetApiName(void)`||
|`void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep)`||
|`void aeSetAfterSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *aftersleep)`||
|`int aeGetSetSize(aeEventLoop *eventLoop)`|获取事件循环的大小|
|`int aeResizeSetSize(aeEventLoop *eventLoop, int setsize)`|调整事件循环的大小|

## IO多路复用接口
*Redis*根据系统所提供的支持，会使用最优的IO多路复用的实现接口：
|源文件|IO多路复用实现|
|-----|------------|
|*src/ae_epoll.c*|对于实现了`epoll`接口的系统，使用`epoll`作为IO多路复用的接口。|
|*src/ae_kqueue.c*|对于*Mac OS*系统，使用`kqueue`作为IO多路复用的接口。|
|*src/ae_select.c*|对于老的系统，使用`select`作为IO多路复用的接口。|

而在事件循环的数据结构中：
```c
typedef struct aeEventLoop {
    ...
    void *apidata;               /*用于Poll的API调用的数据*/
    ...
} aeEventLoop;
```
这个`apidata`字段会根据系统使用的IO多路复用接口的不同，保存所需要的数据结构。
对于使用`epoll`的接口，`apidata`这个字段所保存的数据结构为：
```c
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;   /* Epoll events */
  epoll_data_t data;    /* User data variable */
} __attribute__ ((__packed__));

typedef struct aeApiState {
    int epfd;
    struct epoll_event *events;
} aeApiState;
```
在这些源文件中，定义了一组静态函数，用于封装系统的IO复用接口，供*Redis*事件循环调用：
|函数声明|定义(以`epoll`系统调用为例)|
|-------|---|
|`static int aeApiCreate(aeEventLoop *eventLoop)`|通过调用`epoll_create`创建`epoll`的文件描述符，同时分配`epoll_event`事件队列|
|`static int aeApiResize(aeEventLoop *eventLoop, int setsize)`|调整事件循环中`epoll_event`队列的大小|
|`static void aeApiFree(aeEventLoop *eventLoop)`|释放事件循环中，为`epoll`的相关资源|
|`static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask)`|通过调用`epoll_ctl`系统调用，向内核注册监听一个文件描述符，或者更新一个文件描述符的监听事件|
|`static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask)`|通过调用`epoll_ctl`系统调用，删除一个已监听文件描述符上的事件，或者删除一个已监听的事件描述符|
|`static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp)`|该组静态函数的核心，通过调用`epoll_wait`等待监听的文件描述符上事件触发|

在`aeApiPoll`函数中
```c
    //阻塞在epoll_wait，等待事件ready
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;
            //将从内核中返回的就绪事件提取出出来
            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            //将内核中返回的epoll_event上的文件描述符以及就绪事件拷贝到事件循环aeFiredEvent队列中
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
```

## 事件循环创建、设置相关接口