# Redis中一个基础的事件驱动库
*Redis*数据库最大的一个特点便是其具有高并发性，可以每秒处理数万次的并发读写操作。除了使用使用单线程以无锁的方式处理核心数据逻辑以避免由于*锁争用*而带来的性能下降之外，*Redis*使用了基于*I/O多路复用*的事件驱动的方式来处理客户端通过网络发送来的请求。而基于*I/O多路复用的*的事件驱动模式正是一种可以高效处理网络并发请求的编程模式。

我们知道默认被创建套接字文件描述符都是阻塞的，例如`accept`、`connect`、`read`、`write`这些函数在操作阻塞文件描述符时，如果期待事件没有发生，那么函数会将程序阻塞直到有新连接到来、连接建立、有数据可读或者缓冲区现在可写，那么频繁地阻塞程序势必造成程序性能的下降。

通过前面对套接字通用接口一文的介绍，我们可以调用`anetNonBlock`接口将一个套接字文件描述符设置成非阻塞，然而这样会出现一个问题，当我们针对一个非阻塞套接字文件描述符调用`accept`等函数时，这些函数总是会立即返回。显然单纯的非阻塞套接字无法提高网络程序的并发性能。

我们期望当有事件就绪的时候在正对非阻塞套接字文件描述符调用对应的函数接口，这样才能够发挥出非阻塞套接字的特性提升系统的并发性能。例如当有连接已经到来的时候，我们在去调用`accept`函数来接收新连接；当一条与客户端的连接上有数据可读的时候，我们在调用`read`函数来读取数据。而*I/O多路复用*正式提供了这样的一种支持，与`accept`、`connect`、`read`、`write`这些函数只能操作单一文件描述符不同，*I/O多路复用*可以同时监听多个文件描述符，通过`select`、`poll`、`epoll_wait`这些调用阻塞等待多个文件描述符，只要其中有一个文件描述符就绪，函数就会返回，我们可以调用对应的事件处理函数来处理不同的事件，这将大大提高服务器的处理效率。

*Redis*对于基于*I/O多路复用*的事件驱动的声明与定义在*src/ae.h*、*src/ae.c*、*src/ae_epoll.c*、*src/ae_ecport.c*、*src/ae_kqueue.c*和*src/ae_select.c*这些文件之中。

## 事件驱动库的基础数据结构定义与描述
### 枚举类型描述
#### 事件类型
*Redis*中，事件驱动需要关注两种事件的处理：
1. `#define AE_FILE_EVENTS 1`，文件事件，这类事件主要是用于处理服务器与客户端之间的连接，在Linux系统之中”一切皆文件“，而表示一条TCP连接的套接字实际上也是一个文件描述符。
1. `#define AE_TIME_EVENTS 2`，时间事件，这类事件主要用处理服务器的一些需要周期性被执行的操作。例如服务器心跳处理函数`serverCron`。

#### 可触发事件类型
首先在*Redis*中注册了四种可触发类型:
1. `#define AE_NONE 0`，没有事件被注册
2. `#define AE_READABLE 1`，当描述符可读时触发，对于服务器的监听套接字文件述符，当有新连接到达时，也就是收到客户端发来的**SYN**报文时，文件描述符可读；对于服务器与客户端连接的套接字文件描述符，当内核TCP接收缓冲区中有数据时，文件描述符可读。
3. `#define AE_WRITABLE 2`，当描述符可写时触发，对于表示连接的套接字文件描述符，当内核TCP发送缓冲区有空余空间时，文件描述符可写；对于客户端调用非阻塞`connect`函数后的套接字文件描述符，当连接正式建立时，文件描述符可写。
4. `#define AE_BARRIER 4`，如果在同一个事件循环迭代中，如果有读事件触发，那么即使可写也不触发该事件。这在我们期望在发送反馈信息前将某些数据持久化到磁盘上时很有用。

### 数据结构描述
#### 文件事件类型数据结构
```c
/*文件事件结构体*/
typedef struct aeFileEvent {
    int mask;                 //当前时间监听的可触发事件，AE_READABLE|WRITEABLE|BARRIER
    aeFileProc *rfileProc;    //处理可读事件的回调函数函数指针
    aeFileProc *wfileProc;    //处理可写事件的回调函数函数指针
    void *clientData;         //客户端数据
} aeFileEvent;
```

#### 时间事件类型数据结构
```c
/*时间事件结构体*/
typedef struct aeTimeEvent {
    long long id;                         //时间事件ID
    long when_sec;                        //触发时间秒数
    long when_ms;                         //触发时间毫秒
    aeTimeProc *timeProc;                 //时间事件到期时处理函数的函数指针
    aeEventFinalizerProc *finalizerProc;  //事件最终被删除时处理函数的函数指针
    void *clientData;                     //客户端数据
    struct aeTimeEvent *prev;             //双向链表中前一个时间事件的指针
    struct aeTimeEvent *next;             //双向链表中后一个时间事件的指针
} aeTimeEvent;
```
#### 触发事件数据结构
```c
/*触发事件结构体，用于表示将要被处理的文件事件*/
typedef struct aeFiredEvent {
    int fd;        //被触发事件的文件描述符
    int mask;      //触发事件的掩码
} aeFiredEvent;
```

### 事件循环数据结构类型
对于事件循环的基本数据类型
```c
typedef struct aeEventLoop {
    int maxfd;
    int setsize;
    long long timeEventNextId;
    time_t lastTime;
    aeFileEvent *events;
    aeFiredEvent *fired;
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata;
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```
在上述数据结构之中：
1. `aeEventLoop.maxfd`，记录当前事件循环之中文件事件类型最大的文件描述符数值。
1. `aeEventLoop.setsize`，用于记录当前事件循环之中被追踪的文件描述符的数量。
1. `aeEventLoop.timeEventNextId`，对应于`aeTimeEvent.id`，用于记录下一个时间事件的ID。
1. `aeEventLoop.lastTime`，对应上次处理时间事件的时间戳。
1. `aeEventLoop.events`，一个`aeFileEvent`结构体的指针，指向一个动态分配的数组，其长度为`aeEventLoop.setsize`，用于存储所有被注册的文件事件。
1. `aeEventLoop.fired`，一个`aeFiredEvent`结构体的指针，指向一个动态分配的数组，其长度与`aeEventLoop.events`一致，用于存储从**I/O多路复用**之中返回的已被触发的事件。
1. `aeEventLoop.timeEventHead`，用于指向存储时间事件双向链表的头指针。
1. `aeEventLoop.stop`，用于标记事件循环是否终止的标记。
1. `aeEventLoop.apidata`，用于存储调用底层**I/O多路复用**接口的数据。
1. `aeEventLoop.beforesleep`，每次进入事件循环前调用函数的函数指针。
1. `aeEventLoop.aftersleep`，每次退出事件循环后调用函数的函数指针。

## IO多路复用接口
*Redis*根据系统所提供的支持，会使用最优的IO多路复用的实现接口：
1. *src/ae_epoll.c*，对于实现了`epoll`接口的系统，使用`epoll_wait`作为IO多路复用的接口。
1. *src/ae_kqueue.c*，对于*Mac OS*系统，使用`kqueue`作为IO多路复用的接口。
1. *src/ae_select.c*，对于老的系统，使用`select`作为IO多路复用的接口。

而在事件循环的数据结构中的`aeEventLoop.apidata`这个字段会根据系统使用的IO多路复用接口的不同，保存所需要的数据结构。
对于使用`epoll`接口，`apidata`这个字段所保存的数据结构为`aeApiState`，其定义为：
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
这其中:
1. `aeApiState.epfd`，是调用`epoll_create`系统调用创建的`epoll`文件描述符，系统可以通过阻塞监听这个文件描述符来监听其他一组文件描述符的状态变化。
1. `eaApiState.events`，对应于`aeEventLoop.events`这个字段，也是一个动态分配的数组，存储`epoll`监听事件对应的数据结构。

在上述这些源文件中，定义了一组静态函数，用于封装系统的**IO多路复用**接口，供*Redis*事件循环也就是`aeEventLoop`调用(以`epoll`系统调用为例)：
1. `static int aeApiCreate(aeEventLoop *eventLoop)`，通过调用`epoll_create`创建`epoll`接口的文件描述符，同时为`epoll_event`事件队列分配空间。
1. `static int aeApiResize(aeEventLoop *eventLoop, int setsize)`，调整事件循环中`epoll_event`队列的大小。
1. `static void aeApiFree(aeEventLoop *eventLoop)`，释放事件循环中，为`epoll`接口数据分配的相关资源。
1. `static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask)`，通过调用`epoll_ctl`系统调用，向内核注册监听一个文件描述符，或者更新一个文件描述符的监听事件。
1. `static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask)`，通过调用`epoll_ctl`系统调用，删除一个已监听文件描述符上的事件，或者删除一个已监听的事件描述符。
1. `static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp)`，该组静态函数的核心，通过调用`epoll_wait`等待监听的文件描述符上事件触发。

在`aeApiPoll`函数中：
```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    //阻塞在epoll_wait，等待事件ready
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize, tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;
            //将从内核中返回的就绪事件提取出来
            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            //将内核中返回的epoll_event上的文件描述符以及就绪事件拷贝到事件循环aeFiredEvent队列中
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
}
```
使用`epoll`接口实现的`aeApiPoll`函数其主要流程为：
1. 通过`aeApiAddEvent`函数注册所需要监听的文件描述符。
1. 通过`epoll_wait`这个系统调会以阻塞的方式等待前面注册的文件描述符上的事件，当有事件就绪，该函数会返回就绪事件的个数。
1. 通过循环遍历，将内核中返回的就绪事件提取出来，将其复制到*Redis*事件循环的已触发事件队列`aeEventLoop.fired`上面。

这样，通过上述的步骤，*Redis*的事件循环便可以通过`epoll_wait`接口，从内核中取出一系列的就绪事件，然后对所有的就绪事件进行处理。

## 事件循环接口函数定义
前面我们在系统调用层面介绍了事件驱动的实现细节，下面我们来看一下*Redis*如何应用系统调用来实现事件驱动的事件循环的，首先我们先来看一下相关的函数接口：
1. `aeEventLoop *aeCreateEventLoop(int setsize)`，这个函数用于创建并初始化一个事件循环`aeEventLoop`的结构体，初始可以监听的文件描述符列表`aeEventLoop.events`的长度会被设置为`setsize`。
1. `void aeDeleteEventLoop(aeEventLoop *eventLoop)`，在程序结束时释放整个事件循环结构体。
1. `void aeStop(aeEventLoop *eventLoop)`，用于设置`aeEventLoop.stop`停止标记。
1. `int aeGetSetSize(aeEventLoop *eventLoop)`，获取事件循环中监听队列大小。
1. `int aeResizeSetSize(aeEventLoop *eventLoop, int setsize)`，调整事件循环的监听队列大小。
1. `void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep)`，设置事件驱动进入事件循环前需要执行函数接口。
1. `void aeSetAfterSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *aftersleep)`，设置事件驱动结束一次事件循环时需要执行的函数接口。
1. `int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData)`，向事件循环之中创建一个文件事件，设置其监听的事件掩码`mask`以及事件处理函数`proc`；同时会设置客户端数据`clientData`，这个数据将会被赋值给文件事件结构体`aeFileEvent.clientData`字段中。
1. `void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)`，从事件循环中删除指定文件描述符`fd`的文件事件上监听的`mask`事件。
1. `int aeGetFileEvents(aeEventLoop *eventLoop, int fd)`，根据文件描述符找出对应的文件事件在事件循环之中监听的事件掩码。
1. `long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds, aeTimeProc *proc, void *clientData, aeEventFinalizerProc *finalizerProc)`，向事件循环中添加一个`milliseconds`毫秒后被触发的时间事件，设定其处函数`proc`以及客户端数据`clientData`，并且可以选择性设定结束处理函数`finalizerProc`，*Redis*会根据`aeEventLoop.timeEventNextId`为新的事件事件分配一个序号，记录在`aeTimeEvent.id`字段上。
1. `int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)`，给定一个时间事件序号，将其从事件循环之中删除。
1. `void aeMain(aeEventLoop *eventLoop)`，事件驱动执行主程序。
1. `int aeProcessEvents(aeEventLoop *eventLoop, int flags)`，事件循环处理函数。
1. `int aeWait(int fd, int mask, long long milliseconds)`，在某个文件描述符`fd`上阻塞等待，直到事件`mask`被触发，或者阻塞时间超过`milliseconds`。

下面我们来看一下其中几个比较重要的函数接口。

首先是`aeMain`这个事件驱动的入口函数，其在*Redis*的`main`函数之中被调用，用以启动事件循环：
```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        //如果调用aeSetBeforeSleepProc设置了beforesleep回调，那么在启动一次事件循环开启时，会执行beforesleep调用
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        //启动一次事件循环，处理文件事件以及时间事件，同时会尝试调用aftersleep
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```
这其中`aeProcessEvents`函数是处理一次事件循环的主体，处理每个待处理的时间事件，然后处理每个待处理的文件事件（可以通过刚刚处理的时间事件回调来注册）。 没有特殊标志的情况下，该函数将一直休眠，直到引发某些文件事件或下次发生事件（如果有）时为止。
* 如果`flags`为0，则该函数不执行任何操作并返回。
* 如果`flags`设置了`AE_ALL_EVENTS`，则处理所有类型的事件。
* 如果`flags`设置了`AE_FILE_EVENTS`，则处理文件事件。
* 如果`flags`设置了`AE_TIME_EVENTS`，则将处理时间事件。
* 如果`flags`设置了`AE_DONT_WAIT`，则该函数将尽快返回直到所有。
* 如果`flags`设置了`AE_CALL_AFTER_SLEEP`，则将调用`aftersleep`回调，这些事件可能无需等待就可以处理。

这个函数会返回已处理事件的数量。

`aeProcessEvents`函数的基本流程为：
1. 通过调用`aeSearchNearestTimer`这个静态函数来找到最近一个要到期的时间时间，获取其到期时间以确定`aeApiPoll`等待的`timeval`，这段逻辑可以保证，在即使没有文件事件触发的情况下，`aeApiPoll`仍然可以在最近的一次计时器时间事件到来之前返回，而不会错过对后续时间事件的处理。其具体实现代码为：
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

2. 调用封装的`aeApiPoll`这个核心**I/O多路复用**API，等待文件事件上的时间触发触发返回被触发事件的数量，或者超时进入时间事件的处理流程：
```c
    numevents = aeApiPoll(eventLoop, tvp);
```
3. 如果设置了`AE_CALL_AFTER_SLEEP`标记，同时通过调用`aeSetAfterSleepProc`设置了`aftersleep`，那么会在`aeApiPoll`唤醒返回后，执行`aftersleep`逻辑：
```c
    if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
        eventLoop->aftersleep(eventLoop);
```
4. 由于在aeApiPoll接口中已经将从内核返回的就绪事件拷贝到了事件循环中的fired就绪事件列表中了，后续会循环处理所有的就绪事件，通常情况下，我们会先处理可读事件，然后执行可写事件的处理逻辑，这在类似于执行查询操作后立即反馈查询结果这样的场景中很有用。但是如果在`flags`掩码中设置了`AE_BARRIER`标记，那么*Redis*会要求我们将执行顺序翻转过来，也就是在可读事件之后，绝不触发可写事件的处理。例如当我们需要在响应客户端请求之前，通过调用`beforeSleep`执行类似同步文件到磁盘的操作时，会很用：
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

现在我们来看一下处理时间事件的`processTimeEvents`函数，其基础逻辑为：
1. 如果系统的时钟曾经被移动到未来，那么在再次处理时间事件时，需要尝试对时钟进行修正。同时当这种情况发生时，*Redis*将会强制处理所有队列之中时间事件，通过将`aeTimeEvent.when_sec`设置为0来实现强制提前处理，*Redis*的作者认为当系统时间出现错位时，提早处理时间事件要比延后处理这些事件更为安全。
```c
    time_t now = time(NULL);
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    eventLoop->lastTime = now;
```
2. 从双向链表中删除已近被调度过的，同时被设置为`AE_DELETED_EVENT_ID`的时间事件，如果这个时间时间被设置了`finalizerProc`回调，那么在释放这个时间事件之前，会调用`finalizerProc`来处理相关释放逻辑：
```c
    while(te) {
        ...
        /* Remove events scheduled for deletion. */
        if (te->id == AE_DELETED_EVENT_ID) {
            
            ...

            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        ...
    }
```

3. 处理已经到期的时间事件，会调用时间事件`timeProc`的处理函数，来处理到期触发的计时器时间事件，如果这个时间事件是一个一次性触发的时间事件，那么会返回`AE_NOMORE`标记，这时将该事件设置为`AE_DELETED_EVENT_ID`，在下次事件循环中删除这个事件；如果这个时间事件是一个周期触发的时间事件，那么`timeProc`会返回下次触发的毫秒数，通过`aeAddMillisecondsToNow`，将新的时间更新到这个时间事件上：
```c
    while(te) {
        ...
        if (now_sec > te->when_sec || (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;
            id = te->id;
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
```

最后我们来看一下`aeWait`这个函数接口，这个函数会调用`poll`这个**I/O多路复用**接口来等待给定的文件描述符若干毫秒，这个函数接口是在`aeMain`之外的另外一个将程序阻塞的接口，两者的区别在于`aeMain`函数阻塞等待多个文件描述符，而`aeWait`函数只能阻塞给定的文件描述符。为何*Redis*在`aeMain`的基础上还要实现一个`aeWait`接口呢？这个接口主要用于*Redis*的一些同步阻塞操作，例如在*Master*实例与*Slave*实例之间使用**SYNC**命令以同步阻塞的方式同步数据。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*