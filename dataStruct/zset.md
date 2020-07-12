# Redis中有序集合对象类型的实现

在*Redis*中有序集合对象和集合对象类型一样，都是用于多个字符串*key*数据的集合型对象，有序集合与集合对象一致，都不允许出现相同的*key*；但是与集合对象类型不同的是，集合对象类型本质上是采用哈希方法进行编码的，因为其保存的*key*都是无序的，而有序集合会为每一个*key*关联一个`double`类型的分值，基于这个分值，有序集合会对其内部存储的*key*按照分值从小到大的顺序进行排序。

与散列对象、集合对象一样，有序集合对象同样是采用两个底层编码方式来实现。对于有序集合对象来说，底层有压缩链表编码方式`OBJ_ENCODING_ZIPLIST`以及跳跃表编码方式`OBJ_ENCODING_SKIPLIST`，对于`OBJ_ENCODING_ZIPLIST`编码来说，其底层就是使用压缩链表来存储数据的；而`OBJ_ENCODING_SKIPLIST`编码，*Redis*定义了一个数据结构：
```c
typedef struct zset
{
	dict *dict;
	zskiplist *zsl;
} zset;
```

同时，*Redis*还定义了两个阈值，`redisServer.zset_max_ziplist_entries`以及`redisServer.zset_max_ziplist_value`，用于在创建有序集合时选择编码方式，以及在操作有序集合时执行编码转换，在默认的情况下，*Redis*会选择压缩链表作为有序集合的编码方式。在这两个阈值中`redisServer.zset_max_ziplist_entries`表示当使用压缩链表编码方式的有序集合最多可以存储元素的个数，当压缩链表中的元素个数超过这个阈值，那么这个有序集合会被转换成跳跃表的编码方式，同时这个阈值可以通过配置文件在启动时进行调整，如果被设置为0，那么可以认为当前的*Redis*永远都会使用跳跃表作为有序集合的编码方式；`redisServer.zset_max_ziplist_value`则是表示使用压缩链表编码时，元素最大的长度，当向一个有序集合中插入的*key*的长度大于这个阈值的时候，便会采用跳跃表的方式进行编码。

## 基于压缩链表的底层操作
```c
double zzlGetScore(unsigned char *sptr);
sds ziplistGetObject(unsigned char *sptr);
int zzlCompareElements(unsigned char *eptr, unsigned char *cstr, unsigned int clen);
unsigned int zzlLength(unsigned char *zl);
void zzlNext(unsigned char *zl, unsigned char **eptr, unsigned char **sptr);
void zzlPrev(unsigned char *zl, unsigned char **eptr, unsigned char **sptr);
int zzlIsInRange(unsigned char *zl, zrangespec *range);
unsigned char *zzlFirstInRange(unsigned char *zl, zrangespec *range);
unsigned char *zzlLastInRange(unsigned char *zl, zrangespec *range);
int zzlLexValueGteMin(unsigned char *p, zlexrangespec *spec);
int zzlLexValueLteMax(unsigned char *p, zlexrangespec *spec);
int zzlIsInLexRange(unsigned char *zl, zlexrangespec *range);
unsigned char *zzlFirstInLexRange(unsigned char *zl, zlexrangespec *range);
unsigned char *zzlLastInLexRange(unsigned char *zl, zlexrangespec *range);
unsigned char *zzlFind(unsigned char *zl, sds ele, double *score);
unsigned char *zzlDelete(unsigned char *zl, unsigned char *eptr);
unsigned char *zzlInsertAt(unsigned char *zl, unsigned char *eptr, sds ele, double score);
unsigned char *zzlInsert(unsigned char *zl, sds ele, double score);
unsigned char *zzlDeleteRangeByScore(unsigned char *zl, zrangespec *range, unsigned long *deleted);
unsigned char *zzlDeleteRangeByLex(unsigned char *zl, zlexrangespec *range, unsigned long *deleted);
unsigned char *zzlDeleteRangeByRank(unsigned char *zl, unsigned int start, unsigned int end, unsigned long *deleted);
```

## 有序集合对象的底层操作
```c
unsigned long zsetLength(const robj *zobj);
void zsetConvert(robj *zobj, int encoding);
void zsetConvertToZiplistIfNeeded(robj *zobj, size_t maxelelen);
int zsetScore(robj *zobj, sds member, double *score);
int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore);
int zsetDel(robj *zobj, sds ele);
long zsetRank(robj *zobj, sds ele, int reverse);
```


## 有序集合对象命令

### ZADD与ZINCRBY命令
```c
void zaddGenericCommand(client *c, int flags);
void zaddCommand(client *c);
void zincrbyCommand(client *c);
```


### ZREM命令
```c
void zremCommand(client *c);
```


### ZREMRANGEBYRANK、ZREMRANGEBYSCORE与ZREMRANGEBYLEX命令
```c
void zremrangeGenericCommand(client *c, int rangetype);
void zremrangebyrankCommand(client *c);
void zremrangebyscoreCommand(client *c);
void zremrangebylexCommand(client *c);
```


### ZUNIONSTORE与ZINTERSTORE命令
```c
void zunionInterGenericCommand(client *c, robj *dstkey, int op);
void zunionstoreCommand(client *c);
void zinterstoreCommand(client *c);
```


### ZRANGE与ZREVRANGE命令
```c
void zrangeGenericCommand(client *c, int reverse);
void zrangeCommand(client *c);
void zrevrangeCommand(client *c);
```


### ZRANGEBYSCORE与ZREVRANGEBYSCORE命令
```c
void genericZrangebyscoreCommand(client *c, int reverse);
void zrangebyscoreCommand(client *c);
void zrevrangebyscoreCommand(client *c);
```


### ZCOUNT命令
```c
void zcountCommand(client *c);
```

### ZLEXCOUNT命令
```c
void zlexcountCommand(client *c);
```


### ZRANGEBYLEX与ZREVRANGEBYLEX命令
```c
void genericZrangebylexCommand(client *c, int reverse);
void zrangebylexCommand(client *c);
void zrevrangebylexCommand(client *c);
```


### ZCARD命令
```c
void zcardCommand(client *c);
```


### ZSCORE命令
```c
void zscoreCommand(client *c);
```


### ZRANK与ZREVRANK命令
```c
void zrankGenericCommand(client *c, int reverse);
void zrankCommand(client *c);
void zrevrankCommand(client *c);
```


### ZSCAN命令
```c
void zscanCommand(client *c);
```


### ZPOPMIN与ZPOPMAX命令
```c
void genericZpopCommand(client *c, robj **keyv, int keyc, int where, int emitkey, robj *countarg);
void zpopminCommand(client *c);
void zpopmaxCommand(client *c);
```

### BZPOPMIN与BZPOPMAX命令
```c
void blockingGenericZpopCommand(client *c, int where);
void bzpopminCommand(client *c);
void bzpopmaxCommand(client *c);
```
***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
