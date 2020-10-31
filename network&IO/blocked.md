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
1. `redisDb.blocking_keys`，这是一个哈希表对象，实现了键到一组客户端链表的映射，表示记录当前数据库键空间之中被等待就绪的键以及对应阻塞在该键上的客户端列表。
1. `redisDb.ready_keys`，这是一个作为集合使用哈希表，就用表示当前数据库键空间之中此时就绪的键。

而在客户端对象结构体`client`中与阻塞操作相关的字段为：
```c
struct blockingState {
    mstime_t timeout;
    dict *keys;
    robj *target;
    ...
};

struct client {
    ...
    int btype;
    blockingState bpop;
};
```
其中各个字段的含义为：
1. `client.btype`，用于记录当前客户端对象的阻塞类型，`BLOCKED_XXX`。
1. `client.bpop`，用于记录当前客户端对象在阻塞状态之中的一些参数数据：
    1. `blockingState.timeout`，用于记录阻塞操作的等待时间，如果为0则是会一直等待下去直到就绪为止。
    1. `blockingState.keys`，用于记录当前客户端阻塞等待键，类似**BLPOP**这样的命令可以阻塞多个键，因此这里使用一个哈希表来进行存储。
    1. `blockingState.target`，阻塞操作的目标对象，类似**BRPOPLPUSH**这样的命令需要提供目标键，对于这样的命令或者操作，会将目标键存储在这个字段之中。

介绍完*Redis*阻塞操作的相关数据结构支持之后，我们下面来详细介绍一下阻塞操作的相关业务逻辑。

首先我们看一下，客户端进入阻塞状态的通用接口
```c
void blockClient(client *c, int btype);
void blockForKeys(client *c, int btype, robj **keys, int numkeys, mstime_t timeout, robj *target, streamID *ids);
```
以**BRPOP**命令和**BLPOP**命令为例：
```c
void blockingPopGenericCommand(client *c, int where) {
    robj *o;
    mstime_t timeout;
    if (getTimeoutFromObjectOrReply(c,c->argv[c->argc-1],&timeout,UNIT_SECONDS) != C_OK) return;
    ...
    //如果列表对象为空或者键不存在，那么必须通过blockForKeys接口阻塞客户端
    blockForKeys(c,BLOCKED_LIST,c->argv + 1,c->argc - 2,timeout,NULL,NULL);
}
```
通过上面的代码片段以及其他类似命令的实现函数，我们可以知道所有的阻塞相关命令都是通过上面`blockForKeys`这个接口为客户端设置阻塞状态的。这个函数接口会让客户端阻塞等待指定的键就绪，并设置指定的等待时间，它将为`client.bpop.timeout`以及`client.bpop.target`字段进行复制，并将被等待的键加入到客户端的`client.bpop.keys`这个哈希表中，同时将这个客户端插入到数据库键空间`redisDb.blocking_keys`对应键的等待列表的尾部；最终调用`blockClient`这个接口为客户端设置上`CLIENT_BLOCKED`这个标记，并更新服务器全局数据中关于阻塞客户端的计次`redisServer.blocked_clients`以及`redisServer.blocked_clients_by_type`。

下面在来看一下客户端退出阻塞状态的逻辑，首先需要明确的一点是客户端可以通过两种方式退出阻塞状态，分别是到达等待的超时间`client.bpop.timeout`以及所等待的键就绪。

那么我们先来看一下等待超时的情况。
前面在介绍客户端的内容是，我们介绍过`clientsCronHandleTimeout`这个函数接口：
```c
int clientsCronHandleTimeout(client *c, mstime_t now_ms)
{
    time_t now = now_ms/1000;
    ...
    if (c->flags & CLIENT_BLOCKED)
    {
        if (c->bpop.timeout != 0 && c->bpop.timeout < now_ms) {
            replyToBlockedClientTimeOut(c);
            unblockedClient(c);
        }
        ...
    }
    ...
}
```
应用这个接口，我们可以关闭空闲时间过长的客户端，同时这个函数还会处理阻塞状态等待超时的情况，也就是当客户端拥有`CLIENT_BLOCKED`时，如果`client.bpop.timeout`这个超时时间非0且小于当前时间的话，那么就会通过`ublickedClient`这个接口接触客户端的阻塞状态。


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
上面这段代码，便是*Redis*调用`handleClientsBlockedOnKeys`函数的具体场景。现在我们以及列表对象上的阻塞操作为例，看一下这个函数的实现逻辑：
```c
void handleClientsBlockedOnKeys(void)
{
    while(listLength(server.ready_keys) != 0)
    {
        list *l;
        l = sever.ready_keys;
        server.ready_keys = listCreate();

        while(listLength(l) != 0) {
            listNode *ln = listFirst(l);
            readyList *rl = ln->value;

            dictDelete(rl->db->ready_keys, rl->key);

            robj *o = lookupKeyWrite(rl->db, rl->key);

            if (o != NULL && o->type == OBJ_LIST)
            {
                dictEntry *de;

                de = dictFind(rl->db->blocking_keys, rl->key);
                if (de) {
                    list *clients = dictGetVal(de);
                    int numclients = listLength(clients);

                    while (numclients--)
                    {
                        listNode *clientnode = listFirst(clients);
                        client *receiver = clientnode->value;

                        if (receiver->btype != BLOCKED_LIST)
                        {
                            listDelNode(clients, clientnode);
                            listAddNodeTail(clients, receiver);
                            continue;
                        }
                        ...
                    }
                }
            }

        }
        listRelease(l);
    }
}
```
这个函数在每次执行完客户端命令时，会遍历所有服务器全局数据之中已就绪的键`redisServer.ready_keys`，对于其中每一个就绪的键，会从对应的数据库键空间阻塞列表之中，找到阻塞等待这个键的客户端列表，按照先进先出的策略，将这个就绪的键通知给列表之中最早等待的客户端，这是一个相对公平的分配方案，之所以是相对公平，是因为如果客户端列表中头部的客户端可能等待有序集合类型的键，然而当前就绪键的类型是列表类型，对于这种情况，*Redis*便会就将列表头部的客户端转移到列表的尾部，这就是会造成一定程度不公平的原因所在。

另外通过代码，我们发现在`handleClientsBlockedOnKeys`实现逻辑中，实质上是对`redisServer.ready_keys`中的数据进行了两层嵌套的遍历，之所以这样做是因为列表对象具有一个**BRPOPLPUSH**命令，该命令会阻塞等待某个给定的列表对象类型键，从中弹出一个数据之后将弹出数据插入到另外一个列表对象类型的键之中。详细一个场景，一个客户端X阻塞等待列表A，并期待将列表A中弹出数据插入列表B；而另外一个客户端Y则阻塞等待列表B，而恰好列表A和B都是空的列表，这时如果向A列表中压入一个元素，则会导致两个客户端都被接触阻塞状态。这种连锁接触阻塞的逻辑便是通过代码中双层嵌套的遍历来实现的：
1. 第一轮遍历，将就绪的列表A通知给客户端X，客户端X弹出数据之后，将数据压入列表B，导致列表B被加入`redisServer.ready_keys`之中。
1. 第二轮遍历，将就绪的列表B通知给客户端Y。

最后，我们看一下，对解锁客户端的处理逻辑，前面我们已经介绍到，客户端在处于阻塞状态时，其客户端应用层缓存依然会接受连接上发送来的数据，那么在接触客户端的阻塞状态时，我们有必要对客户端应用层缓存之中的数据进行及时的处理。
```c
void unblockClient(client *c);
void queueClientForReprocessing(client *c);
```
*Redis*会通过`unblockClient`接口接触客户端的阻塞状态并清理相关的数据，其中最重要的一点是解除客户端`client.flags`上的`CLIENT_BLOCKED`阻塞状态标记，然后*Redis*会通过`queueClientForReprocessing`接口为客户端设置一个接触阻塞后的中间状态`CLIENT_UNBLOCKED`，并将这个客户端插入到服务器全局数据`redisServer.unblocked_clients`列表之中。

为了及时处理这些客户端中的数据，*Redis*会在每次进入事件循环之前，通过`beforeSleep`函数，来调用下面这个接口来执行数据处理：
```c
void processUnblockedClients(void);
```
这个函数接口会遍历`redisServer.unblocked_clients`中的每一个客户端，移除客户端上的`CLIENT_UNBLOCKED`状态标记，同时检测如果客户端的缓冲区之中存在未处理的数据时，调用前面我们介绍过的`processInputBuffer`接口来处理数据。


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*