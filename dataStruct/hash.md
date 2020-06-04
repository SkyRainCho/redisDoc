# Redis中散列对象类型实现
通过*src/object.c*文件中的代码，我们看到散列对象只有一个默认的构造函数`createHashObject`，这个函数会使用压缩链表作为底层数据结构，这也就是意味着初始的散列对象都是使用压缩链表，当满足一定条件之后，会从压缩链表转化为`dict`作为底层数据结构。

*Redis*中对于散列对象的操作函数主要被定义在`src/t_hash.c`这个文件中。

## 散列对象基础操作
### 散列数据底层实现转化
```c
void hashTypeTryConversion(robj *o, robj **argv, int start, int end);
void hashTypeConvertZiplist(robj *o, int enc);
void hashTypeConvert(robj *o, int enc);
```

### 散列数据查询操作
```c
sds hashTypeGetFromHashTable(robj *o, sds field);
int hashTypeGetFromZiplist(robj *o, sds field, unsigned char **vstr, unsigned int *vlen, long long *vll);
```

```c
int hashTypeGetValue(robj *o, sds field, unsigned char **vstr, unsigned int *vlen, long long *vll);
robj *hashTypeGetValueObject(robj *o, sds field);
```

```c
size_t hashTypeGetValueLength(robj *o, sds field);
```

```c
int hashTypeExists(robj *o, sds field);
```

### 散列数据插入操作
```c
int hashTypeSet(robj *o, sds field, sds value, int flags);
```

```c
robj *hashTypeLookupWriteOrCreate(client *c, robj *key);
```

### 散列数据删除操作
```c
int hashTypeDelete(robj *o, sds field);
```

### 散列数据迭代与遍历
```c
hashTypeIterator *hashTypeInitIterator(robj *subject);
void hashTypeReleaseIterator(hashTypeIterator *hi);
int hashTypeNext(hashTypeIterator *hi);
```

```c
void hashTypeCurrentFromZiplist(hashTypeIterator *hi, int what, unsigned char **vstr, unsigned int *vlen, long long *vll);
```

```c
sds hashTypeCurrentFromHashTable(hashTypeIterator *hi, int what);
```

```c
void hashTypeCurrentObject(hashTypeIterator *hi, int what, unsigned char **vstr, unsigned int *vlen, long long *vll);
```

```c
sds hashTypeCurrentObjectNewSds(hashTypeIterator *hi, int what);
```

## 散列对象命令实现
### HSET系列命令
```c
void hsetCommand(client *c);
void hsetnxCommand(client *c);
```

### HGET系列命令
```c
void hgetCommand(client *c);
void hmgetCommand(client *c);
void genericHgetallCommand(client *c, int flags);
void hgetallCommand(client *c);
void hkeysCommand(client *c);
void hvalsCommand(client *c);
```

### HINCRBY系列命令
```c
void hincrbyCommand(client *c);
void hincrbyfloatCommand(client *c);
```

### HDEL命令
```c
void hdelCommand(client *c);
```

### HLEN命令
```c
void hlenCommand(client *c);
```

### HSTRLEN命令
```c
void hstrlenCommand(client *c);
```

### HEXISTS命令
```c
void hexistsCommand(client *c);
```

### HSCAN命令
```c
void hscanCommand(client *c);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
