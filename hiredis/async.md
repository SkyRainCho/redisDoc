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

这样，当我们向服务器发送一条查询命令之后，便可以通过调用`redisAsyncContext.ev.addRead`接口注册可读事件，进而在查询数据返回时触发可读事件回调函数，来处理数据。

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

这里在向`redisAsyncContext`对象注册`onConnect`回调函数时，同时也会向事件驱动之中注册监听可写事件，这样当异步发起的连接被正式建立的时候，

### 异步API命令的发送与回调处理

### 异步API连接的断开

### 异步API状态迁移



























***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*