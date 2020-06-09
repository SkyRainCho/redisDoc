# Redis中散列对象类型实现
通过*src/object.c*文件中的代码，我们看到散列对象只有一个默认的构造函数`createHashObject`，这个函数会使用压缩链表作为底层数据结构，这也就是意味着初始的散列对象都是使用压缩链表，当满足一定条件之后，会从压缩链表转化为`dict`作为底层数据结构。

*Redis*中对于散列对象的操作函数主要被定义在*src/t_hash.c*这个文件中。另外在*src/server.h*头文件中，还定义了一个更高一级的散列对象迭代器，用于包装压缩链表迭代器以及哈希表迭代器：
```c
typedef struct {
	robj *subject;
	int encoding;
	unsigned char *fptr, *vptr;
	dictIterator *di;
	dictEntry *de;
} hashTypeIterator;
```
在这个散列对象迭代器的结构提中：
1. `hashTypeIterator.subject`以及`hashTypeIterator.encoding`两个字段用于表示当前迭代器绑定的*Redis*对象以及对象所使用的编码类型。
2. `hashTypeIterator.fptr`以及`hashTypeIterator.vptr`两个字段用于在使用`OBJ_ENCODING_ZIPLIST`编码时的底层迭代器。
3. `hashTypeIterator.di`以及`hashTypeIterator.de`两个字段用在使用`OBJ_ENCODING_HT`编码时的底层迭代器。

## 散列对象基础操作
### 散列数据底层数据结构转化
对于*Redis*中，使用`OBJ_ENCODING_ZIPLIST`编码的散列对象在满足下面两个条件之一的时候就会转化成`OBJ_ENCODING_HT`编码的形式：
1. 如果散列表中元素的数量超过`#define OBJ_HASH_MAX_ZIPLIST_ENTRIES 512`，便会转化为`OBJ_ENCODING_HT`编码的形式。
2. 如果散列表中元素的*key*或者*value*的长度超过`#define OBJ_HASH_MAX_ZIPLIST_VALUE 64`，便会转化为`OBJ_ENCODING_HT`编码的形式。

当*Redis*使用`OBJ_ENCODING_ZIPLIST`对散列对象进行编码的时候，数据以`<key>-<value>-<key>-<value>`这样的方式顺序地存储在压缩链表的所占用的内存中。

*Redis*会使用下面的三个函数来实现散列对象的转换
```c
void hashTypeConvertZiplist(robj *o, int enc);
void hashTypeConvert(robj *o, int enc);
void hashTypeTryConversion(robj *o, robj **argv, int start, int end);
```
其中`hashTypeConvertZiplist`函数是转换的基础，这个函数会调用`dictCreate`函数创建一个新的哈希表，遍历原始的散列，将从中取出的*key*以及*value*将其插入到新创建的哈希表中，最后在底层使用这个哈希表来替换原有的压缩链表。

`hashTypeConvert`函数实际上是对`hashTypeConvertZiplist`的封装，其常用的逻辑是在向散列中插入元素后，会检查散列对象的长度，如果超过了`OBJ_HASH_MAX_ZIPLIST_ENTRIES`这个阈值，那么就会调用`hashTypeConvert`函数，对散列对象进行转换。

`hashTypeTryConversion`需要特殊说明一下，它的含义是，如果`argv`参数列表中从`start`开始到`end`为止的参数里存在长度大于`OBJ_HASH_MAX_ZIPLIST_VALUE`的情况，那么对散列对象`o`调用`hashTypeConvert`进行转换，看似`argv`参数列表与散列对象`o`没有什么关联，为什么对`argv`的检查会导致对象`o`的编码方式被转换？考虑下面一种情况，当我们使用**HSET**命令对散列进行设置的时候，我们可以对客户端传来的参数进行校验，如果客户端参数存在长度超过`OBJ_HASH_MAX_ZIPLIST_VALUE`的情况，那么预先对*key*对应的散列对象执行`hashTypeConvert`，然后在执行设置操作。而这种前置的转换操作，对比先设置再校验的方式，效率要更好，可以校验更少的数据，同时转化更少的数据；同时前置的转化可以避免我们过早地向散列对象的底层压缩链表中插入数据，因为涉及到内存的重新分配，而无效的重新分配内存，势必会降低系统的性能。

### 散列数据查询操作
```c
sds hashTypeGetFromHashTable(robj *o, sds field);
int hashTypeGetFromZiplist(robj *o, sds field, unsigned char **vstr, unsigned int *vlen, long long *vll);
```
上面的两个函数分别实现了在`OBJ_ENCODING_ZIPLIST`编码下以及`OBJ_ENCODING_HT`编码下给定*key*用来查找对应的*value*的功能：
1. 对于`OBJ_ENCODING_ZIPLIST`编码格式，`hashTypeGetFromZiplist`会调用`ziplistFind`函数针对底层压缩链表对`field`执行一次顺序查找。如果查找成功，那么`ziplistNext`函数会返回对应*value*的指针，最后通过`ziplistGet`获得*value*数据的具体信息。
2. 对于`OBJ_ENCODING_HT`编码格式，`hashTypeGetFromHashTable`会调用`dictFind`在底层哈希表中进行查找，如果找到，在通过`dictGetVal`获取数据的具体内容。

```c
int hashTypeGetValue(robj *o, sds field, unsigned char **vstr, unsigned int *vlen, long long *vll);
robj *hashTypeGetValueObject(robj *o, sds field);
```
上述两个函数是对散列对象查找操作的更高一层的函数定义，其中`hashTypeGetValue`函数会根据散列对象的具体编码方式，调用`hashTypeGetFromHashTable`或者`hashTypeGetFromZiplist`对给定的`field`进行查找，如果对应的*value*是一个字符串数据，那么结果会通过`vstr`以及`vlen`进行返回；如果对应的*value*是整数数据，那么会通过`vll`进行返回。而函数`hashTypeGetValueObject`与`hashTypeGetValue`的功能类似，不过其查找的结果是会封装成一个*Redis*对象进行返回，而不是返回数据的原始指针。

```c
size_t hashTypeGetValueLength(robj *o, sds field);
```
*Redis*会通过`hashTypeGetValueLength`这个函数获取给定`field`对应的*value*的长度。

```c
int hashTypeExists(robj *o, sds field);
```
`hashTypeExists`这个函数用于判断一个给定的`field`是否存在于散列对象中。

### 散列数据插入操作
```c
int hashTypeSet(robj *o, sds field, sds value, int flags)
{
	if (o->encoding == OBJ_ENCODING_ZIPLIST) {
		fptr = ziplistIndex(zl, ZIPLIST_HEAD);
		if (fptr != NULL) {
			fptr = ziplistFind(fptr, (unsigned char*)field, sdslen(field), 1);
			if (fptr != NULL) {
				vptr = ziplistNext(zl, fptr);
				update = 1;
				zl = ziplistDelete(zl, &vptr);
				zl = ziplistInsert(zl, vptr, (unsigned char*)value, sdslen(value));
			}
		}
		if (!update) {
			zl = ziplistPush(zl, (unsigned char*)field, sdslen(field), ZIPLIST_TAIL);
			zl = ziplistPush(zl, (unsigned char*)value, sdslen(value), ZIPLIST_TAIL);
		}
		o->ptr = zl;
		if (hashTypeLength(o) > server.hash_max_ziplist_entries)
			hashTypeConvert(o, OBJ_ENCODING_HT);
	} else if (o->encoding == OBJ_ENCODING_HT) {
		dictEntry *de = dictFind(o->ptr,field);
		if (de) {
			sdsfree(dictGetVal(de));
			if (flags & HASH_SET_TAKE_VALUE) {
				dictGetVal(de) = value;
				value = NULL;
			} else {
				dictGetVal(de) = sdsdup(value);
			}
		} else {
			...
			dictAdd(o->ptr,f,v);
		}
	}
}
```
对于散列对象的设置操作，*Redis*遵循这样的原则，如果`field`存在，那么会更新覆盖对应的`value`；如果`field`不存在，那么将这个新的*key-value*对插入散列对象中。从上面的代码片段中，我们可以看到：
1. 如果是一个`OBJ_ENCODING_ZIPLIST`编码方式的散列对象，首先会对给定的`field`进行查找，如果找到，则调用`ziplistDelete`将对应的*value*删除，然后通过`ziplistInsert`将新的`value`插入到`field`后面；如果`field`没有找到，那么调用两次`ziplistPush`以此将`field`以及`value`插入到底层压缩链表之中。
2. 如果是一个`OBJ_ENCODING_HT`编码方式的散列对象，调用`dictFind`对`field`进行查找，如果查找成功，那么调用`dictGetVal`宏对*value*进行覆盖，否则调用`dictAdd`将新的*key-value*加入到散列对象中。

### 散列数据删除操作
```c
int hashTypeDelete(robj *o, sds field);
```
由`hashTypeDelete`定义的散列数据删除操作与前面的`hashTypeSet`类似，会根据编码类型，如果是`OBJ_ENCODING_ZIPLIST`编码方式，在查找到`field`之后，连续调用两次`ziplistDelete`函数，将*key-value*从压缩链表中删除；如果是`OBJ_ENCODING_HT`编码方式，会调用`dictDelete`函数，将*key-value*从哈希表中删除。

### 散列数据长度获取
*Redis*通过下面这个函数来实现散列对象的长度获取。
```c
unsigned long hashTypeLength(const robj *o);
```
其实现逻辑为：
1. 如果对于`OBJ_ENCODING_ZIPLIST`编码，长度为`ziplistLen`结果的二分之一，以为*key-value*是按照顺序的方式存储在压缩链表中的。
2. 如果对于`OBJ_ENCODING_HT`编码，则会通过`dictSize`接口来获取长度。

### 散列数据迭代与遍历
```c
hashTypeIterator *hashTypeInitIterator(robj *subject);
void hashTypeReleaseIterator(hashTypeIterator *hi);
int hashTypeNext(hashTypeIterator *hi);
```
上述的三个函数分别用于散列对象迭代器的初始化、释放与遍历操作，其底层会根据编码类型，分别调用压缩链表或者哈希表的迭代器相关操作来实现。

而下面这四个函数都是用于从散列对象迭代器中获取数据的操作，这四个函数都有一个参数`what`，这表明需要获取数据的类型，对此*Redis*给出了两个定义：
1. `#define OBJ_HASH_KEY 1`
2. `#define OBJ_HASH_VALUE 2`

这两个定义表示希望从迭代器中获取*key*或者*value*。
```c
void hashTypeCurrentFromZiplist(hashTypeIterator *hi, int what, unsigned char **vstr, unsigned int *vlen, long long *vll);
sds hashTypeCurrentFromHashTable(hashTypeIterator *hi, int what);
void hashTypeCurrentObject(hashTypeIterator *hi, int what, unsigned char **vstr, unsigned int *vlen, long long *vll);
sds hashTypeCurrentObjectNewSds(hashTypeIterator *hi, int what);
```
在这四个函数里，`hashTypeCurrentFromZiplist`和`hashTypeCurrentFromHashTable`函数分别对应从`OBJ_ENCODING_ZIPLIST`以及`OBJ_ENCODING_HT`编码的散列对象中，按照参数`what`获取*key*或者*value*，并通过指针返回；而`hashTypeCurrentObject`作为一个更加通用的接口，会根据对象的编码类型来调用`hashTypeCurrentFromZiplist`和`hashTypeCurrentFromHashTable`函数；最后一个函数`hashTypeCurrentObjectNewSds`功能与`hashTypeCurrentObject`相似，但是其获取到的*key*或者*value*使用一个新创建的`sds`数据来进行返回，可以理解为返回了*key*或者*value*的拷贝。

## 散列对象命令实现

在执行*Redis*散列的系列命令前，这里首先定义了一个辅助函数，`hashTypeLookupWriteOrCreate`，其函数原型为：

```c
robj *hashTypeLookupWriteOrCreate(client *c, robj *key);
```

应用这个函数，会在*Redis*的内存数据库中调用`lookupKeyWrite`查找给定`key`所对应的散列对象。如果这个对象不存在，那么会使用`createHashObject`创建一个新的散列对象，并通过`dbAdd`将其加入到内存数据库之中；如果对象存在但不是散列类型，那么将错误信息返回给客户端；否则便将这个散列对象从函数中返回。

### HSET系列命令

**HSET**命令的格式为：`HSET key field value [field value ...]`

这个命令用于设定*key*所存储的散列对象中执行字段*field*的值，如果散列对象不存在，那么会向*Redis*内存数据库中存入一个新建的散列对象；如果*field*在散列中存在，那么它的值将会被覆盖。

**HSETNX**命令的格式为：`HSETNX key field value`

这个命令的行为与**HSET**命令相似，唯一的区别是，如果*field*存在于散列中，那么设定命令操作无效。
```c
void hsetCommand(client *c);
void hsetnxCommand(client *c);
```

上述两个便是用来实现**HSET**命令以及**HSETNX**命令的函数原型，其基础的逻辑都是通过调用`hashTypeSet`这个接口函数来进行*key-value*的设置，区别在于`hsetCommand`会循环调用`hashTypeSet`以实现命令设置多个*key-value*的目的；而`hsetnxCommand`函数在调用`hashTypeSet`前，会先调用`hashTypeExists`函数来检查*field*是否存在于散列对象之中。

### HGET系列命令

**HGET**命令的格式为：`HGET key field`
这个命令用于返回*key*所指定的散列对象中*field*字段所关联的值，如果散列对象不存在或者*field*不存在的时候，命令会返回`nil`空值。
**HMEGET**命令的格式为：`HMGET key field [field ...]`
这个命令用于返回*key*所制定的散列对象中*field*列表所对应的值的列表，如果其中某一个*field*不存在，那么返回的值列表中对应的位置为`nil`值。
**HGETALL**命令的格式为：`HGETALL key`
这个命令用于返回*key*所对应的散列对象中所有*key-value*所组成的列表。
**HKEYS**命令的格式为：`HKEYS key`
这个命令用于返回*key*所对应的散列对象中所有键所组成的列表。
**HVALS**命令的格式为：`HVALS key`
这个命令用于返回*key*所对应的散列对象中所有值所组成的列表。
```c
void hgetCommand(client *c);
void hmgetCommand(client *c);
void genericHgetallCommand(client *c, int flags);
void hgetallCommand(client *c);
void hkeysCommand(client *c);
void hvalsCommand(client *c);
```

### HINCRBY系列命令
*Redis*为散列对象也提供了针对某个字段的数值*value*的加减操作的命令，**HINCRBY**和**HINCRBYFLOAT**这两个命令，这两个命令的格式为：
`HINCRBY key field increment`
`HINCRBYFLOAT key field increment`
这两个命令用于给*key*所指定的散列对象中的*field*字段所对应的值上加上一个指定的数值*increment*，如果这个值不存在，那么会为其先设定一个0的初始值，然后在执行加减操作。
```c
void hincrbyCommand(client *c);
void hincrbyfloatCommand(client *c);
```

上述这两个函数用于实现这组**INCRBY**命令，其基础逻辑是统一的，也就是使用`hashTypeGetValue`函数在散列对象中查找到对应*field*的值，然后对其执行完加减操作后，在调用`hashTypeSet`函数将新值写回散列对象之中。

### HDEL命令

*Redis*为散列对象提供了一个删除给定*field*的命令**HDEL**，这个命令的格式为：

`HDEL key field [field ...]`

这个命令用于删除指定*key*的散列对象中的给定的*field*，命令直接结束后，会返回被成功删除的元素的个数。

```c
void hdelCommand(client *c);
```

*Redis*在**HDEL**命令的实现函数`hdelCommand`中，通过循环调用`hashTypeDelete`函数来将*field*从散列对象中删除，最后如果散列被删空，那么会通过`dbDelete`函数将这个对象从内存数据库中删除。

### HLEN命令

**HLEN**命令用于获取给定*key*的散列对象中元素的个数，如果这个对应的散列对象不存在的时候，会被认为是一个空的散列对象而返回0，这个命令的格式为：`HLEN key`

```c
void hlenCommand(client *c);
```

`hlenCommand`函数，是通过`hashTypeLength`这个接口来获取散列对象的长度，借以实现**HLEN**命令的功能。

### HSTRLEN命令

**HSTRLEN**命令的格式为：`HSTRLEN key field`，这个命令用于获取给定*key*对应的散列对象之中，*ffield*所对应的*value*数据的长度，如果散列对象不存在或者*field*字段不存在，那么这个命令或返回0。

```c
void hstrlenCommand(client *c);
```

`hstrlenCommand`函数通过调用散列对象的`hashTypeGetValueLength`接口来实现**HSTRLEN**命令的逻辑。

### HEXISTS命令

**HEXISTS**这个命令的格式为：`HEXISTS key field`，使用这个命令，我们可以知道给定的*field*是否存在于*key*所对应的散列对象中，如果*field*存在，那么这个命令会返回1，否则命令会返回0。

```c
void hexistsCommand(client *c);
```

很显然，这个命令函数在底层使用的是散列对象的`hashTypeExists`的接口来实现的。

### HSCAN命令

**HSCAN**命令的格式为：`HSCAN key cursor [MATCH pattern] [COUNT count]`

这个命令于前面我们介绍哈希表时所介绍的**SCAN**命令类似，只不过**HSCAN**命令用于扫描给定*key*所对应的散列对象，而**SCAN**命令则是对整个*Redis*的内存数据库进行扫描。

```c
void hscanCommand(client *c);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
