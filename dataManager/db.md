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
2. `redisDb.expires`，这个字段也是一个哈希表类型，如果为数据库键空间中的某一个键设置了超时时间，那么会有一条记录存在这个哈希表中。
3. `redisDd.id`，这个字段用于表示当前数据库的序号。

除了上述三个字段之外，对于该数据库结构体的其他字段的介绍将在后续的内容之中进行展开。

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

### 数据库键空间

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

### 键的有效期

在*Redis*之中，有些命令可以通过携带参数的方式为一个*key*设定其有效期，例如在**SET**命令中，可以通过*expiraion*这个参数，为给定的*key*设定有效时间；或者通过类似**EXPIRE**以及**EXPIREAT**这样的命令，为一个给定的*key*来设定其有效期：
1. **EXPIRE**，这个命令的格式为
    `EXPIRE key seconds`
    这个命令会为给定的*key*设置一个秒级别的有效期。
2. **EXPIREAT**，这个命令的格式为
    `EXPIREAT key timestamp`
    这个命令会为给定的*key*s设置一个UNIX时间戳形式有效期截止时间。

上述对于键有效期的设定，都是将数据存储在数据库的`redisDb.expires`这个字段中，可以理解为`residDb.dict`中存储着*key-value*数据；而在`redisDb.expires`中则存储着*key-timestamp*数据。当我们针对键空间也就是`redisDb.dict`中的某一个*key*通过命令设定其有效期时，会通过一系列接口向`redisDb.expires`中插入对应的数据。

*Redis*首先定义了一个接口用于为给定的*key*设定有效期：

```c
void setExpire(client *c, redisDb *db, robj *key, long long when);
```
这个函数会向一个已经存在于`redisDb.dict`中的`key`设定一个毫秒级的过期时间戳`when`，然后通过调用`dictAddOrFind`以及`dictSetSignedIntegerVal`这两个哈希表接口将这个`key`以及`when`插入到数据库的`redisDb.expires`字段之中。

例如在字符串对象类型的**SET**命令的实现中：
```c
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply)
{
    ...
    if (expire) setExpire(c,c->db,key,mstime()+milliseconds);
    ...
}
```

对于一个给定的*key*，我们可以使用下面这个`getExpire`接口来查询其对应的过期时间戳：
```c
long long getExpire(redisDb *db, robj *key);
```
当这个给定的`key`没有一个关联的过期时间戳的话，这个函数会返回-1；否则这个函数会返回这个`key`对应的过期时间戳。

通过上面`getExpire`这个函数接口获取到一个*key*的过期时间戳之后，*Reids*定义了用于检查一个*key*是否过期的接口：
```c
int keyIsExpired(redisDb *db, robj *key);
```
这个函数中集成三个判断*key*是否过期的逻辑。如果键过期，那么这个函数会返回1，没有过期或者没有设置过期时间则接口会返回0。不过需要注意的是，这个接口仅仅用于判断*key*是否过期，对于一个已经过期的*key*的处理，则需要用到下面的两个接口：
```c
void propagateExpire();
int expireIfNeeded(redisDb *db, robj *key);
```
简单来说，通过调用`expireIfNeeded`这个接口，可以讲一个已经过期的*key*从*Redis*的键空间之中删除，当我们使用查找函数从键空间中查找对应的*key*时，会先调用`expireIfNeeded`这个接口，尝试对过期的*key*进行删除，然后在从键空间中进行查找，例如键空间的查找接口`lookupKeyReadWithFlags`：
```c
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags)
{
    if (expireIfNeeded(db, key) == 1)
    {
        ...
    }
    val = lookupKey(db, key, flags);
    ...

    return val;
}
```
如果要从更细节的层面来讨论这个`expireIfNeeded`接口时，我们需要考虑两个问题：
1. 过期*key*的删除策略
2. 处理*Redis*主从模式的过期策略 

#### 过期删除策略
*Redis*给出了一种名为**惰性删除**的策略，应用**惰性删除**策略，如果删除一个*key*时，其对应的*value*不会立即被释放，而是被加入到惰性删除队列，以异步的形式被释放，与之相反的，如果服务器采用**非惰性删除**策略，那么在删除一个*key*时，其对应的*value*会立即同步地被释放删除。这个策略的开关被定义在`redisServer`这个全局状态结构体之中：
```c
struct redisServer
{
    ...
    /* Lazy free */
    int lazyfree_lazy_eviction;
    int lazyfree_lazy_expire;
    int lazyfree_lazy_server_del;
    int lazyfree_lazy_user_del;
    ...
};
```
以上的四个策略开关的含义为：
1. `redisServer.lazyfree_lazy_eviction`，是否在淘汰某个*key*时，使用**惰性删除**策略。
2. `redisServer.lazyfree_lazy_expire`，是否在过期某个*key*时，使用**惰性删除**策略。
3. `redisServer.lazyfree_lazy_server_del`，服务器端删除某个*key*时，是否使用**惰性删除**策略。
4. `redisServer.lazyfree_lazy_user_del`，客户端通过命令删除某个*key*时，是否使用**惰性删除**策略。  

## 数据库的基础API


如果我们不希望某一个*key*过期的话，*Redis*则是通过`removeExpire`这个接口来实现删除过期的逻辑的：
```c
int removeExpire(redisDb *db, robj *key);
```
这个函数会通过`dictDelete`函数，将`key`从`redisDb.expires`中删除。

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

在这几个函数中，`lookupKey`是一个底层的查找接口，

```c
robj *lookupKey(redisDb *db, robj *key, int flags);
```

这个接口是一个非常底层的查找接口，其主要逻辑只有两个：

1. 通过调用`dictFind`函数以及`dictGetVal`函数，来从键空间之中查找到*key*对应的*value*。
2. 在没有子进程进行*RDB*保存或者*AOF*重写时，这个函数会根据参数`flags`，更新*value*数据对象上用来表示**LRU**或者**LFU**数据字段`robj.lru`。

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



****

![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
