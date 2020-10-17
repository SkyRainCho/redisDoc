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

对于一个普通的客户端，`processInputBufferAndReplicate`实际上是调用`processInputBuffer`这个函数来处理应用层输入缓冲区中的数据将其解析成命令的格式
### 执行客户端命令

## 客户端数据输出过程


## 客户端关闭释放过程


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*