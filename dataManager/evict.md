# Redis中的LRU淘汰机制与其他策略

在前面一段，我们对*Redis*对于*key*的过期策略进行了讨论，这部分比较容易理解，*key*在逻辑上具有一个有效期，当到达*key*的有效期时，*Redis*会自动对过期的*key*进行回收。

*Redis*的淘汰机制主要用在将*Redis*用作缓存的场景，当内存不足时，将某些认为未来不会被访问到的数据从数据库的键空间之中移除。当然这种淘汰机制不适用于将*Redis*作为存储的场景之中，因为作为存储用途的话，便不能随意的将一个*key*从数据库键空间之中删除，除非这个*key*是逻辑上被删除。

*Redis*现在实现了多种*key*的淘汰机制，按照淘汰*key*选择的范围可以分成三类：
1. 在*Redis*的内存到达上限时，不对*key*进行淘汰，而是禁止新的写入操作。
    `MAXMEMORY_NO_EVICTION`
2. 在*Reids*的内存到达上限时，会从设置了过期时间的键中，也就是`redisDb.expires`中，选择*key*进行淘汰，选择*key*的策略有四种：
    1. `MAXMEMORY_VOLATILE_LRU`，
    2. `MAXMEMORY_VOLATILE_LFU`，
    3. `MAXMEMORY_VOLATILE_TTL`，
    4. `MAXMEMORY_VOLATILE_RANDOM`，
3. 在*Redis*的内存到达上限时，会从整个数据库的键空间中，也就是`redisDb.dict`中，选择*key*进行淘汰，选择*key*的策略有三种：
    1. `MAXMEMORY_ALLKEYS_LRU`，
    2. `MAXMEMORY_ALLKEYS_LFU`，
    3. `MAXMEMORY_ALLKEYS_RANDOM`，

在代码层面，前面我们在介绍*Redis*对象时，曾经提过在`redisObject`对象中存在一个用于维护**LRU**数据或者**LFU**数据的字段，
```c
typedef struct redisObject {
    ...
    unsigned lru:LRU_BITS;
    ...
} robj;
```
同时，在前面介绍数据库键空间的底层接口时，我们也介绍过通过`lookupKey`这个接口我们可以对这个字段进行更新。

****
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
