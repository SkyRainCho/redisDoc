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

**SADD**这个命令用于添加一个或多个*member*到给定*key*对应集合对象中，如果*member*已经存在于集合中，则会忽略对这个*member*的插入；如果这个*key*对应的集合不存在，那么会先为这个*key*创建一个集合对象，然后在执行后续的插入操作。

```c
void saddCommand(client *c);
```
`saddCommand`这个**SADD**命令实现函数会通过`lookupKeyWrite`从内存数据库之中查找*key*对应的集合对象，如果对象不存在，那么会调用`setTypeCreate`创建一个集合对象，将其加入到内存数据库之中。获得到集合对象之后，通过调用集合的插入接口`setTypeAdd`，将*key*插入到集合对象之中。

### SREM命令
命令格式为：`SREM key member [member ...]`

**SREM**这个命令用于从给定的*key*集合中移除一个或者多个*member*，如果给定的*member*不存在于集合之中，那么会忽略对这个*member*的删除操作。

```c
void sremCommand(client *c);
```
`sremCommand`这个**SREM**命令实现函数在通过`lookupKeyWriteOrReply`查找到*key*的集合对象，循环调用`setTypeRemove`将*key*从集合之中删除，如果集合被清空，那么会调用`dbDelete`将集合对象从内存数据库中删除。

### SMOVE命令
命令格式为：`SMOVE source destination member`

**SMOVE**命令用于将*member*元素从*source*集合中移动到*destination*集合中，如果*member*不存在于*source*集合中，那么**SMOVE**命令则不会执行任何操作。

```c
void smoveCommand(client *c);
```
`smoveCommand`这个**SMOVE**命令的实现函数，是通过对`source`集合对象调用`setTypeRemove`函数将*meber*元素弹出，然后在对`destination`集合对象调用`setTypeAdd`函数来实现的数据移动的。


### SISMEMBER命令
命令格式为：`SISMEMBER key member`
**SISMEMBER命令**这个命令用于判断*member*是否存在与集合对象*key*之中。

```c
void sismemberCommand(client *c);
```

`sismemberCommand`这个**SISMEMBER**命令的实现函数是通过集合对象的`setTypeIsMember`函数来实现。

### SCARD命令

命令格式为：`SCARD key`
**SCARD**命令用于返回集合对象*key*之中所有的元素的数量，对于不存在的*key*则认为是一个空的集合对象返回0。
```c
void scardCommand(client *c);
```

`scardCommand`这个**SCARD**命令的实现函数是通过集合对象的`setTypeSize`函数来实现的。

### SPOP命令

命令格式为：`SPOP key [count]`
**SPOP**命令用于从集合对象*key*中随机移除并返回一个或者多个元素。

```c
void spopWithCountCommand(client *c);
void spopCommand(client *c);
```

上述两个**SPOP**命令的实现函数，本质上都是通过`setTypeRandomElement`函数从集合对象之中随机获取一个元素，在更具集合对象的编码类型，分别调用`intsetRemove`函数或者`setTypeRemove`函数，将这个元素从集合对象之中删除。

在默认执行一个元素的删除操作时，流程比较简单，当要删除多个元素的时候，*Redis*定义了一个`spopWithCountCommand`这个函数专门用于处理这种情况。对于多个元素的删除，*Redis*划分了三种情况分别进行处理：

1. 移除的数量大于等于集合对象的长度，对于这种情况，只需要将整个集合对象返回给客户端，然后在调用`dbDelete`函数将这个集合对象从内存数据库之中删除就可以。
2. 移除的数量小于集合对象的长度，当移除后剩余元素的个数大于原有集合元素个数的20%时，就属于这种情况。对于这种情况，所执行的操作，相当于循环*count*，每次从集合中调用`setTypeRandomElement`来选择一个元素，然后调用`intsetRemove`或者`setTypeRemove`函数将其删除。
3. 移除的数量接近集合对象的长度，对于这种情况，*Redis*应用了另外一个策略，首先计算出移除操作后集合中的剩余元素个数`remaining`，首先从原始集合对象中移除`remaining`个元素，将这些元素插入一个新的集合对象`newset`中，返回原始集合对象中剩余`count`个元素，然后用`newset`集合替换内存数据库中的原始集合对象。

### SRANDMEMBER命令

命令格式为：`SRANDMEMBER key [count]`
**SRANDMEMBER**命令用于从集合对象*key*中随机返回一个或者多个元素，与**SPOP**命令不同，这个命令不会删除集合对象中的元素。
```c
void srandmemberWithCountCommand(client *c);
void srandmemberCommand(client *c);
```

**SRANDMEMBER**命令的实现函数与**SPOP**命令的实现函数类似，其基础逻辑是一致的，只不过对于**SRANDMEMBER**命令不需要执行删除操作。同时区别情况2与情况3的阈值不是百分二十，而是三分之一。

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

上面这两个*Redis*命令都是用于计算多个集合对象*key*的交集的结果，如果某一个集合对象*key*不存在，则认为这是一个空的集合对象，并使用这个空集合参与集合的交集运算。**SINTER**命令与**SINTERSTORE**命令的差异在于，**SINTERSTORE**命令会将交集的结果存储在集合对象*destination*之中，如果这个集合存在，那么会覆盖这个*destination*集合对象。
```c
void sinterGenericCommand(client *c, robj **setkeys, unsigned long setnum, robj *dstkey);
void sinterCommand(client *c);
void sinterstoreCommand(client *c);
```

上述三个函数是*Redis*用于实现**SINTER**命令的，其中`sinterGenericCommand`函数是交集操作的核心逻辑，另外两个函数都是通过对`sinterGenericCommand`的封装来实现的。

`sinterGenericCommand`这个函数的逻辑为：

1. 收集所有的*key*对应的集合对象到一个集合对象数组`sets`之中。
2. 调用`qsort`函数，对收集到的集合对象按照内部元素从少到多的顺序进行排序。
3. 遍历元素数量最小的那个集合对象中的每一个元素，检查其是否存在于其他的集合之中，如果元素存在于剩余的所有集合对象之中，那么说明其是交集操作的结果之一；否则将这个元素将被忽略。

#### 集合并集差集命令

命令格式为：
1. `SUNION key [key ...]`
2. `SUNIONSTORE destination key [key ...]`
3. `SDIFF key [key ...]`
4. `SDIFFSTORE destination key [key ...]`

前两个命令用于计算多个集合对象的并集结果，与集合交集命令相似，对于不存在的key，命令会认为这是一个空的集合对象，**SUNION**命令会将并集结果返回给客户端，而**SUNIONSTORE**会将结果存储在*destination*集合对象中。

后两个命令用于计算多个集合的差集操作，也就是会返回存在于第一个*key*的集合中，同时不存在于后续*key*集合中的元素，**SDIFF**命令会将差集结果返回给客户端，而**SDIFFSTORE**会将结果存储在*destination*集合对象中。
```c
void sunionDiffGenericCommand(client *c, robj **setkeys, int setnum, robj *dstkey, int op);
void sunionCommand(client *c);
void sunionstoreCommand(client *c);
void sdiffCommand(client *c);
void sdiffstoreCommand(client *c);
```

*Redis*使用`sunionDiffGenericCommand`这个函数来作为并集与差集操作的通用接口：

1. 对于集合的交集操作，*Redis*的实现很简单，就是遍历所有收集到的集合对象，将其调用`setTypeAdd`将其插入一个目标集合中，因为`setTypeAdd`函数可以保证加入到集合中元素的唯一性，也就是满足了并集操作的要求，这个算法的时间复杂度为`O(N)`，其中`N`是所有集合中元素的个数。
2. 对于集合的差集操作，对于差集操作，*Redis*提供了两种算法：
   1. 算法1中，会遍历第一个集合对象中的每一个元素，如果这个元素不存在于后续的集合对象中，那么这个元素便作为差集的结果之一，这个算法的时间复杂度为`O(N*M)`，其中`N`为第一个集合对象中元素的个数，而`M`为集合对象的个数。
   2. 算法2中，会遍历所有集合对象中的每一个元素，如果是第一个集合对象中的元素，那么就将其加入到一个目标临时集合中；否则就尝试从目标集合中将这个元素删除，在完成所有元素的遍历之后，目标集合中存储的便是差集的结果，这个算法的时间复杂度为`O(N)`，其中`N`是所有集合对象中元素个数的总和。

#### 集合扫描命令

命令格式为：`SSCAN key cursor [MATCH pattern] [COUNT count]`
这个命令与前面散列对象**HSCAN**命令类似，用于对一个集合对象*key*执行增量的扫描。
```c
void sscanCommand(client *c);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
