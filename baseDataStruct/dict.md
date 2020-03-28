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

基于这个哈希表的基本节点数据结构，*Redis*定义了自己的哈希表数据结构：
```c
typedef struct dictht {
    dictEntry **table;        //table是一个数组结构，其中的每个元素都是一个dictEntry指针
    unsigned long size;       //
    unsigned long sizemask;   //
    unsigned long used;       //
} dictht;
```

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
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
*Redis*在`dict`中使用两个*hashtable*的主要用意是方便在进行*ReHash*操作时，进行数据转移。



## Redis哈希表的基础底层操作

## Redis哈希表的构造与初始化接口