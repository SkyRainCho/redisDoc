# Redis中哈希表的实现

在文件*src/dict.h*文件之中定义了*Redis*关于哈希表的实现。
在*Redis*中，实现了一个内存哈希表，这个哈希支持如下的操作：
* 插入
* 删除
* 替换
* 查找
* 随机获取元素

哈希边可以在需要时进行扩容，将容量按照当前容量的二倍大小进行扩容。
同时使用拉链的方法来解决哈希碰撞的问题。

## Redis哈希表的基础数据结构
在*src/dict.h*头文件之中，首先定义了基于拉链的哈希表的链表基本节点数据结构。
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
    struct dictEntry *next;   //由于使用基于拉链的哈希表实现，next用于保存桶中下一个key-value对。
} dictEntry;
```
在这个*key-value*的数据结构之中，既可以使用一个64位整数作为*value*，也可以使用一个双精度浮点数作为*value*，
同时还可以使用一个任何一段内存作为*value*，而*value*之中也保存这段内存的指针。

基于这个哈希表的基本节点数据结构，*Redis*定义了自己的哈希表数据结构：
```c
typedef struct dictht {
    dictEntry **table;        //table是一个数组结构，其中的每个元素都是一个dictEntry指针
    unsigned long size;       //table中桶的数量
    unsigned long sizemask;   //table中桶数量的掩码
    unsigned long used;       //该哈希表之中，已经保存的*key-value*的数量
} dictht;
```
在`dictht`中这个`sizemask`数据主要视为实现快速取余的用途，这个掩码被设定为`size-1`的大小，
对于一个给定的哈希值`h`使用`h & sizemask`可以获得哈希值对`szie`的取余操作。根据余数，
可以决定这个`ky-value`数据落在哪个哈希表的哪个桶中。

同时，*src/dict.h*头文件中，定义了一个结构体，存储基础哈希操作的函数指针，
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

使用`dictht`以及`dictType`两个数据结构，*Redis*定义自己的*key-value*的数据结构`dict`:
```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx;         //重哈希索引，如果没有在进行中的重哈希，那么这个rehashidx的值为-1
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
*Redis*在`dict`中使用两个*hashtable*的主要用意是方便在进行*ReHash*操作时，进行数据转移。
*Redis*在执行重哈希操作时，不会一次性将所有的数据进行重哈希，因为如果在哈希表中有大量的*key-value*数据的话，
对所有数据进行重哈希操作，会导致系统阻塞在重哈希操作中，无法退出，而*Redis*本身又是单进程单线程的模式，
这将导致*Redis*无法对外提供服务。为解决这个问题，*Redis*在`dict`中给出了保存了两个哈希表，在进行重哈希操作时，
*Redis*会将第二个哈希表进行扩容，然后定期将第一个哈希表中的数据重哈希到第二个哈希表中。而此时，
而这时，保存在第一个哈希表中没有来得及进行重哈希的数据，对于客户端用户来说，依然是可用的，而当第一个全部数据重哈希结束后，
*Redis*会把数据从第二个哈希表中转移至第一个哈希表中，结束重哈希操作。



## Redis哈希表的基础底层操作

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

## Redis哈希表的构造与初始化接口

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
### Redis用于向哈希表中添加删除元素的操作接口
```c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing);
int dictAdd(dict *d, void *key, void *val);
int dictReplace(dict *d, void *key, void *val);
dictEntry *dictAddOrFind(dict *d, void *key);
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree);
int dictDelete(dict *ht, const void *key);
dictEntry *dictUnlink(dict *ht, const void *key);
void dictFreeUnlinkedEntry(dict *d, dictEntry *he);
```