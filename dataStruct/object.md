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

### 对象创建与释放：

```c
robj *createObject(int type, void *ptr);
```

用于根据给定的`type`通过调用`zmalloc`分配内存，创建一个*Redis*对象，并对*Redis*对象的字段进行初始化，默认*Redis*对象的编码方式`robj.encoding`都是采用原始方式进行编码，并将引用计数`robj.refcount`设置为1，同时还会根据系统的淘汰策略，为`robj.lru`设置初始值。

`createObject`是创建*Redis*对象的基础函数，*Redis*同时提供了一组创建不同类型*Redis*对象的默认接口函数：

| 对象类型         | 编码类型                 | 函数接口                              |
| ---------------- | ------------------------ | ------------------------------------- |
| 列表数据类型     | `OBJ_ENCODING_QUICKLIST` | `robj *createQuicklistObject(void)`   |
| 列表数据类型     | `OBJ_ENCODING_ZIPLIST`   | `robj *createZiplistObject(void)`     |
| 集合数据类型     | `OBJ_ENCODING_HT`        | `robj *createSetObject(void)`         |
| 集合数据类型     | `OBJ_ENCODING_INTSET`    | `robj *createIntsetObject(void)`      |
| 散列数据类型     | `OBJ_ENCODING_ZIPLIST`   | `robj *createHashObject(void)`        |
| 有序集合数据类型 | `OBJ_ENCODING_SKIPLIST`  | `robj *createZsetObject(void)`        |
| 有序集合数据类型 | `OBJ_ENCODING_ZIPLIST`   | `robj *createZsetZiplistObject(void)` |
| Stream数据类型   | `OBJ_ENCODING_STREAM`    | `robj *createStreamObject(void)`      |
| Module数据类型   | `OBJ_ENCODING_RAW`       | `robj *createModuleObject(void)`      |

通过上面这张表格我们可以了解到不同对象数据其底层所使用的数据类型，不过需要注意的是，对于散列类型，默认使用的是压缩链表作为底层实现，但是在一定条件下，会转化为由`dict`数据结构作为底层实现的形式。而这张表中缺失了字符串数据类型的内容，对于字符串数据类型，会在后续单独做专门讲解。

同时*Redis*还提供了一组对应的用于释放对象的操作接口：

```c
void freeStringObject(robj *o);
void freeListObject(robj *o);
void freeSetObject(robj *o);
void freeZsetObject(robj *o);
void freeHashObject(robj *o);
void freeModuleObject(robj *o);
void freeStreamObject(robj *o);
```

最后，*Redis*还为对象提供了一个类型检查接口：

```c
int checkType(client *c, robj *o, int type)
{
  if (o->type != type)
  {
    addReply(c.shared.wrongtypeerr);
    return 1;
  }
  return 0;
}
```

上面这个函数主要的应用场景在于，当*Redis*客户端针对某一个*key*执行某些命令时，需要判断这个*key*对应的*value*对象的类型，是否符合命令对应的类型，如果对一个列表数据执行字符串命令显然是不合适的，因此这个函数函数主要用于执行客户端命令前对数据类型进行校验。

### 对象共享

*Redis*为对象的共享引用提供了几个操作接口：

```c
robj *makeObjectShared(robj *o);
void incrRefCount(robj *o);
void decrRefCount(robj *o);
void decrRefCountVoid(void *o);
robj *resetRefCount(robj *obj);
```

1. `makeObjectShared`用于将对象的引用计数`robj.refcount`设置为一个特定的标记数值`OBJ_SHARED_REFCOUNT`，这个数值在*Redis*中被定义为`#define OBJ_SHARED_REFCOUNT INT_MAX`，也就是被定义为`INT_MAX`。这个函数用于在*Redis*启动时，初始化一些较小的整形数对象作为共享对象，这些对象将在*Redis*整个生命期中一直存在，当其他现场引用这些对象的时候，不需要增加引用计数，当然解绑对象时，也不会减少其引用计数的数量：
   
    ```c
    void createSharedObjects(void) {
      	...
        for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
            shared.integers[j] = makeObjectShared(createObject(OBJ_STRING,(void*)(long)j));
            shared.integers[j]->encoding = OBJ_ENCODING_INT;
        }
    ...
    }
    ```
    
2. `increRefCount`这个函数用于对对象的引用计数`robj.refcount`执行加一操作，这个操作的前提是`robj.refcount`被没有被设置为`OBJ_SHARED_REFCOUNT`。

3. `decrRefCount`这个函数用于对对象的引用计数`robj.refcount`执行减一操作，如果当前的引用计数已经为1，那么执行这个函数会根据对象的类型来调用对应释放函数，来释放对象；如果当前的引用计数没有被设置为`OBJ_SHARED_REFCOUNT`，那么会对引用计数执行减一的操作。

4. `decrRefCountVoid`这个函数实际上是`decrRefCount`函数的一个变形形式，它是接受`void*`作为参数，而不是使用`robj*`作为参数。

5. `resetRefCount`这个函数用于重置对象的引用计数，将`robj.refcount`字段重置为0。通过调用这个函数，我们可以不用释放对象，就可以将其引用计数设置为0。有些函数自身就会增加对象的引用计数，但是如果我们要将一个新的对象（此时引用计数已经为 1）参数该函数时，使用`resetRefCount`就很有用了：

   ```c
   obj = createObject(...);
   functionThatWillIncrementRefCount(obj);
   decrRefCount(obj)
   ```

   如果应用`resetRefCount`这函数，上面的调用就可以简化为下面的形式：

   ```c
   functionThatWillIncrementRefCount(resetRefCount(CreateObject(...)));
   ```

### 长度计算

```c
size_t objectComputeSize(robj *o, size_t sample_size);
```

`objectComputeSize`这个函数用于计算*Redis*对象在内存中所占用的字节数，不过这个返回的字节数是一个近似值，尤其是对于聚合数据类型，当给定了采样大小`sample_size`的时候。`objectComputeSize`函数会根据不同的对象类型，以不同的方式来计算对象占用内存的大小，下面截取计算列表数据类型内存占用的代码段，可以了解其具体的实现细节：

```c
size_t objectComputeSize(robj *o, size_t sample_size) {
  ...
  if (o->type == OBJ_LIST) {
    if (o->encoding == OBJ_ENCODING_QUICKLIST) {
      quicklist *ql = o->ptr;
      quicklistNode *node = ql-head;
      do {
        elesize += sizeof(quicklistNode)+ziplistBlobLen(node=>zl);
        samples++;
      } while ((node = node->next) && samples < sample_size);
      asize += (double)elesize/samples*ql->len;
    }
    else if (o->encoding == OBJ_ENCODING_ZIPLIST) {
      asize = sizeof(*o)+ziplistBlobLen(o->ptr);
    }
    else {
      serverPanic("Unknown list encoding");
    }
  }
  ...
}
```



### OBJECT命令

`OBJECT`命令可以用来查看*Redis*对象内部的细节，按照*Redis*官方文档给出的描述是：

> The [OBJECT](https://redis.io/commands/object) command allows to inspect the internals of Redis Objects associated with keys. It is useful for debugging or to understand if your keys are using the specially encoded data types to save space. Your application may also use the information reported by the [OBJECT](https://redis.io/commands/object) command to implement application level key eviction policies when using Redis as a Cache.

也就是说，`OBJECT`用于检查或者了解你的keys是否用到了特殊编码 的数据类型来存储空间。 当*Redis*作为缓存使用的时候，你的应用也可能用到这些由`OBJECT`命令提供的信息来决定应用层的key的驱逐策略。

应用这个`OBJECT`这个命令，可以通过子命令查看对象的引用计数（`refcount`），编码方式（`encoding`），空闲时间（`idletime`）或者使用频次（`freq`）等信息，其基本的操作流程是：

1. 根据命令参数对应的*Key*查找到对应的*Redis*对象。
2. 再根据子命令的不同，获取*Redis*对象的不同字段用以返回。

```c
void objectCommand(client *c)
{
  ...
  if (!strcasecmp(c->argv[1]->ptr, "refcount") && c->argc == 3)
  {
    if ((o == objecCommandLookupOrReply(c, c->argv[2].shared.nullbulk)) == NULL)
    {
      return;
    }
    addReplyLongLong(c,o->refcount);
  }
  ...
}
```

上面这段代码以查看引用计数子命令为例，展示了`OBJECT`命令的实现方式。

`redisObject`在*Redis*内部被广泛使用，但是为了避免间接访问所带来的的额外开销，最近在很多地方，*Redis*使用没有被包装在`redisObject`中的单纯的动态*String*来表示数据。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*