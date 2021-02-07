# Redis客户端异步API

## 异步API概述
Hiredis之中提供了一组异步API，可以很容易的配合各种事件库进行工作。在*deps/hiredis/examples*目录下，提供了两个例子，展示了Hiredis是如何于libev库以及libevent库配合工作的。本小节内容翻译自Hiredis官方文档。

### 建立连接
使用`redisAsyncConnect`函数，可以建立一条与*Redis*服务器之间的非阻塞的网络连接。这个函数会返回一个新被创建`redisAsyncContext`对象指针。获取到这个`redisAsyncContexr`对象指针指针之后，我们需要检查`redisAsyncContext.err`字段上的数据，以判断在创建连接的过程之中是否有报错信息。之所以需要检查`err`字段，是因为将要创建的连接使用的是非阻塞模式，如果给定的地址与端口可以接受连接，内核是无法立即地将连接返回的。

需要注意的是，`redisAsyncContext`对象，不是一个线程安全的对象。

```c
redisAsyncContext *c = redisAsyncConnect("127.0.0.1", 6379);
if (c->err)
{
    printf("Error: %s\n", c->errstr);
    // 处理错误信息
}
```

`redisAsyncContext`对象可以只有一个断开连接的回调函数，当连接被断开时(可以时因为错误而断开，也可以是用户主动断开)，这个回调函数会被调用。这个函数应该具有如下的函数原型：
```c
void(const redisAsyncContext *c, int status);
```
当发生连接断开时，如果是用户主动断开连接的行为，那么`status`参数将会被设置为`REDIS_OK`；相反的如果是网络发生异常错误而导致连接断开的，`status`参数将会被设置为`REDIS_ERR`。当参数为`REDIS_ERR`时，可以通过访问`redisAsyncContext.err`字段来查找引发错误的原因。

一旦断开连接的回调函数被触发之后，对应的`redisAsyncContext`对象总是会被释放调用。尤其是在需要重连的时候，断开连接的回调函数将是一个很好的方法。

每一个`redisAsyncContext`对象尽可以被设置依次断开连接的回调函数。为同一个对象，重复设置断开连接的回调函数，将会返回`REDIS_ERR`。这个设置回调函数的接口为：
```c
int redisAsyncSetDisconnectCallback(redisAsyncContext *ac, redisDisconnectCallback *fn);
```

### 发送命令以及对应的回调
在异步API之中，命令会基于事件循环的特性被自动地组成Pipeline流水线的形式。因此与同步API具有多种发送命令的形式不同，异步接口之中只有一种发送命令的形式。同时由于命令是被异步地发送到*Redis*服务器的，所以在发出一个命令的时候需要一个回调函数，在收到服务器的回复数据时，这个回调函数将会被调用，以便处理命令的回复数据，这个回调函数的原型应该为：
```c
void(redisAsyncContext *c, void *reply, void *privdata);
```
这个函数原型里，`privdata`参数可以存储任意的数据，并在这个回调函数被调用的地方传递给它。

在异步API之中，用于发送命令的函数为：
```c
int redisAsyncCommand(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata, const char *format, ...);
int redisAsyncCommandArgv(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata, int argc, const char **argv, const size_t *argvlen);
```
上述这两个函数的工作方式与对应的阻塞操作相类似，当命令数据被成功地加入到输出缓冲区之中时，函数会返回`REDIS_ERR`，否则函数会返回`REDIS_ERR`。例如，当用户主动断开连接时，便无法像输出缓冲区之中添加新的命令数据，这时调用`redisAsyncCommand`函数将返回`REDIS_ERR`。

如果命令的返回数据被一个`NULL`的回调函数接收时，这个返回数据将会立即被释放。当命令回调函数不为空时，这个返回数据的内存在回调函数执行结束之后，也会立即被释放，这也就是说返回数据仅在回调函数的生命周期之中有效。

当一个`redisAsyncContext`对象遇到错误时，所有挂起的回调函数都将使用`NULL`的`reply`参数进行调用。

### 断开连接

当用户期望主动地断开一个异步连接时，我们可以使用下面这个函数来进行：
```c
void redisAsyncDisconnect(redisAsyncContext *ac);
```
当调用这个函数的时候，连接不会被立即地被中断。实际上，调用`redisAsyncDisconnect`之后，对应的`redisAsyncContext`对象将不会接收新的命令，而只有当输出缓冲区之中的所有命令数据都已经被写入网络，并且这些命令的返回数据都已经被读取并被各自的回调函数处理完成，这时连接才会真正的被中断。在这之后，前面我们介绍的`status`参数为`REDIS_OK`的回调函数将会被执行，`redisAsyncContext`对象也会被释放。

### 异步API挂载事件库
在*deps/hiredis/adapters*目录下，定义了若干的钩子，用于在`redisAsyncContext`对象被创建之后，挂载到事件库上。

## 异步API的实现细节

### 异步API的数据结构定义
在*deps/hiredis/async.h*头文件之中定义了关于异步API使用的`redisAsyncContext`数据结构。与同步API类似，`redisAsyncContext`表示用户与*Redis*服务器之间的连接。
```c
typedef struct redisAsyncContext 
{
    redisContext c;
    int err;
    char *errstr;
    void *data;
    struct {
        void *data;
        void (*addRead)(void *privdata);
        void (*delRead)(void *privdata);
        void (*addWrite)(void *privdata);
        void (*delWrite)(void *privdata);
        void (*cleanup)(void *privdata);
    } ev;

    redisDisConnectCallback *onDisConnect;
    redisConnectCallback *onConnect;

    redisCallbackList replies;

    struct {
        redisCCallbackList invalid;
        struct dict *channels;
        struct dict *patterns;
    } sub;
} redisAsyncContext;
```
在`redisAsyncContext`这个数据接口之中：
- `redisAsyncContext.c`，这个字段是一个`redisContext`类型的数据，用于实际地记录用户与*Redis*服务器之间的连接信息。
- `redisAsyncContext.err`与`redisAsyncContext.errstr`，当连接或者对于`redisAsyncContext`对象数据的操作出现错误时，这两个字段存储了对应的错误信息。而实际上这两个字段是从`redisAsyncContext.c`这个`redisContext`类型数据中的`redisContext.err`以及`redisContext.errstr`字段之中复制过来的。
- `redisAsyncContext.ev`，这个字段是`redisAsyncContext`对象与事件驱动库交互的接口，需要Hiredis的用户按照声明的函数原型自行定义相关接口函数的实现，并赋值给`redisAsyncContext.ev`这个字段。
- `redisAsyncContext.onDisConnect`，`redisAsyncContext`对象在连接断开时的回调函数。
- `redisAsyncContext.onConnect`，由于`redisAyncContext`对象建立连接时也是采用异步的方式建立的，而这个字段存储的便是连接被真正建立起来时，需要调用的回调函数。
- `redisAsyncContext.replies`，这里以单链表的形式存储着查询命令返回数据的回调函数，由于*Redis*服务器在处理核心数据的时候，采用的是单线程的方式进行处理，因此用户执行的查询命令都是按照顺序执行的，而`redisAsyncContext.replies`字段之中也是按照先进先出的顺序存储查询命令对应的回调函数的。
- `redisAsyncContext.sub`，这里存储在**订阅/发布**模式下的消息回调函数。对于普通的查询命令，无论执行成功与否，都是一条查询命令对应一个返回数据的形式；而在**订阅/发布**模式中，在执行过订阅命令后，可能回收到多条返回消息，也就是对一个频道或者模式的订阅命令对应多个返回数据的一对多的形式。因此异步API之中采用了`redisAsyncContext.sub`这个字段独立存储**订阅/发布**模式下的消息回调函数。

对于`redisAsyncContext.onDisConnect`以及`redisAsyncContext.onConnect`这两个函数指针对应的回调函数，需要Hiredis库的使用者自行定义，不过函数的定义应该遵循下面的函数原型：
```c
void(const redisAsyncContext *c, int status);
```
这两个回调函数应该根据`status`参数的值来判断连接是否正常建立，或者说导致连接断开的具体的原因，并作出相应的处理。

而对于前面介绍的回调函数队列`redisAsyncContext.replies`，其数据结构的定义为：
```c
typedef void (redisCallbackFn)(struct redisAsyncContext*, void*, void*);
typedef struct redisCallback {
    struct redisCallback *next;
    redisCallbackFn *fn;
    void *privdata;
} redisCallback;

typedef struct redisCallbackList {
    redisCallback *head, *tail;
} redisCallbackList;
```
这里面`redisCallbackFn`为查询命令返回数据所触发的回调函数，Hiredis库的用户可以按照自己的需求为查询命令定义自己的回调函数，以处理查询命令的返回数据，回调函数应该接收一个`redisAsyncContext`对象、一个`redisReply`对象以及一个用户自定义的数据`privdata`做为参数。

`redisCallbackList`这个单链表中按照顺寻存储了查询命令的回调数据`redisCallback`。`redisCallback`对象包含了对应的回调函数，以及用户自定义的数据`privdata`。

### 异步API接口与事件驱动库的绑定
Hiredis异步API的高效之处就在，用户通过异步API执行查询命令之后，不需要原地阻塞地等待服务器的返回数据。用户在执行完查询命令之后，可以继续去处理其他的业务逻辑，在查询命令的返回数据就绪之后，Hiredis在通知用户读取数据，通过回调函数来处理返回数据。而这种通知的机制就是整个异步API的核心，事件驱动库主要的工作便是执行上述的通知机制。在前面的文章之中我们介绍了*Redis*服务器所使用的自己实现的事件驱动库，相信各位读者对事件驱动库的核心概念已经有了一个了解。

除了*Redis*服务器自身实现的事件驱动库之外，业内比较流行开源的事件驱动库包括**Libevent**、**Libev**等等，同时也包括了各个公司内部自己实现的事件驱动库。作为Hiredis库的使用者，如果你想在你的项目之中集成Hiredis的异步API，那么你需要将Hiredis与自己项目之中事件驱动库进行绑定。本小节会就如何与事件驱动库进行绑定进行介绍。

为了实现与事件驱动库的绑定，Hiredis库的用户首先需要定义一个事件对象类型，用于在事件驱动库之中表示这个`redisAsyncContext`对象。我们来看一下，针对*Redis*自己的事件驱动库以及**Libevent**这个事件驱动库来说，这个事件对象类型的定义：

```c
typedef struct redisAeEvents {
    redisAsyncContext *context;
    aeEventLoop *loop;
    int fd;
    int reading, writing;
} redisAeEvents;

typedef struct redisLibeventEvents {
    redisAsyncContext *context;
    struct event *rev, *wer;
} redisLibeventEvents;
```

这里我们可以发现，Hiredis异步API所需要的事件类型的定义中，至少应该包含有其对应的`redisAsyncContext`对象的信息；区分输入或者输出的标记，以及对应的事件驱动库对象的数据，例如`redisAeEvents.loop`以及`redisLibeventEvents.rev`和`redisLibeventEvents.wer`（由于在**Libevent**库之中，`struct event`本身就携带着事件驱动库对象`struct event_base`的信息，因此在结构体之中包含了`struct event`数据也就相当于包含了`struct event_base`这个事件驱动库的数据）。

在Hiredis异步API之中已经定义了两个函数接口作为可读事件以及可写事件的回调处理函数：

```c
void redisAsyncHandleRead(redisAsyncContext *ac);
void redisAsyncHandleWrite(redisAsyncContext *ac);
```

对于这两个函数接口，我们需要根据自己绑定的事件驱动库所对应的`Events`上进行一个简单的封装，以**Libevents**对应可读事件的处理函数为例：

```c
static void redisLibeventReadEvents(int fd, short event. void *arg)
{
    redisLibeventEvents *e = (redisLibeventEvents)arg;
    redisAsyncHandleRead(e->context);
}
```

也就是说，实际上注册到事件驱动库之中的可读事件处理函数为`redisLibeventReadEvents`，而这个函数最终是通过调用`redisAsyncHandleRead`来完成对可读事件的处理的。

这样一来，我们自定义一个函数接口，将应用之中的事件驱动库以及`redisAsyncContext`对象绑定起来，还是以**Libevent**库为例：

```c
static int redisLibeventAttach(redisAsyncContext *ac, struct event_base *base)
{
    redisContext *c = &(ac->c);
    redisLibeventEvents *e;
    if (ac->ev.data != NULL)
        return REDIS_ERR;
    e = (redisLibeventEvents*)malloc(sizeof(*e));
    e->context = ac;
    ...
    ac->ev.data = e;
    e->rev = event_new(base, c->fd, EV_READ, redisLibeventReadEvent, e);
    e->wev = event_new(base, c->fd, EV_WRITE, redisLibeventWriteEvent, e);
    event_add(e->rev, NULL);
    event_add(e->wev, NULL);
    return REDIS_OK;
}
```

通过上面这个函数接口，我们便初始化了一个`redisLibeventEvents`对象，并将其赋值给`redisAsyncContext.ev.data`对象，这样就建立起了`redisAsyncContext`对象与事件驱动库的绑定关系。

回头我们再来看看异步API的数据结构定义：

```c
struct {
	void *data;
	void (*addRead)(void *privdata);
	void (*delRead)(void *privdata);
	void (*addWrite)(void *privdata);
 	void (*delWrite)(void *privdata);
	void (*cleanup)(void *privdata);
} ev;
```

这里处理`ev.data`字段用于储存我们自定义的`Events`对象之外，还有五个函数指针，同样的这也需要我们自己根据对应的事件驱动库来定义对应的函数接口，这些函数接口的含义为：

- `ev.addRead`，我们需要自定义这个函数接口用于向事件驱动之中注册监听可读事件。
- `ev.delRead`，我们需要自定义这个函数接口从事件驱动之中移除对可读事件的监听。
- `ev.addWrite`，我们需要自定义这个函数接口用于向事件驱动之中注册监听可写事件。
- `ev.delWrite`，我们需要自定义这个函数接口从事件驱动之中移除对可写事件的监听。
- `ev.cleanup`，这个函数用清理`redisAsyncContext`对象，会从事件驱动之中移除可读事件以及可写事件的监听。

这样，当我们向服务器发送一条查询命令之后，便可以通过调用`redisAsyncContext.ev.addRead`接口注册可读事件，进而在查询数据返回时触发可读事件回调函数，来处理命令的返回数据。

### 异步API连接的建立

```c
redisAsyncContext *redisAsyncConnect(const char *ip, int port);
redisAsyncContext *redisAsyncConnectBind(const char *ip, int port, const char *source_addr);
redisAsyncContext *redisAsyncConnectBindWithReuse(const char *ip, int port, const char *source_addr);
```

通过上面的三个接口，我们可以异步地向*Redis*服务器发起连接，并返回`redisAysyncContext`对象，不过需要注意的是，上述三个函数返回的返回`redisAsyncContext`对象不可以直接使用，因此此时连接没有完全建立。我们需要`redisAysncContext`对象注册`onConnect`回调函数：

```c
#define _EL_ADD_WRITE(ctx) do { \
		if ((ctx)->ev.addWrite) (ctx)->ev.addWrite((ctx)->ev.data); \
	} while(0)
int redisAsyncSetConnectCallback(redisAsyncContext *ac, redisConnectCallback *fn)
{
    if (ac->onConnect == NULL)
    {
        ac->onConnect = fn;
        _EL_ADD_WRITE(ac);
        return REDIS_OK;
    }
}
```

这里在向`redisAsyncContext`对象注册`onConnect`回调函数时，同时也会向事件驱动之中注册监听可写事件。这样当异步发起的连接被正式建立的时候，会触发Socket文件描述符上的可写事件，在`redisAsyncHandleWrite`这个处理写事件的接口之中，便可以调用我们注册的`onConnect`函数，完成连接的建立。

### 异步API命令的发送与回调处理
```c
int redisvAsyncCommand(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata, const char *format, va_list ap);
int redisAsyncCommand(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata, const char *format, ...);
int redisAsyncCommandArgv(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata, int argc, const char **argv, const size_t *argvlen);
int redisAsyncFormattedCommand(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata, const char *cmd, size_t len);
```
上面的四个函数接口是异步API之中用于向*Redis*服务器异步地发送查询命令的接口。在这些接口之中，查询命令数据会被写入到`redisAsyncContext`对象的应用层输出缓冲区之中，同时通过`_EL_ADD_WRITE`这个宏，调用`redisAsyncContext.ev.addWrite`向事件驱动之中注册监听可写事件。

这样，当系统内核空间的输出缓冲区有足够空间时，事件驱动库便会触发可写事件的处理函数`redisAsyncHandleWrite`，将应用层缓冲区之中数据写入内核空间：
```c
void redisAsyncHandleWrite(redisAsyncContext *ac)
{
    redisContext *c = &(ac->c);
    int done = 0;
    ...
    if (redisBufferWrite(c, &done) == REDIS_ERR)
        __redisAsyncDisconnect(ac);
    else {
        if (!done)
            _EL_ADD_WRITE(ac);
        else
            _EL_DEL_WRITE(ac);

        _EL_ADD_READ(ac);
    }
}
```
这里会通过`redisBufferWrite`向内核缓冲区之中写入数据。如果数据输出完成，则会从事件驱动库之中将注册的可写事件的监听移除。最后还会向事件驱动中注册可读事件的监听`_EL_ADD_READ`，以期待后续接收从服务器收到的返回数据。

我们回来再看一下向服务器发送查询命令的接口，调用这些接口除了我们需要给出查询命令的数据之外，我们还可以向接口之中传递一个命令回调函数的函数指针，用于处理这个命令的返回数据，如果传递一个`NULL`值则意味着我们不需要处理该命令的返回数据；同时我们可以根据自己的应用的需求传递一个私有的数据`privdata`，用来配合命令返回数据的回调函数，这个数据如果不需要也可以传递一个`NULL`的值。

正常情况下，我们执行一个普通的查询命令时，Hiredis会用回调函数`redisCallbackFn *fn`以及用户的私有数据`void *privdata`构造一个`redisCallback`对象，并将之插入到`redisAsyncContext.replies`回调队列之中。前面我们介绍了，在Hiredis之中，普通的查询命令以及**发布/订阅**命令之间的差异，因此如果我们通过异步API执行订阅命令时，对应的回调对象`redisCallback`不会被插入`redisAsyncContext.replies`队列之中，而是会根据订阅频道还是订阅模式的不同，被插入到`redisAsyncContext.sub.channels`以及`redisAsyncContext.sub.patterns`这两个哈希表之中。

在服务器处理完用户的查询命令之后，将返回数据发送给客户端，这是在用户一侧，会触发`redisAsyncContext`上的可读事件的回调函数`redisAsyncHandleRead`：
```c
void redisAsyncHandleRead(redisAsyncContext *ac) {
    redisContext *c = &(ac->c);
    ...
    if (redisBufferRead(c) == REDIS_ERR) 
        __redisAsyncDisconnect(ac);
    else {
        _EL_ADD_READ(ac);
        redisProcessCallbacks(ac);
    }
}
```
这个Hiredis在通过`redisBufferRead`函数完成数据读取之后，通过`redisProcessCallbacks`这个接口，执行特定的回调函数处理逻辑。
```c
void redisProcessCallbacks(redisAsyncContext *ac) {
    redisContext *c = &(ac->c);
    redisCallback cb = {NULL, NULL, NULL};
    void *reply = NULL;
    int status;

    while((status = redisGetReply(c,&reply)) == REDIS_OK) {
        ...
        if (cb.fn != NULL)
        {
            __redisRunCallback(ac,&cb,reply);
            c->reader->fn->freeObjecy(reply);

            if (c->flags & REDIS_FREEING)
            {
                __redisAsyncFree(ac);
                return;
            }
        }
        else {
            c->reader->fn->freeObject(reply);
        }
    }
    ...
}
```
这里我们可以看到，查询命令的返回数据`redisReply`会被Hiredis自动释放，这也就意味着，我们无法取出`redisReply`返回数据，如果我们希望操作这些返回数据，只能通过注册自定义的回调函数的方式来进行。

### 异步API连接的断开

在异步API之中，`redisAsyncContext`对象可以通过下面这个接口主动断开连接，并释放`redisAsyncContext`对象：
```c
void redisAsyncDisconnect(redisAsyncContext *ac)
{
    redisContext *c = &(ac->c);
    c->flags |= REDIS_DISCONNECTING;
    if (!(c->flags & REDIS_IN_CALLBACK) && ac->replies.head == NULL)
        __redisAsyncDisconnect(ac);
}
```
在这个`redisAsyncDisconnect`函数之中，只有`redisAsyncContext.replies`这个挂起的回调队列为空，并且当前的对象没有在处理回调函数时，才会立即调用`__redisAsyncDisconnect`函数断开连接；否则不会立即将连接断开，而是会将连接设置为`REDIS_DISCONNECTING`的状态，在这种状态下`redisAsyncContext`对象不会再接受新的查询命令，但是对于已经进入输出缓冲区之中的查询命令，Hiredis依然会执行查询逻辑以及处理返回数据的回调逻辑。

当`redisAsyncContext`对象处理完全部的挂起回调之后，同样也会调用底层的`__redisAsyncDisconnect`来执行断开连接的处理流程。
```c
void redisProcessCallbacks(redisAsyncContext *ac) {
    redisContext *c = &(ac->c);
    void *reply = NULL;
    int status;
    while((status = redisGetReply(c,&reply)) == REDIS_OK) {

    }
    if (status != REDIS_OK)
        __redisAsyncDisconnect(ac);
}
```
同时通过上面读取处理返回数据的`redisProcessCallbacks`代码片段我们也可以看到，当读取返回数据发生错误的时候，也会调用底层的`__redisAsyncDisconnect`函数来释放连接。

在`__redisAsyncDisconnect`处理断开连接的过程之中会执行如下的逻辑：
1. 因为断开连接有可能发生在数据读取的过程之中，因此会存在`redisAsyncContext.replies`以及`redisAsyncContext.sub`之中残留有挂起的回调对象的情况。在连接被真正断开之前，首先会依次触发所有被挂起的对调对象，只不过回调函数的`reply`参数为`NULL`，这也意味着我们自定义的回调函数，需要检查`reply`参数，并对参数为`NULL`的情况给出相应的处理。
1. 清理`redisAsyncContext`对象在事件驱动模块之中注册的可读与可写事件的监听。
1. 如果`redisAsyncContext`对象定义了`onDisconnect`回调，则调用该回调函数，执行额外的清理逻辑。
1. 最后关闭连接，释放对象的应用层缓冲区。

### 异步API状态迁移

在前面介绍Hiredis同步API的文章中，我们了解到`redisContext.flags`这个字段会用于标记当前与*Redis*服务器的连接对象的状态信息。由于`redisContext`同样也是异步API中`redisAsyncContext`对象的字段，因此这些状态信息对于异步API依然有效，本小节将对异步API之中可能会设计到的状态进行一个简要的介绍：
- **`REDIS_CONNECTED`**
    - 该状态表示`redisAsyncContext`对象与*redis*服务器之间已经建立起了完整的网络连接，由于是通过异步`connect`来建立的连接，因此调用`redisAsyncConnect`接口获得的`redisAsyncContext`对象是没有`REDIS_CONNECTED`这个状态的。只有在`redisAsycnContext`对象在事件驱动模块之中触发了`__redisAsyncHandleConnect`回调正常地完成连接的建立之后，`redisAsyncContext`对象才会被设置上`REDIS_CONNECTED`这个标记。
- **`REDIS_IN_CALLBACK`**
    - 该状态表示`redisAsyncContext`对象正在执行回调函数逻辑，在`__redisRunCallback`这个接口之中，当我们真正调用回调函数之前，会为`redisAsyncContext`对象附加上这个状态，当回调函数结束之后，会将这个状态移除。这个状态可以保护`redisAsyncContext`对象在执行回调函数的过程之中不会被其他的线程断开连接或者释放，在断开连接的函数`redisAsyncDisconnect`以及释放对象的函数`redisAsyncFree`之中都会检查这个`REDIS_IN_CALLBACK`状态。
- **`REDIS_SUBSCRIBED`**
    - 该状态表示`redisAsyncContext`对象正在订阅某个频道或者某个模式，用户在通过`__redisAsyncCommand`在`redisAsyncContext`对象上执行**SUBSCRIBE**命令时，会为对象附加上这个状态；而当用户执行**UNSUBSCRIBE**命令后，`REDIS_SUBSCRIBED`状态将会被移除。我们在前面介绍**订阅/发布**机制的时候有讲过服务器端`client`对象在进入订阅模式之后，将无法执行其他的非**订阅/发布**的命令。这里当`redisAsyncContext`对象处于`REDIS_SUBSCRIBED`时，如果通过`__redisAsyncCommand`执行一条普通的查询命令时，这个查询命令的回调函数不会被加入到`redisAsyncContext.replies`队列之中，而是会加入到`redisAsyncContext.sub.invalid`这个队列里。
- **`REDIS_DISCONNECTING`**
    - 该状态表示`redisAsyncContext`对象处于连接将要被断开的状态，在我们调用`redisAsyncDisconnect`函数主动断开与*Redis*服务器之间的连接时，对象会被附加上这个状态标记。拥有这个标记的`redisAsyncContext`对象无法执行新的查询命令，当这个对象所有被挂起的回调对象全部被处理完的时候，这个对象的连接才会被真的断开。
- **`REDIS_FREEING`**
    - 该状态表示`redisAsyncContext`对象处于将要被释放的状态，在我们调用`redisAsyncFree`尝试释放一个`redisAsyncContext`对象的时候，为会对象设置上`REDIS_FREEING`的状态，在所有挂起的数据处理结束之后，这个对象才会真正地被释放。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*