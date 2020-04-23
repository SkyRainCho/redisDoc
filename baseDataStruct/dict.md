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

## Redis哈希表的构造与增删改查

### Redis用于初始化创建与释放清理哈希表的接口
```c
static void _dictReset(dictht *ht);
int _dictInit(dict *d, dictType *type, void *privDataPtr);
int _dictClear(dict *d, dictht *ht, void(callback)(void *));
```
其中`_dictReset`函数对一个给定的`dictht`进行重置初始化。
`_dictInit`函数，使用一个`dictType`以及一个私有数据指针`privDataPtr`来初始化一个`dict`数据。

```c
dict *dictCreate(dictType *type, void *privDataPtr);
void dictRelease(dict *d);
```
通过这个接口，给定一个`dictType`以及一个私有内存数据的指针，内部通过调用`_dictInit`，
最终创建一个`dict`数据。

### Redis用于扩展哈希表容量的接口
```c
static unsigned long _dictNextPower(unsigned long size);
static int _dictExpandIfNeeded(dict *ht);
int dictExpand(dict *d, unsigned long size);
int dictResize(dict *d);
```
`__dictNextPower`用于通过给定的`size`来计算需要扩容的大小，从`DICT_HT_INITIAL_SIZE`开始，
每次翻倍，知道找到第一个大于等于`size`的数值，即为扩容后的大小。

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

而函数`dictExpand`则会根据给定的`size`对这个`dict`进行扩容，*Redis*会根据`size`计算出真正的调整容量:
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

### Redis用于进行重哈希的操作接口
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

`_dictRehashStep`函数是一个通常在内部调用的函数，如果在没有安全迭代器的情况下，这个函数会对其中1个元素执行重哈希:
```c
    if (d->iterators == 0) 
        dictRehash(d,1);
```

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

而函数`dictReplace`执行的是一个插入或者覆盖的操作，如果是插入新元素，那么放回1，如果覆盖已有元素，则返回0。

函数`dictAddOrFind`可以让我们给定一个`key`来查找这个指定`key`对应的`dictEntry`指针，如果`key`不存在`dict`中，
那么会想其中插入一个`dictEntry`，并返回。
而函数`dictFind`，会根绝给定的`key`来查找对应的`dictEntry`，如果查找不到，会返回`NULL`。

## Redis哈希表的迭代与遍历

### Redis中用于处理哈希表迭代器的操作接口

### Redis中用于随机访问哈希表元素的操作接口


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*