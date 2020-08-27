# Redis中的LRU淘汰机制与其他策略

在前面一段，我们对*Redis*对于*key*的过期策略进行了讨论，这部分比较容易理解，*key*在逻辑上具有一个有效期，当到达*key*的有效期时，*Redis*会自动对过期的*key*进行回收。

*Redis*的淘汰机制主要用在将*Redis*用作缓存的场景，当内存不足时，将某些认为未来不会被访问到的数据从数据库的键空间之中移除。当然这种淘汰机制不适用于将*Redis*作为存储的场景之中，因为作为存储用途的话，便不能随意的将一个*key*从数据库键空间之中删除，除非这个*key*是逻辑上被删除。

*Redis*现在实现了多种*key*的淘汰机制，按照淘汰*key*选择的范围可以分成三类：
1. 在*Redis*的内存到达上限时，不对*key*进行淘汰，而是禁止新的写入操作。
    `MAXMEMORY_NO_EVICTION`
2. 在*Reids*的内存到达上限时，会从设置了过期时间的键中，也就是`redisDb.expires`中，选择*key*进行淘汰，选择*key*的策略有四种：
    1. `MAXMEMORY_VOLATILE_LRU`，这种策略会使用近似的**LRU**策略，选择*key*进行淘汰。
    2. `MAXMEMORY_VOLATILE_LFU`，这种策略会使用**LFU**策略，选择*key*进行淘汰。
    3. `MAXMEMORY_VOLATILE_TTL`，这种策略会选择过期时间最早的*key*进行淘汰。
    4. `MAXMEMORY_VOLATILE_RANDOM`，这种策略会从`redisDb.expires`的*key*中随机选择进行淘汰。
3. 在*Redis*的内存到达上限时，会从整个数据库的键空间中，也就是`redisDb.dict`中，选择*key*进行淘汰，选择*key*的策略有三种：
    1. `MAXMEMORY_ALLKEYS_LRU`，这种策略会使用近似的**LRU**策略，选择*key*进行淘汰。
    2. `MAXMEMORY_ALLKEYS_LFU`，这种策略会使用**LFU**策略，选择*key*进行淘汰。
    3. `MAXMEMORY_ALLKEYS_RANDOM`，这种策略会从`redisDb.dict`的*key*中随机选择进行淘汰。

在代码层面，前面我们在介绍*Redis*对象时，曾经提过在`redisObject`对象中存在一个用于维护**LRU**数据或者**LFU**数据的字段，
```c
typedef struct redisObject {
    ...
    unsigned lru:LRU_BITS;
    ...
} robj;
```
同时，在前面介绍数据库键空间的底层接口时，我们也介绍过通过`lookupKey`这个接口我们可以对这个字段按照服务器的配置，更新**LRU**数据或者**LFU**数据：
```c
robj *lookupKey(redisDb *db, robj *key, int flags) {
    ...
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        updateLFU(val);
    }
    else
    {
        val->lru = LRU_CLOCK();
    }
    ...
}
```
解析来我们会对*Redis*所使用的**LRU**策略、**LFU**策略以及*Redis*所使用的淘汰策略进行讲解与介绍。

## Redis中的LRU策略
首先我们需要明确的一点是，*Redis*中使用的是一种近似的**LRU**策略，而不是绝对精确的**LRU**策略。这也就意味着，*Redis*会尽可能的将空闲时间较长的*key*从数据库的键空间之中淘汰，但是不保证被淘汰的*key*一定是键空间中空闲时间最长的。因为*Redis*在实现**LRU**策略时，并不是采用**LRU**的经典实现方式，而是选择随机算法，以采样的方式选择一部分*key*，并将这其中空闲时间最长的*key*作为一次选择的结果。

对于使用**LRU**策略的*Redis*，会在`redisObject.lru`字段之中存储一个**LRU**跳数，系统默认设置的**LUR**跳数分辨率为1000毫秒1次：
```c
#define LRU_CLOCK_RESOLUTION 1000
```
同时*Redis*提供了一个接口用于获取基于当前毫秒时间戳的跳数：
```c
unsigned int getLRUClock(void);
```
这个接口会通过系统调用`mstime()`来获取系统的毫秒时间戳，因为这个调用设计用户态和内核态之间的切换，因此频繁地直接调用会大大地降低系统的性能，因此*Redis*在服务器的全局数据之中定义了一个字段用户缓存当前的**LRU**时间戳跳数`redisServer.lruclock`，*Redis*会在系统事件循环的心跳`serverCron`之中调用`getLRUClock`接口，将跳数缓存下来。系统的频率由`redisServer.hz`来决定，默认是每秒钟10次。

*Redis*给出了一个接口，用于更新*Redis*对象的`lru`数据信息：
```c
unsigned int LRU_CLOCK(void);
```
正常情况下，**LRU**跳数的频率为1000毫秒1次，而系统心跳则是1000毫秒10次，对于这种情况`LRU_CLOCK`函数会之中从`redisServer.lruclock`缓存中获取当前的**LRU**时间戳跳数；当然我们可以通过修改代码的方式提高**LRU**跳数的频率，并且可以通过配置降低系统心跳的频率，对于系统心跳频率低于**LRU**频率时，`redisServer.lruclock`缓存无法及时更新，对于这种情况，通过`getLRUClock`函数来获取时间戳跳数。

最后，*Redis*定义了一个接口`estimateObjectIdleTime`，通过这个接口，我们可以计算出一个*Redis*对象没有被访问的空闲时间毫秒数。
```c
unsigned long long estimateObjectIdleTime(robj *o);
```

## Redis中的LFU策略
**LFU**策略又被称为最不经常使用策略，和**LRU**策略的区别在于：
1. **LRU**认为只有一个数据最近被访问过，那么在未来其被访问的概率也就更高。
2. **LFU**则认为一个数据在最近一段时间内被访问的次数越多，那么在未来其被访问的概率也就越高。

与*Redis*中实现的**LRU**策略一样，*Redis*中的**LFU**策略同样是一个近似的**LFU**策略，而不是一个准确的**LFU**策略。同时在开启了**LFU**策略后，*Redis*会使用`redisServer.lruclock`来存储*Redis*对象的**LFU**信息。这个24位的`redisServer.lruclock`字段被划分为两个部分：

         16 bits      8 bits
    +----------------+--------+
    + Last decr time | LOG_C  |
    +----------------+--------+

1. `LOG_C`，这个字段占用8个比特，是一个对数计数器，用于标记一个对象的访问频数，这个字段不但可以随着对数据的访问而增加，同时也可以随着时间递减，之所以这么处理，是为了防止过去很久一段时间曾经被大量访问的对象无法被淘汰。

2. 剩余的16个比特用于存储一个递减的时间，由于只有16位，因此存储的是一个由Unix时间戳简化存储的分钟时间戳。

键空间之中新建的对象，其对应的`LOG_C`计数器并非是从0开始的，这是为了防止一个新的对象立刻被*Redis*的淘汰机制给删除，初始的对象的计数器被设定为`COUNTER_INIT_VAL`，其定义为
```c
#define COUNTER_INIT_VAL 5
```
在代码中体现在创建*Redis*对象的函数接口`createObject`中：
```c
robj *createObject(int type, void *ptr) {
    ...
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    }
    else
    {
        o->lru = LRU_CLOCK();
    }
}
```

对于*Redis*中的**LFU**策略，在*src/evict.c*文件中定义四个相关的函数接口。
```c
unsigned long LFUGetTimeInMinutes(void);
```
`LFUGetTimeInMinutes`这个函数用于获取16位的当前的基于Unix时间戳的分钟时间戳。

```c
unsigned long LFUTimeElapsed(unsigned long ldt);
```
`LFUTimeElapsed`这个接口与**LRU**策略中的`estimateObjectIdleTime`接口相同，用于计算对象上次访问时间戳距今的空闲时间。

```c
uint8_t LFULogIncr(uint8_t counter);
```
`LFULogIncr`这个接口用于对数地对计数器进行递增操作。这个接口会概率性地对计数器进行递增，简单来说`counter`的数值越高，`counter`计数器保持不变的概率就越大，同时`counter`的上限为255。

```c
unsigned long LFUDecrAndReturn(robj *o);
```
`LFUDecrAndReturn`这个接口用于对*Redis*对象的**LFU**计数器进行递减操作，*Redis*在全局数据中定义了一个**LFU**递减因子`redisServer.lfu_decay_time`，通过**LFU**空闲时间以及这个递减因子，我们可以完成对计数器的更新。

在*src/db.c*源文件中，我们可以看到如何应用上述的几个函数来更新对象的**LRU**信息：
```c
void updateLFU(robj *val) {
    unsigned long counter = LFUDecrAndReturn(val);
    counter = LFULogIncr(counter);
    val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}
```

## Redis中的淘汰逻辑
前面我们提到，*Redis*在对*key*进行淘汰时，采用的都是近似的**LRU**、**LFU**以及**TTL**等策略。也就是基于随机算法，通过采样来选择最佳的待淘汰*key*。因此*Redis*定义了一个辅助的数据结构，用以实现随机采样：
```c
#define EVPOOL_SIZE 16
struct evictionPoolEntry {
    unsigned long long idle;
    sds key;
    sds cached;
    int dbid;
};
static struct evictionPoolEntry *EvictionPoolLRU;
```
通过`evictionPoolAlloc`这个接口，*Redis*会为这个**淘汰池**分配内存并进行初始化。

对于*key*的淘汰，*Redis*只会在设置了内存上限，并且所使用的内存已经达到上限时才会进行。在服务器的全局数据之中定义了数据库可以使用的内存上限`redisServer.maxmemory`，如果这个值被设置成0，那么意味着数据没有内存上限，因此这种情况将不会触发淘汰机制。

对于设置了内存上限的情况，*Redis*定义了两个函数用于统计内存使用，判断是否当前需要进行淘汰：
```c
size_t freeMemoryGetNotCountedMemory(void);
int getMaxmemoryState(size_t *total, size_t *logical, size_t *tofree, float *level);
```

对于**AOF**所使用的内存缓存，以及主从模式下**Slave**的输出缓存所使用的内存，*Redis*不希望将其记录在内存统计之内，因此*Redis*需要通过`freeMemoryGetNotCountedMemory`这个接口来获取上述这部分缓存所使用的内存，将总的内存用量减去上述这部分内存，便获得实际需要被统计的内存使用量。

而通过`getMaxmemoryState`接口，我们可以判断当前*Redis*服务器的内存使用状态，进而判断是否需要启动淘汰机制，如果该函数返回`C_OK`表明当前内存使用量没有达到上限，无需进行淘汰；反之如果函数返回`C_ERR`则表明当前内存使用量已经达到上限，需要启动淘汰机制。在函数返回`C_ERR`时，通过函数传入的指针参数还可以获得当前*Redis*内存使用的具体数据：

1. `total`，通过该字段可以返回当前*Redis*使用的总内存数量。
2. `logical`，这个参数会返回*Redis*使用的逻辑内存，也就是`total`的内存减去`freeMemoryGetNotCountedMemory`之后的数量。
3. `tofree`，这个参数会返回*Redis*需要被释放的内存数量。
4. `level`，这个参数会返回当前内存的使用率，即`logical`/`redisServer.maxmemory`的结果。

而*Redis*淘汰机制的核心是通过`freeMemoryIfNeeded`这个接口来实现的：

```c
int freeMemoryIfNeeded(void);
```

*Redis*会在每次执行客户端发来的命令前执行该函数，尝试对*key*进行淘汰。在了解这个函数的实现之前，我们首先看一个辅助函数`evictionPoolPopulate`，收集合适的*key*，将相关的数据填充到`EvictionPoolLRU`这个**淘汰池**中：
```c
void evictionPoolPopulate(int dbid, dict *sampledict, dict *keydict, struct evictionPoolEntry *pool);
```
`evictionPoolPopulate`这个接口，用于从给定编号`dbid`数据库的待采样哈希表`sampledict`中进行采样，如果当前系统采用的是**LRU**策略，那么这个待采样哈希表就是`redisDb.expires`；否则的话便采用数据库的键空间`redisDb.dict`作为待采样数据集。每次调用该接口会从待采样数据集中通过`dictGetSomeKeys`接口最多采集`redisServer.maxmemory_samples`个样本，**淘汰池**中的数据按照闲置时间的升序方式进行排序，不同策略的闲置时间的获取方式如下面的代码片段所示：
```c
void evictionPoolPopulate(int dbid, dict *sampledict, dict *keydict, struct evictionPoolEntry *pool) {
    ...
        if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
            idle = estimateObjectIdleTime(o);
        }
        else if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
            idle = 255-LFUDecrAndReturn(o);
        }
        else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
            idle = ULLONG_MAX - (long)dictGetVal(de);
        }
    ...
}
```
有了这个`evictionPoolPopulate`辅助函数，我们就可以来看看`freeMemoryIfNeeded`的实现细节了，这个函数的基本逻辑可以由下面几点进行简单的描述：
1. 通过调用`getMaxmemoryState`获得当前系统的内存使用信息。
2. 如果需要执行淘汰策略回收内存，那么记录需要被释放的内存数量`mem_tofree`。
3. 对*Redis*的每一个数据库调用`evictionPoolPopulate`进行采样，这样一来，**淘汰池**中变记录了当前一轮采样中最**闲置**的一组数据。
4. 从这一组**闲置**数据之中，选择最为**闲置**的哪一个数据进行淘汰。
5. 如果淘汰数据释放的内存满足`mem_tofree`，那么结束淘汰流程，否侧重复步骤2，3。

****
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
