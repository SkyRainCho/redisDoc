# Redis中列表对象类型的实现
现在看起，对于*Redis*中存储在内存里的列表对象类型都是使用快速链表作为底层的数据结构实现的，也就是都是使用`OBJ_ENCODING_QUICKLIST`作为编码方式。因为虽然*Redis*定义了`createZiplistObject`函数用于创建一个使用压缩链表进行编码的列表对象的接口函数，但是这个函数却没有其它地方在被调用。

回顾*src/object.c*中的`createQuicklistObject`函数接口：
```c
robj *createQuicklistObject(void)
{
	quicklist *l = quicklistCreate();
	robj *o = createObject(OBJ_LIST, l);
	o->encoding = OBJ_ENCODING_QUICKLIST;
	return o;
}
```
创建一个列表对象时，首先会创建一个快速链表，然后再将快速链表的指针赋值给`robj.ptr`字段，最后再设定劣币对象的编码字段`robj.encoding`。

同时在*src/server.h*这个头文件还定义了关于列表对象的两个数据结构：
```c
typedef struct {
	robj *subject;
	unsigned char encoding;
	unsigned char direction; /* Iteration direction */
	quicklistIter *iter;
} listTypeIterator;

typedef struct {
	listTypeIterator *li;
	quicklistEntry entry; /* Entry in quicklist */
} listTypeEntry;
```
上述两个数据结构可以被认为是对应快速链表中的`quicklistIter`和`quicklistEntry`这两个数据结构，分别是列表对象的迭代器以及在遍历中用于表示列表对象中的数据节点。

还有一点需要注意的是，*Redis*中的列表对象只能用于以链表的形式存储一组字符串对象，*Redis*不支持嵌套的列表对象，也就是说在*Redis*之中，无法创建一个存储列表对象的列表对象。

对于列表对象的操作除了创建与释放操作被定义在*src/object.c*文件中之外，其余的内容都被定义在*src/t_list.c*源文件中，其中主要包含两个部分：
1. 列表对象基础操作定义。
2. 列表对象命令实现。

## 列表对象基础操作

### 列表对象的PUSH操作
`listTypePush`这个函数用于向一个指定的列表对象`subject`的头部或者尾部位置插入一个新的数据`value`。
```c
void listTypePush(robj *subject, robj *value, int where)
{
	...
	value = getDecodedObject(value);
	size_t len = sdslen(value->ptr);
	quicklistPush(subject->ptr, value->ptr, len, pos);
	decrRefCount(value);
	...
}
```
通过`listTypePush`函数的代码片段，我们可以了解到一些列表对象在执行**PUSH**操作时的一些细节：
1. **PUSH**操作的`value`只能是字符串类型对象，因为内部调用`getDecodedObject`这个函数只能操作字符串类型对象，这也正好印证了前面提到的，无法创建嵌套存储列表对象的列表对象这个性质。
2. 被插入列表对象的数据实际上是`value`数据的拷贝，而不是直接将数据插入到列表中，这一点可以通过`quicklistPush`这个函数的实现细节了解到。

### 列表对象的POP操作
`listTypePop`这个函数用于从给定的列表对象`subject`的头部或者尾部弹出一个数据节点，并将弹出的数据通过函数返回。
```c
void *listPopSaver(unsigned char *data, unsigned int sz);
robj *listTypePop(robj *subject, int where);
```
`listTypePop`这个函数通过调用`quicklistPopCustom`将数据从底层的快速链表之中弹出，同时通过`listPopSaver`这个辅助函数，使用弹出的数据创建一个字符串对象，并将这个对象通过函数进行返回。

### 列表对象迭代与遍历
```c
listTypeIterator *listTypeInitIterator(robj *subject, long index, unsigned char direction);
void listTypeReleaseIterator(listTypeIterator *li);
int listTypeNext(listTypeIterator *li, listTypeEntry *entry);
```

### 列表对象的插入与删除
```c
void listTypeInsert(listTypeEntry *entry, robj *value, int where);
void listTypeDelete(listTypeIterator *iter, listTypeEntry *entry);
```

### 列表对象其他操作
```c
unsigned long listTypeLength(const robj *subject);
robj *listTypeGet(listTypeEntry *entry);
int listTypeEqual(listTypeEntry *entry, robj *o);
void listTypeConvert(robj *subject, int enc);
```
其中：
1. `listTypeLength`通过`quicklistCount`来获取给定的列表对象`subject`的**数据节点**的个数。
2. `listTypeGet`用于从`listTypeEntry`对应的**数据节点**通过调用`createStringObject`或者`createStringObjectFromLongLong`来获取一个新的字符串对象。
3. `listTypeEqual`用于判断`listTypeEntry`对应的**数据节点**与`robj`对象在内容上是否相等。
4. `listTypeConvert`用于对列表对象进行类型转换，只能够从`OBJ_ENCODING_ZIPLIST`编码转换为`OBJ_ENCODING_QUICKLIST`编码，通过`quicklistCreateFromZiplist`这个函数实现。

## 列表对象命令
### 列表对象的PUSH系列命令
#### PUSH命令
```c
void pushGenericCommand(client *c, int where);
void lpushCommand(client *c);
void rpushCommand(client *c);
```

#### PUSHX命令
```c
void pushxGenericCommand(client *c, int where);
void lpushxCommand(client *c);
void rpushxCommand(client *c);
```

#### LINSERT命令
```c
void linsertCommand(client *c);
```

### 列表对象的POP系列命令
#### POP命令
```c
void popGenericCommand(client *c, int where);
void lpopCommand(client *c);
void rpopCommand(client *c);
```

### 列表对象的RPOPLPUSH系列命令
```c
void rpoplpushHandlePush(client *c, robj *dstkey, robj *dstobj, robj *value);
void rpoplpushCommand(client *c);
```

### 列表对象的阻塞POP系列命令
```c
int serveClientBlockedOnList(client *receiver, robj *key, robj *dstkey, redisDb *db, robj *value, int where);
void blockingPopGenericCommand(client *c, int where);
void blpopCommand(client *c);
void brpopCommand(client *c);
void brpoplpushCommand(client *c);
```

### 列表对象的删除系列命令
#### LTRIM命令
```c
void ltrimCommand(client *c);
```

#### LREM命令
```c
void lremCommand(client *c);
```

### 列表对象的RANGE命令
#### LRANGE命令
```c
void lrangeCommand(client *c);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
