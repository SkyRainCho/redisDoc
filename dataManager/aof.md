# Redis中AOF持久化策略
通过前面我们了解到**RDB**的持久化策略，可以使用一个最为精简的方式将内存中的数据序列化到磁盘的文件之中。但是**RDB**策略存在一个它自身固有的问题，便是每次**RDB**持久化都是对于内存数据的一个全量镜像的备份，这种策略对于系统的消耗比较大，因此备份无法做到过于的频繁。一旦系统的软件或者硬件出现问题，那么将遗失从上一次生成**RDB**文件之后的所有数据变化。

因此*Redis*还给出了另外一种**AOF**持久化策略，这种策略会将每一条被执行的*Redis*命令以记录的形式追加到**AOF**文件之中；在启动服务器时，*Redis*会解析**AOF**文件中被记录的每一条命令并按照顺序执行该命令，借以实现数据的恢复。服务器可以配置将每条命令在执行时都追加到文件之中；也可以配置每1秒钟将累积的命令追加到文件之中；或者是由操作系统决定在内核文件缓冲区已满的情况下，将命令写入文件之中。**AOF**这种方式可以最大限度的降低数据丢失的情况，如果选择每执行一条命令都追加到文件中的这种策略，那么将只会丢失一个事件循环之中的数据变化；当然这种策略将大大地降低系统的性能。即使选择每1秒钟写入一次的策略，**AOF**策略最多也只会丢失1秒钟内变化的数据。因此这种持久化策略具有较高的数据安全性。

但是**AOF**策略也有一个问题，它不同于**RDB**选择将内存数据直接序列化到文件中的策略，因此**AOF**文件中会附加很多命令的附加信息，同时随着系统的运行，被追加到**AOF**文件中命令记录也越来越多，这不但增加了**AOF**文件的大小，在服务器启动进行数据恢复加载时，过多的命令记录也将增加*Redis*启动所消耗的时间。因此*Redis*给出了一种重写**AOF**的机制，应用这种机制可以对**AOF**文件之中的冗余数据进行剔除，压缩文件大小，同时提升启动加载时的速度。例如执行下面两台命令：
    
    SET key:1 hello
    SET key:1 world

上面两条命令在正常的**AOF**策略时，都会被追加到文件之中，但是显然第一条命令对于恢复数据并没有任何意义，因为最终存储在键空间之中的`key:1`对应的*value*为`world`。**AOF**重写机制可以将第一条**SET**命令的记录从**AOF**文件之中剔除，通过将**AOF**文件中冗余以及无效的命令删除，以达到压缩文件大小，同时提升启动加载时的速度的目的。

这篇文章将从**AOF**持久化策略以及重写**AOF**机制两个方面进行讲解。

## AOF持久化
我们可以先了解一下*Redis*服务器全局数据之中关于**AOF**持久化的一些数据字段：
```c
struct redisServer
{
    ...
    int aof_state;
    int aof_fsync;
    char *aof_filename;
    sds aof_buf;
    int aof_fd;
    int aof_selected_db;
    ...
};
```
在这些字段中：
1. `redisServer.aof_state`标记了当前服务器**AOF**策略的开启策略，*Redis*一共定义了三种策略：
    1. `#define AOF_OFF 0`，当前*Redis*关闭了**AOF**持久化。
    1. `#define AOF_ON 1`，当前*Redis*开启**AOF**持久化。
    1. `#define AOF_WAIT_REWRITE 2`
1. `redisServer.aof_fsync`标记了命令记录被同步写入磁盘**AOF**文件的策略，*Redis*定义了三种策略：
    1. `#define AOF_FSYNC_NO 0`，这种策略*Redis*只负责将数据写入文件缓冲区，操作系统自行决定何时将缓冲区中的数据写入磁盘。
    1. `#define AOF_FSYNC_ALWAYS 1`，这种策略*Redis*会在每次进入事件循环前，会强制清理一下文件缓冲区，将其中的内容写入磁盘，这样可以保证当软件或者硬件出现故障时，丢失的数据最少。
    1. `#define AOF_FSYNC_EVERYSEC 2`，这种策略会每秒钟将缓冲区中的数据写入磁盘，属于前两种策略折中的办法。
1. `redisServer.aof_filename`，**AOF**文件的文件名。
1. `redisServer.aof_buf`，**AOF**策略的内存缓冲，命令记录先被写入这个内存缓冲之中，然后在写入文件的内核缓冲区。
1. `redisServer.aof_fd`，**AOF**文件对应的文件描述符。
1. `redisServer.aof_selected_db`，上一条被记录的命令对应的数据库编号，如果新的命令对应数据库发生变化，需要补充追加一条**SELECT**命令的记录。

*Redis*在启动服务器时，会根据配置，打开**AOF**文件，并将对应文件描述符记录在`redisServer.aof_fd`字段上：
```c
void initServer(void)
{
    ...
    if (server.aof_state == AOF_ON)
    {
        server.aof_fd = open(server.aof_filename, O_WRONLY|O_APPEND|O_CREATE, 0644);
        ...
    }
    ...
}
```

*Redis*在每次执行客户端命令时，会通过调用`propagate`函数，将这个命令传递给**AOF**内存缓存，或者发送给集群中的*Slave*实例：
```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc, int flags)
{
    if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
        feedAppendOnlyFile(cmd, dbid, argv, argc);
    ...
}
```
这里`feedAppendOnlyFile`函数是追加命令记录的核心接口，在了解这个接口之前我们先了解一下另外两个辅助函数接口：
```c
sds catAppendOnlyGenericCommand(sds dst, int argc, robj **argv);
```
这个函数用于将一条*Redis*命令按照一定格式写入一个`sds`缓冲区内，命令写入的格式为：

    *参数数量X\r\n$参数1长度\r\n参数1数据\r\n ... \r\n$参数X长度\r\n参数X数据\r\n

对于命令的名称，将作为命令的第一个参数被写入到缓冲区之中，而整个命令的格式可以总结为：
1. 以`\r\n`来作为不同字段之间的间隔符号。
1. `*`这个特殊标记表示后面的一个字段用于存储命令的个数。
1. `$`这个特殊标记表示后面的两个字段用于存储一个命令参数，其中第一个字段为参数长度，第二个字段为参数数据。

在这个函数的基础上，*Redis*实现了用于将**EXPIRE**系列命令转换为**EXPIREAT**系列命令的函数接口：
```c
sds catAppendOnlyExpireAtCommand(sds buf, struct redisCommand *cmd, robj *key, robj *seconds);
```
之所以需要执行这个命令的转换，是因为**EXPIRE**系列命令记录的是一个相对的时间段，这在加载**AOF**文件重新执行命令时，是没有意义的，因此需要将其转化为**EXPIREAT**系列命令。

现在我们可以看一下`feedAppendOnlyFile`这个函数接口：
```c
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc);
```
这个函数会执行如下的逻辑：
1. 如果操作的数据库`dictid`与上一条被执行的命令的数据库`server.aof_selected_db`不一致，那么补充追加一条**SELECT**命令记录。
1. 如果执行的是**EXPIRE**系列命令，那么通过`catAppendOnlyExpireAtCommand`接口转化为**EXPIREAT**系列命令。
1. 如果是**SETEX**系列命令，或者是携带过期参数的**SET**命令，那么将其拆分成两个命令，一个**SET**命令，一个**EXPIREAT**命令进行记录。
1. 其他命令则直接调用`catAppendOnlyGenericCommand`函数进行记录。

现在我们了解，*Redis*是如何将一条命令记录在内存缓存`redisServer.aof_buf`之中的。现在我们来看一下，**AOF**内存缓存是如何被写入到磁盘文件之中的。

我们首先来了解一下*Redis*服务器全局变量中，关于这部分的一些字段的描述：
```c
struct redisServer
{
    ...
    off_t aof_current_size;
    off_t aof_fsync_offset;
    time_t aof_last_fsync;
    time_t aof_flush_postponed_start;
}
```
这些字段的含义为：
1. `redisServer.aof_current_size`表示当前**AOF**文件的大小。
1. `redisServer.aof_fsync_offset`表示当前已经被写入磁盘的**AOF**偏移。
1. `redisServer.aof_last_fsync`记录了上次**AOF**数据写入磁盘的时间戳。

## 重写**AOF**文件


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*