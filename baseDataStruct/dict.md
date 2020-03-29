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

```c
static void _dictReset(dictht *ht);
int _dictInit(dict *d, dictType *type, void *privDataPtr);
```
其中`_dictReset`函数对一个给定的`dictht`进行重置初始化。
`_dictInit`函数，使用一个`dictType`以及一个私有数据指针`privDataPtr`来初始化一个`dict`数据。

```c
dict *dictCreate(dictType *type, void *privDataPtr);
```
通过这个接口，给定一个`dictType`以及一个私有内存数据的指针，内部通过调用`_dictInit`，
最终创建一个`dict`数据。