# Redis命令支持

在了解*Redis*命令执行之前，我们先来看一下代表*Redis*命令的结构体的定义，在*src/server.h*这个头文件中：
```c
typedef void redisCommandProc(client *c);
typedef int *redisGetKeysProc(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags;
    int flags;
    redisGetKeysProc *getkeys_proc;
    int firstkey;
    int lastkey;
    int keystep;
    long long microseconds, calls;
};
```
首先讲解一下，这个结构体中常用的字段的含义：
1. `redisCommand.name`，用于记录命令的小写的命令名，例如`get`，`set`，`sadd`等等。
1. `redisCommand.proc`，对应命令的实行函数的函数指针。
1. `redisCommand.arity`，命令的参数个数限定，如果`arity`的值为正数`N`，那么命令参数必须为`N`个；如果为负值`-N`，则参数个数必须大于`N`。
1. `redisCommand.microseconds`，`redisCommand.calls`，这两个字段用于对命令的执行情况进行统计，分别用于记录命令累计执行时间毫秒数以及累计执行次数。
1. `redisCommand.sflags`，`redisCommand.flags`，用于标记当前命令属性标记，分别以字符形式标记以及掩码形式标记。

命令标记，在*src/server.h*头文件之中，定义了用于标记命令属性的标记：
1. `CMD_WRITE`，对应字符标记`w`，表示是一条写命令，有可能会修改服务器的键空间。
1. `CMD_READONLY`，对应字符标记`r`，表示是一条只读命令。
1. `CMD_DENYOOM`，对应字符标记`m`，标记这条命令被调用时有可能会增加内存的使用量。
1. `CMD_MODULE`
1. `CMD_ADMIN`，对应字符标记`a`，表示这是一条管理命令，例如**SAVE**或者**SHUTDOWN**。
1. `CMD_PUBSUB`，对应字符标记`p`，对应是对应发布功能相关的命令。
1. `CMD_NOSCRIPT`，对应字符标记`s`，表示这个命令不允许在脚本中执行。
1. `CMD_RANDOM`，对应字符标记`R`，表示这是一个随机命令。
1. `CMD_SORT_FOR_SCRIPT`，对应字符标记`S`，如果这条命令在脚本中被调用，命令的输出将会被排序。
1. `CMD_LOADING`，对应字符标记`l`，表示该命令允许在加载数据库的阶段被执行
1. `CMD_STALE`，对应字符标记`t`，表示这个命令允许在**Slave**实例中有陈旧数据的时候被执行。
1. `CMD_SKIP_MONITOR`，对应字符标记`M`，
1. `CMD_ASKING`，对应字符标记`k`
1. `CMD_FAST`，对应字符标记`F`，表示这是一条以`O(1)`或者`O(log(N))`时间复杂度执行的快速命令。
1. `CMD_MODULE_GETKEYS`
1. `CMD_MODULE_NO_CLUSTER`

在*src/server.c*源文件之中，*Redis*使用硬编码的方式定义了一个`redisCommand`的数组`redisCommandTable`，用于为*Redis*的命令结构体进行初始化：
```c
struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,"as",0,NULL,0,0,0,0,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    ...
};
```
同时在*Redis*服务器全局数据中，使用一个哈希表存储了所有的*Redis*命令，使用命令名作为键用来索引`redisCommand`结构体指针：
```c
struct redisServer {
    ...
    dict *commands;
    ...
}
```
*Redis*会通过下面这个函数`populateCommandTable`，在初始化服务器配置`initServerConfig`接口之中，将`redisCommandTable`数组中的数据插入导`redisServer.commands`这个哈希表之中：
```c
void populateCommandTable(void);
```
通过下面这两个函数，我们可以通过给定命令名称来从`redisServer.commands`中搜索`redisCommand`结构体指针：
```c
struct redisCommand *lookupCommand(sds name);
struct redisCommand *lookupCommandByCString(char *s);
```

前面在介绍*Redis*客户端的文章之中，我们简略提到过客户端可以通过`processCommand`函数来执行命令，现在我们来看一下这个函数的实现细节：
```c
int processCommand(client *c);
```
回顾一下前面我们介绍的*Redis*客户端的相关内容，客户端使用`client.argc`用于存储命令参数的个数，`client.argv`来存储命令参数的具体内容，同时`client.argv[0]`中存储的是命令的名称，现在我们来看一下这个函数的实现细节：
1. 首先通过`lookupCommand`函数接口来查找对应命令的结构提数据，同时判断参数个数是否合法。
```c
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    if (!c->cmd)
    {
        ...
    } else if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) || 
            (c->argc < -c->cmd->arity))
    {
        ...
    }
```
2. 如果服务器开启了认证状态，验证客户端的验证标记`client.authenticated`，未认证的客户端仅允许执行**AUTH**认证命令。
```c
    if (server.requirepass && !c->authenticated && c->cmd->proc != authCommand)
    {
        return C_OK;
    }
```
3. 如果服务器开启了集群`redisServer.cluster_enabled`，那么执行集群重定向。
4. 如果服务器配置了最大内存上限`redisServer.maxmemory`，则会通过`freeMemoryIfNeededAndSafe`接口检测并执行内存淘汰的逻辑，淘汰机制的具体内容，可以参考前面的文章进行回顾。
```c
    if (server.maxmemory && !server.lua_timeout)
    {
        int out_of_memory = freeMemoryIfNeedAndSafe() == C_ERR;
        ...
    }
```
5. 如果当前服务器磁盘出现了问题，导致无法对数据进行持久化，那么会拒绝执行`CMD_WRITE`标记的写入命令。
6. 在*Master-Slave*模式下，如果**Master**实例没有足够的可用**Slave**实例，那么会拒绝执行写入命令。
7. 如果是只读的**Slave**实例，同样禁止执行写入命令。
8. 如果当前客户端处于订阅模式`c->flags & CLIENT_PUBSUB`，那么只允许执行**PING**、**PING**、**SUBSCRIBE**、**UNSUBSCRIBE**、**PSUBSCRIBE**、**PUNSUBSCRIBE**这几个特定的命令。
9. 判断如果服务器处于加载状态，那么只能执行具有`CMD_LOADING`标记的命令。
10. 如果Lua脚本执行过慢，那么只允许指定特定的几个命令**AUTH**，**REPLCONF**，**SHUTDOWN**。

在完成了上述的前置条件判断之后，我们便可以进入命令的具体形式过程，*Redis*通过`call`这个函数接口，执行给定客户端上绑定的命令处理函数：
```c
void call(client *c, int flags);
```
这个函数接口是*Redis*执行命令的核心，`flags`参数是命令的执行标记，可以选择的标记有：
1. `CMD_CALL_NONE`，没有任何标记，不执行任何额外行为。
1. `CMD_CALL_SLOWLOG`，检查命令的执行速度，如果需要的话会将这条命令记录到慢日志之中
1. `CMD_CALL_STATS`，会对命令进行统计。
1. `CMD_CALL_PROPAGATE_AOF`，如果命令修改了数据库，那么会将这个命令追加到**AOF**文件之中。
1. `CMD_CALL_PROPAGATE_REPL`，如果命令修改了数据库，那么会将这个命令发送到**Slave**实例节点上。
1. `CMD_CALL_PROPAGATE`，这个标记相当于`CMD_CALL_PROPAGATE_AOF|CMD_CALL_PROPAGATE_REPL`。
1. `CMD_CALL_FULL`，这个是一个更加全面的标记，相当于`CMD_CALL_PROPAGATE|CMD_CALL_STATS|CMD_CALL_SLOWLOG`。

接下来，我们可以来看一下`call`这个函数接口执行客户端命令的具体细节：
```c
void call(client *c, int flags)
{
    ...
    //执行命令，并记录消耗时间已经是否引起数据库键空间中数据的变化
    dirty = server.dirty;
    start = server.ustime;
    c->cmd->proc(c);
    duration = ustime() - start;
    drity = server.dirty - dirty;

    //如果设置了CMD_CALL_SLOWLOG标记，那么如果需要的话，将其记录到系统慢日志之中
    if (flags & CMD_CALL_SLOWLOG)
    {
        char *latency_event = (c->cmd->flags & CMD_FAST) ? "fast-command" : "command";
        latencyAddSampleIfNeeded(latency_event,duration/1000);
        slowlogPushEntryIfNeeded(c,c->argv,c->argc,duration);
    }

    //如果设置了CMD_CALL_STATS，那么更新该命令的总用时以及执行次数
    if (flags & CMD_CALL_STATS)
    {
        real_cmd->microseconds += duration;
        real_cmd->calls++;
    }

    //如果设置了了CMD_CALL_PROPAGATE，那么将该命令追加到AOF文件中并发送给Slave示例
    if (flags & CMD_CALL_PROPAGATE)
    {
        ...
        propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags);
    }
}
```
以上便是`call`接口执行用户命令的全部流程。


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
