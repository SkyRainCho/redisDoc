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

对于列表对象的操作除了创建与释放操作被定义在*src/object.c*文件中之外，其余的内容都被定义在*src/t_list.c*源文件中，其中主要包含两个部分：
1. 列表对象基础操作定义。
2. 列表对象命令实现。

## 列表对象基础操作

### 列表对象的PUSH操作
```c
void listTypePush(robj *subject, robj *value, int where);
```

### 列表对象的POP操作
```c
void *listPopSaver(unsigned char *data, unsigned int sz);
robj *listTypePop(robj *subject, int where);
```

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
int listTypeEqual(listTypeEntry *entry, robj *o);
unsigned long listTypeLength(const robj *subject);
robj *listTypeGet(listTypeEntry *entry);
void listTypeConvert(robj *subject, int enc);
```

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
