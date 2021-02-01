# Redis客户端C语言库

Hiredis是*Redis*数据库的一个最小的C语言客户端库。这里需要注意一点的是，本文所提到的客户端，是指用户可以通过它连接*Redis*服务器，并将服务器发送查询命令，并获取命令返回数据的客户端。而不是前面所介绍的在服务器端用于表示与用户之间网络连接的客户端对象`client`。

之所以说Hiredis是一个最小化的C语言客户端库，是因为Hiredis仅仅提供了对于*Redis*客户端协议最低限度的支持。Hiredis只支持二进制安全的*Redis*协议，因此可以支持任何版本号大于*1.2*的*Redis*服务器。Hiredis支持多种API，其中包括同步API以及异步API，本文将对同步API部分的内容进行讲解。本文前半部分，翻译自*Redis*官方给出的说明文档。

## 同步API概述
若要使用同步API，仅需要引入下面几个函数接口便可以：
```c
redisContext *redisConnect(const char *ip, int port);
void *redisCommand(redisContext *c, const char *format, ...);
void freeReplyObject(void *reply);
```

### 建立连接
```c
typedef struct redisContext {
    int err;
    char errstr[128];
    int fd;
    int flags;
    char *obuf;
    redisReader *reader;

    enum redisConnectionType connecion_type;
    struct timeval *timeout;
    struct {
        char *host;
        char *source_addr;
        int port;
    } tcp;

    struct {
        char *path;
    } unix_sock;

} redisContext;
```

`redisConnect`函数用户创建一个`redisContext`对象。这里的`redisContext`结构体在Hiredis库之中用户存储与维护与*Redis*服务器之间连接的状态信息。当这个网络连接处于错误状态的时候，那么`redisContext.err`这个字段将会被设置为响应的错误码，而`redisContext.errstr`字段将存储对应的错误信息。关于错误信息的更多内容，可以查看后续的部分。因此这也就要求我们在通过`redisConnect`接口获取到一个`redisContext`对象之后，需要检查一下`redisContext.err`字段，以确认连接是否成功被建立。
```c
redisContext *c = redisConnect("127.0.0.1", 6379);
if (c == NULL || c->err) {
    if (c) {
        printf("Error: %s\n", c->errstr);
        // handle error
    } else {
        printf("Can't allocate redis context\n");
    }
}
```
这里需要注意的是，`redisContext`对象不是线程安全的对象。

### 发送命令
Hiredis库中提供了多种向*Redis*服务器发送命令的方式。其中第一种便是`redisCommand`接口，这个接口采用了类似于`printf`的形式来进行命令的发送，我们可以采用如下最简单的格式来向*Redis*服务器发送命令：
```c
reply = redisCommand(context, "SET foo bar");
```

使用`%s`这个通配符，我们可以像命令字符串数据之中格式化地写入一段C语言风格的字符串，字符串的长度有`strlen`函数自动获取：
```c
reply = redisCommand(context, "SET foo %s", value);
```

当我们需要像命令数据之中传递一段二进制安全的字符串，那么就需要是使用`%b`这个通配符，同时使用指向字符串的指针以及这段字符串的长度：
```c
reply = redisCommand(context, "SET foo %b", value, (size_t) valuelen);
```

与`redisCommand`命令类似，函数`redisCommandArgv`同样可以被用来发送查询命令，这个函数具有如下的函数原型：
```c
void *redisCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);
```
这个函数通过`argc`参数来接收查询命令参数的个数，而查询命令每个参数的具体数据则是通过字符串数组`argv`来传递的，另外一个整数数组`argvlen`则是用于记录每个参数数据的长度。当然如果`argv`数组之中的每一个参数都是C语言风格的字符串时，我们可以使用`strlen`来获取参数数据的长度，那么`argvlen`可以传递为`NULL`。而当我们需要传递二进制安全的参数时，就需要通过`argvlen`来指定参数的长度。

### 返回数据
```c
typedef struct redisReply {
    int type;
    long long integer;
    size_t len;
    char *str;
    size_t elements;
    struct redisReply **element;
} redisReply;
```

`redisCommand`成功执行之后的返回值，用一个`redisReply`对象来保存。当命令执行出现错误时，`redisCommand`会返回`NULL`指针，同时会在`redisContext.err`字段上设置对应的错误码。一旦`redisContext`对象上出现了错误，那么用户将不能继续在这个对象上继续发送消息。如果后续还有向服务器发送查询命令的请求，则应该重新建立一个`redisContext`对象来使用。

在返回数据`redisReply`对象之中，`redisReply.type`字段用于表示接收到的返回数据的类型：
- **`REDIS_REPLY_STATUS`**
    - 这个类型对应状态信息的返回。具体的返回状态信息会被存储到`redisReply.str`字段之中，这个字符串的长度则会被存储在`redisReply.len`字段之中。
- **`REDIS_REPLY_ERROR`**
    - 这个类型对应错误信息的返回。具体返回错误信息的存储与`REDIS_REPLY_STATUS`相同。
- **`REDIS_REPLY_INTEGER`**
    - 这个类型对应一个整数数据的返回。返回的整数数据存储在`redisReply.integer`字段之中。
- **`REDIS_REPLY_NIL`**
    - 这个类型对应返回了一个空数据。对于这个类型没有可供读取的返回数据。
- **`REDIS_REPLY_STRING`**
    - 这个类型对应块数据（字符串）的返回。具体返回的数据会被存储到`redisReply.str`字段之中，这个字符串的长度则会被存储在`redisReply.len`字段之中。
- **`REDIS_REPLY_ARRAY`**
    - 这个类型对应多块数据的返回。多块返回数据之中子数据的个数会被存储到`redisReply.elements`字段上，在`redisReply`上的每一个字数据依然是一个`redisReply`对象，这个嵌套的`redisReply`对象可以通过`redisReply.elements[index]`进行访问。

返回数据`redisReply`需要使用`freeReplyObject()`函数来释放。需要注意的是，这个函数会迭代地释放嵌套的`sub-reply`对象数据，因此我们不要用自己来主动释放这些`sub-reply`数据。

### 流水线

为了解释Hiredis在同步API之中对于流水线的支持，我们需要了解Hiredis发送查询命令的内部执行流程。

当调用`redisCommand`系列函数发送*Redis*查询命令时，Hiredis首先将命令按照*Redis*协议的格式进行格式化。接下来格式化好的命令数据将被写入`redisContext`的输出缓冲区之中。这个输出缓冲区本质上是一个动态字符串，它可以存储任意数量的查询命令。当数据被写入输出缓冲区之后，Hiredis将会调用`redisGeReply`函数，这个函数将按照下面的逻辑进行执行：
1. 当`redisContext`对象中的输入缓冲区非空时：
    - 尝试从输入缓冲区之中解析初一个`redisReply`数据，并返回这个`redisReply`。
    - 如果无法解析出完整的`redisReply`数据，那么会执行下面的步骤2。
1. 当`redisContext`对象中的输入缓冲区为空时：
    - 将整个输出缓冲区之中的数据，通过`write`系统调用全部写入Socket上。
    - 从Socket网络连接上调用`read`系统调用获取返回数据，直到可以解析出一个完整的`redisReply`数据为止。

`redisGetReply`函数作为Hiredis库的API的一部分，可以在连接上有返回数据时调用这个函数`redisGetReply`来获取返回数据。对于流水线命令，唯一需要做的就是将命令数据填充到`redisContext`的输出缓冲区之内。对于这种情况，我们可以使用下面的这两个函数来实现填充数据但不发送数据的行为：
```c
void redisAppendCommand(redisContext *c, const char *format, ...);
void redisAppendCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);
```
当调用这两个函数将查询命令数据写入输出缓冲区若干次后，再调用`redisGetReply`函数可以用来将缓冲区之中的命令数据一次性地发送到网络上并接收后续查询命令的返回数据。这个函数的返回值为`REDIS_OK`或者`REDIS_ERR`，当函数返回`REDIS_ERR`时，意味着在读取返回数据时出现错误，这时将会在`redisContext.err`字段上设置出现错误的具体原因。

下面这个例子向我们展示了一个简单的流水线代码的编写范式：
```c
redisReply *reply;
redisAppendCommand(context,"SET foo bar");
redisAppendCommand(context,"GET foo");
redisGetReply(context,&reply); // reply for SET
freeReplyObject(reply);
redisGetReply(context,&reply); // reply for GET
freeReplyObject(reply);
```

除了查询命令的流水线机制外，`redisGetReply`还可以在发布/订阅机制之中实现一个阻塞的订阅者：
```c
reply = redisCommand(context,"SUBSCRIBE foo");
freeReplyObject(reply);
while(redisGetReply(context,&reply) == REDIS_OK) {
    // consume message
    freeReplyObject(reply);
}
```

### 错误信息

当一次函数调用没有执行成功，那么`redisContext.err`字段将会被设置成一个非零的错误码，用于标记错误信息：
- **`REDIS_ERR_IO`**：

    **I/O**错误发生在创建连接，尝试调用`write`系统调用向连接之中写数据，或者调用`read`系统调用从连接之中读取数据时。如果我们在代码之中包含了`errno.h`头文件，那么我们可以通过`errno`这个全局变量来查找出错误原因。

- **`REDIS_ERR_EOF`**：

    当`read`系统调用返回空数据的时候，意味着服务器一段关闭了网络连接。

- **`REDIS_ERR_PROTOCOL`**：

    在解析协议数据时出错

- **`REDIS_ERR_OTHER`**：

    任何可能出现的其他错误，现在唯一被用到的场景便是连接一个指定的域名，地址却无法被解析。

当`redisContext.err`字段之中，出现了上述的错误码之后，`redisContext.errstr`字段相应地将会存储对应错误描述字符串。

### 清理数据

当我们需要断开连接并释放`redisContext`对象时，我们可以使用如下的函数：
```c
void redisFree(redisContext *c);
```
这个函数会立即关闭连接，然后释放`redisContext`对象之中已经被创建的数据。

## 同步API代码实现

### 建立连接
在*deps/hiredis/net.h*以及*deps/hiredis/net.c*这两个文件之中定义了Hiredis库中关于网络连接的一些接口。首先在*deps/hiredis/net.c*源文件之中，定义了若干关于网络操作的静态函数接口，由于面前对于*Redis*服务器的底层网络API已经做过介绍，因此这里仅对相关函数进行一个简单的罗列：
|静态函数接口|功能|
|------------|---|
|`int redisCreateSocket(redisContext *c, int type)`|为`redisContext`对象创建一个Socket文件描述符|
|`void redisContextCloseFd(redisContext *c)`|关闭`redisContext`对象上的Socket文件描述符|
|`int redisSetReuseAddr(redisContext *c)`|设置`redisContext`对象连接地址可重用|
|`int redisSetBlocking(redisContext *c, int blocking)`|设置`redisContext`对象连接的阻塞状态|
|`int redisSetTcpNoDelay(redisContext *c)`|关闭`redisContext`对象连接上的**Nagle**设置|
|`int redisContextTimeoutMsec(redisContext *c, long *result)`|获取`redisContext`对象上设置的超时间事件毫秒数|

除了上面这些基础的静态函数接口之外，*deps/hiredis/net.c*源文件之中还定义了两个比较重要的静态函数接口，
首先是等待连接就绪的函数接口：
```c
int redisContextWaitReady(redisContext *c, long msec)
{
    struct pollfd wfd[1];
    wfd[0].fd = c->fd;
    wfd[0].events = POLLOUT;

    if (errno == EINPROGRESS)
    {
        int res;
        if ((res = poll(wfd, 1, msec) == -1))
        {
            ...
        }
        ...
        return REDIS_OK;
    }
    return REDIS_ERR;
}
```
这个函数的主要用途便是调用`poll`系统调用等待`redisContext`连接上的可写事件。我们在前面介绍*Redis*底层API的时候曾经讲解过的非阻塞`connect`操作。而在Hiredis库之中，客户端调用API向服务器发起连接都是采用非阻塞地`connect`进行的，因此我们需要一个接口来检查连接上是否触发了可写事件。`redisContextWaitReady`这个接口便是为了这个目的而实现的，如果`redisContextWaitReady`函数从`poll`系统调用的阻塞之中返回，且没有返回错误，那么说明与服务器之间的网络连接已经完全建立；否则便说明连接建立失败。

另外一个接口便是客户端主动向服务器发起连接的函数：
```c
int _redisContextConnectTcp(redisContext *c, const char *addr, int port, const struct timeval *timeout, const char *source_addr)
{
    int blocking = (c->flags & REDIS_BLOCK);
    ...
    s = socket(p->ai_family,p->ai_socktype,p->ai_protocol);
    c->fd = s;
    redisSetBlocking(c,0);
    if (connect(s,p->ai_addr,p->ai_addrlen) == -1) {
        ...
        else if (errno == EINPROGRESS && !blocking) {

        }
        ...
        else
        {
            if (redisContextWaitReady(c,timeout_msec) != REDIS_OK)
                goto error;
        }
    }
    if (blocking && redisSetBlocking(c,1) != REDIS_OK)
        goto error;
}
```
通过上面这个代码片段，我们可以发现不论`redisContext.flags`字段上是否携带了`REDIS_BLOCK`标记，客户端与服务器之间建立连接的过程都是采用非阻塞`connect`的调用来进行的，唯一区别在于同步的API在非阻塞`connect`之后，会通过`redisContextWaitReady`接口阻塞地等待连接建立；而在异步API之中，则不会通过`redisContextWaitReady`函数接口等待连接建立，取而代之的会将这个`redisContext`对象加入事件循环之中，通过回调函数来建立连接。

在*deps/hiredis/hiredis.h*以及*deps/hiredis/hiredis.c*这两个文件之中，定义了多干个用于发起连接的API：
```c
redisContext *redisConnect(const char *ip, int port);
redisContext *redisConnectWithTimeout(const char *ip, int port, const struct timeval tv);
redisContext *redisConnectNonBlock(const char *ip, int port);
redisContext *redisConnectBindNonBlock(const char *ip, int port, const char *source_addr);
redisContext *redisConnectBindNonBlockWithReuse(const char *ip, int port, const char *source_addr);
```
在这几个API之中`redisConnect`以及`redisConnectWithTimeout`用于在同步API之中发起连接，而另外三个API则是用于在异步API之中向服务器发起连接。

### 命令格式化
Hiredis库对于查询命令的格式化以及发送的相关逻辑代码主要定义在*deps/hiredis/hiredis.h*以及*deps/hiredis/hiredis.c*这两个代码文件之中。

在`redisContext`这个结构体之中，`redisContext.obuf`字段代表着这个连接所对应的应用层输出缓冲区，所有的查询命令都会首先被写入这个输出缓冲区，然后再通过`write`系统调用写入网络。Hiredis给出了一组用于对输出命令进行格式化的函数接口，这组接口会按照*Redis*的协议格式对客户端给出的查询命令数据进行格式化：
```c
int redisvFormatCommand(char **target, const char *format, va_list ap);
int redisFormatCommand(char **target, const char *format, ...);
int redisFormatCommandArgv(char **target, int argc, const char **argv, const size_t *argvlen);
int redisFormatSdsCommandArgv(sds *target, int argc, const char ** argv, const size_t *argvlen);
```

而客户端对于流水线机制的支持，即`redisAppendCommand`系列函数便都是借用上面的这些函数接口，将格式化后的查询命令数据写入到应用层缓冲区`redisContext.obuf`之中来实现的。以`redisvAppendCommand`这个函数接口为例：
```c
int redisvAppendCommand(redisContext *c, const char* format, va_list ap)
{
    char * cmd;
    int len;
    len = redisvFormatCommand(&cmd, format, ap);
    ...
    __redisAppendCommand(c, cmd, len);

    free(cmd);
    return REDIS_OK;
}
```
这里我们可以看到，`redisvAppendCommand`这个函数接口首先会通过`redisvFormatCommand`将查询命令的数据格式化到一段内存`cmd`之中，在将这段存储着格式化命令数据的内存`cmd`追加到`redisContext.obuf`输出缓冲区之中。

而非流水线的同步命令查询的`redisCommand`系列接口，则是在`redisAppendCommand`系列函数接口的基础上来实现的，其本质就是将格式化后的查询命令追加到`redisContext.obuf`之后，立即将缓冲区之中的查询命令数据发送到网络，并同步地等待查询命令返回数据，以`redisvCommand`接口为例：
```c
void *redisvCommand(redisContext *c, const char *format, va_list ap)
{
    if (redisvAppendCommand(c, format, ap) != REDIS_OK)
        return NULL;
    return __redisBlockForReply(c);
}
```

### 数据的发送与接收
在底层将`redisContext`输出缓冲区之中的数据发送出去则是通过下面这个接口来实现的：
```c
int redisBufferWrite(redisContext *c, int *done);
```
这个函数会尝试调用`write`系统调用，将`redisContext.obuf`之中的数据写入网络的内核输出缓冲区。如果输出失败，函数会返回`REDIS_ERR`，否则会返回`REDIS_OK`。同时，`redisContext.obuf`之中的数据是否已经全部被发送，则会通过`done`参数进行返回。

而将数据从内核缓冲区读取到应用层缓冲区的工作，则是由`redisBufferRead`这个函数来完成的：
```c
int redisBufferRead(redisContext *c);
```
除了将数据从内核缓冲区读取到应用层缓冲区之外，`redisBufferRead`这个接口还会检测服务器端是否主动断开连接。如果`read`系统调用返回，但是读取的字节数为0，那么则意味着服务器端主动将连接断开。此时，会将`redisContext.err`这个字段设置为`REDIS_ERR_EOF`用以标记连接已经被断开。

### 返回数据解析
关于返回数据解析的主要逻辑，都被定义在*deps/hiredis/read.h*以及*deps/hiredis/read.c*这两个文件之中。这里Hiredis定义了一个`redisReader`的数据结构用于表示应用层缓冲区，同时也负责返回数据的解析工作：
```c
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
在`redisContext`对象之中负责接收与解析返回数据的`redisContext.reader`字段表示一个`redisReader`类型的数据。

同时Hiredis还为这个`redisReader`类型定义几个公开的API，用于实现数据的接收与解析。

```c
int redisReaderFeed(redisReader *r, const char *buf, size_t len);
```
`redisReaderFeed`这个接口便是用于将一段从内核缓冲区之中读取到的数据填充到`redisReader.buf`这个缓冲区之中，`redisContext`用于从网络之中读取数据的接口`redisBufferRead`便是用`redisReaderFeed`这个函数实现数据的拷贝的。

在`redisReader`获取到数据之后，便可以通过下面这个函数来实现数据的解析：
```c
int redisReaderGetReply(redisReader *r, void **reply);
```
`redisReaderGetReply`这个函数会尝试解析`redisReader.buf`之中的数据，如果数据足够组成一个`redisReply`数据，那么会通过`reply`参数返回`redisReply`数据的指针。

以上便是关于Hiredis之中同步API的一个简要的介绍，谢谢大家阅读。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*