# Redis客户端返回数据的解析

Hiredis库提供了一组用于解析命令返回数据的**API**，应用这个组**API**我们可以实现与更高级语言的绑定，在介绍完了Hiredis之中同步**API**以及异步**API**之中，本文将就Hiredis之中的数据解析**API**进行介绍。

## 解析API概述

```c
redisReader *redisReaderCreate(void);
void redisReaderFree(redisReader *r);
int redisReaderFeed(redisReader *r, const char *buf, size_t len);
int redisReaderGetReply(redisReader *r, void **reply)
```

我们可以理解为`redisReader`是一个集成了应用层缓冲区的数据解析器。在不考虑`redisReader`对象结构体的细节的情况下，上面4个函数便是Hiredis之中解析返回数据的主要接口。

### API用法

函数`redisReaderCreate`用于创建一个`redisReader`的对象，这个`redisReader`用于存储尚未被解析的返回数据，以及数据协议解析的过程状态。

通过`redisReaderFeed`函数接口，Hiredis可以将输入数据（通常是同过Socket网络套接字获得的）存储到`redisReader`的内部缓冲区之中。这个函数会将数据从`buf`的内存之中拷贝到`redisReader`之中。

当Hiredis调用`redisReaderGetReply`函数的时候，`redisReader`之中缓存的序列化数据解析成`redisReply`数据，这个函数接口将会返回一个整数的状态返回值，并通过`reply`参数将`redisReply`结果返回。函数的返回值可能是`REDIS_OK`或者`REDIS_ERR`，后一种返回值表示在解析的过程之中发生了某些错误，可能是协议格式的错误，也可能是内存的错误。

当一个`redisReader`对象不在被需要的时候，我们可以调用`redisReaderFree`函数接口将对象释放。

### 最大缓冲区

无论是直接使用`redisReader`的解析API，还是在正常的*Redis*上下文之中间接地只用`redisReader`，`redisReader`结构体都将会使用一段缓冲区来累积收集来自*Redis*服务器的数据。

当缓冲区之中的数据被清空时并且缓冲区的未使用的内存大小超过了16K字节的时候，为了防止内存的浪费，缓冲区将会被销毁并重新创建。

然而在处理过大缓冲区的销毁时，有可能会显著地影响系统的性能，因此我们可以是当地修改最大的未使用缓冲区大小的上限。我们也可以将缓冲区的这个上限设置为0，这意味着未使用缓冲区的大小不存在上限，缓冲区将永远不会被释放。

## 数据解析的代码实现

关于返回数据解析的主要逻辑，都被定义在*deps/hiredis/read.h*以及*deps/hiredis/read.c*这两个文件之中。

```c
typedef struct redisReadTask {
    int type;
    int elements;
    int idx;
    void *obj;
    struct redisReadTask *parent;
    void *privdata;
} redisReadTask;

typedef struct redisReplyObjectFunctions {
    void *(*createString)(const redisReadTask*, char*, size_t);
    void *(*createArray)(const redisReadTask*, int);
    void *(*createInteger)(const redisReadTask*, long long);
    void *(*createNil)(const redisReadTask*);
    void (*freeObject)(void*);
} redisReplyObjectFunctions;

typedef struct redisReader {
    int err;
    char errstr[128];

    char *buf;
    size_t pos;
    size_t len;
    size_t maxbuf;

    redisReadTask rstack[9];
    int ridx;
    void *reply;

    redisReplyObjectFunctions *fn;
    void *privdata;
} redisReader;
```

在`redisContext`对象之中负责接收与解析返回数据的`redisContext.reader`字段表示一个`redisReader`类型的数据，上面便是这个`redisReader`数据结构的定义：

- `redisReader.err`以及`redisReader.errstr`，解析器的错误标记，以及对应的错误信息。
- `redisReader.buf`，这是一个`sdd`动态字符串作为解析器的输入缓冲区。
- `redisReader.pos`，这是一个游标，用于记录当前解析器处理的位置。
- `redisReader.len`，输入缓冲区之中数据的大小。
- `redisReader.maxbuf`，这个字段便是前面我们介绍的缓冲区的最大长度上限。
- `redisReader.reply`，当解析出一个`redisReply`对象的话，这个字段用于指向这个`redisReply`对象。
- `redisReader.fn`，这是一组函数接口，包含了各个类型返回数据的解析函数。

*Redis*查询命令的返回数据有可能是一组多块`multi-block`数据，同时还有可能出现嵌套的数据，因此返回数据本质上是一个树形的数据结构。而从*Redis*服务器发来的返回数据则是一维的序列化数据，从一维数据转化为树形的数据结构，则需要我们使用一个辅助的栈结构来实现：

- `redisReader.rstack`，辅助的栈结构，用于实现数据的解析。
- `redisReader.ridx`，这个字段则用于记录当前正在运行的`redisReadTask`。

***

![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*

