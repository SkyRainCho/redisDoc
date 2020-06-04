# Redis中字符串对象类型实现

对于*Redis*中字符串对象的类型的代码主要分布在两个文件之中，其中在*src/object.c*文件中主要是实现了字符串数据类型的构造相关的操作，另外在*src/t_string.c*文件中则实现了字符串的相关命令。

对于*Redis*中的字符串对象，可以使用三种编码类型，分别是：
1. `OBJ_ENCODING_RAW`
2. `OBJ_ENCODING_INT`
3. `OBJ_ENCODING_EMBSTR`

其中当字符串的长度较短的时候，*Redis*会采用`OBJ_ENCODING_EMBSTR`的编码方式，这个长度阈值的定义为`#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44`，当超过这个长度的字符串则会才用`OBJ_ENCODING_RAW`的编码方式，而当这个字符串实际上是一个整形数的时候，*Redis*则会采用`OBJ_ENCODING_INT`对其进行编码。

字符串对象类型是*Redis*中最基本的数据类型，虽然名为字符串数据对象，但是其可以存储的数据除了数字，文本数据之外，还可以存储例如一张图片或者其他的序列化数据，其本质原因是，字符串对象类型底层采用的是`sds`数据类型，而这个类型是二进制安全的，因此可以存储任何数据，每一个字符串对象最大可以存储512MB的数据，这个限制是通过*src/t_string.c*文件中的静态函数`checkStringLength`来定义的。

## Redis字符串对象的基础操作
### Redis字符串对象的构造
#### 字符串型对象的构造
*Redis*对于字符串型的字符串对象定义了两种编码方式：
1. 如果使用的是`OBJ_ENCODING_RAW`这种编码方式，那么表示对象的`robj`与表示底层数据的`sds`在内存分布上是相互分离的，通过`robj.ptr`指针指向`sds`数据，来实现对象对数据的引用。
2. 如果使用的是`OBJ_ENCODING_RAW`这种编码方式，那么对象`robj`数据与底层数据的`sds`是处于一个连续的内存块中的，`sds`数据存储在`robj`数据之后。

*Redis*为创建字符串型对象提供了三个接口函数：
```c
robj *createRawStringObject(const char *ptr, size_t len);
robj *createEmbeddedStringObject(const char *ptr, size_t len);
robj *createStringObject(const char *ptr, size_t len);
```
其中`createRawStringObject`与`createEmbeddedStringObject`函数用于创建`OBJ_ENCODING_RAW`编码或者`OBJ_ENCODING_EMBSTR`编码的字符串对象，而`createStringObject`则是对上述两个函数更高级的封装，会判断`len`是否大于`#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44`，如果大于这个阈值，那么会按照`OBJ_ENCODING_RAW`编码，调用`createRawStringObject`接口进行创建，否则的话则会调用`createEmbeddedStringObject`来创建字符串对象。

另外，我们可以通过`createEmbeddedStringObject`的实现来了解`OBJ_ENCODING_EMBSTR`编码对象的内存分布：
```c
robj *createEmbeddedStringObject(const char *ptr, size_t len)
{
	robj *o = zmalloc(sizeof(robj) + sizeof(struct sdshdr8) + len + 1);
	struct sdshdr8 *sh = (void *)(o + 1);
	o->type = OBJ_STRING;
	o->encoding = OBJ_ENCODING_EMBSTR;
	o->ptr = sh + 1;
	o->refcount = 1;
	...
}
```

#### 数值型对象的构造
##### 整形数值
处于节约内存的考虑，一部分整形数值是直接存储在`robj.ptr`这个指针中，对于这种形式的对象，*Redis*将其设定为`OBJ_ENCODING_INT`的编码类型；而为了进一步的节约内存，对于比较小的整数`#define OBJ_SHARED_INTEGERS 10000`，*Redis*会在启动的时候自动创建这么一组整数形式的字符串对象`shared.integers`供后续共享。如果用户需要一个数值小于`OBJ_SHARED_INTEGERS`的整数型字符串对象，那么他可以直接从`shared.integers`中进行引用，而不需要重复创建。

*Redis*为创建整形数值字符串对象提供了三个接口函数：
```c
robj *createStringObjectFromLongLongWithOptions(long long value, int valueobj);
robj *createStringObjectFromLongLong(long long value);
robj *createStringObjectFromLongLongForValue(long long value);
```
其中`createStringObjectFromLongLongWithOptions`是这个三个函数中的基础，这个函数会给定一个整形数`value`，返回对应的字符串对象，如果`valueobj`参数为1，那么*Redis*会强制为`value`创建一个独立的字符串对象，否侧如果`value`小于`OBJ_SHARED_INTEGERS`的话，*Redis*则会为`value`则会返回系统创建的共享对象。其具体的执行步骤为：
1. 判断`value`以及`valueobj`的数值，如果满足`value >= 0 && value < OBJ_SHARED_INTEGERS && valueobj == 0`，那么直接从`shared.integers`中返回共享对象。
2. 如果不满足1中的条件，那么判断`value >= LONG_MIN && value <= LONG_MAX`，如果满足这种情况，那么使用`OBJ_ENCODING_INT`编码方式创建一个字符串对象，并使用`o->ptr = (void *)((long)value);`将`value`存储在`robj.ptr`指针上。
3. 如果上述两个条件都不满足，那么*Redis*会通过`sdsfromlonglong`创建一个`sds`数据，并使用这个数据创建一个字符串对象。

而`createStringObjectFromLongLong`和`createStringObjectFromLongLongForValue`则是对于`createStringObjectFromLongLongWithOptions`函数的一个封装，分别相当与`valueobj == 0`以及`valueobj == 1`的情况。

##### 浮点数数值
```c
robj *createStringObjectFromLongDouble(long double value, int humanfriendly);
```
上面这个函数用于从一个长浮点数`value`中创建一个字符串对象。

### Redis数值型字符串对象的解码

#### 整形数值的解码
```c
int isSdsRepresentableAsLongLong(sds s, long long *llval);
int isObjectRepresentableAsLongLong(robj *o, long long *llval);
```
上面两个函数分别用于从一个`sds`数据和`robj`数据中尝试解码出整形数据，如果解码失败函数会返回`C_ERR`，否侧返回`C_OK`，并解码出的数值存储在`llval`之中。而这个`isObjectRepresentableAsLongLong`现在在*Redis*并未被使用，取代它的是使用更加广泛的`getLongLongFromObject`的函数。

```c
int getLongLongFromObject(robj *o, long long *target);
int getLongLongFromObjectOrReply(client *c, robj *o, long long *target, const char *msg);
int getLongFromObjectOrReply(client *c, robj *o, long *target, const char *msg);
```
上述三个函数都是用于尝试从一个给定的`robj`字符串对象中解码出一个整形数值，`getLongLongFromObjectOrReply`和`getLongFromObjectOrReply`在尝试解码失败的时候，会通知给对应的客户端`client`。

#### 浮点数值的解码
```c
int getDoubleFromObject(const robj *o, double *target);
int getDoubleFromObjectOrReply(client *c, robj *o, double *target, const char *msg);
int getLongDoubleFromObject(robj *o, long double *target);
int getLongDoubleFromObjectOrReply(client *c, robj *o, long double *target, const char *msg);
```
上述四个函数用于尝试从一个`robj`对象中尝试解码浮点数数据。

### Redis字符串对象的比较
```c
int compareStringObjectsWithFlags(robj *a, robj *b, int flags);
int compareStringObjects(robj *a, robj *b);
int collateStringObjects(robj *a, robj *b);
int equalStringObjects(robj *a, robj *b);
```
上面四个函数用于实现对两个字符串对象的比较操作，`compareStringObjectsWithFlags`是比较操作的的基础实现，后面的三个函数都是基于`compareStringObjectsWithFlags`这个函数进行的封装。

`compareStringObjectsWithFlags`函数接口用来根据传入的`flags`参数来调用`memcpy`或者`strcoll`来对底层数据进行比较。如果待比较的字符串对象是整数型的字符串，那么调用这个接口进行比较的话，会将数字转化为字符串，然后在通过`memcpy`或者`strcoll`进行比较。

*Redis*定义了两个比较的*flags*：
```c
#define REDIS_COMPARE_BINARY (1 << 0)
#define REDIS_COMPARE_COLL (1 << 1)
```
上述这两个*flags*分别对应使用`memcpy`和`strcoll`进行检查，也就是对字符串对象进行二进制比较或者根据本地环境，基于字典顺序进行比较。

### Redis字符串对象的其他操作

#### 字符串对象的裁剪
```c
void trimStringObjectIfNeeded(robj *o)
{
	if (o->encoding == OBJ_ENCODING_RAW && sdsavail(o->ptr) > sdslen(o->ptr) / 10) {
		o->ptr = sdsRemoveFreeSpace(o->ptr);
	}
}
```
从前面对于`sds`介绍，我们知道，为了防止频繁的分配内存，`sds`通常会多分配出一段内存以备后用。同时`sds`中还提供`sdsRemoveFreeSpace`接口用于释放空闲的内存。基于上述这两个特性，*Redis*定义了一个对于字符串对象的剪裁操作函数，用于优化对象的内存占用，当底层`sds`中空闲内存的大小超过已被使用内存的10%的时候，那么会调用`sdsRemoveFreeSpace`对`sds`的空闲内存进行释放。

```c
robj *tryObjectEncoding(robj *o);
```
而`tryObjectEncoding`这个函数则是应用一组规则尝试对一个字符串对象的内存进行优化：其基本的逻辑为：
```flow

```

#### 字符串对象的其他操作
```c
robj *getDecodedObject(robj *o);
```
`getDecodeObject`这个函数会返回一个新的对象，如果原始对象是一个字符串类型对象，那么会增加其引用计数，并将这个原始对象返回；当原始对象是一个整数型对象时，则会将这个整数转化为一个字符串形式，然后为其创建一个字符串类型的新对象。

```c
robj *dupStringObject(const robj *o);
```
`dupStringObject`这个函数会赋值一个字符串对象，并确保返回的对象与原始对象由相同的编码类型。

```c
size_t stringObjectLen(robj *o);
```
`stringObjectLen`这个函数则用于获取一个字符串对象底层数据的大小。


## Redis字符串对象的命令实现
### SET命令的实现
*Redis*中对于**SET**命令的描述为：
> Set key to hold the string value. If key already holds a value, it is overwirtten, regardless of its type. Any previous time to live associated with the key is discarded on successful SET operation.

也就是说，这个命令会将*key*设置为指定的字符串*value*，如果这个*key*已经存储了其他的*value*，那么*value*会被覆盖，并且会忽略原始*value*的类型，最后如果**SET**命令执行成功，那么之前关联这个*key*的过期时间将失效。

这个命令格式为:
`SET key value [expiration EX seconds|PX milliseconds] [NX|XX]`
其中：
* **EX**表示为这个*key*设置一个秒级超时的过期时间。
* **PX**表示为这个*key*设置一个毫秒级超时的过期时间。
* **NX**表示只有在*key*不存在时，**SET**命令才会成功。
* **XX**表示只有在*key*存在时候，**SET**命令才会成功。

除了这个**SET**命令之外，*Redis*还给出了三个类似的命令:
1. **SETEX**，`SETEX key seconds value`，用于以秒级过期来对*key*进行设置。
2. **PSETEX**，`PSETEX key milliseconds value`，用于以毫秒级过期来对*key*进行设置。
3. **SETNX**，`SETNX key value`，用于在*key*不存在的情况的下，设置新*key*。

可以发现，上面的这三个命令，都可以通过使用**SET**命令进行实现。

```c
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply);
void setCommand(client *c);
void setnxCommand(client *c);
void setexCommand(client *c);
void psetexCommand(client *c);
```
在代码层面，*Redis*通过上面的五个函数来实现**SET**系列命令的，而`setGenericCommand`函数时实现**SET**的基础，其基础的逻辑为：
1. 如果**SET**指令携带了`NX`参数或者`XX`参数，那么会调用`lookupKeyWrite`对参数`key`进行查找，如果查找结果与参数预期不符，那么将错误返回给客户端。
2. 通过`setKey`函数，将*key-value*写入内存数据库。
3. 如果设置了过期时间，那么通过`setExpire`将过期时间设置给对应的*key*。

### GET命令的实现


这个命令用于返回给定*key*对应的*value*，需要注意的是这个命令只能获取到字符串对象类型的*value*，其命令格式为：`GET key`。
```c
int getGenericCommand(client *c);
void getCommand(client *c);
```
在代码层面，*Redis*通过上面的两个函数实现的**GET**命令，对于**GET**命令的代码实现比较简单，可以分为两个步骤：
1. 通过`lookupKeyReadOrReply`查找给定的*key*是否存在与内存数据库中。
2. 如果找到，通过`robj.type`验证给定*key*对应的*value*是否是字符串对象类型。

### GETSET命令的实现

**GETSET**这个命令用于从*Redis*中查找给定*key*对应的*value*，
```c
void getsetCommand(client *c);
```

### SETRANGE命令的实现
```c
void setrangeCommand(client *c);
```

### GETRANGE命令的实现
```c
void getrangeCommand(client *c);
```

### MSET命令的实现
```c
void msetGenericCommand(client *c, int nx);
void msetCommand(client *c);
void msetnxCommand(client *c);
```

### MGET命令的实现
```c
void mgetCommand(client *c);
```

### INCR和DECR命令的实现
```c
void incrDecrCommand(client *c, long long incr);
void incrCommand(client *c);
void decrCommand(client *c);
void incrbyCommand(client *c);
void decrbyCommand(client *c);
void incrbyfloatCommand(client *c);
```

### APPEND命令的实现
```c
void appendCommand(client *c);
```

### STRLEN命令的实现
```c
void strlenCommand(client *c);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
