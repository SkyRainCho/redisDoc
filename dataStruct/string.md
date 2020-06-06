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
1. 只有`OBJ_ENCODING_RAW`编码以及`OBJ_ENCODING_EMBSTR`编码的字符串才有优化内存空间的必要，这一点可以通过调用`sdsEncodedObject`函数来进行验证。
2. 对于共享对象的内存优化是不安全的，因为这个对象可能在整个*Redis*的内存空间的各个地方被引用，甚至有可能在其他地方正在处理中。对象是否被共享，可以通过`robj.refcount`字段来进行判断。
3. 检查该字符串是否可以表示一个长整形数，可以通过`len <= 20 && string2l(s, len, &value)`这个条件进行判断：
	1. 如果解析出来整数处于*Redis*的共享整数对象`shared.integers`范围内，那么直接返回系统共享对象。
	2. 如果解析出的整数数值不在共享整数对象：
		1. 如果`robj.encoding`为`OBJ_ENCODING_RAW`，则释放底层`sds`数据，将整数保存在`robj.ptr`指针上。
		2. 如果`robj.encoding`为`OBJ_ENCODING_EMBSTR`，那么这种情况则需要释放整个对象，然后调用`createStringObjectFromLongLongForValue`来创建一个新的`OBJ_ENCODING_INT`编码的字符串对象。
4. 如果没法编码成`OBJ_ENCODING_INT`对象，则会检查`sds`数据的长度，如果小于`OBJ_ENCODING_EMBSTR_SIZE_LIMIT`，那么通过`createEmbeddedStringObject`函数将对象转化为`OBJ_ENCODING_EMBSTR`对象。
5. 最后通过`trimStringObjectIfNeeded`函数，尝试对底层`sds`数据所分配的内存进行进一步的优化。

#### 字符串对象的其他操作
```c
robj *getDecodedObject(robj *o);
```
`getDecodeObject`这个函数只对字符串对象生效，会从传入参数所对应的对象返回一个新的对象，如果原始对象是一个字符串类型对象，那么会增加其引用计数，并将这个原始对象返回；当原始对象是一个整数型对象时，则会将这个整数转化为一个字符串形式，然后为其创建一个字符串类型的新对象。

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

### MSET命令的实现
这一系列命令有**MSET**以及**MSETNX**两个命令，用于批量地为一组*keys*设定它们的*values*，这系列命令具有原子性，也就是说所有的*key*是一次性被设置的，因此对于客户端来说，不会存在一部分*key*被设置了，而另外一部分的数据没有被设置的情况。当需要批量地对某些*key*进行设置的时候，应该选择使用这个**MSET**命令，这比客户端循环地执行**SET**命令效率更高。**MSET**命令与**MSENX**命令的区别在于，**MSETNX**命令所涉及的*key*不能是已存在的*key*。

这两个命令的格式为：
* **MSET**，`MSET key value [key value ...]`
* **MSETNX**，`MSETNX key value [key value ...]`

```c
void msetGenericCommand(client *c, int nx);
void msetCommand(client *c);
void msetnxCommand(client *c);
```
上述的三个函数用于实现*Redis*的批量设置命令，通过`msetGenericCommand`这个基础操作，我们可以了解命令具体的实现逻辑：
```c
void msetGenericCommand(client *c, int nx) {
	//如果设置了nx选项，那么在执行set之前，需要先验证所有的key都不存在
	if (nx) {
		for (j = 1; j < c->argc; j += 2) {
			if (lookupKeyWrite(c->db, c->argv[j]) != NULL) {
				return;
			}
		}
	}
	//通过for循环对所有的key进行设置操作
	for (j = 1; j < c->argc; j += 2) {
		setKey(c->db, c->argv[j], c->argv[j + 1]);
	}
}
```

### GET命令的实现
*Redis*对于**GET**命令的描述为：
> Get the value of key. If the key does not exits the special value nil is returned. An error is rerturned if the value stored at key is not a string, because GET only handles string values.

这个命令用于返回给定*key*对应的*value*，需要注意的是这个命令只能获取到字符串对象类型的*value*。

命令格式为：`GET key`
```c
int getGenericCommand(client *c);
void getCommand(client *c);
```
在代码层面，*Redis*通过上面的两个函数实现的**GET**命令，对于**GET**命令的代码实现比较简单，可以分为两个步骤：
1. 通过`lookupKeyReadOrReply`查找给定的*key*是否存在与内存数据库中。
2. 如果找到，通过`robj.type`验证给定*key*对应的*value*是否是字符串对象类型。

### MGET命令的实现
**MGET**命令用于获取一组给定的*key*所对应的*value*，这个命令只对字符串数据类型生效，对于每个不存在的*key*或者不是字符串类型的*value*，都会返回特殊的`nil`值。

这个命令的格式：`MGET key [key ...]`

```c
void mgetCommand(client *c);
```
`mgetCommand`函数的实现，可以结合**GET**命令以及**MSET**命令一起进行理解。

### GETSET命令的实现
*Redis*对于**GETSET**命令的描述为：

> Atomically sets key to value and returns the old value stored at key. Returns an error when key exists but does not hold a atring value.

**GETSET**这个命令用于自动将*key*对应到*value*并且返回原来*key*对应的*value*。如果*key*存在但是对应的*value*不是字符串，就返回错误。

```c
void getsetCommand(client *c);
```
*Redis*使用上面这个`getsetCommand`函数来实现**GETSET**命令，这个函数首先会调用`getGenericCommand`函数，相当于执行一次**GET**指令，将这个*key*对应的*value*返回给客户端，最后在调用`setKey`函数为*key*设定新的*value*值。

### SETRANGE命令的实现
**SETRANGE**命令用于覆盖*key*所对应的字符串从给定的*offset*开始的一部分内容。如果修改成功，那么这个命令会返回修改后的字符串对象的长度。如果*offset*比当前*key*对应的字符串还要长的话，那么会在这个字符串对象后面追加0以达到*offset*的长度。如果给定的*key*不存在的话，那么相当于执行一次**SET**命令操作，为这个*key*设定了*value*。当然这个命令会限定修改后的数据不能超过*Redis*对字符串数据对象512MB字节的限制。

这个命令的格式为：`SETRANGE key offset value`
```c
void setrangeCommand(client *c);
```

*Redis*使用`setrangeCommand`这个函数来实现**SETRANGE**这个命令，这个函数的逻辑可以用下面的逻辑来简述：
1. 调用`lookupKeyWrite`函数来查找*key*所对应的对象
2. 如果对象不存在的话，那么会尝试创建一个新的字符串对象，其对象长度为`offset+sdslen(value)`
	1. *Redis*会通过调用`checkStringLength`来检查长度有没有超过512MB字节的限制。
	2. 如果长度合法，那么会通过`createObject`创建一个长度为`offset+sdslen(value)`的字符串对象，并调用`dbAdd`将其加入内存数据库之中。
3. 如果对象存在的话:
	1. 由于**SETRANGE**只对字符串对象有效，因此需要调用`checkType`来检查这个已经存在的对象是否是字符串对象。
	2. 同样也会通过`checkStringLength`来检查长度是否超过限制。
	3. 调用`dbUnshareStringValue`函数，如果这个已存在的对象是一个共享对象，那么将其从共享中解绑，基于*CopyOnWrite*原则，创建一个对象的拷贝。
4. 通过上面2、3步之后，现在一定存在一个对应的字符串对象，会调用`sdsgrowzero`尝试对`sds`数据进行扩展。
5. 通过`memcpy`来将数据拷贝到`sds`数据之中。


### GETRANGE命令的实现

命令的格式为：`GETRANGE key start end`

**GETRANGE**命令用于获取一个*key*所对应的字符串的子串，这个子串是由*start*以及*end*来决定的，*start*以及*end*参数都可以使用负数来实现从后先前地截取子串。
```c
void getrangeCommand(client *c);
```

### INCR和DECR命令的实现
*Redis*为数值型字符串对象提供了一组操作，可以允许客户端以原子操作的方式对一个数值型字符串对象增加某个数值，或者减去某个数值，如果这个*key*在内存数据库之中不存在，那么会为这个*key*创建一个值为0的数据，然后在执行后续的加减操作：
|命令|描述|格式|
|----|----|----|
|**INCR**|对存储在指定*key*的数值执行原子的加1操作|`INCR key`|
|**DECR**|对存储在指定*key*的数值执行原子的减1操作|`INCRBY key increment`|
|**INCRBY**|对存储在指定*key*的数值执行原子的加上给定数值的操作|`DECR key`|
|**DERBY**|对存储在指定*key*的数值执行原子的减去给定数值的操作|`DECRBY key decrement`|
|**INCRBYFLOAT**|对存储在指定*key*的数值执行原子的加上给定浮点数值的操作|`INCRBYFLOAT key increment`|

```c
void incrDecrCommand(client *c, long long incr);
void incrCommand(client *c);
void decrCommand(client *c);
void incrbyCommand(client *c);
void decrbyCommand(client *c);
void incrbyfloatCommand(client *c);
```
在*Redis*实现上述五个命令的代码中，`incrDecrCommand`是四个整数加减基础实现逻辑，`incrbyfloatCommand`则是浮点数加减操作的基础逻辑，二者存在一些差异，但是本质上的逻辑是大致相同的，那么以`incrDecrCommand`为例，其具体实现为：
```c
void incrDecrCommand(client *c, long long incr) 
{
	...
	//根据给定的key，搜索对应的value对象o，此处可以查找不到
	o = lookupKeyWrite(c->db, c->argv[1]);
	//如果查找到o，那么检查对象是否是字符串对象
	if (o != NULL && checkType(c, o, OBJ_STRING)) return;
	//从对象o中解码出对应的数值，存储在value变量中，如果对象o为空，那么value会被赋值0
	if (getLongLongFromObjectOrReply(c, o, &value, NULL) != C_OK) return;
	...
	value += incr;

	if (o && o->refcount == 1 && o->encoding == OBJ_ENCODING_INT &&
		(value < 0 || value >= OBJ_SHARED_INTEGERS) && value >= LONG_MIN &&
		value <= LONG_MAX) {
		/* 如果对象o存在，并且没有被其他对象共享，同时在long长度的表示范围内，
		 * 那么直接将新的数值存储在robj.ptr的指针位上
		 */
		new = o;
		o->ptr = (void *)((long)value);
	} else {
		//不满足上面的情况，则需要位这个新的数值创建一个新的字符串对象
		new = createStringObjectFromLongLongForValue(value);
		if (o) {
			dbOverwrite(c->db, c->argv[1], new);
		} else {
			dbAdd(c->db, c->argv[1], new);
		}
	}
	...
}
```

### APPEND命令的实现
**APPEND**命令格式为：`APPEND key value`

这个命令用于将*value*的内容追加到给定*key*存储的内容之后。这个追加操作只对字符串类型的数据有效，如果给定的*key*不存在，那么会将*key*以及*value*作为新的数据存储在内存数据库中；如果*key*存在，那么还需要检查追加数据之后，是否会超过*Redis*对字符串对象512MB的长度限制。
```c
void appendCommand(client *c);
```
上面这个函数便是**APPEND**命令的具体实现，从本质上来说，这个追加操作可以看作是一种特殊的**SETRANGE**命令，相当于`offset`是字符串对象的长度，因此可以通过`setrangeCommand`来了解这个追加命令的实现细节。

### STRLEN命令的实现
**STRLEN**命令格式为：`STRLEN key`

这个命令用于获取给定*key*所存储的字符串类型*value*的长度。
```c
void strlenCommand(client *c);
```
函数`strlenCommand`用于实现这个**STRLEN**命令，该命令首先会在内存数据库中调用`lookupKeyReadOrReply`来查找给定的*key*，然后在调用`checkType`验证对应的*value*是否为字符串类型数据，最后通过`stringObjectLen`函数获取字符串对象长度。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
