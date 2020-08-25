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

## Redis中的LFU策略

## Redis中的淘汰逻辑

****
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
