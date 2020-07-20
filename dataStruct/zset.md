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

在这个数据结构之中，`zset.dict`这个哈希表中存储了*key*与*score*之间的映射关系，而`zset.zskiplist`这个跳跃表则是维护了按照分值的升序方式对所有*key*的排序信息。

同时，*Redis*还定义了两个阈值，`redisServer.zset_max_ziplist_entries`以及`redisServer.zset_max_ziplist_value`，用于在创建有序集合时选择编码方式，以及在操作有序集合时执行编码转换，在默认的情况下，*Redis*会选择压缩链表作为有序集合的编码方式。在这两个阈值中`redisServer.zset_max_ziplist_entries`表示当使用压缩链表编码方式的有序集合最多可以存储元素的个数，当压缩链表中的元素个数超过这个阈值，那么这个有序集合会被转换成跳跃表的编码方式，同时这个阈值可以通过配置文件在启动时进行调整，如果被设置为0，那么可以认为当前的*Redis*永远都会使用跳跃表作为有序集合的编码方式；`redisServer.zset_max_ziplist_value`则是表示使用压缩链表编码时，元素最大的长度，当向一个有序集合中插入的*key*的长度大于这个阈值的时候，便会采用跳跃表的方式进行编码。

## 基于压缩链表的底层操作

对于使用压缩链表编码方式的有序集合，用户数据都是按照*key-score*的方式依次存储在压缩链表之中，存储在压缩链表中的*key-score*数据是按照*score*分值从小到大进行存储的。在向压缩链表之中插入一个*key-socre*的时候，需要先从压缩链表的头部向后遍历，找到一个合适的位置，然后在调用两次压缩链表的插入接口，将*key-score*分别插入到表中。

*Redis*在*src/t_zset.c*源文件之中，在压缩链表基础操作的基础上，又进行了一层封装，用于实现有序集合的一些底层操作，这里仅仅简介相关函数的用途，不会对其具体实现进行讲解，相关的内容可以参考跳跃表的文章。

### 数据的获取与比较

```c
double zzlGetScore(unsigned char *sptr);
sds ziplistGetObject(unsigned char *sptr);
int zzlCompareElements(unsigned char *eptr, unsigned char *cstr, unsigned int clen);
unsigned int zzlLength(unsigned char *zl);
```

上述者四个函数中：

1. `zzlLength`这个函数用于获取压缩链表中存储的*key-score*数据的个数。
2. `zzlGetScore`，用于从给定的数据指针之中返回其所表示的分值*score*。
3. `ziplistGetObject`，用于从给定的数据指针中返回其所表示的数据*key*。
4. `zzlCompareElements`，用于比较`eptr`所表示的*key*与`cstr`和`clen`所表示的数据。

### 数据的遍历

```c
void zzlNext(unsigned char *zl, unsigned char **eptr, unsigned char **sptr);
void zzlPrev(unsigned char *zl, unsigned char **eptr, unsigned char **sptr);
```

上述两个函数，都是给定一个压缩链表指针，以及对应的*key-score*的指针，分别会返回前一个*key-score*或者后一个`key-score`的指针

### 数据的范围操作

```c
int zzlIsInRange(unsigned char *zl, zrangespec *range);
unsigned char *zzlFirstInRange(unsigned char *zl, zrangespec *range);
unsigned char *zzlLastInRange(unsigned char *zl, zrangespec *range);
int zzlLexValueGteMin(unsigned char *p, zlexrangespec *spec);
int zzlLexValueLteMax(unsigned char *p, zlexrangespec *spec);
int zzlIsInLexRange(unsigned char *zl, zlexrangespec *range);
unsigned char *zzlFirstInLexRange(unsigned char *zl, zlexrangespec *range);
unsigned char *zzlLastInLexRange(unsigned char *zl, zlexrangespec *range);
```

上面的几个函数可以分成两类，分配是用于处理浮点数范围区间`zrangespec`以及字典范围区间`zlexrangespec`，以浮点树范围区间操作为例：

1. `zzlIsInRange`，这个函数用于判断这个一个有序集合底层的压缩链表是否有一部分落在给定的范围`range`内。
2. `zzlFirstInRange`，这个函数用于获取压缩链表之中落在`range`范围内的第一个元素。
3. `zzlLastInRange`，这个函数用于获取压缩链表之中落在`range`范围内的最后一个元素。

### 数据的插入、查找

```c
unsigned char *zzlFind(unsigned char *zl, sds ele, double *score);

unsigned char *zzlInsertAt(unsigned char *zl, unsigned char *eptr, sds ele, double score);
unsigned char *zzlInsert(unsigned char *zl, sds ele, double score);
```

在这几个函数之中：

1. `zzlFind`这个函数使用顺序查找的方式，从压缩链表之中查找`ele`以及`score`对应的*key-score*，如果找到，那么会返回指向*key*数据的指针，否则返回`NULL`指针。
2. `zzlInsertAt`用于将一个由`ele`和`score`构成的*key-score*插入到`eptr`所指向的元素后面，需要注意的是，这个函数是一个比较底层的函数，它在执行的时候，不会对数据的合法性进行校验，需要调用者在调用前保证数据的合法性。
3. `zzlInsert`用于向压缩链表中插入一个由`ele`和`score`构成的*key-score*，这个函数会在插入前先搜索到合适的插入位置，然后在调用`zzlInsertAt`进行数据插入，因此这个函数会保证插入后的压缩链表的顺序。

### 数据的批量删除

```c
unsigned char *zzlDelete(unsigned char *zl, unsigned char *eptr);
unsigned char *zzlDeleteRangeByScore(unsigned char *zl, zrangespec *range, unsigned long *deleted);
unsigned char *zzlDeleteRangeByLex(unsigned char *zl, zlexrangespec *range, unsigned long *deleted);
unsigned char *zzlDeleteRangeByRank(unsigned char *zl, unsigned int start, unsigned int end, unsigned long *deleted);
```

1. `zzlDelete`这个函数是删除操作的基础，用于从压缩链表中删除由`eptr`所指向的*key-score*，并返回删除后的压缩链表指针。
2. `zzlDeleteRangeByScore`、`zzlDeleteRangeByLex`、`zzlDeleteRangeByRank`这三个函数都是用于执行批量删除的函数，分别对应按照分值范围、字典范围以及下标范围进行删除，函数会返回删除后的压缩链表指针，同时通过参数`deleted`来返回给调用者被删除元素的个数。

## 有序集合对象的底层操作

首先*Redis*定义了一个获取有序集合对象大小的接口：

```c
unsigned long zsetLength(const robj *zobj);
```

这个接口会更根据底层编码类型的不同，分别调用`zzlLength`或者直接访问`zskiplist.length`字段来获取有序集合对象的大小。

接下来，我们来看一下*Redis*中有序集合的编码转换函数：

```c
void zsetConvert(robj *zobj, int encoding);
void zsetConvertToZiplistIfNeeded(robj *zobj, size_t maxelelen);
```

在这里，有序集合与其他*Redis*的对象类型不同的地方在于，其他的具有多种底层编码类型的类型转换都是单一方向的；而有序集合的编码转换则是双向的，也就是说可以实现从压缩链表实现方式转换到跳跃表的编码方式，同时还可以进行反向的转换。这样当有序集合中的元素个数以及单一元素大小已经处于系统的阈值之内，那么我们便可以通过调用`zsetConvertToZiplistIfNeeded`这个函数将一个跳跃表数据转化为压缩链表数据实现。

在了解了有序集合的编码转换函数之后，我们可以看一下如何在有序集合之中获取给定元素对应的分值：

```c
int zsetScore(robj *zobj, sds member, double *score);
```

这个`zsetScore`函数用户获取给定`member`数据对应的分值，通过参数`score`来返回这个分值结果：

1. 对于使用`OBJ_ENCODING_SKIPLIST`编码的有序集合，可以直接从`zset`的底层哈希表中，通过`dictFind`函数，来查找`member`对应的分值。
2. 对于使用`OBJ_ENCODING_ZIPLIST`编码的有序集合，则调用上面的`zzlFind`函数从底层的压缩链表搜索数据。

接下来，我们看一下用于向有序集合之中插入与更新数据的函数接口：

```c
int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore);
```

这个函数用于向有序集合之中插入一个新的元素，或者更新一个已有元素的分值。根绝参数`flags`传入的数值不同，这个函数会执行不同的策略：

1. `ZADD_INCR`，在当前元素分值的基础上加上给定的分值，用以更新元素分值；如果对应元素的*key*不存在，那么就假定其分值为0。
2. `ZADD_NX`，拥有这个标记，那么只会在给定的*key*元素不存在的时候，才会执行操作。
3. `ZADD_XX`，拥有这个标记，那么指挥在给定的*key*元素存在的时候，才会执行操作。

同时，操作的结果标记也会通过`flags`这个参数返回，而能够返回的标记有：

1. `ZADD_NAN`，这是一个失败的结果，由于分值不是数值所导致。
2. `ZADD_ADDED`，元素已经被添加。
3. `ZADD_UPDATED`，元素已经被更新。
4. `ZADD_NOP`，由于带着`ZADD_NX`或者`ZADD_XX`标记而导致操作失败。

最终，如果函数执行成功，那么会返回1，同时通过`flags`返回标记；如果函数执行发生错误，那么会返回0。在向采用`OBJ_ENCODING_ZIPLIST`编码的有序集合中插入新元素的时候，如果有序集合中元素的个数达到阈值，或者单一元素的长度达到阈值，那么会调用`zsetConvert`函数，将其转换为`OBJ_ENCODING_SKIPLIST`编码格式。

在了解向有序集合之中插入元素的函数之后，我们可以通过下面这个函数来实现从有序集合之中删除一个给定的元素：

```c
int zsetDel(robj *zobj, sds ele);
```

这个函数在执行成功时，会返回1，当待删除的元素不存在于有序集合中的时候，会返回0。在删除元素的同时，会检查删除

最后便是如何获取一个给定的元素在有序集合之中排序：

```c
long zsetRank(robj *zobj, sds ele, int reverse);
```

这个函数可以根据`reserve`参数的不同，返回一个从0开始的正向排序值，或者一个反向排序值。针对使用`OBJ_ENCODING_ZIPLIST`编码的有序集合，会遍历整个底层的压缩链表，通过`ziplistCompare`函数，来进行查找；对于使用`OBJ_ENCODING_SKIPLIST`的有序集合，则会先从`zset`中的哈希便利，查找对应的分值，在使用分值在跳跃表中通过`zslGetRank`来查找对应的顺序。



## 有序集合对象命令

### ZADD与ZINCRBY命令

*Redis*提供了两个命令用于向一个有序集合之中插入元素或者修改元素的分值：

1. **ZADD**命令，该命令的格式为：

   `ZADD key [NX|XX] [CH] [INCR] score member [score member ...]`

   这个命令用于将所有指定成员添加到给定`key`所对应的有序集合之中，添加的时候，可以添加多个成员。如果待添加的元素已经存在于有序集合之中，那么会更新对应元素的分值。该命令在`score`与`member`前支持一些参数：

   1. `NX`，如果携带该参数，则只添加新的元素成员。
   2. `XX`，如果携带该参数，则是只会更新已有的元素成员，但是这个参数与`NX`参数互斥。
   3. `INCR`，如果携带这个参数，则会对元素的分值进行递增操作，如果给定的元素不存在，那么会向有序集合中插入这个新的元素，并指定其分值为0，然后在执行分值的递增操作。
   4. `CH`，如果设置了这个参数，那么命令会返回被修改的元素的个数，而在默认的情况下，这个命令则会返回被新增的元素的数。

2. **ZINCRBY**命令，该命令的格式为：

   `ZINCRBY key increment member`

   这个命令用于向`key`对应的有序集合之中的`member`元素的分值上递增`increment`分值。这个命令与携带了`INCR`参数的**ZADD**命令相似。

由于本质上**ZINCRBY**命令是一种特殊形式的**ZADD**命令，因此*Redis*对于这两个命令的实现，使用了统一的通用接口`zaddGenericCommand`，而两个命令对应的函数都是通过调用`zaddGenericCommand`来实现的。

```c
void zaddGenericCommand(client *c, int flags);
void zaddCommand(client *c);
void zincrbyCommand(client *c);
```

在`zaddGenericCommand`这个函数中：

1. 调用`lookupKeyWrite`在内存数据库中查找给定`key`对应的有序集合对象。
2. 如果对象不存在，那么判断输入数据的大小，选择`createZsetObject`或者`createZsetZiplistObject`接口i来创建一个有序集合对象。
3. 循环调用`zsetAdd`将客户端输入的数据更新到`key`所对应的有序集合之中。

### ZREM命令

在了解有序集合对象插入新元素的命令之后，我们可以来看一个删除有序集合内元素的命令**ZREM**，这个名的格式为：

`ZREM key member [member ...]`

这个命令的含义为从`key`对应的有序集合之中删除一个或者多个`member`元素。参数`member`所对应的元素可以不存在与有序集合之中，这个命令在执行成功之后，会返回被删除元素成员的个数。

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
