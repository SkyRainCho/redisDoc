# Redis数据持久化RDB策略
众所周知*Redis*是一款内存数据库，所有的数据都被存储在内存之中，然而如果数据仅仅被存储在内存中的话，那么一旦服务器进程出现停机，那么所有的数据都将丢失，因此*Redis*需要支持数据的持久化，将内存之中的数据存储在磁盘之中。当*Redis*进程启动时，会从磁盘之中将数据恢复到内存之中。

## RDB概述
**RDB**持久化是*Redis*支持的一种持久化策略，*Redis*会将服务器的状态信息以及所有数据库中的数据序列化到磁盘的文件之中，相当于保存了服务器数据的一个快照。在*Redis*启动时，会从文件中反序列化恢复数据到内存之中。

*Redis*可以通过主动或者被动的方式来创建**RDB**文件：
1. 主动方式，通过客户端数据的命令**SAVE**以及**BGSAVE**，来启动**RDB**持久化。
    1. 如果客户端执行**SAVE**，那么*Redis*会同步地生成**RDB**文件，此时*Redis*的主线程会被阻塞起来，直到**RDB**文件生成完成，*Redis*都不会响应客户端的命令请求。
    2. 如果客户端执行**BGSAVE**，那么*Redis*会异步地生成**RDB**文件，此时*Redis*会主动创建一个子进程用于生成**RDB**文件，而*Redis*此时依然可以处理客户端的其他命令请求。
2. 被动方式，*Redis*检测一段时间内修改次数累积到一定的阈值时，会启动异步生成**RDB**文件的逻辑。

对于**RDB**文件的自动备份，这里需要专门介绍一下，*Redis*会通过下面这个结构体来描述这个自动备份的阈值：
    ```c
    struct saveparam {
        time_t seconds;
        int changes;
    };
    ```
这表示，如果*Redis*在`saveparam.seconds`秒数内累积了`saveparam.changes`次变化的化，便会启动备份。*Redis*可以保有多个这样的阈值，存储在服务器全局数据之中：
```c
struct redisServer {
    ...
    struct saveparam *saveparams;
    int saveparamslen;
};
```
同时*Redis*也存储了自上次备份以来被修改的数据数量以及上次备份的时间，同样保存在服务器全局数据之中：
```c
struct redisServer {
    ...
    long long dirty;
    time_t lastsave;
};
```
*Redis*会在服务器的心跳`serverCron`中，对累积的变化数据进行检测，多个阈值只要有一个满足，那么便会触发生成**RDB**文件的逻辑。需要注意的时，这种情况下，*Redis*是采用异步的方式来生成**RDB**文件的。

*Redis*在生成**RDB**文件时，则是使用前面我们介绍的`rio`对象，通过操作`FILE`类型的`rio`对象，我们可以实现内存数据到文件的序列化过程以及文件数据到内存的反序列化过程。

## RDB基础数据的读写
为了实现序列化，出了需要将内存中的数据转化为一段连续数据之外，还需要附加上一个类型标记，通过这个类型标记，我们可以知道被序列化的连续数据将被反序列化为何种类型的数据。另外还需要在序列化数据之中附加上长度信息，通过这个信息，我们可以在反序列化时，确定一段完整的数据应该截止于何处。

*Redis*在*src/rdb.h*头文件之中，通过`define`定义了**RDB**序列化数据的类型：
- `RDB_TYPE_STRING`，这个类型表示序列化的是字符串对象类型
- `RDB_TYPE_LIST`，这个类型已经不再使用
- `RDB_TYPE_SET`，这个类型表示序列化的是使用`OBJ_ENCODING_HT`编码方式的集合对象
- `RDB_TYPE_ZSET`，这个类型已经不再使用
- `RDB_TYPE_HASH`，这个类型表示序列化的是使用`OBJ_ENCODING_HT`编码方式的散列对象
- `RDB_TYPE_ZSET_2`，这个类型表示序列化的是使用`OBJ_ENCODING_SKIPLIST`编码方式的有序集合对象
- `RDB_TYPE_MODULE`，这个类型已经不再使用
- `RDB_TYPE_MODULE_2`，这个类型表示序列化的是*Module*对象数据

上述这几个类型都是用于标记对象数据类型的序列化，并且上述类型的数据都是对应底层使用哈希表或者跳跃表这类离散数据的序列化。通过前面的介绍我们知道在一些特殊的情况下，*Redis*会使用连续分配的内存段来事项对象数据类型，下面这几个类型的定义便是对应于这种情况：
- `RDB_TYPE_HASH_ZIPMAP`，这个类型已经不在使用
- `RDB_TYPE_LIST_ZIPLIST`，这个类型已经不在使用
- `RDB_TYPE_SET_INTSET`，这个类型表示序列化的是使用`OBJ_ENCODING_INTSET`编码方式的集合对象
- `RDB_TYPE_ZSET_ZIPLIST`，这个类型表示序列化的是使用`OBJ_ENCODING_ZIPLIST`编码方式的有序集合对象
- `RDB_TYPE_HASH_ZIPLIST`，这个类型表示序列化的是使用`OBJ_ENCODING_ZIPLIST`编码方式的散列对象
- `RDB_TYPE_LIST_QUICKLIST`，这个类型表示序列化的是使用`OBJ_ENCODING_QUICKLIST`编码方式的列表对象
- `RDB_TYPE_STREAM_LISTPACKS`，这个类型表示序列化的是*Stream*dui对象数据

上述的这些对应数据对象类型的序列化类型，可以通过下面的两个函数进行读写：
```c
int rdbSaveObjectType(rio *rdb, robj *o);
int rdbLoadObjectType(rio *rdb);
```

除了标记数据对象的序列化类型，*Redis*还定义了几个辅助数据序列化的类型：
- `RDB_OPCODE_MODULE_AUX`，这个类型表示后序列化的是*Module*辅助数据。
- `RDB_OPCODE_IDLE`，这个类型表示后面序列化的是一个*key*的**LRU**空闲时间。
- `RDB_OPCODE_FREQ`，这个类型表示后面序列化的是一个*key*的**LFU**频率数据。
- `RDB_OPCODE_AUX`，这个类型表示后面序列化的是一个**RDB**的辅助数据。
- `RDB_OPCODE_RESIZEDB`，这个类型表示后面序列化的是数据库键空间的长度以及过期哈希表的长度。
- `RDB_OPCODE_EXPIRETIME_MS`，这个类型表示后面序列化的是一个*key*的毫秒级有效期时间戳。
- `RDB_OPCODE_EXPIRETIME`，这个类型表示后面序列化的是一个*key*的秒级有效时间时间戳，但是现在已经不在继续使用。
- `RDB_OPCODE_SELECTDB`，这个类型表示后面序列化的是数据库对应的编号数据。
- `RDB_OPCODE_EOF`，这个类型用于标记**RDB**文件的结束符。

上述这些辅助序列化的类型，可以通过下面这两个函数进行读写：
```c
int rdbSaveType(rio *rdb, unsigned char type);
int rdbLoadType(rio *rdb);
```
上面两个`rdbSaveObjectType`以及`rdbLoadObjectType`函数也是通过`rdbSaveType`以及`rdbLoadType`来实现的。


接下来*Redis*定义了一些基础层数据的序列化与反序列化的操作函数，这些函数的实现细节较为简单，也较为相似，因此只简要介绍一下各个函数的含义与用途，具体的实现可以参考*src/rdb.c*源文件：
```c
int rdbSaveTime(rio *rdb, time_t t);
time_t rdbLoadTime(rio *rdb);
int rdbSaveMillisecondTime(rio *rdb, long long t);
long long rdbLoadMillisecondTime(rio *rdb, int rdbver);
```
上述这四个函数用于实现秒数级时间戳与毫秒时间戳的读写。

```c
int rdbSaveLen(rio *rdb, uint64_t len);
uint64_t rdbLoadLen(rio *rdb, int *isencoded);
int rdbLoadLenByRef(rio *rdb, int *isencoded, uint64_t *lenptr);
```
这三个函数则是用于实现长度信息的读写。


## RDB对象数据的读写

对于*Redis*中数据对象类型的序列化与反序列化，是通过下面这两个函数来实现的：
```c
ssize_t rdbSaveObject(rio *rdb, robj *o, robj *key);
robj *rdbLoadObject(int rdbtype, rio *rdb, robj *key);
```
对象序列化的过程会根据对象的底层编码类型，如果是使用快速列表、哈希表、或跳跃表这类离散型数据结构的话，那么遍历对象中的子数据，将其依次序列化到文件之中；如果是使用压缩链表、整数集合这类的连续内存数据结构，那么直接将这段连续的内存序列化到文件之中。需要注意的是，调用`rdbSaveObject`函数只会将数据对象序列化到文件中，而不会写入对象的类型，因此需要显式调用`rdbSaveObjectType`将对象的类型进行序列化。

而数据对象的反序列化，则是通过`rdbLoadObject`函数进行的，与序列化函数`rdbSaveObject`对应，调用这个函数前，需要首先反序列化出对象的类型`rdbtype`，然后`rdbLoadObject`这个函数会通过`rdbtype`参数，选择对应的反序列化方式，从文件之中读出数据。

对于存储在数据库之中的数据，是以*key-value*的形式存储在键空间之中；如果设置了有效期，则会在`redisDb.expires`中保存一份*key-timestamp*记录，对于这样一组数据，*Redis*通过下面这个函数进行序列化：
```c
int rdbSaveKeyValuePair(rio *rdb, robj *key, robj *val, long long expiretime);
```
这个函数会按照如下的策略对数据进行序列化：
1. 如果设置了有效期，那么通过`RDB_OPCODE_EXPIRETIME_MS`类型，将*key*的有效期序列化到文件之中；
2. 如果服务器开启了**LRU**或者**LFU**策略，那么通过`RDB_OPCODE_IDLE`或者`RDB_OPCODE_FREQ`将相关数据序列化到文件之中；
3. 调用`rdbSaveObjectType`接口，将`val`的对象类型序列化到文件之中；
4. 由于*key*一定都是字符串对象类型，调用`rdbSaveStringObject`接口，将`key`序列化到文件之中；
5. 调用`rdbSaveObject`接口，将`val`序列化到文件之中。

这也就意味着，*Redis*键空间之中的每一条数据都将以下面的几种存储格式之一序列化到文件之中：

    ----------------------------------------------------------
    |Expire Info|LRU/LFU Info|Object Type|Key Data|Value Data|
    ----------------------------------------------------------

    ----------------------------------------------
    |LRU/LFU Info|Object Type|Key Data|Value Data|
    ----------------------------------------------
    
    ---------------------------------------------
    |Expire Info|Object Type|Key Data|Value Data|
    ---------------------------------------------
    
    ---------------------------------
    |Object Type|Key Data|Value Data|
    ---------------------------------

## RDB辅助数据的读写
此处说的辅助数据是指记录生成**RDB**数据时，系统的一些状态信息，*Redis*使用下面这个函数接口来向**RDB**数据之中写入辅助数据：
```c
ssize_t rdbSaveAuxField(rio *rdb, void *key, size_t keylen, void *val, size_t vallen)
{
    ...
    rdbSaveType(rdb, RDB_OPCODE_AUX);
    rdbSaveRawString(rdb, key, keylen);
    rdbSaveRawString(rdb, val, vallen);
}
```
通过上面的代码片段可以看到，**RDB**中辅助数据的格式为：

    ------------------------------------------
    |RDB_OPCODE_AUX|key-str-data|val-str-data|
    ------------------------------------------

而下面这个接口用于将一组辅助数据写入**RDB**数据之中：
```c
int rdbSaveInfoAuxField(rio *rdb, int flags, rdbSaveInfo *rsi);
```
这些辅助信息中比较重要的一些有：
1. *redis-ver*，用来标记当前**RDB**格式的版本。
1. *redis-bits*，用来记录当前*Redis*是32位系统还是64位系统。
1. *ctime*，用来记录创建**RDB**文件的时间戳。
1. *used-mem*，用来记录当前*Redis*使用的内存。

## RDB数据库备份的读写
在了解了前面一基础数据信息的读写之后，我们可以来了解一下整个**RDB**文件的格式以及生成步骤：
```c
int rdbSaveRio(rio *rdb, int *error, int flags, rdbSaveInfo *rsi);
```
*Redis*通过这个函数向**RDB**文件之中写入数据库的整个备份数据，下面我们可以通过简略的代码片段来了解，如何生成一个**RDB**文件：
```c
int rdbSaveRio(rio *rdb, int *error, int flags, rdbSaveInfo *rsi)
{
    if (server.rdb_checksum)
        rdb->update_cksum = rioGenericUpdateChecksum;
    snprintf(magic, sizeof(magic), "REDIS%04d", RDB_VERSION);
    rdbWriteRaw(rdb);
    rdbSaveInfoAuxField(rdb,flags,rsi);    
    rdbSAveModuleAux(rdb, REDISMODULE_AUX_BEFORE_RDB);

    for (j = 0; j < server.dbnum, j++)
    {
        redisDd *db = server.db + j;
        dict *d = db->dict;
        if (dictSize(d) == 0) continue;
        
        rdbSaveType(rdb, RDB_OPCODE_SELECTDB);
        rdbSaveLen(rdb, j);
        
        rdbSaveType(rdb, RDB_OPCODE_RESIZEDB);
        rdbSaveLen(rdb, db_size);
        rdbSaveLen(rbd, expires_size);
        
        while((de = dictNext(di)) != NULL) {
            rdbSaveKeyValuePair(rdb, &key, o, expire);
        }
    }

    rdbSAveModulesAux(rdb, REDISMODULE_AUX_AFTER_RDB);

    rdbSaveType(rdb, RDB_OPCODE_EOF);

    cksum = rdb->cksum;
    ridoWrite(rdb, &cksum, 8);
}
```
由此我们可以看出，整个**RDB**文件中数据的分布格式，如下面的图片可视：

![RDB文件数据格式]()

而从**RDB**文件之中进行数据的反序列化逻辑色是通过`rdbLoadRio`这个函数实现的。这个加载**RDB**文件最为常用的场景，便是*Redis*在启动时，从磁盘**RDB**文件中将数据恢复到内存之中，这个逻辑被定义在*Redis*的`loadDataFromDisk`函数之中：
```c
void loadDataFromDisk(void)
{
    if (server.aof_state == AOF_ON)
    {
        ...
    }
    else {
        rdbSaveInfo rsi = RDB_SAVE_INFO_INIT;
        rdbLoad(server.rdb_filename, &rsi);
    }
}
```
也就是说，如果我们没有开启**AOF**持久化策略，那么在*Redis*启动时，会调用`rdbLoad`这个函数从磁盘**RDB**文件中加载数据；而`rdbLoad`这个函数的逻辑主体正式`rdbLoadRio`这个接口，下面我们可以看一下这个接口的代码片段：
```c
int rdbLoadRio(rio *rdb, rdbSaveInfo *rsi, int loading_aof)
{
    ...
    while(1) {
        robj *key, *val;

        type = rdbLoadType(rdb);
        //RDB_OPCODE_*的类型编码，按照不同类型的类型选择对应的反序列化规则
        ...

        //如果不是RDB_OPCODE_*类型，则按照数据对象类型进行反序列化
        key = rdbLoadStringObject(rdb);
        val = rdbLoadObject(rdb, rdb, key);

        if (server.masterhost == NULL && !loading_aof && expiretime != -1 && expiretime < now)
        {
            decrRefCount(key);
            decrRefCount(val);
        }
        else
        {
            dbAdd(db, key, val);
            if (expiretime != -1) setExpire(NULL, db, key, expiretime);
            objectSetLRUorLFU(val, lfu_freq, lru_idle, lru_clock);
            decrRefCount(key);
        }
    }
    ...
}
```
从上面这段代码片段，我们可以发现，当从**RDB**文件中反序列化出一条键空间中的记录时，如果*Redis*是**主从模式**中的*主实例*启动的，并且当前反序列化出的记录以及超过了它的有效期，那么就不会将其加载到键空间之中；换而言之，如果是按照*从实例*启动的*Redis*，即使记录超过有效期，也依然会将其加载到键空间之中，之所不需要忽略加载过期的记录，是因为以**主从模式**启动的*Redis*服务器，*主实例*会在启动后将整个键空间的数据同步给**从实例**，因此不需要在加载过程中在*从实例*上处理过期的记录。



## RDB持久化策略

### 同步生成RDB文件
*Redis*可以通过**SAVE**命令来主动执行同步生成**RDB**文件过程，这个命令的格式为：

    SAVE -

其对应的实现函数为：
```c
void saveCommand(client *c);
```
这个函数会通过调用前面介绍的`rdbSave`接口，来执行生成**RDB**文件的逻辑，这里我们可以对`rdbSave`函数的逻辑做一个简单的讲解：
```c
int rdbSave(char *filename, rdbSaveInfo *rsi) {
    ...
    snprintf(tmpfile, 256, "temp-%d.rdb", (int) getpid());
    fp = fopen(tmpfile, "w");
    ...
    rioInitWithFile(&rdb, fp);

    if (server.rdb_save_incremental_fsync)
        rioSetAutoSync(&rdb, REDIS_AUTOSYNC_BYTES);

    rdbSaveRio(&rdb, &error, RDB_SAVE_NONE, rsi);
    fflush(fp);
    fsync(fileno(fp));
    fclose(fp);
    
    rname(tmpfile, filename);
    ...
    server.dirty = 0;
    server.lastsave = time(NULL);
    server.lastbgsave_status = C_OK;
    
    ...
}
```
在创建**RDB**文件结束后，会清空服务器的修改计数器`redisServer.dirty`，并重置存库时间`redisServer.lastsave`。

### 异步生成RDB文件
异步生成**RDB**文件可以通过**BGSAVE**命令执行，也可以通过服务器心跳中检测累积的修改是否达到了阈值自动运行，**BGSAVE**命令的格式为：

    BGSAVE -

这个命令的实现函数为：
```c
void bgsaveCommand(client *c)
{
    if (server.rdb_child_pid != -1)
    {
        ...
    }
    else if (server.aof_child_pid != -1)
    {
        ...
    }
    else if (rdbSaveBackground(server.rdb_filename, rsiptr))
    {
        ...    
    }
    ...
}
```
通过上面的函数，我们可以发现，只有在没有后台**RDB**以及**AOF**子进程运行时，才可以开启新的后台异步**RDB**流程。

下面我们介绍一下*Redis*用于实现后台异步生成**RDB**文件的函数接口`rdbSaveBackground`：
```c
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi)
{
    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;

    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);

    if ((childpid = fork()) == 0)
    {
        closeListeningSockets(0);
        retval - rdbSave(filename, rsi);
        if (retval == C_OK)
        {
            ...
            sendChildInfo(CHILD_INFO_TYPE_RDB);
        }
        exitFromChild((retval == C_OK) ? 0:1);
    }
    else
    {
        ...
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_chile_type = RDB_CHILD_TYPE_DISK;
    }
}
```
通过上面的代码片段，我们可以总结出异步生成**RDB**文件的流程：
1. 检查`redisServer.aof_child_pid`以及`redisServer.rdb_child_pid`，确保同时只有一个后台子进程在运行。
1. 由于**RDB**文件是通过子进程异步生成，在生成文件的过程中，父进程中依然可能发生数据的变化，因此将`redisServer.dirty`备份到`redisServer.dirty_before_bgsave`字段中，用于标记在开始生成**RDB**文件的那一刻被修改的数据的计数。
1. 通过`fork`系统调用创建子进程，`fork`会一次调用两次返回，在子进程中会返回0；在父进程之中则会返回子进程的`pid`。
1. 在父进程中，将开始异步生成**RDB**文件的时间记录在`redisServer.rdb_save_time_start`字段中；将子进程的`pid`记录在`redisServer.rdb_child_pid`字段中。完成这些数据的记录之后，*Redis*的父进程就可以继续处理客户端的请他命令请求，包括对于`redisServer.dirty`计数器的更新。
1. 在子进程之中，子进程会调用`rdbSave`这个接口生成**RDB**文件，生成结束之后，子进程便会退出。

那么为何子进程可以对父进程中的数据进程备份呢？我们可以看一下《UNIX环境高级编程》中对`fork`调用以及子进程的描述：
> 子进程是父进程的副本。例如，子进程获得父进程数据空间、堆和栈的副本。 ......  由于在fork之后经常跟随着exec，所以现在的很多实现并不执行一个父进程数据段、栈和堆的完全副本。作为替代，使用了写时复制（Copy-On-Write,COW）技术，这些区域有父进程和子进程共享，而且内核将它们的访问权限改为只读。如果父进程和子进程中的任一个试图修改这些区域，则内核只为修改区域的那块内存制作一个副本。

这相当于在调用`fork`系统调用时，操作系统自动为*Redis*将其内存之中的数据镜像到子进程之中，子进程可以使用这份数据镜像生成**RDB**文件，而基于**COW**技术，父进程可以在子进程生成文件时，继续操作修改父进程内存中的数据，而不会影响子进程生成**RDB**文件。

当*Redis*的子进程完成**RDB**文件生成而退出时，父进程需要能够及时获得子进程状态的变化，父进程会在`serverCron`心跳调用中检查子进程的状态：
```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)
{
    ...
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {
        if ((pid = wait3(&statloc, WNOHANG, NULL)) != 0) {
            ...
            if (pid == server.rdb_child_pid)
            {
                backgroundSaveDoneHandler(exitcode, bysignal);
            }
            ...
        }
    }
    ...
}
```
通过`wait3`系统调用以`WNOHANG`参数非阻塞地检测所有子进程的状态，如果返回终止的子进程`pid`是`redisServer.rdb_child_pid`，则意味着生成**RDB**的子进程结束，父进程会回调`backgroundSaveDoneHandler`处理备份结束后的一些操作。

`backgroundSaveDoneHandler`这个接口则会调用`backgroundSaveDoneHandlerDisk`来清理一些在开启备份时缓存的数据：
```c
void backgroundSaveDoneHandlerDisk(int exitcode, int bysignal)
{
    server.dirty = server.dirty - server.dirty_before_bgsave;
    ...
    server.rdb_child_pid = -1;
    server.rdb_child_type = RDB_CHILD_TYPE_NONE;
    ...
}
```


****
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
