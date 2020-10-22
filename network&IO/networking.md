# Redis客户端逻辑的支持
需要注意的一点是，文本中所指的客户端，是指在*Redis*服务器端用于表示客户端的数据结构，而非用户端操作的客户端*redis-cli*。关于客户端的数据结构主要被定义在*src/server.h*这个头文件之中，而相关的函数接口则是在*src/networking.c*源文件中实现的。

## 客户端相关数据结构
首先我们看一下，在*Redis*之中用户表示一个客户端的数据结构`client`：
```c
typedef struct client {
    uint64_t id;            /* 自助的客户端唯一标识ID */
    int fd;                 /* 客户端套接字连接文件描述符 */
    redisDb *db;            /* 指向当前客户端选择的数据库键空间的指针 */
    robj *name;             /* 客户端的名字 */
    ...
    time_t ctime;           /* 客户端被创建的时间戳 */
    time_t lastinteraction; /* 客户端上次交互的时间戳，用于判断超时 */
    time_t obuf_soft_limit_reached_time;
    int flags;              /* 客户端标记 */
    int authenticated;      /* 客户端认证标记，当服务器开启密码时有效 */
    ...
    dict *pubsub_channels;  /* 存储客户端订阅的频道 */
    list *pubsub_patterns;  /* 存储客户端订阅的模式 */

    listNode *client_list_node; /* 该客户端对应在服务器双端链表中的节点指针 */

    /* 静态输出缓存 */
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
    list *reply;            /* 一个双端链表，存储将要发送给客户端的反馈对象数据 */
    unsigned long long reply_bytes; /* 反馈链表中对象数据的总的字节数量 */
    size_t sentlen;         /* Amount of bytes already sent in the current
                               buffer or object being sent. */
} client;
```

下面我们来看一下，客户端结构体数据是如何被存储在*Redis*的服务器全局数据之中的：
```c
struct redisServer {
    ...
    list *clients;            
    list *clients_to_close;
    list *clients_pending_write;
    client *current_client;
    rax *clients_index;
    int client_paused;
    mstime_t clients_pause_end_time;
    uint64_t next_client_id;
    int protected_mode;
    ...
};
```
在这些字段之中：
1. `redisServer.clients`，*Redis*服务器中存储所有客户端结构体数据的双端链表。
1. `redisServer.clients_to_close`，*Redis*中需要被异步地关闭的客户端对象的双端链表。
1. `redisServer.clients_pending_write`，等待输出的客户端双端链表，其中的每个客户端的应用层输出缓冲区中都有数据等待写入到TCP的内核输出缓冲区里。
1. `redisServer.current_client`，*Redis*服务器当前执行命令的客户端结构体数据的指针。
1. `redisServer.clients_index`，使用一个基数树来存储客户端ID与客户端结构体数据的映射关系，用于通过ID快递检索客户端数据。
1. `redisServer.client_paused`，*Redis*服务器暂停标记，如果这个字段被设置，则所有的客户端都会被挂起。
1. `redisServer.clients_pause_end_time`，客户端被挂起状态的持续时间。
1. `redisServer.next_client_id`，自增的ID，用于标记客户端的唯一标识记录。
1. `redisServer.protected_mode`，*Redis*服务器是否开启保护模式，如果开启保护模式之后，服务器只能接受本地连接，外网无法访问。

## Redis客户端的创建

*Redis*会通过`createClient`这个接口从一个TCP连接套接字对应的文件描述符中创建一个新的*Redis*客户端对象：
```c
client *createClient(int fd) {
    client *c = zmalloc(sizeof(client));

    if (fd != -1) {
        anetNonBlock(NULL,fd);
        anetEnableTcpNoDelay(NULL,fd);
        if (server.tcpkeepalive)
            anetKeepAlive(NULL,fd,server.tcpkeepalive);
        if (aeCreateFileEvent(server.el,fd,AE_READABLE, readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        }
    }
    ...
    if (fd != -1) linkClient(c);
    initClientMultiState(c);
    return c;
```
`createClient`这个函数会为客户端对象分配内存，并初始化其相应的字段。如果文件描述符不为-1，说明这个是一个正式的客户端对象，那么*Redis*会将这个客户端连接设置为非阻塞模式，同时关闭这条连接上的Nagle算法，并为这个客户端连接在事件循环之中注册一个可读事件，同时设置其可读事件的事件处理函数`readQueryFromClient`用于在该连接可读时读取客户端的输入数据。最后会调用`linkClient`函数，将这个客户端添加到*Redis*服务器的全局数据之中：
```c
void linkClient(client *c) {
    listAddNodeTail(server.clients,c);
    c->client_list_node = listLast(server.clients);
    uint64_t id = htonu64(c->id);
    raxInsert(server.clients_index,(unsigned char*)&id,sizeof(id),c,NULL);
}
```
`linkClient`这个函数则会将客户端对象插入到服务器用于存储客户端对象的双端链表`redisServer.clients`的尾部，并将这个初始化客户端`client.client_list_node`这个字段，实现一个双向的引用。最后还会根据客户端的ID，将其插入到服务器的基数树之中`redisServer.clients_index`。

那么*Redis*服务器会在何种时机通过`createClient`函数接口来创建客户端对象呢？

*Redis*服务器为监听套接字在事件循环之中注册一个可读事件以及对应的事件处理函数，当事件循环之中监听套接字的可读事件被触发时，事件处理函数会接收新的连接，并使用新连接对应的文件描述符分配并初始化一个`client`结构体。

就如前面TCP套接字接口一文中介绍的，首先在*Redis*初始化的过程之中为监听套接字文件描述符注册可读事件以及对应的处理函数`acceptTcpHandler`：
```c
void initServer(void)
{
    ...
    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,acceptTcpHandler,NULL);
    }
    ...
}
```
而事件处理函数`acceptTcpHandler`则是通过`acceptCommonHandler`这个函数接口将`accept`系统调用返回的套接字转化为一个*Redis*客户端对象的：
```c
static void acceptCommonHandler(int fd, int flags, char *ip) {
    client *c;
    if ((c = createClient(fd)) == NULL) {
        ...
        close(fd);
        return;
    }
    ...
```

## 客户端命令执行过程
在了解客户端执行命令之前，我们先来了解一下*Redis*中客户端对象相关的字段：
```c
typedef struct client {
    ...
    sds querybuf;
    size_t qb_pos;
    sds pending_querybuf;
    size_t querybuf_peak;
    int argc;
    robj **argv;
    struct redisCommand *cmd, *lastcmd;
    int reqtype;
    int multibulklen;
    long bulklen;
    ...
} client;
```
在这些字段之中：
1. `client.querybuf`，用于收集客户端查询数据的应用层缓冲区。
1. `client.qb_pos`，用于标记缓冲区之中已经处理处理的数据的偏移量。
1. `client.pending_querybuf`，对应标记为master的客户端，此缓冲区代表我们从主服务器接收到的复制数据流中尚未应用的部分。
1. `client.querybuf_peak`，最近一段时间以来`client.querybuf`大小的峰值。
1. `client.argc`，用于存储客户端查询命令参数的个数。
1. `client.argv`，用于存储重缓冲区之中解析出的命令参数。
1. `client.cmd`，用于记录本次执行的命令实体。
1. `client.lastcmd`，记录上一次执行命令实体指针。
1. `client.reqtype`，客户端查询命令的请求类型。
1. `client.multibulklen`，剩下的没有被处理的批量参数的个数。
1. `client.bulklen`，多批量请求中批量参数的长度。


### 处理客户端对象应用区缓存
由于TCP是以面向连接的字节流形式来传输数据的，在这种情况下，TCP无法区分出两组传输数据之间的边界。因此就需要应用程序自己来处理解析TCP连接上接收到的数据，这样也就需要每个客户端对象都要有一个对应自己的TCP连接的应用层缓冲区。

首先*Redis*通过创建客户端对象时注册的可读事件处理函数`readQueryFromClient`将TCP连接上内核输入缓冲区中的数据通过`read`系统调用读入到客户端对象的应用层缓冲区`client.querybuf`之中。
```c
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata;
    int nread, readlen;
    size_t qblen;

    readlen = PROTO_IOBUF_LEN;
    ...
    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    nread = read(fd, c->querybuf+qblen, readlen);
    ...
    sdsIncrLen(c->querybuf,nread);
    c->lastinteraction = server.unixtime;
    ...
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        ...
        freeClient(c);
        return;
    }
    processInputBufferAndReplicate(c);
}
```
这里在完成`client.querybuf`的写入之后，会检查缓冲区的大小，如果超过了系统的限制`redisServer.client_max_querybuf_len`，这意味着这个客户端有可能处于一种异常的状态，这是需要将这个异常的客户端对象释放掉。而`readQueryFromClient`在完成将内核数据写入应用层缓冲区之后，会调用`processInputBufferAndReplicate`来解析缓冲区中的命令数据。

对于一个普通的客户端，`processInputBufferAndReplicate`实际上是调用`processInputBuffer`这个函数来处理应用层输入缓冲区中的数据将其解析成命令的形式。这里*Redis*所能处理的命令的由两种形式，通过客户端的`client.reqtype`字段来标记具体的命令协议类型，以命令`SET key newvalue`为例：
1. `#define PROTO_REQ_INLINE 1`，对于这种协议格式，前面的命令将会以`SET key newvalue\r\n`的实行在网络上进行传输，也就是说每一条命令都是以单行的形式编码的。
1. `#define PROTO_REQ_MULTIBULK 2`，而这种协议格式则会将命令编码成多行的形式，同时以`*`作为命令的起始字符，那么前面的命令使用这种协议格式的话，则编码后的内容为`*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$8\r\nnewvalue\r\n`。也就是说每一行便是一个字段，`*`以及后面的数字标记了该条命令参数的个数；`$`以及后面的数字则标记了后面参数数据的长度，基于上述的规则，便可以从客户端传送的数据之中解析出命令的参数内容。这种形式与前面`PROTO_REQ_INLINE`相比，由于携带了参数个数以及各个参数数据的长度信息，因此是二进制安全的，可以用于更为复杂的命令数据的传输。

而下面这两个函数便是分别用于解析上述两种协议格式的数据：
```c
int processInlineBuffer(client *c);
int processMultibulkBuffer(client *c);
```
这两个函数会按照上面的规则从客户端的应用层缓存`client.querybuf`之中尝试解析数据，并将解析出来的命令参数赋值给`client.argc`以及`client.argv`字段之中。如果应用层缓存之中的数据不足以构成一条完整的命令时，则上述两个函数会返回`C_ERR`，客户端会重新进入事件循环，继续等待客户端的输入。如果应用层缓冲区之中可以解析出完整的命令，那么函数会返回`C_OK`，不过需要注意的是，如果缓冲区中的数据可以解析成多个命令时，这两个函数每次调用也只会解析出一个命令。

那么现在可以看一下，解析客户端输入数据的`processInputBuffer`接口的实现细节。
```c
void processInputBuffer(client *c) {
    server.current_client = c;
    while (c->qp_pos < sdslen(c->querybuf)) {
        ...
        if (!c->reqtype) {
            if (c->querybuf[c->qb_pos] == '*') {
                c->reqtype = PROTO_REQ_MULTIBULK;
            } else {
                c->reqtype = PROTO_REQ_INLINE;
            }
        }

        if (c->reqtype == PROTO_REQ_INLINE) {
            if (processInlineBuffer(c) != C_OK) break;
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != C_OK) break;
        } else {
            serverPanic("Unknown request type");
        }
        if (c->argc == 0)
        {
            resetClient(c);
        } else {
            processCommand(c);
        }
    }
    if (server.current_client != NULL && c->qb_pos) {
        sdsrange(c->querybuf,c->qb_pos,-1);
        c->qb_pos = 0;
    }
    server.current_client = NULL;
}
```
通过上面的代码片段，我们可以发现推断出如果客户端应用层缓存之中累计了多条命令的数据，那么`processInputBuffer`函数会循环处理累积的每一条命令，直到所有可以被处理的命令都被处理完成为止，才会切换到下一个客户端的处理。

### 执行客户端命令
完成了客户端命令参数的解析之后，我们便可以下面这个函数完成命令的执行过程：
```c
int processCommand(client *c);
```
当执行完命令之后，如果客户端依然可用且合法，那么函数会返回`C_OK`；则函数会返回`C_ERR`，例如通过**QUIT**命令关闭释放客户端。这里需要注意的是，`client.argv`中的第一个参数便是所执行的客户端命令的命令名称。

对于*Redis*命令的具体细节，将会在后面的文章之中进行详细介绍。

## 客户端数据输出过程
在了解客户端输出过程前，我们还是依照惯例来看一下客户端结构体之中关于数据输出的相关字段。*Redis*所有发送给客户端的数据都会被写入到客户端对象的输出缓冲区之中，然后在通过TCP连接发送给用户。客户端对象中的输出缓冲区分为两种，第一种是一个固定长度的输出缓冲区，用于输出长度较小的数据：
```c
typedef struct client {
    ...
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
    ...
};
```
客户端的这个固定长度的缓冲区的大小为16K；另外客户端还有一个边长的输出缓冲区，而这个缓冲区则是以双端链表形式存储的，当输入过大固定缓冲区无法存储时，缓存输出数据。
```c
typedef struct client {
    ...
    list *reply;
    unsigned long long reply_bytes;
    size_t sentlen;
    ...
};
```
*Redis*通过下面这两个底层接口，将数据写到客户端的输出缓冲区之中：
```c
int _addReplyToBuffer(client *c, const char *s, size_t len);
void _addReplyStringToList(client *c, const char *s, size_t len);
```
这两个函数中`_addReplyToBuffer`用于将数据写入客户端的固定长度缓冲区之中，如果数据可以成功写入，那么函数会返回`C_OK`，如果数据长度超过了定长缓冲区的容量，或者在变长缓冲区之中已经存在数据，那么该函数不会讲述如写入，而函数会返回`C_ERR`。而函数`_addReplyStringToList`则是用于将数据写入客户端的变长缓冲区之中，同时在`_addReplyToBuffer`尝试向定长缓冲区写入失败后调用该函数。

在上述两个函数的基础上，*Redis*提供了一组用于向客户端输出数据的函数接口：
```c
void addReplyString(client *c, const char *s, size_t len);
void AddReplyFromClient(client *c, client *src);
void addReplyBulk(client *c, robj *obj);
void addReplyBulkCString(client *c, const char *s);
void addReplyBulkCBuffer(client *c, const void *p, size_t len);
void addReplyBulkLongLong(client *c, long long ll);
void addReply(client *c, robj *obj);
void addReplySds(client *c, sds s);
void addReplyBulkSds(client *c, sds s);
void addReplyError(client *c, const char *err);
void addReplyStatus(client *c, const char *status);
void addReplyDouble(client *c, double d);
void addReplyHumanLongDouble(client *c, long double d);
void addReplyLongLong(client *c, long long ll);
void addReplyMultiBulkLen(client *c, long length);
void addReplyHelp(client *c, const char **help);
```
上述这些函数的具体实现细节大致上都是一致的，因此我们不做过多的描述，我们仅就一个通用的客户端输出过程进行介绍。
所有的客户端输出逻辑都是一个异步的过程，`addReplyXXX`这一系列函数在将需要输出的数据写入客户端输出缓冲区之前，需要通过`prepareClientToWrite`这个函数接口对应的客户端对象加入服务器的输出列表`redisServer.clients_pending_write`，等待后续将数据异步地输出到网络上，例如`addReply`这个函数接口：
```c
void addReply(client *c, robj *obj)
{
    if (prepareClientToWrite(c) != C_OK) return;
    ...
    if (_addReplyToBuffer(c, obj->ptr, sdslen(obj->pyr)) != C_OK)
        _addReplyStringToList(c, obj->ptr, sdslen(obj->ptr));

    ...
}
```

那么*Redis*服务器是在何时将这些被挂起客户端的输出缓冲区中数据发送到网络上的呢？通过代码我们可以发现，*Redis*会在每次进入事件循环等待之前的`beforeSleep`接口中，尝试将`redisServer.clients_pending_write`里的客户端的输出数据发送到网络上：
```c
void before(struct aeEventLoop *eventLoop)
{
    UNUSED(eventLoop);
    ...
    handleClientsWithPendingWrites();
    ...
}
```
这里`handleClientsWithPendingWrites`便是*Redis*整个输出逻辑的核心，简单来说*Redis*当使用`epoll`系统调用的作为底层事件驱动的实现时，采用的是**水平触发**的模式，也就是`handleClientsWithPendingWrites`会遍历`redisServer.clients_pending_write`中的每一个客户端，对每一个客户端对象调用`writeToClient`函数接口，将客户端输出缓冲区之中的数据通过`write`系统调用发送到网络连接上。如果对应TCP连接的内核写缓冲区空间不足导致无法接收应用层缓存中数据的话，那么为这个客户端在事件循环之中注册一个可写事件以及相应的事件处理函数`sendReplyToClient`。当TCP连接的对端确认接收数据之后，该客户端会触发可写事件，此时*Redis*会调用`sendReplyToClient`函数，继续将客户端应用层缓冲区的数据通过`write`系统调用写入内核。
```c
int handleClientWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    
    listRewind(server.clients_pending_write, &li);
    while ((ln = listNext(&li))){
        client *c = listNodeValue(ln);
        listDelNode(server.clients_pending_write, ln);
        ...
        if (writeToClient(c->fd, c, 0) == C_ERR) continue;
        
        if (clientHasPendingReplies(c))
        {
            ...
            aeCreateFileEvent(server.el, c->fd, ae_flags, sendReplyToClient, c);
            ...
        }
    }
    ...
}
```
## 客户端关闭释放过程

### 客户端的异步释放
```c
void freeClientAsync(client *c);
void freeClientInAsyncFreeQueue(clinet *c);
void asyncCloseClientOnOutputBufferLimitReached(client *c);
```
在服务器全局变量中`redisServer.clients_to_close`存储了需要被异步地关闭的客户端对象的指针。通过`freeClientAsync`这个函数接口，*Redis*可以将一个客户端对象数据插入`redisServer.clients_to_close`这个双端链表之中。而在服务器的心跳函数`serverCron`中调用`freeClientInAsyncFreeQueue`这个接口，将`redisServer.clients_to_close`中的客户端依次进行释放。主要需要执行异步释放客户端的情况有两种：
1. 当调用`_addReplyStringToList`向客户端的应用层输出缓存中写入数据之后，如果该客户端的输出缓存的长度超过了系统的最大上限，那么`asyncCloseClientOnOutputBufferLimitReached`函数会将这个客户端加入到`redisServer.clients_to_close`链表之中。
1. 在`writeToClient`函数中，如果向事件循环中注册对应客户端的可写事件失败，那么也会调用`freeClientAsync`接口，将客户端对象的指针加入到服务器的异步释放队列之中。

### 客户端的同步释放
```c
int clientsCronHandleTimeout(client *c, mstime_t now_ms);
void freeClient(client *c);
```
*Redis*服务器的同步释放逻辑之中，主要有两种应用场景：
1. 首先便是对长时间没有操作的客户端，*Redis*会在服务器心跳中，通过`cleintsCronHandleTimeout`函数来计算每一个客户端的当前空闲时间，通过当前时间与`client.lastinteraction`时间戳的差值进行计算。如果这个空闲时间大于服务器设置的最大客户端空闲时间上限`redisServer.maxidletime`，那么会调用`freeClient`函数接口直接将这个空闲的客户端对象释放掉。
1. 另外一种同步的释放方式，是客户端主动调用**QUIT**指令执行关闭流程。当客户端执行**QUIT**命令时，*Redis*会为这个客户端对象的标志字段上设置一个`CLIENT_CLOSE_AFTER_REPLY`掩码，在*Redis*将这个客户端的所有应用层缓存数据都发送到网络后，会调用`freeClient`接口将这个客户端对象同步地释放掉。
```c
int writeToClient(int fd, client *, int handler_installed)
{
    ...
    if (!clientHasPendingReplies(c))
    {
        if (c->flags & CLIENT_CLOSE_AFTER_REPLY) {
            freeClient(c);
            return C_ERR;
        }
    }
    return C_OK;
}
```


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*