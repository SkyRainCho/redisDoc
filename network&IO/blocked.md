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

*Redis*服务器全局变量中关于阻塞操作的相关字段：
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
1. 

数据库中与阻塞操作相关的字段：
```c
struct redisDb {
    ...
    dict *blocking_keys;
    dict *ready_keys;
};
```

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

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*