# Redis中的惰性删除策略

在前面对于数据库键空间的基础操作时，我们已经了解到，在*Redis*在从键空间中删除一个*key*可以采取一种**惰性删除**的策略，应用这种策略的原因在于对于某些数据对象的释放需要消耗过多的系统资源，如果在*Redis*的主线程中采用同步的方式去删除以及释放这样的*key-value*数据，那么会导致系统长时间的阻塞在释放数据操作上，而无法处理其他的业务逻辑。对于这种情况，我们以**惰性删除**的策略，使用一个后台线程，通过异步的方式来对数据对象进行释放，无疑是一种较为合适的选择。

这个策略的开关被定义在`redisServer`这个全局状态结构体之中：
```c
struct redisServer
{
    ...
    /* Lazy free */
    int lazyfree_lazy_eviction;
    int lazyfree_lazy_expire;
    int lazyfree_lazy_server_del;
    ...
};
```
以上的三个策略开关的含义为：
1. `redisServer.lazyfree_lazy_eviction`，是否在淘汰某个*key*时，使用**惰性删除**策略。
2. `redisServer.lazyfree_lazy_expire`，是否在过期某个*key*时，使用**惰性删除**策略。
3. `redisServer.lazyfree_lazy_server_del`，服务器端删除某个*key*时，是否使用**惰性删除**策略。

默认情况下，*Redis*服务器会启用**惰性删除**的策略，如果希望关闭这个策略，那么可以修改*Redis*的配置文件中的配置项，将这个策略关闭：

    lazyfree-lazy-eviction off
    lazyfree-lazy-expire off
    lazyfree-lazy-server-del off

在*src/lazyfree.c*源文件中定义了关于*Redis*中**惰性删除**策略的相关逻辑，在这个源文件中代码的实现主要分为两个部分：
1. 对于数据的异步释放，这里会将待释放的数据加入到*Redis*中一个后台任务队列之中，由一个专门的线程，按照**FIFO**的策略，依次将待释放的数据进行释放。
1. 为释放线程提供释放数据所用到的函数接口。

## 惰性删除的异步接口

### 异步操作阈值
通过前面对于数据库基本操作以及键过期的代码，我们可以发现，只要*Redis*服务器开启了**惰性删除**策略，那么在从键空间删除释放一个数据是，都会使用**惰性删除**的异步接口。但是**惰性删除**策略被设计的初衷是防止较为复杂的释放过程过多地占用主线程，可以想象如果在不开启**惰性删除**策略时，从数据库键空间中释放一个包含1000万条记录的散列对象，那么主线程阻塞在遍历释放散列对象中的1000万条记录时，对于整个*Redis*的并发性将会产生何等的灾难。

但是，如果不分青红皂白将要释放的数据一律扔给后台线程来处理，是否是一种合理的操作呢？显然这并不合理，例如释放一个类似于字符串对象的数据，或者一个长度较小的列表对象，如果选择在主线程中同步地，立即地进行释放，这会大大提高内存回收的效率，同时不至于对性能产生较大的影响。

因此*Redis*给出一个用于判断数据对象是否需要异步释放的接口
```c
size_t lazyFreeGetFreeEffort(robj *obj);
```
影响一个对象被释放效率的因素主要在于该对象占用了多少块被分配的内存。接口`lazyFreeGetFreeEffort`函数，主要用于获取一个数据对象大致占用的内存块数。对于字符串对象，以及使用压缩链表实现的散列对象、有序集合对象，或者使用整数集合实现的集合对象，则认为他们都只占用了一块被分配的内存；而对于列表对象、使用哈希表实现的散列对象、集合对象，以及使用跳跃表实现的有序集合对象，这个函数则会返回这些对象中包含元素的个数。

同时*Redis*定义了一个需要异步释放的阈值
```c
#define LAZYFREE_THRESHOLD 64
```
如果`lazyFreeGetFreeEffort`获取的结果如果小于`LAZYFREE_THRESHOLD`这个阈值，那么便会使用同步释放的方式；否则会采用异步释放的**惰性删除**策略。

### 异步数据库删除
```c
int dbAsyncDelete(redisDb *db, robj *key);
```
`dbAsyncDelete`这个接口用于从数据库的键空间中**惰性删除**一对*key-value*，如果*Redis*服务器开启了前面介绍的**惰性删除**开关时，在执行删除时，便会调用`dbAsyncDelete`这个接口来处理。这个函数的具体实现逻辑为：
```c
int dbAsyncDelete(redisDb *db, robj *key)
{
    if (dictSize(db->expires) > 0) dictDelete(db->expires, key->ptr);
    
    dictEntry *de = dictUnlink(db->dict, key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        size_t free_effort = lazyfreeGetFreeEffort(val);

        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1)
        {
            bioCreateBackgroundJob(BIO_LAZY_FREE, val, NULL, NULL);
            dictSetVal(db->dict, de, NULL);
        }
    }
    
    if (de) {
        dictFreeUnlinkedEntry(db->dict, de);
    }
    else
    {
        return 0;
    }
}
```
通过这个函数，我们可以看到其中的几个细节：
1. 由于`redisDb.expires`中存储的是*key-timestamp*数据，不存在释放压力，因此直接调用`dictDelete(db->expires, key->ptr);`进行数据删除。
2. 通过哈希表`dictUnlink`接口，将*key-value*对应的`dictEntry`从哈希表中移除，但不直接释放。
3. 通过`dictGetVal`来获取*value*的对象，如果对象的引用计数为1，同时达到了`LAZYFREE_THRESHOLD`阈值，那么通过`bioCreateBackgroundJob`接口向后台任务队列添加任务，同时通过`dictSetVal`接口将*value*对象与`dictEntry`解绑。
4. 最后通过`dictFreeUnlinkedEntry`接口，将`dictEntry`数据进行释放，因为前面已经将*value*与`dictEntry`解绑，因此这里的释放操作相当于只释放了*key*，因此不会对主线程造成过大的负担。

我们可以发现，不论是对数据库键空间的同步删除`dbSyncDelete`，还是异步删除`dbAsyncDelete`，对于客户端来说都是透明的，因为这两种操作都是会将*key*从数据库的键空间中移除，唯一的区别就在于，对*value*数据的释放，是同步操作还是异步操作。

### 异步释放对象
```c
void freeObjAsync(robj *o);
```
`freeObjAsync`这个接口用于异步地释放一个*Redis*对象数据，这个接口只用在对键空间的覆盖操作，用于释放旧的*value*数据。
```c
void dbOverwrite(redisDb *db, robj *key, robj *val)
{
    dictEntry *de = dictFind(db->dict, key-ptr);
    robj *old = dictGetval(de);
    ...
    if (server.lazyfree_lazy_server_del)
    {
        freeObjAsync(old);
        ...
    }
    ...   
}

```
### 异步清空数据库
```c
void emptyDbAsync(redisDb *db);
```
`emptyDbAsync`接口用于异步地情况整个数据库的键空间，当用户执行清库数据库命令时，如果给出了`async`参数，那么便会调用`emptyDbAsync`接口对数据库进行清空。这个函数会为`redisDb.dict`以及`redisDb.expires`分配两个新的哈希表，对于原有的两个哈希表，则将其包装为一个后台任务，加入到后台任务中等待时候。

由于这个函数为`redisDb.dict`以及`redisDb.expires`分配两个新的哈希表，因此即使释放原始哈希表是异步操作，对于用户来说，这一步是透明的，执行完改操作之后，展现给用户的值两个空的哈希表，就好像同步地执行了清空释放操作一样。

### 集群Slot异步清空
```c
void slotToKeyFlushAsync(void);
```
对于*Redis*集群，我们会在后续的内容之中进行讲解，这里只需要了解，*Redis*集群中会存在一个基数树的数据结构`rax`，在异步清理时，与上面清空数据库的操作类似，重新分配一个新的基数树，并将原始的基数树加入任务队列中，等待异步被释放。

## 后台线程调用的释放接口
通过前面的接口函数，*Redis*将需要被异步释放的数据加入后台的任务队列，等待后台线程异步地去执行释放逻辑。从上面介绍的接口来看，实际的释放工作只有三种：
1. 释放一个单一的数据对象，对应接口为`dbAsyncDelete`以及`freeObjAsync`。
2. 释放整个哈希表，对应接口`emptyDbAsync`。
3. 释放基数树，对应接口`slotToKeyFlushAsync`。

而后台线程在开始处理**惰性删除**任务时，便会调用下面这三个函数，分别来处理上述的三种情况：
```c
void lazyfreeFreeObjectFromBioThread(robj *o)；
void lazyfreeFreeDatabaseFromBioThread(dict *ht1, dict *ht2);
void lazyfreeFreeSlotsMapFromBioThread(rax *rt);
```
上面三个函数分别对调用对应数据对象的释放函数，完成对数据的释放操作。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*