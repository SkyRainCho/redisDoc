# Redis中哈希表的实现
在*Redis*中，哈希表不但是我们可以使用的几种基础数据结构之一，同时还是整个*Redis*数据存储结构的核心。
究其根本，*Redis*是一种基于*Key-Value*的内存数据库，所有的用户数据，*Redis*在底层都是使用哈希表进行保存的。
可以说，了解*Redis*中哈希表的实现方式，是了解*Redis*存储的一个关键。

## Redis哈希表概述
基于链表的哈希表是实现哈希表的一个主流方式，除了*Redis*中的哈希表实现，*g++*中的`unordered_map`也是使用基于链表的哈希表实现的，
但是两者之间还是有一些细微的差别，这在后续会进行介绍。

在*src/dict.h*头文件之中，首先定义了哈希表中的链表基本节点数据结构。
```c
typedef struct dictEntry
{
    void *key;                //用于存储Key
    union
    {
        void *val;    //可以用来保存一段具体的内存二进制数据
        uint64_t u64; //可以用来存储一个64位无符号整形数据
        int64_t s64;  //可以用来存储一个64位有符号整形数据
        double d;     //可以用来存储一个双精度浮点数据
    } v;                      //使用union来保存value数据
    struct dictEntry *next;   //由于使用基于拉链的哈希表实现，next用于指向桶中下一个key-value对。
} dictEntry;
```
在这个*key-value*的数据结构之中，*value*是使用一个联合体`union`来表示的，通过这个联合体，*Redis*既可以使用一个64位整数作为*value*，
也可以使用一个双精度浮点数作为*value*，同时还可以使用一个任何一段内存作为*value*，而*value*之中也保存这段内存的指针。

基于这个哈希表的基本节点数据结构，*Redis*定义了自己的哈希表数据结构：
```c
typedef struct dictht {
    dictEntry **table;        //table是一个数组结构，其中的每个元素都是一个dictEntry指针
    unsigned long size;       //table中桶的数量
    unsigned long sizemask;   //table中桶数量的掩码
    unsigned long used;       //该哈希表之中，已经保存的*key-value*的数量
} dictht;
```
根据`dictht`给出的定义，我们可以知道这个哈希表在内存中的结构可以如下图所示：

![dictht内存分布](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/dictht%E5%86%85%E5%AD%98%E5%88%86%E5%B8%83.PNG)

在`dictht`中这个`sizemask`数据主要视为实现快速取余的用途，这个掩码被设定为`size-1`的大小。
这里体现了*Redis*哈希表与 *g++* 中`unordered_map`的一个不同之处，*g++* 总是会选取一个素数作为桶的数量，
而在*Redis*之中，桶的数量一定是2的*n*次方个，那么当`dictht.size`为16的时候，`dictht.sizemask`对应而二进制形式变为`1111`,
这样对于一个给定的哈希值`h`使用`h & sizemask`可以获得哈希值对`size`的取余操作结果。
根据余数，可以决定这个`key-value`数据落在哪个哈希表的哪个桶中。

同时，*src/dict.h*头文件中，定义了一个结构体，存储基础哈希操作的函数指针，这里体现了*Redis*代码实现中的函数式编程的思想，
用户可以对指定的函数指针赋值自己定义的函数接口。
```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);            //通给可定key，计算对应的哈希值
    void *(*keyDup)(void *privdata, const void *key);     //用于复制key的函数指针
    void *(*valDup)(void *privdata, const void *obj);     //用于复制value的函数指针
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); //两个key的比较函数
    void (*keyDestructor)(void *privdata, void *key);     //用于处理key的释放
    void (*valDestructor)(void *privdata, void *obj);     //用于处理val的释放
} dictType;
```
*Redis*为不同的哈希表给出了不同的`dictType`，例如在*src/server.c*文件中，
```c
/* Db->dict, key是sds动态字符串, vals则是Redis对象类型 */
dictType dbDictType = {
    dictSdsHash,                /* hash function */
    NULL,                       /* key dup */
    NULL,                       /* val dup */
    dictSdsKeyCompare,          /* key compare */
    dictSdsDestructor,          /* key destructor */
    dictObjectDestructor        /* val destructor */
};
```
*Redis*会使用上述这组`dictType`来描述*Redis*数据库中的键空间，`server.db[j].dict = dictCreate(&dbDictType,NULL);`。

基于`dictht`以及`dictType`这两个数据结构，*Redis*定义最终的哈希表数据结构`dict`:
```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx;         //重哈希索引，如果没有在进行中的重哈希，那么这个rehashidx的值为-1
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

![dict内存分布](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/dict%E5%86%85%E5%AD%98%E5%88%86%E5%B8%83.PNG)

上面这张图，便是描述了*Redis*的哈希表在内存中的分布结构，我们可以注意到其中的一个特别设计之处，
也就是在`dict`结构中会有两个底层的哈希表数据结构`dict.ht[2]`，
这与*g++*中的`unordered_map`的设计有着显著的不同，
```cpp
class _Hashtable
{
    ...
    __bucket_type*    _M_buckets;
    size_type         _M_bucket_count;
    size_type         _M_element_count;
    ...
};
template<class _Key, class _Tp,
        class _Hash = hash<_Key>,
        class _Pred = std::equal_to<_Key>,
        class _Alloc = std::allocator<std::pair<const _Key, _Tp> > >
class unordered_map
{
    ...
    _Hashtable _M_h;
    ...
};
```
从*g++*的`unordered_map`源代码可以知道，它在内部只维护了一个底层哈希表数据结构。

*为什么Redis的哈希表要有两个呢？？？*

之所以*Redis*在`dict`中使用两个*hashtable*，其主要用意是方便在进行*ReHash*操作时，进行数据转移。
*Redis*在执行重哈希操作时，不会一次性将所有的数据进行重哈希，而是采用一种增量的方式，
逐步地将数据转移到新的桶中。而这又是其与`unordered_map`的不同之处，`unordered_map`在进行重哈希的时候，
会一次性地将表中的所有元素移动到新的桶中，而不是增量进行的。究其原因，`unordered_map`只是*C++*中存储数据的
手段之一，其有特定的应用场景，因此不需要增量地进行重哈希。而在*Redis*中，虽然官方给出了多种基础数据类型，
但是其在底层进行检索的时候，都是以哈希表进行存储的，同时*Redis*定义为是一种数据库，那么其在哈希表中所存储的数据
的量级要远远大于通用的*C++*程序，如果在哈希表中有大量的*key-value*数据的话，对所有数据进行重哈希操作，
会导致系统阻塞在重哈希操作中无法退出，而*Redis*本身对于核心数据的操作又是单线程的模式，
这将导致*Redis*无法对外提供服务。为解决这个问题，*Redis*在`dict`中给出了保存了两个哈希表，在进行重哈希操作时，
*Redis*会将第二个哈希表进行扩容或者缩容，然后定期将第一个哈希表中的数据重哈希到第二个哈希表中。
而这时，保存在第一个哈希表中没有来得及进行重哈希的数据，对于客户端用户来说，依然是可用的。
当第一个哈希表中全部数据重哈希结束后，*Redis*会把数据从第二个哈希表中转移至第一个哈希表中，结束重哈希操作。

![重哈希dict内存分布](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E9%87%8D%E5%93%88%E5%B8%8C%E7%9A%84dict%E5%86%85%E5%AD%98%E5%88%86%E5%B8%83.PNG)

上图便是*Redis*在对哈希表执行重哈希操作过程中，其在内存中的分布情况。在进行重哈希的过程中`dict.rehashidx`字段
标记了第一个哈希表中下一次需要被重哈希的桶的索引，当重哈希结束后，这个字段会被设置为`-1`。

而字段`dict.iterators`则是关联在这个哈希表上的安全迭代器的数量，关于安全迭代器的内容，会在后续进行介绍。



### Redis哈希表的基础底层操作

在*src/dict.h*头文件之中，*Redis*定义了一组用于完成基本底层操作的宏：
| 宏定义 | 具体含义 |
|-------|---------|
|`#define dictFreeVal(d, entry)`|使用`dict`中`type`的`valDestructor`来释放`entry`节点的`v.val`|
|`#define dictSetVal(d, entry, _val_)`|为`entry`中的`v.val`进行赋值|
|`#define dictSetSignedIntegerVal(entry, _val_)`|为`entry`中的`v.u64`进行赋值|
|`#define dictSetUnsignedIntegerVal(entry, _val_)`|为`entry`中的`v.s64`进行赋值|
|`#define dictSetDoubleVal(entry, _val_)`|为`entry`中的`v.d`进行赋值|
|`#define dictFreeKey(d, entry)`|使用`dict`中`type`的`valDestructor`来释放`entry`节点的`key`|
|`#define dictSetKey(d, entry, _key_)`|为`entry`中的`key`进行赋值|
|`#define dictCompareKeys(d, key1, key2)`|对`key1`和`key2`进行比较|
|`#define dictHashKey(d, key)`|为`key`计算其对应的哈希值|
|`#define dictSlots(d)`|获取`dict`中桶的个数，由两个哈希表中的`size`数据的加和组成|
|`#define dictSize(d)`|获取`dict`中存储元素的个数，由两个哈希表中`used`数据的加和组成|
|`#define dictIsRehashing(d)`|判断`dict`是否处于重哈希过程中|

## Redis哈希表的构造释放与增删改查

### Redis用于初始化创建与释放清理哈希表的接口
```c
static void _dictReset(dictht *ht);
int _dictInit(dict *d, dictType *type, void *privDataPtr);
dict *dictCreate(dictType *type, void *privDataPtr);
```
其中：
1. `_dictReset`函数对一个给定的`dictht`进行重置初始化。
2. `_dictInit`函数，使用一个`dictType`以及一个私有数据指针`privDataPtr`来初始化一个`dict`数据；
并调用`_dictReset`函数来初始化`dict`的两个底层哈希表。
3. `dictCreate`函数，会调用`zmalloc`函数来分配一个`dict`，并调用`_dictInit`对数据进行初始化。

```c
void dictRelease(dict *d);
int _dictClear(dict *d, dictht *ht, void(callback)(void *));
void dictEmpty(dict* d, void(callbacl)(void*));
```
上述三个函数用于清空与释放一个给定的哈希表，其中`_dictClear`函数是整个释放操作的基础，
该函数用于释放哈希表中的某一个给定的底层哈希表，逐个释放每个桶中的每个元素；
`dictRelease`函数与`dictEmpty`都会调用`_dictClear`来清空释放哈希表中的元素，
二者的区别是，`dictRelease`在释放哈希表中元素后会释放整个哈希表，
而`dictEmpty`则不会释放哈希表`dict`这个数据结构。
在`_dictClear`函数释放数据的时候，这里有一个特殊的操作：
```c
int _dictClear(dict *d, dictht *ht, void(callback)(void *))
{
    ...
    for (i = 0; i < ht->size && ht->used > 0; i++)
    {
        ...
        if (callback && (i & 65535) == 0) callback(d->privdata);
        ...
        //free key-value pair in the bucket
    }
    ...
}
```
我们可以看到，如果在清空哈希表的时候，传入了`callback`回调函数作为参数的话，
那么*Redis*会在清空哈希表时，每清空`65536`个桶时，会调用`callback`函数，处理响应的逻辑。
这段代码是*Redis*作者在2013年11月11日添加的，作者自己给出的提交信息是：
> dict.c: added optional callback to dictEmpty().
> 
> Redis hash table implementation has many non-blocking fearures like
> incremental rehashing, however while deleting a large hash table there
> was no way to have a callback called to do some incremental work.
> 
> This commit adds this support, as an optiona callback argument to
> dictEmtpy() that  is currently called at a fixed interval (one time every
> 65k deletions)

按照作者的说法，在*Redis*的哈希表操作很多的是以非阻塞的方式进行的，但是释放与清空的操作却是
以阻塞的方式进行的，当删除一个很大的哈希表时，缺少一种增量逐步执行某些操作的机制。
因此，作者在这次提交中引入了这个机制，可以在删除哈希表时，以一个固定的间隔来执行回调函数。

一个简单的例子，在使用主从结构的*Redis*集群时，*slave*节点在异步读取从*master*节点
收到的*SYNC*数据时，会涉及到删除自己的旧数据以加载新数据，*slave*节点阻塞在从哈希表中删除旧数据，
而无法响应其他请求时，*master*节点可能会认为该节点超时，为了防止这种情况发生，我们可以在清空哈希表时，
传入如下的回调函数，定期向*master*节点发送一个*newLine*数据，确保*master*不会误认为改节点超时：
```c
void replicationEmptyDbCallback(void *privdata)
{
    replicationSendNewlineToMaster();
}
```

### Redis用于对哈希表进行重哈希的接口
因为*Redis*是使用链表的形式来实现哈希表的，哈希值冲突的键值对会以链表的形式存储在一个桶中，
如果桶中平均的链表长度过长的话，会严重降低搜索的效率；同时如果键值对过少，而哈希表中的桶数过多，
则会浪费内存的空间。因此需要在哈希表的装载因子过大或者过小的时候，对哈希表进行重哈希，扩容或者缩容，
以达到提高搜索速度，或者优化内存存储的目的。
```c
void dictEnableResize(void);
void dictDisableResize(void);
```
上述的两个函数用于设置静态全局变量`dict_can_resize`，对于这个全局变量的描述：
1. `dict_can_resize`为0，不会对哈希表进行缩容；当哈希表的装载因子大于强制重哈希阈值`dict_force_resize_ratio`时，
仍然会进行重哈希扩容操作。
2. `dict_can_resize`为1，会对哈希表进行缩容；如果哈希表的装载因子大于1，就会对哈希表执行重哈希扩容操作。

#### Reis用于调整哈希表大小的接口
```c
static unsigned long _dictNextPower(unsigned long size);
static int _dictExpandIfNeeded(dict *ht);
int dictExpand(dict *d, unsigned long size);
int dictResize(dict *d);
```
`__dictNextPower`用于通过给定的`size`来计算需要扩容的大小，从`DICT_HT_INITIAL_SIZE`开始，
每次翻倍，直到找到第一个大于等于`size`的数值，即为扩容后的大小。

在我们对一个`dict`数据添加一个新的*key-value*时，都会尝试调用`_dictExpandIfNeeded`来尝试，
是否需要对于`dict`进行扩容，其判断是否需要扩容，则依照下面的两种情况进行判断:
1. 如果这个`dict`中的哈希表是空的情况，那么会将哈希表扩容到`DICT_HT_INITIAL_SIZE`:
```c
    if (d->ht[0].size == 0) 
        return dictExpand(d, DICT_HT_INITIAL_SIZE);
```
2. 在元素个数以及桶数量的比例达到*1:1*的情况下，如果设置了`dict_can_resize`，
或者装载因子超过了`dict_force_resize_ratio`。
```c
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize || d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
```

而函数`dictExpand`则会根据给定的`size`对这个`dict`的容量进行调整，*Redis*会根据`size`计算出真正的调整容量:
```c
    unsigned long realsize = _dictNextPower(size);
```
如果`dict`中的第一个哈希表为空，表明这是创建一个新的哈希表的操作，那么直接将扩容出的哈希表赋值给第一个哈希表：
```c
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }
```
否则的话，意味着需要对这个`dict`进行重哈希操作，将扩容出的哈希表赋值给`dict`的第二个哈希表，同时设置`rehashidx`。
```c
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
```
而函数`dictResize`会将这个`dict`调整至最小的可以容纳所有元素的大小。
在*src/server.c*源文件中定义了接口用于判断一个哈希表是否需要进行缩容
```c
int htNeedsResize(dict *dict)
{
    long long size, used;
    size = dictSlots(dict)
    used = dictSize(dict)

    return (size > DICT_HT_INITIAL_SIZEE && (used*100/size < HASHTABLE_MIN_FILL));
}
```

上述这几个接口不会真的执行移动数据的操作，只是会对哈希表的尺寸进行调整，同时将哈希表设置成进入重哈希的状态，
真正的数据迁移是在后续重哈希接口中进行的。

#### Redis用于进行重哈希的操作接口
```c
int dictRehash(dict *d, int n);
int dictRehashMilliseconds(dict *d, int ms);
static void _dictRehashStep(dict *d);
```
其中`dictRehash`函数，是重哈希操作的基础，其含义是给定一个`dict`数据以及需要重哈希的元素的个数`n`,
由于哈希表中会存在空的桶，如果这一次重哈希操作遍历到`10 * n`个空的桶时，终止这次重哈希操作。
当需要进行重哈希操作时，其基础逻辑是，`rehashidx`表示当前操作的第一个哈希表中桶的位置索引，
把第一个哈希表中的某个元素移动到第二个哈希表中：
```c
    de = d->ht[0].table[d->rehashidx];
    /* Move all the keys in this bucket from the old to the new hash HT */
    while(de) {
        uint64_t h;

        nextde = de->next;
        /* Get the index in the new hash table */
        h = dictHashKey(d, de->key) & d->ht[1].sizemask;
        de->next = d->ht[1].table[h];
        d->ht[1].table[h] = de;
        d->ht[0].used--;
        d->ht[1].used++;
        de = nextde;
    }
    d->ht[0].table[d->rehashidx] = NULL;
    d->rehashidx++;
```
当第一个哈希表中所有的元素都已经转移到第二个哈希表时，会把第二个哈希表转移给第一个哈希表，
同时重置`rehashidx`，并返回0表示重哈希已经结束。当还有元素没有进行重哈希时，会返回1。

而`dictRehashMilliseconds`会在1毫秒的时间片内，执行若干次`dictRehash`操作，直到所有数据都已经重哈希，
或者执行时间超过1毫秒的时间片。

在*Redis*的服务器中，系统会以1毫秒一次的心跳来执行`databaseCron`接口，用于增量处理*Redis*数据库操作：
```c
void databaseCron(void)
{
    ...
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
    {
        ...
        /* Resize */
        for (j = 0; j < dbs_per_call; j++)
        {
            tryResizeHashTables(resize_db % server.dbnum);
            resize_db ++;
        }

        /* Rehash */
        if (server.activerehashing)
        {
            for (j = 0; j < dbs_per_call; j++)
            {
                int work_done = incrementallyRehash(rehash_db);
                if (work_done)
                {
                    break;
                }
            }
        }
        ...
    }
    ...
}
```
上述的代码便是*Redis*渐进式重哈希的核心，函数`tryResizeHashTables`会调用`dictResize`对哈希表进行缩容，
函数`incrementallyRehash`会调用`dictRehashMilliseconds`对哈希表进行重哈希。

`_dictRehashStep`函数是一个通常在内部调用的函数，如果在没有安全迭代器的情况下，这个函数会对其中1个元素执行重哈希:
```c
    if (d->iterators == 0) 
        dictRehash(d,1);
```
设计这个函数的意义在于，在执行添加、删除、查找操作时被动执行一次单步重哈希，用来提高系统重哈希的速度。
对于涉及安全迭代器的内容，在后续的内容之中会进行介绍。

### Redis用于向哈希表中添加删除搜索元素的操作接口
```c
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing);
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing);
int dictAdd(dict *d, void *key, void *val);
int dictReplace(dict *d, void *key, void *val);
dictEntry *dictFind(dict *d, const void *key);
dictEntry *dictAddOrFind(dict *d, void *key);
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree);
int dictDelete(dict *ht, const void *key);
dictEntry *dictUnlink(dict *ht, const void *key);
void dictFreeUnlinkedEntry(dict *d, dictEntry *he);
```

#### Redis哈希表的添加与查找

其中`_dictKeyIndex`函数是一个基础的底层操作，给定一个`key`以及其对应的`hash`值，
获取这个返回其所应该放置的桶的索引，如果这个`key`已经存在，那么返回-1，
同时在`key`已经存在的情况下，如果传入了`existing`，那么这个对应的已存在`dictEntry`会被赋值给`existing`。
有一点需要注意的是，如果`dict`没有处在重哈希状态中，那么只会在第一个哈希表中进行查找，否则会在两个哈希表中都进行查找。
```c
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
```

而`dictAddRaw`同样是一个底层的操作，给定一个`key`，向`dict`中插入一个对应这个`key`的`dictEntry`,
如果插入成功，会返回这个插入的`dictEntry`的指针，否则会返回一个`NULL`，在这种情况下，
如果我们传入了`existing`，那么这个`key`对应的已经存在的`dictEntry`会赋值给`existing`。
需要注意的是：
1. 这个接口不会设置`dictEntry`的*val*，需要调用者在获取到`dictEntry`指针后，自己处理。
2. 如果`dict`处于重哈希状态，那么这个新*key-value*会被插入第二个哈希表中，否则加入到第一个哈希表中，如代码所示：
```c
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;
```

函数`dictAdd`执行了最简单插入操作，如果成功，就调用`dictSetVal`设置*val*，否则的话，返回`DICT_ERR`

而函数`dictReplace`执行的是一个插入或者覆盖的操作，如果是插入新元素，那么返回1，如果覆盖已有元素，则返回0。

函数`dictAddOrFind`可以让我们给定一个`key`来查找这个指定`key`对应的`dictEntry`指针，如果`key`不存在`dict`中，
那么会向其中插入一个`dictEntry`，并返回。
而函数`dictFind`，会根据给定的`key`来查找对应的`dictEntry`，如果查找不到，会返回`NULL`。

#### Redis哈希表的删除
其中函数`dictGenericDelete`是哈希表删除操作的基础，这个函数会接受一个参数`nofree`，如果该参数被设置为0，
那么会将查找到的节点直接释放；如果该参数为1，那么会将查找到节点从哈希表中分离出来，返回给调用者自己操作。
```c
int dictDelete(dict *ht, const void *key)
{
    return dictGenericDelete(ht, key, 0) ? DICT_OK : DICT_ERR;
}
```
`dictDelete`函数便是以`nofree = 0`来调用`dictGenericDelete`函数直接删除释放节点。

```c
dictEntry *dictUnlink(dict *ht, const void *key)
{
    return dictGenericDelete(ht, key, 1);
}
```
这个函数会返回从哈希表中分离出的节点。那么设计这个函数的意义在于，如果我们希望从哈希表中找到某个节点，对该节点执行某些操作后，释放该节点的话，
如果使用先查找在删除的方式来进行的话，需要执行两次查找
```c
entry = dictFind(...);
//do something
dictDelete(dictionary, entry);
```
如果应用`dictUnlink`接口，那么只需要执行一次查找就可以。
```c
entry = dictUnlink(...);
//do something
dictFreeUnlinkedEntry(entry);
```
最后使用`dictFreeUnlinkedEntry`接口，可以实现分离节点的释放。

## Redis哈希表的迭代与遍历
*Redis*中对于哈希表的迭代与遍历，是整个哈希表实现中最为巧妙，同时也是最复杂难以理解的一个部分，
相当一部分写法，很难揣测作者的意图。只能暂且根据*Redis*的整体功能来进行推测，当然笔者的推测无法保证完全的准确，
如果随着对代码理解的深入，笔者会随时更新对这部分内容的理解。

### Redis哈希表的随机采样
在*Redis*中，给出了两个接口函数用于对哈希表进行随机采样，不过作者自己也认为，这两个接口在采样数据分布的随机性上并不是特别的完善。
```c
dictEntry* dictGetRandomKey(dict *d);
unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count);
```
`dictGetRandomKey`这个接口从名字上理解起来很简单，就是从哈希表中，随机的返回一个*key-value*元素，
其具体的算法为：
1. 从所有的桶中，随机选择一个非空的桶。
2. 从这个选定的桶中的链表里，随机返回一个元素。

可以看出，每个元素并不是等概率地被采样出来的，如果某个桶中的链表长度比较短的话，那么该桶中的元素，相比较与其他链表较长的桶的元素，
有更高的概率被`dictGetRandomKey`这个接口所返回。

`dictGetSomeKeys`这个接口用于从哈希表中，随机地获取给定`count`数量的元素，当然作者也给出注释：
1. 该接口无法保证，一定会获取到`count`个采样结果，最终的采样数量通过函数进行返回，这一点很好理解，当哈希表中的元素个数小于`count`时，
该接口返回足够的采样数据。
2. 该接口无法保证，所返回的采样结果一定是没有重复的。

```c
    while(stored < count && maxsteps--) {
        for (j = 0; j < tables; j++) {
            if (tables == 2 && j == 0 && i < (unsigned long) d->rehashidx) {
                if (i >= d->ht[1].size)
                    i = d->rehashidx;
                else
                    continue;
            }
            if (i >= d->ht[j].size) continue; /* Out of range for this table. */
            dictEntry *he = d->ht[j].table[i];

            if (he == NULL) {
                emptylen++;
                if (emptylen >= 5 && emptylen > count) {
                    i = random() & maxsizemask;
                    emptylen = 0;
                }
            } else {
                emptylen = 0;
                while (he) {
                    *des = he;
                    des++;
                    he = he->next;
                    stored++;
                    if (stored == count) return stored;
                }
            }
        }
        i = (i+1) & maxsizemask;
    }
```

其基本逻辑为：
1. 从两个哈希表中选取最大的`maxsizemask`，以此为依据生成随机索引。
2. 根据该随机索引，选择对应的桶，将该桶中的元素尽可能的加入到返回结果中，如果达到`count`的数量，那么直接结束调用。
3. 如果没有达到所需的`count`，那么选择下一个桶进行采样，`i = (i+1) & maxsizemask;`。
4. 如果对应的桶为空，则选择下一个桶进行采样；如果连续五个桶都为空，则重新生成随机索引，这一步有可能会导致数据被重复采样。
5. 如果生成的随机索引大于当前哈希表的`size`，则用该索引查看另外一个哈希表，
而第一步的`maxsizemask`则可以保证随机索引在两个哈希表中至少有一个是有效的。
6. 如果对桶的采样超过`10 * count`的次数，则停止采样，防止*Redis*长时间的阻塞在这个调用中，
这也是导致该接口无法保证返回`count`个采样结果的原因之一。

### Redis中用于处理哈希表迭代器的操作接口

### Redis中用于扫描哈希表的操作接口
```c
unsigned long dictScan(dict *d, unsigned long v, dictScanFunction *fn, dictScanBucketFunction *bucketfn, void *privdata);
```
这段代码，作者自己的评价是：
> somewhat hard to understand at first

也就是第一次看，很令人费解。

想要理解这段代码，必须要先了解*Redis*的*SCAN*命令：
> SCAN命令以及相关SSCAN，HSCAN和ZSCAN命令都用于增量迭代一个集合元素
>
> 该组命令每次调用都会以O(1)的时间复杂度来返回若干结果，以O(N)的时间复杂度完成整个迭代。
>
> SCAN命令是一个基于游标的迭代器。这意味着命令每次被调用都需要使用上一次这个调用返回的游标作为该次调用的游标参数，以此来延续之前的迭代过程
> 当SCAN命令的游标参数被设置为0时， 服务器将开始一次新的迭代， 而当服务器向用户返回值为0的游标时， 表示迭代已结束。

按照正常的理解，遍历应该按照*0->1->2->3->...*这样的顺序来进行的，但是*Redis*对哈希表的扫描却不是按照这个顺序进行的，
以拥有8个桶且没有在重哈希过程中的哈希表来说，其扫描的顺序为*0->4->2->6-1->5->3->7->0*，如果对于这个扫描顺序没有头绪的话，
那么将其转化为二进制来看一下，*000->100->010->110->001->101->011->111->000*，我们其扫描顺序的规律：
1. 从0开始，高位加1
2. 向地位进位，到0结束

*那么为什么Redis要使用如此抽搐的扫描方式呢？*

其原因在于，对于哈希表的扫描，不是一次性完成的，而是客户端通过调用*SCAN*命令增量地进行的，而在这个过程中，有可能会发生哈希表的重哈希操作，
如果采用*0->1->2->3->...*这样的扫描顺序，则有可能漏掉某些元素，或者对某些元素进行了重复扫描。重复扫描可以通过客户端自己维护一个数据集合，
来对重复数据进行过滤，但是如果某些元素被漏掉，那么则有可能会产生一些错误。

*什么这种跳跃的扫描方式可以避免元素被遗漏呢？*

首先，我们需要明确的一点是，*Redis*中桶的个数一定是2的n次方个，以拥有8个桶的哈希表为例：
* 如果扩容到16个桶，那么0号桶中的元素，也就是二进制`000`中的元素一定会被分散到`0000`和`1000`这两个桶中，也就是0号桶和8号桶中。
* 如果缩容到4个桶，那么0号桶和4号桶中的元素，也就是`000`和`100`中的元素会被集中到新的0号（二进制`00`号）桶中。


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*