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
2. 这个函数会更新*value*数据对象上用来表示**LRU**或者**LFU**数据字段`robj.lru`，除非当前有后台的**AOF**或者**RDB**持久化，或者`flags`参数之中携带者`LOOKUP_NOTOUCH`标记。

通过`lookupKey`这个函数实现了两组查找操作，分别对应于对于*key*的读操作以及写操作，这两个操作的区别在于，针对读操作的查找，会根据结果更新系统的键空间的命中计数器以及未命中计数器，而对于写操作的查找则不会更新这两个计数器，这两个统计计数器被定义在`redisServer`数据结构之中：
```c
struct redisServer{
    ...
    long long stat_keyspace_hits;
    long long stat_keyspace_misses;
    ...
};
```

### 数据库插入相关接口
针对数据库键空间的插入操作，本质上都是通过调用哈希表的`dictAdd`接口实现的，
```c
void dbAdd(redisDb *db, robj *key, robj *val);
void dbOverwrite(redisDb *db, robj *key, robj *val);
```
*Redis*提供了两个上述两个接口用于向数据库的键空间插入新的*key-value*，以及为覆盖已存在的*key*，通过这两个接口*Redis*定义了一个更高一级的接口用于实现键空间的插入（覆盖）操作：
```c
void setKey(client *c, redisDb *db, robj *key, robj *val);
```
这个接口会根据键是否存在于键空间之中选择调用`dbAdd`或者`dbOverwrite`这两个函数，同时如果是对已有的*key*进行覆盖，那么还会尝试移除它的有效期。

### 数据库查找相关接口
```c
int dbExists(redisDb *db, robj *key);
robj *dbRandomKey(redisDb *db);
```
函数`dbExists`通过`dictFind`接口来判断一个给定的*key*是否存在于键空间之中。

而函数`dbRandomKey`则通过`dictGEtRandomKey`这个接口从键空间之中返回一个随机的*key*。

### 数据库删除相关接口
```c
int dbSyncDelete(redisDb *db, robj *key);
int dbDelete(redisDb *db, robj *key);
```
*Redis*定义了`dbSyncDelete`接口用于将一个*key*从键空间之中同步地删除并释放，显而易见*Redis*还定义了一个异步删除释放的接口`dbAsyncDelete`，这个接口被定义在*src/expire.c*之中，用于执行一种**惰性释放**的策略，对于这个策略的介绍会在后续的内容中进行讲解。是否执行**惰性释放**，*Redis*会根据`redisServer.lazyfree_lazy_server_del`这个开关进行控制。

而`dbDelete`则是一个更高级的接口，它会根据服务器的开关，来决定使用`dbSyncDelete`还是`dbAsyncDelete`对*key*进行删除与释放。

下面这个`emptyDb`接口则是*Redis*用于清空整个键空间的接口。
```c
long long emptyDb(int dbnum, int flags, void(callback)(void*));
```
当`dbnum`为-1的话，则是清空所有数据库的键空间，否则清空指定数据库；而`flags`标记则会控制*Redis*是采取同步清空的逻辑，还是采取基于**惰性释放**策略的异步逻辑。

## 键空间操作命令
### FLUSHDB与FLUSHALL命令
*Redis*提供了两个用于清空数据库键空间命令**FLUSHDB**以及**FLUSHALL**，这两个命令的格式为：

    FLUSHDB [ASYNC]
    FLUSHALL [ASYNC]

**FLUSHDB**命令用于清空当前客户端选中的数据库键空间；**FLUSHALL**命令用于清空所有数据库的键空间。默认这两个命令在清空键空间时，都是采用同步的方式对键空间进行清空，这也就意味着如果键空间过于庞大的时候，同步清空会长期阻塞主线程，因此这对于较大的键空间，可以通过给定参数`ASYNC`来进行一种异步的**惰性释放**的策略，应用这种**惰性释放**策略，*Redis*会通过一个后台线程来实现对被删除键空间哈希表的释放。无论是否采用**惰性释放**，命令执行后，对于客户端来说，键空间都是处于一种被清空的状态。
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

上述的两个命令都可以接受多个*key*作为参数，并返回被删除*key*的个数。不过区别在于**DEL**命令使用的是同步的删除释放策略；而**UNLINK**命令则是与面携带`ASYNC`参数的**FLUSHDB**命令相似，采用异步的**惰性释放**策略，通过一个后台线程异步地对数据进行释放。
```c
void delGenericCommand(client *c, int lazy);
void delCommand(client *c);
void unlinkCommand(client *c);
```
上面三个函数中`delGenericCommand`是整个删除逻辑的基础，需要注意的是在对某一个*key*执行删除操作前需要调用`expireIfNeeded`来尝试处于过期*key*，防止那些已过期的*key*污染返回的删除个数，因为被`expireIfNeeded`删除的*key*本质上是属于“已经被删除”的*key*。

### EXISTS命令
**EXISTS**这个命令的格式为：
    EXISTS key [key ...]

该命令用于返回给定的*key*列表中存在于数据库键空间中的*key*的个数，该命令的实现函数为：
```c
void existsCommand(client *c);
```

### SELECT命令
**SELECT**命令的格式为：
    SELECT index
该命令的实现函数为：
```c
void selectCommand(client *);
```
这个命令通过调用数据库的`selectDb`接口，切换当前连接对应的客户端选择的数据库。

### RANDOMKEY命令
**RANDOMKEY**这个命令用于从数据库的键空间之中随机返回一个*key*，该命令的格式为：
    RANKDOMKEY

这个命令对应的实现接口为：
```c
void randomkeyCommand(client *c);
```
这个函数会调用`dbRankdomKey`接口来获取一个随机的*key*。

### KEYS命令
**KEYS**命令用于从数据库的键空间之中返回符合给定`pattern`的*key*的集合。该函数的格式为：
    KEYS pattern

这个命令对应的实现函数为：
```c
void keysCommand(client *c);
```
这个命令会遍历整个数据库的键空间来收集与`pattern`匹配的*key*来返回给客户端，主要注意的是，如果对于较大的键空间上调用的该命令，会对整个*Redis*的性能造成一定影响。

### DBSIZE命令
**DBSIZE**这个命令用于获取键空间中*key*的个数，
```c
void dbsizeCommand(client *c);
```
`dbsizeCommand`这个函数会通过`dictSize`这个接口来获取键空间对应哈希表中元素的个数。

### TYPE命令
**TYPE**命令用于获取给定*key*对应*value*的对象类型，该命令的格式为：
    TYPE key

该命令的实现函数为：
```c
void typeCommand(client *c);
```
该函数会查找到*key*对应*value*的`robj`指针，通过`robj.type`来返回对象的类型。

### RENAME系列命令
这一系列命令有两个，**RENAME**以及**RENAMENX**，这两个命令的格式为：
    RENAME key newkey
    RENAMENX key newkey

这两个命令的用于为将*key*更名为*newkey*，两个命令的区别在于，如果*newkey*存在，那么**RENAME**命令会将`newkey`这个对应的*key*覆盖掉；而**RENAMENX**命令只有在*newkey*不存在的情况下，才会重新命名成功。
这两个命令的实现函数为：
```c
void renameGenericCommand(Client *c, int nx);
void renameCommand(client *c);
void renamenxCommand(clinet *c);
```
这两个命令的基础逻辑，从键空间中查找到*key*对应的*key-value*，将其从键空间中删除，并将新的*newkey-value*插入到键空间中。

### MOVE命令
**MOVE**这个命令用于将给定的*key*从当前选中的键空间之中移动到另外一个键空间之中，该命令的格式为：
    MOVE key db

该命令的实现函数为：
```c
void moveCommand(client *c);
```
这个命令的基础逻辑为，将*key*从当前数据库键空间中取出，将其调用`dbAdd`接口加入到另外一个数据库的键空间之中。

### SWAPDB命令
**SWAPDB**命令用于交换*Redis*中两个给定数据库的键空间，其格式为：
    SWAPDB index index

该命令的实现函数为：
```c
int dbSwapDatabases(int id1, int id2);
void swapdbCommand(client *c);
```
其基础逻辑为交换两个数据库`redisDb`的成员指针内容，实现数据库键空间的快速交换。

****

![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
