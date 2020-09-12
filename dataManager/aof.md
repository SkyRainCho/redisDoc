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
    int aof_last_write_status;
}
```
这些字段的含义为：
1. `redisServer.aof_current_size`表示当前**AOF**文件的大小。
1. `redisServer.aof_fsync_offset`表示当前通过`write`系统调用写入文件的**AOF**偏移。
1. `redisServer.aof_last_fsync`记录了上次**AOF**数据写入磁盘的时间戳。
1. `redisServer.aof_flush_postponed_start`标记是否延时启动**AOF**写入，如果当前有正在后台运行的线程执行**AOF**磁盘写入，那么这个标记将会被设置，通过`wirte`系统调用写入**AOF**的操作将会被推迟。
1. `redisServer.aof_last_write_status`记录了上次调用`write`系统调用的结果，可以为`C_OK`或者`C_ERR`，如果内核的文件缓冲区已满，那么`write`有可能失败。

*Redis*将内存中`redisServer.aof_buf`缓存写入**AOF**文件是通过`flushAppendOnlyFile`这个函数来实现的。这个函数有若干个被调用的场合：
- 在*Redis*主线程每次进去事件循环等待之前，该函数会被调用：
```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ...
    if (server.aof_flush_postponed_start)
        flushAppendOnlyFile(0);

    run_with_period(1000) {
        if (server.aof_last_write_status == C_ERR)
            flushAppendOnlyFile(0);
    }
    ...
}
```
- 在*Redis*服务器的心跳函数中，该函数会被调用：
```c
void beforeSleep(struct aeEventLoop *eventLoop)
{
    ...
    flushAppendOnlyFile(0);
    ...
}
```
- 在*Redis*服务器准备关闭之前，这个函数会被调用：
```c
int prepareForShutDown(int flags) {
    ...
    if (server.aof_state != AOF_OFF)
    {
        ...
        flushAppendOnlyFile(1);
        redis_fsync(server.aof_fd);
    }
    ...
}
```

根据*Redis*设置的**AOF**文件写入磁盘的策略不同，*Redis*会执行如下不同的逻辑：
1. `AOF_FSYNC_ALWAYS`，会调用`write`系统调用将`redisServer.aof_buf`写入文件，然后立刻调用`fsync`将文件写入磁盘。
1. `AOF_FSYNC_EVERYSEC`，会调用`write`系统调用将`redisServer.aof_buf`写入文件，每隔1秒钟，启动一个后台任务，用于异步地将文件通过`fsync`写入磁盘。
1. `AOF_FSYNC_NO`，只通过调用`write`将`redisServer.aof_buf`写入文件，至于文件何时被写入磁盘，则由操作系统自行决定。

现在我们可以看一下`flushAppendOnlyFile`的实现细节，从中我们可以了解到命令记录是如何被追加到**AOF**文件之中：
```c
void flushAppendOnlyFile(int force) {
    ...
    if (sdslen(server.aof_buf) == 0)
    {
        //对于使用AOF_FSYNC_EVERYSEC策略的AOF持久化。
        //如果内存缓存已经全部通过write写入文件内核缓冲区，
        //但是内核缓冲区没有被fsync全部写入磁盘，同时没有在后台运行的fsync线程，
        //那么尝试将缓冲区的内容调用fsync写入磁盘
    }

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        sync_in_progress = aofFsyncInProgress();

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force)
    {
        if (sync_in_progress)
        {
            //对于使用AOF_FSYNC_EVERYSEC策略的AOF持久化，
            //如果有在后台运行的异步fsync操作，那么设置aof_flush_postponed_start，
            //同时推迟这次的flushAppendOnlyFile操作
        }
    }

    //通过write系统调用将aof_buf写入文件
    nwritten = aofWrite(server.aof_fd, server.aof_buf, sdslen(server.aof_buf));
    ...
    server.aof_flush_postponed_start = 0;

    if (nwritten != sdslen(server.aof_buf))
    {
        //如果write写入的数据长度不等于aof_buf的长度，那么说明此时内核缓冲区不足
        server.aof_last_write_status = C_ERR;
    }
    ...
try_fsync:
    ...
    if (server.aof_fsync == AOF_FSYNC_ALWAYS)
    {
        redis_fsync(server.aof_fd);    
    }
    else if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
    {
        if (!sync_in_progress)
        {
            aof_backgroud_fsync(server.aof_fd);
        }
    }
}
```

*Redis*在默认情况，会关闭**AOF**持久化策略，我们可以通过修改配置文件来开启**AOF**持久化策略，另外我们可以在*Redis*运行中通过命令来开启**AOF**持久化：

    CONFIG SET appendonly on

那么在运行时开启**AOF**策略的处理逻辑是由`startAppendOnly`函数来实现的：
```c
int startAppendOnly(void);
```
这个函数的逻辑为：
1. 服务器必须处于**AOF**关闭的状态，`server.aof_state == AOF_OFF`。
1. 如果有一个子进程在重写**AOF**文件，那么先杀掉这个子进程。
1. 通过`rewriteAppendOnlyFileBackground()`函数来执行一次重写**AOF**文件，由于**AOF**本质是增量备份，因此这一步相当于先对整个数据库键空间进行一次全量备份。
1. 将`server.aof_state`设置为`AOF_WAIT_REWRITE`，在重写**AOF**结束之后，这个标记会被修改为`AOF_ON`。


## 重写AOF文件
*Redis*对于重写**AOF**文件都是启动一个子进程以异步的方式进行的，启动重写**AOF**文件有两种方式，一种是客户端执行**BGREWRITEAOF**命令；另外一种是在服务器心跳中，检查文件大小，当**AOF**文件大小超过一定阈值时，启动重写机制。

在了解重写**AOF**之前，我们先看一下*Redis*服务器全局变量中相关的一些内容：
```c
struct redisServer {
    int aof_rewrite_min_size;
    int aof_rewrite_base_size;
    int aof_rewrite_perc;
    int aof_rewrite_scheduled;
    ...
};
```
这里：
1. `redisServer.aof_rewrite_min_size`用于开启重写机制的**AOF**文件大小的阈值。
1. `redisServer.aof_rewrite_base_size`与`redisServer.aof_rewrite_perc`，这两个字段用于处理当**AOF**文件大小超过`aof_rewrite_base_size`的百分比，如果超过阈值，那么将开启重写机制。
1. `redisServer.aof_rewrite_scheduled`是一个调度标记，**BGREWRITEAOF**命令并不会立即启动重写机制，而是设置好`aof_rewrite_scheduled`，在下一次心跳中去启动该机制。

这里我们可以看一下*Redis*服务器心跳之中的相关代码：
```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)
{
    ...
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.aof_rewrite_scheduled)
    {
        rewriteAppendOnlyFileBackground();
    }

    ...
    if (server.aof_state == AOF_ON &&
        server.rdb_child_pid == -1 &&
        server.aof_child_pid == -1 &&
        server.aof_rewrite_perc &&
        server.aof_current_size > server.aof_rewrite_min_size)
    {
        long long base = server.aof_rewrite_base_size ? server.aof_rewrite_base_size : 1;
        long long growth = (server.aof_current_size * 100/base) - 100;
        if (growth >= server.aof_rewrite_perc)
        {
            rewriteAppendOnlyFileBackground();
        }
    }

    ...
}
```

在了解`rewriteAppendOnlyFileBackground`这个函数之前，我们先看一些底层的辅助函数，首先我们来看一下*Redis*用于重写对象数据的函数：
```c
int rewriteListObject(rio *r, robj *key, robj *o);
int rewriteSetObject(rio *r, robj *key, robj *o);
int rewriteSoredSetObject(rio *r, robj *key, robj *o);
int rewriteHashObject(rio *r, robj *key, robj *o);
int rewriteStreamObject(rio *r, robj *key, robj *o);
int rewriteModuleObject(rio *r, robj *key, robj *o);
```
应用上面这几个函数，*Redis*可以将键空间之中的对象数据转化为命令的形式被记录在**AOF**文件之中。某些对象数据之中可能具有多个子元素，因此*Redis*定义了一个限制`AOF_REWRITE_ITEMS_PER_CMD`，即每条命令最多可以添加的元素个数，如果键空间中的有一个数据对象的子元素超过了这个限制，那么*Redis*将会将一条过长的命令记录拆解成多条命令记录在**AOF**文件之中。

而下面这个函数则是*Redis*用于将这个数据库键空间转化为命令记录写入**AOF**文件的接口：

```c
int rewriteAppendOnlyFileRio(rio *aof);
```

*Redis*通过这个函数依次遍历每个数据库的键空间中的每一个数据，对于每一个新被遍历的数据，会追加一条**SELECT**命令的记录；而对于键空间之中携带过期时间的数据，则会为其追加一条**nPEXPIREAT**命令的记录。最终所有这些命令会被写入封装了**AOF**文件的`rio`对象之中。

如果阅读*src/aof.c*源码，我们发现`rewriteAppendOnlyFileRio`并不是重写**AOF**文件的顶层接口，这个真正的接口是下面这个函数：

```c
int rewriteAppendOnlyFile(char *filename);
```

那么为什么要在`rewriteAppendOnlyFileRio`函数的基础上在实现一个`rewriteAppendOnlyFile`函数呢？这里就要涉及到*Redis*在重写**AOF**文件时的一个可选策略了，通过上面对于`rewriteAppendOnlyFileRio`这个函数的讲解我们可以了解到，所谓重写**AOF**文件，实际上时执行一次对于数据库键空间的全量的备份，而后续被执行的命令则会被追加到这个新的全量备份之后。然而对于基于命令记录的**AOF**文件，即使是重写时清理的冗余的以及无效的数据，其文件的大小依然要大于使用**RDB**策略生产的全量备份文件。因此*Redis*在重写**AOF**文件时，可以选择一种`aof_use_rdb_preamble`策略，应用这种策略，在对数据库键空间进行全量备份的时候，会使用**RDB**策略生成文件，而后续被执行的命令以**AOF**的格式被追加到这个**RDB**文件之后。这一点可以通过`rewriteAppendOnlyFile`代码片段了解到。

```c
int rewriteAppendOnlyFile(char *filename) {
    rio aof;
    FILE *fp;
    char tmpfile[256];
    char byte;
    
    snprintf(tmpfile,256,"temp-rewriteaof-%d.aof", (int) getpid());
    fp = fopen(tmpfile,"w");
    rioInitWithFile(&aof,fp);
    if (server.aof_rewrite_incremental_fsync)
        rioSetAutoSync(&aof,REDIS_AUTOSYNC_BYTES);
    
    if (server.aof_use_rdb_preamble) {
        rdbSaveRio(&aof,&error,RDB_SAVE_AOF_PREAMBLE,NULL);
    }
    else
    {
        rewriteAppendOnlyFileRio(&aof);
    }
    
    fflush(fp);
    fsync(fileno(fp));
    ...
}
```

而`rewriteAppendOnlyFileBackground`与后台异步生成**RDB**文件的处理方式类似，通过`fork`系统调用创建一个子进程，子进程调用`rewriteAppendOnlyFileBackground`重写**AOF**文件。然而这样会产生另外一个问题，便是在子进程重写**AOF**文件时，如果父进程因为新的命令而引入新的变化时，要如何将这些变化进行持久化。也许有人会问，异步生成**RDB**文件时，也会出现产生新变化的可能，为什么**RDB**策略不需要关注这个问题？这主要是因为，每次生成**RDB**文件时，都是从键空间之中生成全量的备份，而在异步生成**RDB**文件出现的新变化也是存储在数据库的键空间之中。那么在下一次生成**RDB**文件时，这些新的变化会被持续化到新的**RDB**文件之中，而不会造成数据丢失。但是**AOF**策略是基于命令记录的持久化，内存中数据库键空间中只会存储数据而不会存储命令记录，因此如果不进行特殊处理，那么在异步重写**AOF**文件时引入的新变化将会丢失。

对于上面这种情况，*Redis*会在父进程中将异步生成**AOF**文件时，将这个时间段的客户端命令存储在一个重写缓存之中，等待重写完成后在对这些缓存的命令进程处理，首先我们看一下，*Redis*对于重写缓存的数据结构与数据字段：

```c
#define AOF_RW_BUF_BLOCK_SIZE (1024*1024*10)
typedef struct aofrwblock {
    unsigned long used, free;
    char buf[AOF_RW_BUF_BLOCK_SIZE];
} aofrwblock;

struct redisServer {
    ...
    list *aof_rewrite_buf_blocks;
    ...
};
```

*Redis*提供了一个接口用于将格式化的命令记录写入到`redisServer.aof_rewrite_buf_blocks`之中：

```c
void aofRewriteBufferAppend(unsigned char *s, unsigned long len);
```

*Redis*在执行命令时调用`feedAppendOnlyFile`时，如果检测到现在有后台执行**AOF**的子进程时，那么会执行`aofRewriteBufferAppend`，将命令追加到重写缓存中：

```c
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
	...
    if (server.aof_child_pid != -1)
        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));
    ...
}
```

同时在*Redis*服务器的全局变量之中，还定义了若干个管道文件描述符，用于在重写**AOF**文件时，处理父进程与子进程之间的通信：

```c
struct redisServer {
    ...
    int aof_pipe_write_data_to_child;
    int aof_pipe_read_data_from_parent;
    int aof_pipe_write_ack_to_parent;
    int aof_pipe_read_ack_from_child;
    int aof_pipe_write_ack_to_child;
    int aof_pipe_read_ack_from_parent;
    ...
}
```

*Redis*在启动重写**AOF**文件时，在`fork`创建子进程之前，会调用`aofCreatePipes`函数初始化上述的三对管道，同时将`redisServer.aof_pipe_read_ack_from_child`的可读事件注册到服务器的事件循环之中，也就是主进程会在后续等待`redisServer.aof_pipe_read_ack_from_child`上的可读事件。现在我们可以看一下*Redis*在处理重写缓存时的逻辑：

1. 主进程在调用`rewriteAppendOnlyFileBackground`时，调用`aofCreatePipes`初始化管道，并监听`redisServer.aof_pipe_read_ack_from_child`上的可读事件，同时记录`redisServer.aof_child_pid`这个子进程ID。
2. 主进程在后续执行客户端命令时 ，将命令记录调用`aofRewriteBufferAppend`写入重写缓存`redisServer.aof_rewrite_buf_blocks`，并向父进程的事件循环之中注册监听`redisServer.aof_pipe_write_data_to_child`的可写事件。
3. 父进程在事件循环之中检测到`redisServer.aof_pipe_write_data_to_child`可写时，会调用注册的事件处理函数`aofChildWriteDiffData`，这个函数会将`redisServer.aof_write_buf_blocks`中的命令记录通过`redisServer.aof_pipe_write_data_to_child`写入管道，等待子进程读取。
4. 子进程在`rewriteAppendOnlyFile`函数完成文件写入后，会循环数次尝试从`redisServer.aof_pipe_read_data_from_parent`对应的管道中调用`aofReadDiffFromParent`函数读取父进程发送来的命令记录，并将这些数据存储在子进程`redisServer.aof_child_diff`缓存之中。
5. 子进程在`rewriteAppendOnlyFile`函数完成`aofReadDiffFromParent`数据读取时，向`redisServer.aof_pipe_write_ack_to_parent`这个管道的写入端写入一个`!`通知父进程。
6. 父进程从`redisServer.aof_pipe_read_ack_from_child`这个管道的读取端上监听的可读事件返回，调用对应的事件处理函数`aofChildPipeReadable`，这个函数会停止向`redisServer.aof_pipe_write_data_to_child`对应的管道之中写入数据，同时向`redisServer.aof_pipe_write_ack_to_child`对应的管道之中写入一个确认数据`!`。
7. 子进程在函数`rewriteAppendOnlyFile`中，在第5步向`redisServer.aof_pipe_write_ack_to_parent`写入一个确认数据之后，便调用`syncRead`接口同步等待读取父进程在第6步中发送来的确认数据。
8. 子进程在`rewriteAppendOnlyFile`收到父进程的确认信息之后，最后在调用一次`aofReadDiffFromParent`完成数据的读取，然后将记录在`redisServer.aof_child_diff`缓存中的数据追加到重写后的**AOF**文件之中。子进程在完成数据写入之后，成功结束。
9. 父进程在检测到子进程退出后，会调用`backgroundRewriteDoneHandler`这个函数接口，完成后续的处理步骤



## 加载AOF文件


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*