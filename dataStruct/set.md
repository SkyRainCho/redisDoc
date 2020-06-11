# Redis中集合对象类型实现
*Redis*中集合对象是用于存储其他对象的无序集合，其底层可以采用整数集合或者`dict`作为实现的数据结构。如果集合对象中存储的都是整形数，并且集合对象中存储的元素个数小于512时，会使用整数集合作为底层实现；否则会使用`dict`作为底层数据结构的实现。

对于使用`OBJ_ENCODING_INTSET`编码的集合对象，所有的*key*数据全部存储于底层整数集合的连续内存之中；对于使用`OBJ_ENCODING_HT`编码的集合对象，底层哈希表的`dictEntry`用于存储*key*，而`dictEntry`的值字段为空，不存储数据。

同时在*src/server.h*头文件中，还定义了一个集合对象的迭代器，这个迭代器提供了更高的抽象，可以实现对底层整数集合以及哈希表的迭代：
```c
typedef struct {
	robj *subject;
	int encoding;
	int ii;
	dictIterator *di;
} setTypeIterator;
```
在这个机构体中：
1. `setTypeIterator.subject`以及`setTypeIterator.encoding`分别用于表示迭代器对应的集合对象，以及对应的编码方式。
2. `setTypeIterator.ii`用于表示在使用`OBJ_ENCODING_INTSET`时的底层迭代器。
3. `setTypeIterator.di`用于表示在使用`OBJ_ENCODING_HT`时的底层迭代器。

## 集合对象基础操作

### 集合对象的创建
```c
robj *setTypeCreate(sds value);
```
`setTypeCreate`这个函数用于从`value`这个初始值来创建一个集合对象，这个函数会通过`isSdsRepresentableAsLongLong`来判断`value`是否是一个整形数：
1. 如果是一个整数，则会调用`createIntsetObject`来创建一个`OBJ_ENCODING_INTSET`编码的集合对象，
2. 否侧会调用`createSetObject`来创建一个`OBJ_ENCODING_HT`编码的集合对象。

### 集合对象编码转换
```c
void setTypeConvert(robj *setobj, int enc);
```
`setTypeConvert`这个函数是专门用于处理集合对象的编码类型的转换，*Redis*只允许将`OBJ_ENCODING_INTSET`编码的集合对象转换为`OBJ_ENCODING_HT`编码的集合对象，禁止反向的编码转换。调用这个函数的时机有两个：
1. 向一个`OBJ_ENCODING_INTSET`编码的集合对象中插入一个非整数数据。
2. `OBJ_ENCODING_INTSET`编码的集合对象中元素的数量超过`server.set_max_intset_entries`所规定的阈值。

*Redis*对集合类型的编码转换逻辑为:
1. 通过`dictCreate`创建一个哈希表。
2. 调用`dictExpand`为哈希表预先设置长度，防止重哈希。
3. 遍历原有的整数集合，将数值依次调用`dictAdd`加入到哈希表中。
4. 替换集合对象的底层数据以及编码方式。

### 集合对象的插入
```c
int setTypeAdd(robj *subject, sds value);
```
`setTypeAdd`这个函数用于实现将给定的`value`插入到集合对象之中，如果这个`value`在集合不存在，那么插入成功，函数返回1；否则不会进行数据的插入，同时函数返回0。根据集合对象所使用的不同的编码类型：
1. 针对`OBJ_ENCODING_HT`编码，会调用`dictAddRaw`尝试将`value`插入底层哈希表。
2. 针对`OBJ_ENCODING_INTSET`编码，会首先检查`value`是否是整形数，如果是整形则调用`intsetAdd`接口将`value`插入底层整数集合，否则先将集合对象转换为`OBJ_ENCODING_HT`编码，然后通过`dictAdd`进行插入操作。

### 集合对象的删除
```c
int setTypeRemove(robj *setobj, sds value);
```
`setTypeRemove`这个函数会通过在底层调用`dictDelete`或者是`intsetRemove`接口，将`value`从底层数据结构删除。如果当前的对象是采用`OBJ_ENCODING_HT`编码，那么在执行删除后，会检查底层哈希表是否存在重哈希的空间，如果有必要会对底层哈希表执行`dictResize`。

### 集合对象的查找
```c
int setTypeIsMember(robj *subject, sds value);
```
`setTypeIsMember`这个函数会根据编码类型，调用`dictFind`函数执行哈希搜索，或者调用`intsetFind`执行二分搜索。

### 集合对象的长度获取
```c
unsigned long setTypeSize(const robj *subject);
```
函数`setTypeSize`会通过`dictSize`函数以及`intsetLen`函数，来获取一个集合对象的元素的个数。

### 集合对象的随机取值
```c
int setTypeRandomElement(robj *setobj, sds *sdsele, int64_t *llele);
```
`setTypeRandomElement`这个函数用于从一个给定非空的集合对象中，通过调用`dictGetRandomKey`或者`intsetRandom`函数来随机返回一个元素。字符串对象会通过`sdsele`参数进行返回；整数会通过`llele`参数进行返回，同时函数会返回这个集合对象的编码类型，这里需要注意的是，不论当前的集合对象采用什么编码方式，`sdsele`以及`llele`这连个参数都需要提供，不能将其中一个设置为`NULL`。

### 集合对象的遍历
```c
setTypeIterator *setTypeInitIterator(robj *subject);
void setTypeReleaseIterator(setTypeIterator *si);
int setTypeNext(setTypeIterator *si, sds *sdsele, int64_t *llele);
sds setTypeNextObject(setTypeIterator *si);
```
上面四个函数是*Redis*为集合对象的迭代于遍历所提供的接口，这里需要注意的一点的是，*Redis*没有为整数集合提供相应的迭代器类型与操作，但是由于整数集合本质上可以被认为是连续内存上的有序整数数组，因此集合迭代器中使用`setTypeIterator.ii`这个下标索引来作为整数集合的迭代器，用过对下标的加减来实现迭代器的移动。

## 集合对象命令实现
### SADD命令
命令格式为：`SADD key member [member ...]`
```c
void saddCommand(client *c);
```

### SREM命令
命令格式为：`SREM key member [member ...]`
```c
void sremCommand(client *c);
```

### SMOVE命令
命令格式为：`SMOVE source destination member`
```c
void smoveCommand(client *c);
```

### SISMEMBER命令
命令格式为：`SISMEMBER key member`

```c
void sismemberCommand(client *c);
```

### SCARD命令
命令格式为：`SCARD key`

```c
void scardCommand(client *c);
```

### SPOP命令
命令格式为：`SPOP key [count]`

```c
void spopWithCountCommand(client *c);
void spopCommand(client *c);
```

### SRANDMEMBER命令
命令格式为：`SRANDMEMBER key [count]`
```c
void srandmemberWithCountCommand(client *c);
void srandmemberCommand(client *c);
```

### 集合操作命令

#### 辅助操作
```c
int qsortCompareSetsByCardinality(const void *s1, const void *s2);
int qsortCompareSetsByRevCardinality(const void *s1, const void *s2);
```

#### 集合交集命令
命令格式为：
1. `SINTER key [key ...]`
2. `SINTERSTORE destination key [key ...]`

```c
void sinterGenericCommand(client *c, robj **setkeys, unsigned long setnum, robj *dstkey);
void sinterCommand(client *c);
void sinterstoreCommand(client *c);
```

#### 集合差集并集命令
命令格式为：
1. `SUNION key [key ...]`
2. `SUNIONSTORE destination key [key ...]`
3. `SDIFF key [key ...]`
4. `SDIFFSTORE destination key [key ...]`

```c
void sunionDiffGenericCommand(client *c, robj **setkeys, int setnum, robj *dstkey, int op);
void sunionCommand(client *c);
void sunionstoreCommand(client *c);
void sdiffCommand(client *c);
void sdiffstoreCommand(client *c);
```

#### 集合扫描命令
命令格式为：`SSCAN key cursor [MATCH pattern] [COUNT count]`

```c
void sscanCommand(client *c);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
