# Redis中Redis Object实现

在*Redis*中定义了基础的*Redis*对象类型
```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```
这个数据结构基本可以表示所有的*Redis*数据类型，通过这个`redisObject.type`这个4比特的字段，我们获得一个给定`redisObject`数据所代表的数据类型，在*Redis*中，*src/server.h*头文件中给出了`redisObject`数据类型的定义：

|定义|含义|
|-----|----|
|`#define OBJ_STRING 0`|字符串数据类型|
|`#define OBJ_LIST 1`|列表数据类型|
|`#define OBJ_SET 2 `|集合数据类型|
|`#define OBJ_ZSET 3`|有序集合数据类型|
|`#define OBJ_HASH 4`|散列数据类型|
|`#define OBJ_MODULE 5`|Module数据类型|
|`#define OBJ_STREAM 6`|Stream数据类型|

而`redisObject.refcount`这个字段，可以允许我们在多个地方引用同一个对象，而不是需要重复多次的分配过程。最终`redisObject.ptr`这个字段用于指向实际的数据位置，即使相同的的数据类型，基于所使用的不同的编码格式，其最终的内容也有可能不同，因此*Redis*在*src/server.h*头文件中给出了对象编码类型的定义：

|编码方式定义|具体含义|
|----------|-------|
|`#define OBJ_ENCODING_RAW 0 `|使用原始数据进行编码|
|`#define OBJ_ENCODING_INT 1`|按照整数进行编码|
|`#define OBJ_ENCODING_HT 2`|按照哈希表进行编码|
|`#define OBJ_ENCODING_ZIPMAP 3`|按照压缩Map进行编码（暂时不在使用）|
|`#define OBJ_ENCODING_LINKEDLIST 4`|按照双端链表进行编码（已不再使用）|
|`#define OBJ_ENCODING_ZIPLIST 5`|按照压缩链表进行编码|
|`#define OBJ_ENCODING_INTSET 6`|按照整数集合进行编码|
|`#define OBJ_ENCODING_SKIPLIST 7`|按照跳跃变进行编码|
|`#define OBJ_ENCODING_EMBSTR 8`|使用嵌入数据进行编码|
|`#define OBJ_ENCODING_QUICKLIST 9`|按照快速链表进行编码|
|`#define OBJ_ENCODING_STREAM 10`||

`redisObject.lru`这个字段用于处理对象过期的相关逻辑，可以使用**LRU**策略来处理过期逻辑，也可以使用**LFU**策略来处理过期逻辑。关于对象过期的内容，会在后面专门进行讲理，这一部分将着重讲解*Redis*对象的内容。

### 对象创建

```c
robj *createObject(int type, void *ptr);
```

用于根据给定的`type`创建一个*Redis*对象。

### 对象共享

```c
robj *makeObjectShared(robj *o);
void incrRefCount(robj *o);
void decrRefCount(robj *o);
void decrRefCountVoid(void *o);
robj *resetRefCount(robj *obj);
```

### 长度计算

```c
size_t objectComputeSize(robj *o, size_t sample_size);
```

### OBJECT与MEMORY命令

`redisObject`在*Redis*内部被广泛使用，但是为了避免间接访问所带来的的额外开销，最近在很多地方，*Redis*使用没有被包装在`redisObject`中的单纯的动态*String*来表示数据。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*