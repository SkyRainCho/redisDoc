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
上述两个数据结构可以被认为是对应快速链表中的`quicklistIter`和`quicklistEntry`这两个数据结构，分别是列表对象的迭代器以及在遍历中用于表示列表对象中的数据节点，这其中各个字段的意义为：

1. `listTypeIterator.subject`，这个对象指针用于标记当前的迭代器属于哪个列表对象。
2. `listTypeIterator.encoding`，这个字段用于标记其所使用的对象编码类型。
3. `listTypeIterator.direction`，这个字段用于标记迭代器遍历的方向，对于迭代器的方向，*Redis*给出了两个定义：
   1. `#define LIST_HEAD 0`
   2. `#define LIST_TAIL 1`
4. `listTypeIterator.iter`，这个字段用于表示底层快速链表所对应的迭代器。

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

对于列表对象迭代器的初始化是通过`listTypeInitIterator`这个函数来实现的，这个函数会创建一个`listTypeIterator`对象，并通过调用快速链表的`quicklistGetIteratorAtIdx`函数获取到底层快速链表的迭代器，并将其赋值给`listTypeIterator.iter`字段。

因为迭代器也是使用`zmalloc`函数创建的，因此当一个列表对象迭代器不在使用的时候，需要调用`listTypeReleaseIterator`函数将其释放。

最后*Redis*会通过`listTypeNext`函数，实现列表对象迭代器的迭代过程，这个函数内部通过调用`quicklistNext`函数将迭代器当前指向的数据节点存储在给定的`listTypeEntry`之中，同时对迭代器进行迭代，将其指向下一个元素。

### 列表对象的插入与删除

```c
void listTypeInsert(listTypeEntry *entry, robj *value, int where);
void listTypeDelete(listTypeIterator *iter, listTypeEntry *entry);
```

上述两个函数都是通过调用快速链表的相关接口来实现对列表对象的插入与删除操作做。具体是通过`quicklistInsertAfter`以及`quicklistInsertBefore`来实现`listTypeInsert`；使用`quicklistDelEntry`来实现`listTypeDelete`。

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

*Redis*对于列表对象提供了**LPUSH**命令以及**RPUSH**命令，用于将给定的值插入到存储于*key*中的列表头部或者尾部，如果*key*不存在，那么在进行**PUSH**命令时，会创建一个空列表。这两个命令的格式为：

- **LPUSH**，`LPUSH key value [value ...]`
- **RPUSH**，`RPUSH key value [value ...]`

我们可以看到，这两个命令都可以向列表对象中插入多个值，当插入多个值的时候，不论是**LPUSH**还是**RPUSH**命令，值的插入顺序等价于按照*value*的次序，多次执行**PUSH**命令的插入效果。

```c
void pushGenericCommand(client *c, int where);
void lpushCommand(client *c);
void rpushCommand(client *c);
```

上面定义的三个函数是*Redis*用于实现**PUSH**系列命令的函数，`pushGenericCommand`是这三个单数的基础，另外两个函数是在`pushGenericCommand`基础上进行的简单封装。

`pushGenericCommand`这个函数首先会调用`lookupKeyWrite`从*Redis*的内存数据库之中查找到给定*key*所对应的列表对象。如果列表不存在，那么会创建一个空的列表，通过`dbAdd`将其加入到*Redis*的内存数据库中，最后在循环调用前面我们讲解过的`listTypePush`函数，将参数中的一个或者多个*value*压入列表对象之中。

#### PUSHX命令

同时*Reids*还提供了**LPUSHX**命令与**RPUSHX**命令，这两个命令与上面的**LPUSH**命令与**RPUSH**命令的区别在于，**LPUSHX**与**RPUSHX**命令要求*key*所对应的列表对象必须存在于*Redis*的内存数据库之中，同时这两个命令只能一次插入一个值，无法进行数据的批量插入。

这两个命令的格式为：

- **LPUSHX**，`RPUSHX key value`
- **RPUSHX**，`RPUSHX key value`

```c
void pushxGenericCommand(client *c, int where);
void lpushxCommand(client *c);
void rpushxCommand(client *c);
```

上面者三个函数是*Redis*用于实现**PUSHX**系列命令，其本质上与上面的**PUSH**类似，唯一的区别是在查找*key*所对应的列表对象的时候，会使用`lookupKeyOrReply`函数来查找，如果找不到，将错误信息返回给客户端。

#### LINSERT命令

除了上述可以在列表两端执行插入数据的命令之外，*Redis*还提供了一个可以在给定位置执行插入的命令**LINSERT**，这个命令的格式为：

`LINSERT key BEFORE|AFTER pivot value`

应用这个命令，可以把*value*插入存储于*key*的列表中在基准值*pivot*的前面或者后面，如果执行成功，命令会返回执行插入操作后的列表对象的长度。

```c
void linsertCommand(client *c)
{
    ...
	iter = listTypeInitIterator(subject, 0, LIST_TAIL);
    while (listTypeNext(iter, &entry))
    {
        if (listTypeEqual(&entry, c->argv[3]))
        {
            listTypeInsert(&entry, c->argv[4], where);
            inserted = 1;
            break;
        }
    }
    listTypeReleaseIterator(iter);
    ...
}
```

上面这个函数`linsertCommand`是**LINSERT**命令的底层实现，通过上述的代码片段，我们可以了解到，其实现的基础逻辑便在通过列表迭代器对列表对象进行遍历，当找*pivot*的时候，调用`listTypeInsert`函数，将*value*插入到底层快速链表之中。在前面对快速链表的讲解时，曾经提到不能在使用迭代器对快速链表进行遍历的同时执行插入操作，因为这会导致迭代器失效，因此在上面的代码段中，执行插入之后便会从循环之中调用`break`跳出。

### 列表对象的POP系列命令

#### POP命令

对于基础的弹出操作，*Redis*提供了两个命令，分别会从列表对象的头部或者尾部删除一个元素，这两个命令的格式为：

1. **LPOP**，`LPOP key`
2. **RPOP**，`RPOP key`

执行上述两个命令，如果成功，会返回被弹出的元素，而当列表对象为空的时候，弹出失败，则会返回空值。

```c
void popGenericCommand(client *c, int where);
void lpopCommand(client *c);
void rpopCommand(client *c);
```

`popGenericCommand`函数是*Redis*实现**POP**系列命令的基础，通过`lookupKeyWriteOrReply`函数来查找*key*所对应的对象，在校验对象为列表对象之后，会调用`listTypePop`函数来实现具体的弹出功能。

### 列表对象的RPOPLPUSH系列命令

出了上面针对单一列表对象的操作之外，*Redis*还提供正对两个列表对象的操作命令**RPOPLPUSH**，这个命令的格式为：

`RPOPLPUSH source destination`

应用这个命令，可以原子性地将存储在*source*中的列表对象的尾部元素弹出，并将其插入到存储在*destination*的列表对象之中，当命令执行成功后，会返回这个被移动的元素。

```c
void rpoplpushHandlePush(client *c, robj *dstkey, robj *dstobj, robj *value);
void rpoplpushCommand(client *c);
```

上述两个函数会在内部调用`listTypePush`以及`listTypePop`函数来实现命令的功能，其本质上与**PUSH**命令以及**POP**命令相似，这里就不做过多赘述。

### 列表对象的阻塞POP系列命令

前面提到的**LPOP**命令，**RPOP**命令，**RPOPLPUSH**命令都是属于非阻塞的命令，也就是如果列表对象非空，那么命令会执行相应的操作；而当对象为空的时候，命令会直接返回空值，表示当前的命令执行失败。

相应地，*Redis*为上述的三个命令提供了阻塞的版本，也就是说当列表对象中非空的情况下，其行为与非阻塞版本一致；如果列表对象为空的时候，那么会阻塞客户端，直到列表中有新的元素被插入，或者等待超时。

这三个命令的阻塞版本的格式为：

1. **BLPOP**，`BLPOP key [key ...] timeout`
2. **BRPOP**，`BRPOP key [key ...] timeout`
3. **BRPOPLPUSH**，`BRPOPLPUSH source destination timeout`

```c
int serveClientBlockedOnList(client *receiver, robj *key, robj *dstkey, redisDb *db, robj *value, int where);
void blockingPopGenericCommand(client *c, int where);
void blpopCommand(client *c);
void brpopCommand(client *c);
void brpoplpushCommand(client *c);
```

上面的5个函数是*Redis*用于实现阻塞**POP**命令的函数，这里的具体实现细节，会在后续讲解*Redis*的阻塞操作时进行详细介绍。

### 列表对象的删除系列命令

#### LTRIM命令

**LTRIM**命令用于修剪一个已存在的列表对象，这个命令的格式为：

`LTRIM key start stop`

应用这个命令，可以使列表对象只保留由*start*到*end*这个闭区间的数据。

```c
void ltrimCommand(client *c);
```

`ltrimCommand`函数用于实现这个修剪命令，通过*start*以及*end*确定保留的索引范围，通过调用两次`quicklistDelRange`函数，将保留部分两侧的数据从底层快速链表中删除。

#### LREM命令

*Redis*提供了**LREM**命令，这个命令的格式为：

`LREM key count value`

用于从存储在*key*中的列表对象里，删除前*count*次出现的值为*value*的元素。其中对于*count*参数的数值有三种情况：

1. 当*count*为正数的时候，从列表对象的头部往尾部进行删除。
2. 当*count*为负数的时候，从列表对象的尾部向头部进行删除。
3. 当*count*为0的时候，会删除列表对象中的所有值为*value*的元素。

下面的这个函数用于实现**LREM**命令，给出的代码片段中包含了一段快速链表一边遍历一边删除的逻辑：

```c
void lremCommand(client *c)
{
	...
    while (listTypeNext(li, &entry))
    {
        if (listTypeEqual(&entry, obj))
        {
            listTypeDelete(li, &entry);
            server.dirty++;
            removed++;
            if (toremove && removed == toremove) break;
        }
    }
    listTypeReleaseIterator(li);
    ...
}
```

### 列表对象LINDEX与LSET命令

*Redis*提供了两个命令，可以允许客户端对列表对象按照某个下标索引进行操作，分别是**LINDEX**与**LSET**命令，这两个命令的格式为：

1. **LINDEX**，`LINDEX key index`
2. **LSET**，`LSET key index value`

上面的两个命令分别用于获取或者设置给定*key*对应的列表对象中的下标为*index*的元素，当*index*超过列表对象的长度时，**LINDEX**命令会返回一个空值，而**LSET**命令则会向客户端返回一个错误。

```c
void lindexCommand(client *c);
void lsetCommand(client *c);
```

上述两个函数是**LINDEX**命令以及**LSET**命令的实现，底层分别通过调用`quicklistIndex`以及`quicklistReplaceAtIndex`函数来实现相关的功能。

### 列表对象的RANGE命令

#### LRANGE命令

*Redis*中还提供了返回存储在*key*的列表对象里指定范围内元素的命令**LRANGE**，这个命令的格式为：

`LRANGE key start stop`

*start*与*end*是表示范围的偏移量，可以是基于0的非负数；同时也可以是从-1开始负数，用来表示从列表对象的尾部开始计数。这里需要注意的一点是，基于*start*以及*end*标记的范围是一个闭区间，也就是如果执行`LRANGE list 0 10`的话，那么会返回11个元素，而不是10个。

下面这个函数就是负责具体实现**LRANGE**命令的函数，通过下面的代码片段，我们可以了解到命令的具体实现：

```c
void lrangeCommand(client *c)
{
    ...
	if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.emptyarray)) == NULL
        || checkType(c,o,OBJ_LIST)) return;
    //处理下标索引
    ...
   	rangelen = (end - start) + 1;
    if (o->encoding == OBJ_ENCODING_QUICKLIST) {
        listTypeIterator *iter = listTypeInitIterator(o, start, LIST_TAIL);
        while(rangelen--) {
            listTypeEntry entry;
            listTypeNext(iter, &entry);
            quicklistEntry *qe = &entry.entry;
            ...
        }
        listTypeReleaseIterator(iter);
    }
    ...
}
```

### 列表对象的LLEN命令

*Redis*通过**LLEN**命令来获取一个列表对象的长度，这个命令的格式为：`LLEN key`，命令执行成功时，会返回列表对象的长度；当*key*不存在的时候，会被认为是一个空的列表，而返回1；当*key*存储的数据不是一个列表的时候，那么会向客户端返回错误。

```c
void llenCommand(client *c);
```

上面这个函数则会通过调用`listTypeLength`来实现**LLEN**命令对列表对象长度的获取。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
