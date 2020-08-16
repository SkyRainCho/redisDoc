# Redis中键的过期策略

通过前面对数据库键空间的介绍，可以了解到，我们能够为存储在键空间之中的*key*设置一个过期时间，当*key*到达过期时间后，*Redis*便会将其从键空间之中自动删除。这种特性大大方便了我们在使用*Redis*作为存储模块时，清理过期数据的工作。那么在本篇文章中，我们就来介绍，过于*Redis*过期策略的实现细节。

在*Redis*之中，有些命令可以通过携带参数的方式为一个*key*设定其有效期，例如在**SET**命令中，可以通过*expiraion*这个参数，为给定的*key*设定有效时间；或者通过类似**EXPIRE**以及**EXPIREAT**这样的命令，为一个给定的*key*来设定其有效期：
1. **EXPIRE**，这个命令的格式为
    `EXPIRE key seconds`
    这个命令会为给定的*key*设置一个秒级别的有效期。
2. **EXPIREAT**，这个命令的格式为
    `EXPIREAT key timestamp`
    这个命令会为给定的*key*s设置一个UNIX时间戳形式有效期截止时间。

上述对于键有效期的设定，都是将数据存储在数据库的`redisDb.expires`这个字段中，可以理解为键空间`residDb.dict`中存储着*key-value*数据；而在`redisDb.expires`中则存储着*key-timestamp*数据。当我们针对键空间也就是`redisDb.dict`中的某一个*key*通过命令设定其有效期时，会通过一系列接口向`redisDb.expires`中插入对应的数据。

## 过期时间的存取
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

如果我们不希望某一个*key*过期的话，*Redis*则是通过`removeExpire`这个接口来实现删除过期的逻辑的：
```c
int removeExpire(redisDb *db, robj *key);
```
这个函数会通过`dictDelete`函数，将`key`从`redisDb.expires`中删除。

## 过期键的清理策略
前面我们已经了解了如何设置以及获取一个*key*的过期时间，那么对于一个已经过期的*key*，*Redis*是如何将其删除的呢？本小节将就这个问题作出详细的解释。

首先*Redis*清理过期*key*的一个策略为在从键空间查找*key*的时候，首先在`redisDb.expires`中查到判断对应的*key*是否过期，如果过期将这个*key*从*Redis*的键空间之中删除，然后在从键空间`redisDb.dict`中执行查找操作。这个策略的一个优势在于，对于这个操作对于CPU的占用是最小的，因为只有在访问*key*时，才会执行过期清理的策略；而对于没有被请求的*key*，其是否过期以及是否需要被清理自然不需要关系。应用这个策略对于系统性能的影响最低，但是这种策略会造成系统内存的浪费，想象一下若干个*key*已经过期，但是如果不对这些*key*进行访问，那么这些*key*以及其对应的*value*将会被残留在键空间中不会被删除，特别是*value*过大而又得不到释放时，对于内存的浪费更为明显。

为了避免对于内存的浪费，*Redis*还设计了另外一种主动清理过期键的策略，便是定期理清的机制，定期调用清理机制，从数据库的键空间之中清理一定数量的过期键，以增量的方式逐步完成对整个键空间的遍历。当然应用这个策略，可能会短时间阻塞主线程，会对系统的性能造成一定的影响，因此这个策略可以选择性的开启，在`redisServer`全局状态中有对应这个策略的开关：
```c
struct redisServer
{
    ...
    int active_expire_enabled;
    ...
};
```
这个开关在`initServerConfig`函数中被默认设置成了开启状态，同时没有对应的配置文件，如果希望关闭改功能，可以通过修改源代码重新编译的方式来实现。

虽然还没有介绍到相关内容，但是我们可以先简单的介绍一下，*Redis*可以以多机的方式组成一个集群，这样可以提高*Redis*的并发性并增强其可用性。例如*Redis*可以*Master-Slave*的模式启动多个实例，*Master*作为可写的实例，并定期将数据同步给*Slave*实例，*Slave*实例为可读实例，可以分担对数据的读取操作。在*Master-Slave*模式下的*Redis*集群中，为了保持数据的一致性，*Slave*实例不会主动地对过期的*key*进行删除，而是等待*Master*实例显式地向*Slave*实例发送一条**DEL**指令后，在执行删除操作。这样一来可以保证，在*Matser*中存在的*key*也同样会存在于*Slave*实例之中，即使这个*key*是一个过期待删除的*key*。

在*Redis*中，*Master*实例通过下面这个接口向所有的*Slave*实例广播过期*key*的**DEL**删除指令：
```c
void propagateExpire(redisDb *db, robj *key, int lazy);
```

### 在对键的访问过程中清理
有了上面获取一个给定键有效期的接口`getExpire`之后，我们可以用下面这个接口来判断一个*key*是否已经过期：
```c
int keyIsExpired(redisDb *db, robj *key);
```
这个函数被定义在*src/db.c*文件中，这个函数中集成三个判断*key*是否过期的逻辑。如果键过期，那么这个函数会返回1，没有过期或者没有设置过期时间则接口会返回0。这个接口的判断逻辑可以通过其代码细节了解到：
```c
int keyIsExpired(redisDb *db, robj *key) {
    mstime_t when = getExpire(db,key);
    mstime_t now;
    //如果这个给定的key没有有效期，显然说明这是一个永久存在的key
    if (when < 0) return 0; 
    //如果在启动的加载过程中，默认键不过期，会在后续主动定期清理的策略中进行删除
    if (server.loading) return 0;
    //现在需要确定now这个时间戳究竟是从什么时间开始算起
    if (server.lua_caller) {
        now = server.lua_time_start;
    }
    else if (server.fixed_time_expire > 0) {
        now = server.mstime;
    }
    else {
        now = mstime();
    }
    return now > when;
}
```
在上面这个函数中，最重要的一段逻辑便是如何确认`now`这个当前时间，对于当前时间的确认，*Redis*给出了三个中情况：
1. 如果在一个Lua脚本的上下文中，那么我们将执行Lua脚本的起始时间作为`now`的时间戳。
2. 如果在一个命令的执行过程之中，我们将执行明见的起始时间作为`now`。
3. 其他情况，便是使用当前的实际时间来作为`now`，

`keyIsExpired`这个接口只会用于判断一个给定的*key*是否过期，而不会关系如何处理一个过期的*key*。对于如何处理一个已经过期的*key*，则需要使用下面这个`expireifNeeded`接口来处理，现在我们来看一下这个函数接口的实现细节：
```c
int expireIfNeeded(redisDb *db, robj *key)
{
    //如果key没有过期，则不进行清理操作
    if (!keyIsExpired(db,key)) return 0;
    //如果是一个Slave实例，那么我们也不进行清理操作，而是等待Matser发送来的DEL指令
    if (server.masterhost != NULL) return 1;
    //如果是Master实例，或者是单一实例的Redis，那么开始执行清理操作，并尝试通知Slave实例
    server.stat_expiredkeys++;
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED, "expired",key,db->id);
    return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) : dbSyncDelete(db,key);
}
```
在这个函数中，我们可以注意到：
1. 在*Slave*实例中，虽然不会主动删除已经过期的*key*，但是这个函数还是会返回1，通知调用者，这个给定的*key*已经过期，保持与*Master*实例上行为一致的操作。
2. *Redis*通过一个全局的开关`redisServer.lazyfree_lazy_expire`来决定对过期*key*的删除是采用同步的方式还是异步的方式。对于异步的删除方式**惰性删除**，会在后续的内容之中进行介绍。但是不论是哪种删除方式，对于客户端来说都是透明的，*key*会立即从数据库的键空间之中被移除。

`expireIfNeeded`这个接口主要是在键空间针对某一个给定的`key`执行查找操作时背调用，例如`lookupKey*`这一系列函数之中，用以实现一种在对*key*的访问过程之中进行清理的逻辑。

### 主动定时批量清理
*Redis*对过期*key*的主动清理策略的接口被定义在*src/expire.c*源文件之中。相关策略在这个源文件中主要体现为两个函数接口的定义：
```c
int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now);
void activeExpireCycle(int type);
```
这其中`activeExpireCycle`是主动清理的逻辑入口，而`activeExpireCycleTryExpire`函数则是主动清理逻辑的辅助函数。

首先我们便从`activeExpireCycleTryExpire`函数入手，其中`db`参数为当前我们选中的数据库，`de`对应的是存储在`redisDb.expires`中的*key-timestamp*数据，那么这个函数会针对当前时间的时间戳`now`，对给定的*key-timestamp*数据`de`进行过期清理，如果没有过期，那么这个函数会返回0，否则函数返回1。这个这个函数清理过期*key*的过程与`expireIfNeeded`类似，都包含了调用`propagateExpire`接口尝试将删除指令同步给*Slave*实例的步骤；以及根据服务器的开关在键空间中执行对*key*的同步或者异步删除的步骤。

*Redis*有两种运行主动清理逻辑`activeExpireCycle`的方式：
1. `ACTIVE_EXPIRE_CYCLE_FAST`，这种*快速*清理方式会运行不超过`EXPIRE_FAST_CYCLE_DURATION`毫秒对过期的*key*进行清理。*Redis*会在每次进入事件循环之前，调用这个类型类型的主动清理逻辑进行*快速*清理，同时保证在上一次*快速*清理之后`EXPIRE_FAST_CYCLE_DURATION`时间内，不会再次进行*快速*清理。
2. `ACTIVE_EXPIRE_CYCLE_SLOW`，这种*慢速*清理方式时*Redis*中通用的普通清理模式，这种清理模式会在*Redis*的`databasesCron`中被调用，并且每次清理会占用一定百分比的`REDIS_HZ`时间，而这个百分比则是`ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC`定义的。

`activeExpireCycle`这个函数的逻辑我们可以通过其代码片段进行了解：
```c
void activeExpireCycle(int type)
{
    static unsigned int current_db = 0;
    ...
    for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++)
    {
        int expired;
        redisDb *db = server.db + (current_db % server.dbnum);
        current_db++;
        ...
        do
        {
            ...
            iteration++;
            ...
            expired = 0;
            num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;
            while (num--)
            {
                ...
                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                ...
                if (activeExpireCycleTryExpire(db,de,now)) expired++;
                ...
            }
            if ((iteration & 0xf) == 0) {
                elapsed = ustime()-start;
                if (elapsed > timelimit) {
                    timelimit_exit = 1;
                    break;
                }
            }
        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);    
    }
}
```
通过上面的代码片段，我们了解到这个主动清理函数的逻辑为：
1. 该函数每次调用会依次处理*Redis*中16个数据库中的若干个，该函数的静态变量`current_db`来记录当前处理的数据库。下次调用该函数时，可以通过`current_db`这个变量接续上次调用继续遍历处理*Redis*中的数据库。
2. 针对每一个被遍历的数据库，*Redis*则会采取每次通过`dictGetRandomKey`随机从数据库中取出`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP`个*key*进行采样，并调用`activeExpireCycleTryExpire`接口检查是否过期。
3. 每次采样`iteration`中，如果过期的*key*在整个的采样个数中占比不到25%的话，那么结束对这个数据库的检查与清理，进而检查下一个数据库。
4. 每进行16次采样，会检查一次是否达到了时间上限，如果达到，设置`timelimit_exit`，结束整个遍历过程。
***

## 过期时间相关命令
前面已经介绍*Redis*对于*key*有效期的相关内容，下面我们来讲一下关于*key*有效期相关的*Redis*命令。

### EXPIRE相关命令
这一系列命令主要是用于为*key*设定一个有效期。这一系列命令又可以分为两个类型：
1. 用于为*key*设定一个有效期时间段：

    EXPIRE key seconds
    PEXPIRE key milliseconds

这两个命令分别用于为*key*设定一个秒级的有效持续时间以及一个毫秒级的有效持续时间。
2. 用于为*key*设定一个有效期截至的时间点：
    
    EXPIREAT key time
    PEXPIREAT key ms_time

这两个命令分别用于为*key*设定一个有效期截至的秒级时间戳以及毫秒级时间戳。

*Redis*使用下面这个函数接口来实现上述的四个命令：
```c
void expireGenericCommand(client *c, long long basetime, int unit)
```
不论是设置有效期时间段，还是过期的时间点，在*Redis*数据库的`redisDb.expires`中都是以毫秒级过期时间点进行存储的，并且通过去我们前面介绍的`setExpire`接口进行设置。

### TTL相关命令
*Redis*为查询给定*key*的有效期提供了两个命令，分布是**TTL**以及**PTTL**这两个命令，这两个命令的格式为：

    TTL key
    PTTL key

对于一个不存在*key*，这两个命令会返回-2；如果这个*key*存在但是没有设置有效期，那么这两个命令会返回-1；对于一个存在且具有有效期的*key*，**TTL**命令会返回秒级有效时间，**PTTL**命令则会返回毫秒级剩余有效时间
```c
void ttlGenericCommand(client *c, int output_ms);
void ttlCommand(client *c);
void pttlCommand(client *c);
```
**TTL**以及**PTTL**这两个命令都是通过`ttlGenericCommand`这个函数实现的，该函数从`redisDb.expires`中查询给定*key*的有效期时间戳，再与当前时间相减计算*key*的剩余有效时间。

### PERSIST命令
*Redis*通过**PERSIST**移除一个*key*的有效期，使之永久存在于键空间之中，这个命令的格式为：

    PERSIST key

这个命令的实现函数为：
```c
void persistCommand(client *c);
```
该函数内部会通过调用`removeExpire`接口，将*key*从`redisDb.expires`哈希表中删除，使之成为成为永久存在的*key*。

***

![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*