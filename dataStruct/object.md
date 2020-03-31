# Redis中Redis Object实现

在*Redis*中定义了五种实际的数据类型
```c
#define OBJ_STRING 0    //
#define OBJ_LIST 1      //
#define OBJ_SET 2       //
#define OBJ_ZSET 3      //
#define OBJ_HASH 4      //
```
同时还定了基础的*Redis*对象类型
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