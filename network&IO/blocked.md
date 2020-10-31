# Redis对客户端阻塞操作的支持
在前面的关于*Redis*列表对象类型以及有序集合对象类型的文章中，我们了解到*Redis*针对这些数据对象类型提供了一组阻塞命令。以常用的**BLPOP**命令以及**BRPOP**命令为例，执行这些命令可以从指定的列表数据对象之中弹出数据，如果指令的列表对象为空，那么命令的调用者将会进入阻塞状态，直到这个指定的列表对象中有数据为止。

不过这里需要注意的是，此处所说的阻塞，并不是系统编程中所指的阻塞，也就是说阻塞状态并不会阻塞整个服务器，其仅仅是将执行阻塞命令的客户端对象`client`设置成阻塞状态，而服务器依然可以继续为其他客户端的请求提供服务。

在*Redis*之中，与阻塞相关的操作包括，列表对象、有序集合对象等对象类型上的阻塞命令，以及主从模式下的**WAIT**命令。本文接下来将详细讲解*Redis*对于阻塞命令的相关支持。

首先我们来看一下，*Redis*对于阻塞相关的状态定义：
```c
#define CLIENT_BLOCKED (1<<4)
#define CLIENT_UNBLOCKED (1<<7)
```
上述两个状态掩码都是用于设置客户端的标记字段`client.flags`：
1. `CLIENT_BLOCKED`，这个状态用于标记客户端处于阻塞状态，回顾并补充一下前面关于客户端内容的讲解，在这种状态下客户端依然可以通过通过`readQueryFromClient`接口将连接上接收到的数据读取到客户端的应用层缓冲区。但是在处理应用层缓冲区并执行命令的函数接口`processInputBuffer`中，如果发现客户端处于阻塞状态则会终止解析处理的流程。
1. `CLIENT_UNBLOCKED`，这个状态是客户端解除阻塞状态`CLIENT_BLOCKED`后，到恢复到正常状态前的一个中间状态。这是由于前面所说的，客户端在阻塞状态时应用层缓冲区中可能会累积着一些缓存数据没有被处理，那么这些数据需要在客户端解除阻塞状态后及时的被处理，因此*Redis*设计了这个`CLIENY_UNBLOCKED`中间状态，用于告知服务器及时处理应用层缓冲区中的数据；如果不做处理直接将客户端扔回事件循环，那么只有在这个客户端对应的连接上再次触发可读事件时，缓冲区中的数据才会有机会被处理。

除了上面两个阻塞状态定义之外，*Redis*还给出了关于阻塞类型的若干个定义：
```c
#define BLOCKED_NONE    0    //没有处于任何阻塞状态
#define BLOCKED_LIST    1    //阻塞在关于列表对象的阻塞命令上
#define BLOCKED_WAIT    2    //阻塞在WAIT命令上
#define BLOCKED_MODULE  3    //模块导致的阻塞
#define BLOCKED_STREAM  4    //阻塞在关于STREAM对象上的阻塞命令
#define BLOCKED_ZSET    5    //阻塞在关于有序集合对象上的阻塞命令
#define BLOCKED_NUM     6    //阻塞状态的个数
```

*Redis*用于维护阻塞状态相关的内容主要分布在服务器全局变量、数据库键空间定义以及客户端对象结构体之中。

首先我们看一下*Redis*服务器全局变量中关于阻塞操作的相关字段：
```c
struct readyList
{
    redisDb* db;
    robj *key;
};

struct redisServer{
    ...
    unsigned int blocked_clients;
    unsigned int blocked_clients_by_type[BLOCK_NUM];
    list *unblocked_clients;
    list *ready_keys;
    ...
};
```
在上述这些结构体字段之中：
1. `redisServer.blocked_clients`以及`redisServer.blocked_client_by_type`这两个字段分别用于记录当前处于阻塞状态的客户端对象的数量，以及按照`BLOCKED_XXX`分组的被阻塞客户端对象的数量。
1. `redisServer.unblocked_clients`则是一个存储`client`对象指针的双端链表，用于保存当前处于`CLIENT_UNBLOCKED`状态的客户端的列表。
1. `redisServer.ready_keys`是一个存储`readyList`结构体的双端链表，用于存储当前整个*Redis*之中已经就绪可以返回给被阻塞客户端的键的集合，这个每一个`readyList`对应一个就绪的键：
    1. `readyList.db`，表示这个就绪的键对应的数据库。
    1. `readyList.key`，表示这个就绪的键的内容。

数据库键空间中与阻塞操作相关的字段：
```c
struct redisDb {
    ...
    dict *blocking_keys;
    dict *ready_keys;
};
```
上述这两个字段里：
1. `redisDb.blocking_keys`，这是一个哈希表对象，实现了键到一组客户端链表的映射，表示当前
1. `redisDb.ready_keys`

而在客户端对象结构体`client`中与阻塞操作相关的字段为：
```c
struct client {
    ...
    int btype;
    blockingState bpop;
};
```

这里`blockingState`这个结构体的定义为：
```c
struct blockingState {
    mstime_t timeout;
    dict *keys;
    robj *target;
    ...
};
```


而另外另外一种解除客户端阻塞状态的方式，就是客户端等待的键已经就绪，下面这个函数便是用于通知键就绪的接口：
```c
void signalKeyAsReady(redisDb *db, robj *key);
```
被客户端阻塞等待的键一定是一个已经不存在于数据库键空间之中的键，那么如果想要通知客户端键就绪的事件，就需要这个键被重新加入到数据库键空间中时，调用`signalKeyAsReady`接口进行通知：
```c
void dbAdd(redisDb *db, robj *key, robj * val)
{
    ...
    if (val->type == OBJ_LIST ||
        val->type == OBJ_ZSET)
        signalKeyAsReady(db, key);
    ...
}
```
`signalKeyAsReady`这个接口会检查被加入的键是否存在于该数据库键空间的阻塞列表`redisDb.blocking_keys`之中，如果不存在则表明这个键没有被任何一个客户端“阻塞地”等待；反正会将这个就绪的键加入到服务器的就绪队列`redisServer.ready_keys`以及键空间的就绪队列`redisDb.ready_keys`之中，等待后续的处理。

而*Redis*在收到一个键就绪的事件之后，需要解锁并通知阻塞在这个键上的客户端：
```c
void handleClientsBlockedOnKeys(void);
```
上面这个`handleClientsBlockedOnKeys`函数便是用于处理这种情况的接口，由于键的就绪事件都是由于某一个客户端执行命令所产生的，因此这个函数在*Redis*每次执行一条命令时，会调用`handleClientsBlockedOnKeys`这个接口来对阻塞的客户端进行处理：
```c
int processCommand(client *c)
{
    ...
    call(c, CMD_CALL_FULL);
    if (listLength(server.ready_keys))
        handleClientsBlockedOnKeys();
    ...
}
```
上面这段代码，便是*Redis*调用`handleClientsBlockedOnKeys`函数的具体场景。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*