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

上述这四个函数中：

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

这个接口会根据底层编码类型的不同，分别调用`zzlLength`或者直接访问`zskiplist.length`字段来获取有序集合对象的大小。

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

这个函数用于向有序集合之中插入一个新的元素，或者更新一个已有元素的分值。根据参数`flags`传入的数值不同，这个函数会执行不同的策略：

1. `ZADD_INCR`，在当前元素分值的基础上加上给定的分值，用以更新元素分值；如果对应元素的*key*不存在，那么就假定其分值为0。
2. `ZADD_NX`，拥有这个标记，那么只会在给定的*key*元素不存在的时候，才会执行操作。
3. `ZADD_XX`，拥有这个标记，那么只会在给定的*key*元素存在的时候，才会执行操作。

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

这个函数可以根据`reserve`参数的不同，返回一个从0开始的正向排序值，或者一个反向排序值。针对使用`OBJ_ENCODING_ZIPLIST`编码的有序集合，会遍历整个底层的压缩链表，通过`ziplistCompare`函数，来进行查找；对于使用`OBJ_ENCODING_SKIPLIST`的有序集合，则会先从`zset`中的哈希表中，查找对应的分值，在使用分值在跳跃表中通过`zslGetRank`来查找对应的顺序。



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
`zremCommand`这个函数是*Redis*用于实现**ZREM**命令的，在通过调用`lookupKeyWriteOrReply`函数查找到`key`对应的有序集合后，循环调用`zsetDel`将成员元素从有序集合中删除，最后如果有序集合中成员个数为0的话，那么就会将其从内存数据库之中删除。

### ZREMRANGEBYRANK、ZREMRANGEBYSCORE与ZREMRANGEBYLEX命令
在介绍过*Redis*中用于从有序集合中删除元素的命令**ZREM**之后，我们可以看一下*Redis*用于按照条件对有序集合进行批量删除的一组命令：

1. **ZREMRANGEBYRANK**，这个命令用于从`key`所指定的有序集合中删除指定排名区间内的所有元素，这个命令的格式为：

    `ZREMRANGEBYRANK key start stop`

    在这个命令之中的下标`start`以及`stop`可以是从0开始的正向排名，也可以是以-1开始的反向排名。命令执行成功后，会返回被删除的成员的个数。

2. **ZREMRANGEBYSCORE**，这个命令用于从`key`所指定的有序集合中删除指定分值区间内的所有成员，这个命令的格式为：

    `ZREMRANGEBYSCORE key min max`

    这个命令执行成功后，会返回被删除的成员的个数。

3. **ZREMRANGEBYLEX**，这个命令用于从`key`所指定的有序集合中删除名称按照字典顺序区间内的所有成员，这个命令的格式为：

    `ZREMRANGEBYLEX key min max`

    执行这条命令的前提为，有序集合之中的所有的元素的分值必须一致，因为有序集合中，是优先按照分值排序，在分值相同的基础上，在按照元素的字典顺序进行排序的。因此，如果在一个分值不同的有序集合之中指向这个命令，将会导致错误产生。

```c
void zremrangeGenericCommand(client *c, int rangetype);
void zremrangebyrankCommand(client *c);
void zremrangebyscoreCommand(client *c);
void zremrangebylexCommand(client *c);
```
*Redis*对于上述的三个命令定义了一个通过的函数接口`zremrangeGenericCommand`，同时还定义了三个类型，用于标记三种删除类型：
```c
#define ZRANGE_RANK 0
#define ZRANGE_SCORE 1
#define ZRANGE_LEX 2
```
*Redis*在实现`zremrangeGenericCommand`时，会执行下面4步的逻辑：
1. `zremrangeGenericCommand`会根据`rangetype`类型，校验客户端输入的区间参数的合法性。
2. 在*Redis*的内存数据库中查找`key`所对应的有序集合；如果是按照排名进行删除，那么会结合有序集合的长度，对排名进行检查。
3. 根据有序集合对应的编码类型以及删除类型，分别调用前面提到的有序集合批量删除接口，对数据进行删除。
4. 最终将删除的结果返回给客户端调用者。


### ZUNIONSTORE与ZINTERSTORE命令
针对有序集合，*Redis*提供了两个用于处理集合交集与并集的命令：
1. **ZUNIONSTORE**，这个命令的格式为：
    `ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]`
2. **ZINTERSTORE**，这个命令的格式为：
    `ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]`

上述两个命令的用途是计算给定的`numkeys`个有序集合交集或者并集结果，并将结果存储在`destination`对应的有序集合之中，可以选择性的给出`WEIGHTS`参数，用于设定计算分值时所需要的用到的权重因子，如果没有指定这个参数，那么默认的权重因子便是1；通过可选参数`AGGREGATE`，我们可以指定不同有序集合中的相同成员在存储到目标集合中时分值要采用何种聚合方式，默认是`SUM`，如果使用参数`MIN`或者`MAX`，目标集合中成员分值就是所有集合中最小或者最大的分值。

这里需要注意的是`destination`集合会在操作结束之后被覆盖。

```c
void zunionInterGenericCommand(client *c, robj *dstkey, int op);
void zunionstoreCommand(client *c);
void zinterstoreCommand(client *c);
```


### ZRANGE与ZREVRANGE命令
*Redis*为有序集合的区间操作提供了两个命令：
1. **ZRANGE**，这个命令的格式为：

    `ZRANGE key start stop [WITHSCORES]`

    这个命令用于返回在有序集合中的指定排序范围的所有成员。如果携带了`WITHSCORES`参数，那么这个函数会在返回成员元素之外，还会返回成员元素的分值。返回成员元素的顺序是按照分值递增的顺序来排序的。
    
2. **ZREVRANGE**，这个命令的格式为：

    `ZREVRANGE key start stop [WITHSCORES]`

    这个命令的功能与**ZRANGE**相似，区别在于所返回的元素的顺序是按照分值递减的顺序来排序的。
```c
void zrangeGenericCommand(client *c, int reverse);
void zrangeCommand(client *c);
void zrevrangeCommand(client *c);
```

函数`zrangeGenericCommand`函数是上面这两个命令的实现基础：

1. 对于使用`OBJ_ENCODING_ZIPLIST`实现有序集合，首先会调用`ziplistIndex`函数在底层压缩链表中定位到第一个元素的位置，然后循环调用 `zzlPrev`或者`zzlNext`来收集数据。
2. 对于使用`OBJ_ENCODING_SKIPLIST`实现的有序集合，那么会通过`zslGetElementByRank`函数在底层的跳跃表之中定义到起始元素的位置，最后根据跳跃表的的性质，对跳跃表进行遍历，完成数据的收集。

### ZRANGEBYSCORE与ZREVRANGEBYSCORE命令

除了基于排序的区间成员的获取命令，*Redis*还提供了两个基于分值的区间成员获取的命令**ZRANGEBYSCORE**以及**ZREVRANGEBYSCORE**命令：

1. **ZRANGEBYSCORE**命令的格式为：

   `ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]`。

2. **ZREVRANGEBYSCORE**命令的格式为：

   `ZREVRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]`

上述这两个命令都是用于从`key`所指定的有序集合返回落在`min`以及`max`区间内的成员元素，这两个命令都可以携带两个可选参数：

1. `WITHSCORES`，如果命令的调用者给出了这个参数，那么成团的分值也会一并返回。
2. `LIMIT`，这个可选参数主要是用来对返回结果的个数进行限制，其中`offset`用于表示结果的起始位置，而`count`则用来表示返回结果的个数。

```c
void genericZrangebyscoreCommand(client *c, int reverse);
void zrangebyscoreCommand(client *c);
void zrevrangebyscoreCommand(client *c);
```

函数`genericZrangebyscoreCommand`是用来实现上述命令的基础，这个函数首先会给定客户端输入的区间参数，初始化了`zrangespec`数据，根据前面介绍的四个函数：

1. `zzlLastInRange`
2. `zzlFirstInRange`
3. `zslLastInRange`
4. `zslFirstInRange`

通过上面这个四个函数，*Redis*可以使用`zrangespec`在有序集合中查找属于区间的起始节点，进而通过遍历底层数据结构，收集数据，并通过下面这两个函数来判断是否完成收集：

1. `zslValueGteMin`
2. `zslValueLteMax`

### ZRANGEBYLEX与ZREVRANGEBYLEX命令

最后*Redis*还提供了两个基于字典顺顺序获取有序集合区间成员数据的命令，**ZRANGEBYLEX**以及**ZREVRANGEBYLEX**这两个命令，这两个命令所操作的有序集合，其元素分值必须都是一致的。

1. **ZRANGEBYLEX**这个命令的格式为：

   `ZRANGEBYLEX key min max [LIMIT offset count]`

2. **ZREVRANGEBYLEX**这个命令的格式为：

   `ZREVRANGEBYLEX key min max [LIMIT offset count]`

```c
void genericZrangebylexCommand(client *c, int reverse);
void zrangebylexCommand(client *c);
void zrevrangebylexCommand(client *c);
```

**ZRANGEBYLEX**与**ZREVRANGEBYLEX**这两个命令，是使用`genericZrangebylexCommand`作为基础实现函数的，这个函数的逻辑基础与`genericZrangebyscoreCommand`类似，因此在这里不做过多的讲解。

### ZCOUNT命令

*Redis*使用**ZCOUNT**命令来获取有序集合之中落于指定分值之间的成员的个数，这个命令的格式为：

`ZCOUNT key min max`

```c
void zcountCommand(client *c);
```

`zcountCommand`这个函数会根据有序集合底层编码的不用选用不同的策略：

1. `OBJ_ENCODING_ZIPLIST`，对于只用压缩链表实现有序集合，会首先调用`zzlFirstInRange`找到区间的起始成员，在按照遍历的方式，对落在范围区间内的成员进行计数。
2. `OBJ_ENCODING_SKIPLIST`，对于采用跳跃表实现的有序集合，会调用`zslFirstInRange`以及`zslLastInRange`找到区间的第一个以及最后一个成员，然后通过`zslGetRank`来获取成团的排名值，通过两个排名值来计算区间内成员的个数。

### ZLEXCOUNT命令

对于字典区间的计数，*Redis*也提供了一个命令**ZLEXCOUNT**，用获取有序集合中指定成员之间的成员的个数，因为这是基于字典顺序的操作，因此需要有序集合之中的每个成员的分值要都相同，这个命令的格式为：

`ZLEXCOUNT key min max`

```c
void zlexcountCommand(client *c);
```

`zlexcountCommand`这个函数在实现逻辑上与`zcountCommand`相似，不同之处在调用处理`zlexrangespec`的相关函数。

### ZCARD命令

**ZCARD**命令用于获取给定的`key`所对应的成员个数，该命令的格式为：

`ZCARD key`

```c
void zcardCommand(client *c);
```

`zcardCommand`在实现**ZCARD**命令时，是通过调用`zsetLength`接口来实现的。

### ZSCORE命令

**ZSCORE**命令用于返回有序集合中某个指定成员的分值。该函数的格式为：

`ZSCORE key member`

如果`key`对应的有序集合不存在，或者`member`不存在于有序集合之中，则该命令返回`nil`值，否则成功的情况下，则会返回对应的成员的分值。

```c
void zscoreCommand(client *c);
```

`zscoreCommand`函数则是通过调用`zsetScore`函数来实现命令的功能的。

### ZRANK与ZREVRANK命令

对于有序集合的排名操作，*Redis*给出了两个命令：
1. **ZRANK**，这个命令用于获取获取元素按照分值递增顺序的排名，其格式为：

    `ZRANK key member`

    这个命令所返回的排名是以0开始的，也就是说如果函数返回0，那么说明这个成员在当前的有序集合之中是最小的；同时对于一个不存在的成员，命令会返回一个空值`nil`。

2. **ZREVRANK**，这个命令用于获取元素按照分值递减排序的排名，这个命令的格式为：
   
    `ZREVRANK key member`

    这个命令所返回的排名依然是以0开始的，如果函数返回0，那么说明这个成员在当前的有序集合之中是最大的。

```c
void zrankGenericCommand(client *c, int reverse);
void zrankCommand(client *c);
void zrevrankCommand(client *c);
```

`zrankGenericCommand`是实现上述两个命令的基础，其内部是通过调用`zsetRank`这个接口来实现获取元素成员排名操作的。

### ZPOPMIN与ZPOPMAX命令

*Redis*对于有序集合给出了两个用于弹出数据的命令：

1. **ZPOPMIN**，这个命令从`key`所对应的有序集合之中弹出最小的若干个成员。这个命令的格式为：

   `ZPOPMIN key [count]`

   该命令如果不给出`count`，则默认为1，也就是弹出有序集合中最小的那个成员。该命令执行成功后，会返回弹出的成员。

2. **ZPOPMAX**，这个命令从`key`所对应的有序集合之中弹出最大的若干个成员。这个命令的格式为：

   `ZPOPMAX key [count]`

   这个命令与**ZPOPMIN**类似，成功之后都是会返回弹出的成员。

上述的两个命令中的`count`参数可以大于有序集合的元素个数，这种情况下，命令不会返回错误，只会将有序集合之中的全部元素弹出。

```c
void genericZpopCommand(client *c, robj **keyv, int keyc, int where, int emitkey, robj *countarg);
void zpopminCommand(client *c);
void zpopmaxCommand(client *c);
```

用于实现上述两个命令的函数`genericZpopCommand`，其内部逻辑是相对简单的，因为无论是使用压缩链表还是跳跃表实现的有序集合，其底层数据结构中成员数据的存储都是有序的，因此弹出操作只需要在有序集合的头部或者尾部按照个数弹出元素就可以。

### BZPOPMIN与BZPOPMAX命令

对于上面的**ZPOPMIN**以及**ZPOPMAX**命令，*Redis*还提供了带有阻塞功能的版本：

1. **BZPOPMIN**命令，其命令格式为：`BZPOPMIN key [key ...] timeout`
2. **BZPOPMAX**命令，其命令格式为：`BZPOPMAX key [key ...] timeout`

这两个命令都可以处理多个有序集合对象，在参数中的所有有序集合都是空的情况下，会阻塞客户端的连接，`timeout`会指定客户端被阻塞的最大秒数，0则便是永久阻塞。当所有的集合并非都是为空的情况下，会按照命令`key`的输入顺序从第一个非空的集合之中弹出最大或者最小的成员。

```c
void blockingGenericZpopCommand(client *c, int where);
void bzpopminCommand(client *c);
void bzpopmaxCommand(client *c);
```
用于实现上述两个命令的函数`blockingGenericZpopCommand`在给定的有序集合非空时，会直接调用`genericZpopCommand`来将结果返回给客户端，而当所有的有序集合都为空的时候，则会用`blockForKeys`函数阻塞客户端的请求，直到其中一个有序集合非空或者超时为止。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
