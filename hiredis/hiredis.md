# Redis客户端C语言库

Hiredis是*Redis*数据库的一个最小的C语言客户端库。这里需要注意一点的是，本文所提到的客户端，是指用户可以通过它连接*Redis*服务器，并将服务器发送查询命令，并获取命令返回数据的客户端。而不是前面所介绍的在服务器端用于表示与用户之间网络连接的客户端对象`client`。

之所以说Hiredis是一个最小化的C语言客户端库，是因为Hiredis仅仅提供了对于协议最低限度的支持。Hiredis只支持二进制安全的*Redis*协议，因此可以支持任何版本号大于*1.2*的*Redis*服务器。Hiredis支持多种API，其中包括同步API以及异步API，本文将对同步API部分的内容进行讲解。

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

`redisConnect`函数用户创建一个`redisContext`对象。这里的`redisContext`在Hiredis库之中用户存储与维护与*Redis*服务器之间连接的状态信息。当这个网络连接处于错误状态的时候，那么`redisContext.err`这个字段将会被设置为响应的错误码，而`redisContext.errstr`字段将存储对应的错误信息。关于错误信息的更多内容，可以查看后续的部分。同时这也就要求我们在通过`redisConnect`接口获取到一个`redisContext`对象之后，需要检查一下`redisContext.err`字段，以确认连接是否成功被建立。
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
在*deps/hiredis/net.h*以及*deps/hiredis/net.c*这两个文件之中定义了Hiredis库中关于网络连接的一些接口。首先在*deps/hiredis/net.c*源文件之中，定义了若干关于网络操作的静态函数接口，由于面前对于*Redis*服务器的底层网络API已经做过介绍，因此小结进对相关函数进行一个简单的罗列：
|静态函数接口|功能|
|------------|---|
|`int redisCreateSocket(redisContext *c, int type)`|为`redisContext`对象创建一个Socket文件描述符|
|`void redisContextCloseFd(redisContext *c)`|关闭`redisContext`对象上的Socket文件描述符|
|`int redisSetReuseAddr(redisContext *c)`|设置`redisContext`对象连接地址可重用|
|`int redisSetBlocking(redisContext *c, int blocking)`|设置`redisContext`对象连接的阻塞状态|
|`int redisSetTcpNoDelay(redisContext *c)`|关闭`redisContext`对象连接上的**Nagle**设置|
|`int redisContextTimeoutMsec(redisContext *c, long *result)`|获取`redisContext`对象上设置的超时间事件毫秒数|

除了上面这些基础的静态函数接口之外，*deps/hiredis/net.c*源文件之中还定义了两个比较重要的静态函数接口
### 命令格式化与发送

### 返回数据解析


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*