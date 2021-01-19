# Redis客户端异步API

## 异步API概述
Hiredis之中提供了一组异步API，可以很容易的配合各种事件库进行工作。在*deps/hiredis/examples*目录下，提供了两个例子，展示了Hiredis是如何于libev库以及libevent库配合工作的。

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



























***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*