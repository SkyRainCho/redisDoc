# Redis中一个基础的事件驱动库

## 事件驱动库的基础数据结构定义
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
    long when_sec;                        /*触发时间秒数*/
    long when_ms;                         /*触发时间毫秒*/
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

在`aeApiPoll`函数中：
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

## 事件循环接口函数定义

基础操作接口
|函数定义|函数含义|
|-------|------|
|`aeEventLoop *aeCreateEventLoop(int setsize)`|创建一个事件循环`aeEventLoop`|
|`void aeDeleteEventLoop(aeEventLoop *eventLoop)`|删除事件循环，释放对应时间所占的空间|
|`void aeStop(aeEventLoop *eventLoop)`|设置`aeEventLoop.stop`的停止标记为1|
|`int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,...)`|在给定的`aeEventLoop`找那个创建对应的文件事件|
|`void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)`|从给定的事件循环中删除指定文件描述符的文件事件|
|`int aeGetFileEvents(aeEventLoop *eventLoop, int fd)`|根据文件描述符，找出对应的文件的事件|
|`long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,...)`|在事件循环中添加一个多少毫秒后被触发的时间事件|
|`int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)`|从事件循环中删除对应时间事件|
|`int aeProcessEvents(aeEventLoop *eventLoop, int flags)`|处理事件循环中的事件|
|`int aeWait(int fd, int mask, long long milliseconds)`|让某个事件等待|
|`void aeMain(aeEventLoop *eventLoop)`|事件循环执行主程序|
|`char *aeGetApiName(void)`||
|`void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep)`||
|`void aeSetAfterSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *aftersleep)`||
|`int aeGetSetSize(aeEventLoop *eventLoop)`|获取事件循环的大小|
|`int aeResizeSetSize(aeEventLoop *eventLoop, int setsize)`|调整事件循环的大小|

其中`aeMain`函数是整个事件循环的主体：
```c
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        //如果调用aeSetBeforeSleepProc设置了beforesleep回调，那么在启动一次事件循环开启时，会执行beforesleep调用
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        //启动一次事件循环，处理文件事件以及时间事件，同时会尝试调用aftersleep
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
```
而`aeProcessEvents`函数是处理一次事件循环的主体，
处理每个待处理的时间事件，然后处理每个待处理的文件事件（可以通过刚刚处理的时间事件回调来注册）。 
没有特殊标志的情况下，该函数将一直休眠，直到引发某些文件事件或下次发生事件（如果有）时为止。
* 如果`flags`为0，则该函数不执行任何操作并返回。
* 如果`flags`设置了`AE_ALL_EVENTS`，则处理所有类型的事件。
* 如果`flags`设置了`AE_FILE_EVENTS`，则处理文件事件。
* 如果`flags`设置了`AE_TIME_EVENTS`，则将处理时间事件。
* 如果`flags`设置了`AE_DONT_WAIT`，则该函数将尽快返回直到所有。
* 如果`flags`设置了`AE_CALL_AFTER_SLEEP`，则将调用`aftersleep`回调，这些事件可能无需等待就可以处理。

该函数返回已处理事件的数量。

其基本流程为：
1. 确定`aeApiPoll`等待的`timeval`，而这段逻辑可以保证，在即使没有文件事件触发的情况下，
`aeApiPoll`仍然可以在最近的一次计时器时间事件到来之前返回，
而不会错过对后续时间事件的处理。其具体逻辑为：
```c
    //如果设置了AE_TIME_EVENTS标记，同时没有设置AE_DONT_WAIT，搜索最近的时间事件
    if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
        shortest = aeSearchNearestTimer(eventLoop);
    if (shortest) {
        //通过shortest这个时间事件，来计算等待的timeval
        long now_sec, now_ms;
        aeGetTime(&now_sec, &now_ms);
        ...
    } else {
        if (flags & AE_DONT_WAIT) {
            //如果设置了AE_DONT_WAIT，那么设置timeval为0，aeApiPoll立即返回
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        } else {
            //一直等待，知道有事件触发
            tvp = NULL; /* wait forever */
        }            
    }
```

2. 调用封装的`aeApiPoll`多路复用API，等待事件触发，或者超时，或返回触发的时间数量：
```c
    numevents = aeApiPoll(eventLoop, tvp);
```
3. 如果设置了`AE_CALL_AFTER_SLEEP`标记，同时通过调用`aeSetAfterSleepProc`设置了`aftersleep`，
那么会在`aeApiPoll`唤醒返回后，执行`aftersleep`逻辑：
```c
    if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
        eventLoop->aftersleep(eventLoop);
```
4. 由于在aeApiPoll接口中已经将从内核返回的就绪事件拷贝到了事件循环中的fired就绪事件列表中了，
后续会循环处理所有的就绪事件，通常情况下，我们会先处理可读事件，然后执行可写事件的处理逻辑，
这在类似于执行查询操作后立即反馈查询结果这样的场景中
很有用。但是如果在flags掩码中设置了AE_BARRIER标记，那么*Redis*会要求我们将执行顺序翻转过来，也就是在可读事件之后，
绝不触发可写事件的处理。例如当我们需要在响应客户端请求之前，通过调用`beforeSleep`执行类似同步文件到磁盘的操作时，会很用：
```c
    for (j = 0; j < numevents; j++) {
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
        ...
        int invert = fe->mask & AE_BARRIER;
        if (!invert && fe->mask & mask & AE_READABLE) {
            //不需要翻转操作时，调用rfileProc处理文件描述符上的可读事件
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
        }
        if (fe->mask & mask & AE_WRITABLE) {
            if (!fired || fe->wfileProc != fe->rfileProc) {
                //处理文件描述符上的可写事件
                fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
        }
        if (invert && fe->mask & mask & AE_READABLE) {
            if (!fired || fe->wfileProc != fe->rfileProc) {
                //需要翻转操作时，在处理完可写事件后，处理读事件
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
        }
    }
```

5. 调用`processTimeEvents`来处理事件事件
```c
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);
```

processTimeEvents函数，则会处理事件循环之中的由定时器触发的时间事件。