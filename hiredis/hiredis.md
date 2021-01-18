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
### 返回数据

### 流水线

### 错误信息

### 清理数据

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*