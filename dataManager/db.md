# Redis中内存数据库API实现

前面我们已经讲解了在*Redis*之中基础的数据结构，以及五种数据对象类型，那么在这篇文章中，我们将介绍*Redis*之中，所有的数据对象是以何种方式被组织起来的，也就是内存数据库的数据结构与业务逻辑。

## 内存数据库概述

在*Redis*之中对于数据库的描述，被定义在*src/server.h*这个头文件之中：

```c
typedef strcut redisDb
{
    dict *dict;
    dict *expires;
    dict *blocking_keys;
    dict *ready_keys;
    dict *watched_keys;
    int id;
    long long avg_ttl;
    unsigned long expires_cursor;
    list *defrag_later;
} redisDb;
```

以上这个结构体便是*Redis*中对内存数据库的描述，在这个数据结构之中：

1. `redisDb.dict`，该字段是一个哈希表类型，用于维护这个数据库的键空间，前面我们讲解的五种数据对象类型，都按照*key-value*的形式存储在这个哈希表中。
2. `redisDb.expires`，这个字段也是一个哈希表类型，如果为数据库键空间中的某一个键设置了过期时间，那么会有一条记录存在这个哈希表中。
3. `redisDd.id`，这个字段用于表示当前数据库的序号。

对于*Redis*数据库来说，最为核心的便是`redisDb.dict`以及`redisDb.expires`这两个大的哈希表结构，除了上述三个字段之外，对于该数据库结构体的其他字段的介绍将在后续的内容之中进行展开。

在*Redis*中，服务器用于多个`redisDb`数据库，这些数据库被定义在*Redis*服务器全局状态中：

```c
struct redisServer
{
    ...
    redisDb *db;
    int dbnum;
    ...
};
```

在这个数据结构之中，`redisServer.db`是一个存储所有数据库结构体`redisDb`的数组，而`redisServer.dbnum`用于存储对应数据库数组的长度。在*Redis*服务器的初始化函数中，会为这个数据库数组分配空间：

```c
void initServer(void) {
    ...
    server.db = zmalloc(sizeof(redisDb)*server.dbnum);
    ...
}
```

默认当我们连接到*Redis*服务器上，默认是使用0号数据库，使用**SELECT**命令，我们可以切客户端所选择的数据库，这个命令的格式为：

`SELECT index`

在服务器中用于表示客户端连接的数据结构之中，也保存了一个数据库`redisDb`结构体的指针，

```c
typedef struct client
{
    ...
    redisDb *db
    ...
} client;
```

在这个结构体中`client.db`会指向`redisServer.db`数组中的某一个`redisDb`数据库。

## 数据库键空间

通过上面的简单介绍，我们可以看到，虽然在散列对象类型、集合对象类型、有序集合对象类型中，都会使用哈希表作为数据的底层实现方式，但是整个*Redis*还会维护数个大的哈希表，用于实现*key-value*的存储；而上述的五种数据对象类型，则是作为*value*存储在数据库的大哈希表中的，而通过前面的对于数据对象类型的介绍，我们可以发现，*Redis*会使字符串对象类型来作为*value*所对应的*key*，用以在哈希表中实现对数据的快快速查找。

例如，当我们执行如下的命令设置一个字符串时：

    SET key:1 value:1

这条命令会向*Redis*的键空间之中插入一个新的*key-value*，其中*key*是一个字符串对象`key:1`，这个*key*所对应的*value*也是一个字符串对象`value:1`。

同时在前面讲解哈希表数据结构时的一个问题在此处也可以得到解释，便是：

```c
int dictRehashMilliseconds(dict *d, int ms);
static void _dictRehashStep(dict *d);
```

上述两个函数都是用于实现对哈希表进行重哈希操作的，其区别在于：

- `dictRehashMilliseconds`，这个函数是对给定的哈希表进行持续1毫秒的重哈希，在这1毫秒内会尽可能多的实现*key-value*数据的转移。
- `_dictRehashStep`，这函数则是每次对给定哈希表执行单步的重哈希操作，一次调用只会移动一个*key-value*数据。

除了上述已知的区别之外，另外一个重要的一点在此处便可以得到解释：

- `dictRehashMilliseconds`，这个函数是在系统的心跳中被主动调用，处理的也是`redisDb.dict`这个大的哈希表。
- `_dictRehashStep`，这个函数则是在针对某一个哈希表进行操作做时被动调用，这个函数重哈希的对象则是作为*value*存储在`redisDb.dict`中的散列对象、集合对象、有序集合对象这种对象数据底层的哈希表。


### 数据库查找相关接口
```c
robj *lookupKey(redisDb *db, robj *key, int flags);
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags);
robj *lookupKeyRead(redisDb *db, robj *key);
robj *lookupKeyWriteWithFlags(redisDb *db, robj *key, int flags);
robj *lookupKeyWrite(redisDb *db, robj *key);
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply);
robj *lookupKeyWriteOrReply(client *c, robj *key, robj *reply);
```
在这几个函数中，`lookupKey`是一个底层的查找接口，其主要逻辑有两个：
1. 通过调用`dictFind`函数以及`dictGetVal`函数，来从键空间之中查找到*key*对应的*value*。
2. 在没有子进程进行*RDB*保存以及*AOF*重写时，同时如果这个查找操作没有携带这个`LOOKUP_NOTOUCH`标记，这个函数会根据参数`flags`，更新*value*数据对象上用来表示**LRU**或者**LFU**数据字段`robj.lru`。

### 数据库插入相关接口

```c
void dbAdd(redisDb *db, robj *key, robj *val);
int dbAddRDBLoad(redisDb *db, sds key, robj *val);
void dbOverwrite(redisDb *db, robj *key, robj *val);
void genericSetKey(client *c, redisDb *db, robj *key, robj *val, int keepttl, int signal);
void setKey(client *c, redisDb *db, robj *key, robj *val);
```

### 数据库查找相关接口

```c
int dbExists(redisDb *db, robj *key);
robj *dbRandomKey(redisDb *db);
```

### 数据库删除相关接口

```c
int dbSyncDelete(redisDb *db, robj *key);
int dbDelete(redisDb *db, robj *key);
long long emptyDbGeneric(redisDb *dbarray, int dbnum, int flags, void(callback)(void*));
```

## 键空间操作命令
### FLUSHDB与FLUSHALL命令
*Redis*提供了两个用于清空数据库键空间命令**FLUSHDB**以及**FLUSHALL**，这两个命令的格式为：

    FLUSHDB [ASYNC]
    FLUSHALL [ASYNC]

**FLUSHDB**命令用于清空当前客户端选中的数据库键空间；**FLUSHALL**命令用于清空所有数据库的键空间。默认这两个命令在清空键空间时，都是采用同步的方式对键空间进行清空，这也就意味着如果键空间过于庞大的时候，同步清空会长期阻塞主线程，因此这对于较大的键空间，可以通过给定参数`ASYNC`来进行一种异步的**惰性删除**的策略，应用这种**惰性删除**策略，*Redis*会通过一个后台线程来实现对被删除键空间哈希表的释放。无论是否采用**惰性删除**，命令执行后，对于客户端来说，键空间都是处于一种被清空的状态。
```c
int getFlushCommandFlags(client *c, int *flags);
void flushdbCommand(client *c);
void flushallCommand(client *c);
```
这三个函数用于实现**FLUSHDB**以及**FLUSHALL**这两个命令，其内部是通过键空间的`emptyDb`这个接口来实现的。

### DEL与UNLINK命令
**DEL**命令与**UNLINK**命令都是*Redis*提供的，用于从数据库键空间之中删除给定*key*的命令，这两个命令的格式为：
    DEL key [key ...]
    UNLINK key [key ...]

上述的两个命令都可以接受多个*key*作为参数，并返回被删除*key*的个数。不过区别在于**DEL**命令使用的是同步的删除释放策略；而**UNLINK**命令则是与面携带`ASYNC`参数的**FLUSHDB**命令相似，采用异步的**惰性删除**策略，通过一个后台线程异步地对数据进行释放。
```c
void delGenericCommand(client *c, int lazy);
void delCommand(client *c);
void unlinkCommand(client *c);
```


### EXISTS命令

### SELECT命令

### RANDOMKEY命令

### KEYS命令

### DBSIZE命令

### LASTSAVE命令

### TYPE命令

### RENAME命令

### MOVE命令

### SWAPDB命令



****

![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
