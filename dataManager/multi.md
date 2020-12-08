# Redis事务支持

*Redis*提供了对于**事务**的支持，通过**事务**客户端可以一次性地执行多条命令，并保证**事务**中的命令都会被按序执行，同时不会被其他客户端的命令中途打断。

## 事务概述
*Redis*对于**事务**的支持，是通过**MULTI**、**EXEC**、**DISCARD**以及**WATCH**这四个命令来实现的。

在上述的四条命令之中，**MULTI**命令用于将执行命令的客户端设置为事务开启的状态，这个命令一定会执行成功，不会返回错误。在执行**MULTI**命令开启了事务之后，后续用户的命令不会被立即执行，而是被存入一个缓存队列。

当我们需要执行这个**事务**缓存队列中的命令时，我们可以通过**EXEC**命令来提交**事务**。当我们执行**EXEC**命令之后，**事务**缓冲队列之中的命令会被按照顺序依次执行，并且这个多条命令执行的过程不会被其他的客户端命令所打败。当**事务**命令队列中的命令存在语法错误时，例如缺少必要参数时，整个**事务**中的命令将都不会被执行。不过如果**事务**命令队列中的命令存在运行时错误时，例如针对错误的对象类型执行命令，这种情况**事务**仅仅会跳过这条命令，同时会继续执行后续的命令。

当我们在**事务**被**EXEC**命令提交之前，不希望执行整个**事务**，我们可以通过**DISCARD**命令来中断**事务**，在中断之前被加入到**事务**队列中的命令将都不会被执行。

**WATCH**命令，则可以用于监控一个或者多个键，在**事务**被提交前，如果被**WATCH**命令监控的键有被修改，则**事务**在**EXEC**命令被提交时，将会返回错误，所有**事务**中的命令都不会被执行。

## 事务相关数据结构
在*src/server.h*头文件中为了支持**事务**操作，为了客户端定义了三个特殊的状态，可以被设定在`client.flags`字段上：
1. `CLIENT_MULTI`，这个状态表示，对应的客户端当前正处于**事务**的状态，拥有这个状态的客户端，在收到普通客户端命令时，会将命令加入**事务**缓存队列，而不是执行命令；通过**MULTI**命令，可以为客户端设置这个状态；通过**EXEC**命令提交**事务**之后或者**DISCARD**命令中断**事务**之后，客户端会移除这个状态。
1. `CLIENT_DIRTY_EXEC`，这个状态表示，该处于**事务**中的客户端对象，曾经向**事务**队列中添加了一条有语法错误的命令，拥有这个状态的客户端会在**EXEC**命令提交事务时，返回失败，不执行**事务**中的命令。
1. `CLIENT_DIRTY_CAS`，这个状态表示，该客户端通过**WATCH**命令监控的若干个键中，有被修改的键。拥有这个标记的客户端在执行**EXEC**命令提交事务时，也会返回失败，而不会去执行**事务**中的命令。

*Redis*为了维护**事务**中的缓存命令队列，在客户端对象`client`之中定义了下面这些数据结构：
```c
typedef struct multiCmd {
    robj **argv;
    int argc;
    struct redisCommand *cmd;
} multiCmd;

typedef struct multiState {
    multiCmd *commands;
    int count;
    int cmd_flags;
    ...
} multiState;

struct client {
    ...
    multiState mstate;
    ...
}；
```
这其中，`multiCmd`用于表示一条被缓存的**事务**命令，每条命令中都包含了命令参数列表`multiCmd.argv`、命令参数个数`multiCmd.argc`以及命令的执行体`multiCmd.cmd`；而存储在客户端对象中的`client.mstate`则是用于表示当前客户端的**事务**状态；`multiState`会使用动态分配的数组`multiState.commands`以及队列长度`multiState.count`用来维护命令队列。

对于**WATCH**命令所监控的键，*Redis*使用两个数据结构来存储：
1. 在客户端对象中维护了一个双端链表对象`client.wacthed_keys`，表示这个客户端通过**WATCH**命令监控了哪些键，这个链表中的元素为：
```c
typedef struct watchedKey{
    robj *key;
    redisDb *db;
}watchedKey;
```
2. 在数据键空间中维护了一个哈希表对象`redisDb.watched_keys`，这个哈希表中的键为数据库键空间之中的键，而值则是一个存储了客户端对象指针的双端链表，用于存储当前数据库键空间中键以及监控这个键的所有客户端构成的映射关系。


## 事务相关代码逻辑
```c
void multiCommand(client *c);
```
客户端执行**MULTI**命令，其处理函数`multiCommand`的逻辑比较简单，会为客户端设置一个`CLIENT_MULTI`的状态标记。这表示当前这个客户端开启了**事务**的处理。

```c
void queueMultiCommand(client *c);
```
那么具有`CLIENT_MULTI`状态的客户端，在收到其他命令时，不会执行命令的对应处理函数，而是会将其加入到客户端的**事务**缓存命令队列之中，这个工作是通过`queueMultiCommand`接口完成的，这个函数会将*Redis*从读取的网络数据解析出的命令参数`client.cmd`、`client.argc`以及`client.argv`缓存到`client.mstate`之中，等待后续在**EXEC**命令提交时来执行
```c
int processCommadn(client *c)
{
    ...
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c);
        addReply(c, shared.queued);
    }
    else
    {
        call(c, CMD_CALL_FULL);    
    }
}
```

而**WATCH**命令，*Redis*则是通过下面这个接口来实现的：
```c
void watchCommand(client *c);
```
需要注意的是，这个命令必须在调用**MULTI**命令进入**事务**之前来执行，而不能在**事务**的进行中执行。这个命令会将相应的数据插入到客户端监控列表`client.watched_keys`以及键空间的监控列表`redisDb.watched_keys`之中。
```c
void touchWatchedKey(redisDb *db, robj *key);
```
当其他客户端通过命令修改*Redis*键空间中的数据时，则会触发上面`touchWatchedKey`这个函数，如果`key`恰好在`redisDb.watched_keys`之中，则表明这个`key`被若干个客户端的**事务**所监控。对于这种情况，就需要遍历`redisDb.watched_keys[key]`这个链表中的所有的客户端对象，为客户端设置`CLIENT_DIRTY_CAS`，表明这个客户端所监控的键已经被其他客户端所修改。

最后，客户端可以用过**EXEC**这个命令来提交**事务**，一次性地执行**事务**之中的所有命令。
```c
void execCommand(client *c);
```
`execCommand`这个函数会检测对应客户端的状态`client.flags`上是否携带了`CLIENT_DIRTY_CAS`或者`CLIENT_DIRTY_EXEC`这两个状态，如果携带则**事务**中的命令无法被执行；否则的话就会遍历`client.mstate`中命令队列里的每个命令，在循环中依次执行。

以上便是*Redis*对于**事务**机制的一个简要介绍。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
