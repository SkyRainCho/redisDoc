# Redis中字符串对象类型实现

对于*Redis*中字符串对象的类型的代码主要分布在两个文件之中，其中在*src/object.c*文件中主要是实现了字符串数据类型的构造相关的操作，另外在*src/t_string.c*文件中则实现了字符串的相关命令。

对于*Redis*中的字符串对象，可以使用三种编码类型，分别是：
1. `OBJ_ENCODING_RAW`
2. `OBJ_ENCODING_INT`
3. `OBJ_ENCODING_EMBSTR`

其中当字符串的长度较短的时候，*Redis*会采用`OBJ_ENCODING_EMBSTR`的编码方式，这个长度阈值的定义为`#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44`，当超过这个长度的字符串则会才用`OBJ_ENCODING_RAW`的编码方式，而当这个字符串实际上是一个整形数的时候，*Redis*则会采用`OBJ_ENCODING_INT`对其进行编码。

## Redis字符串对象的基础操作
### Redis字符串对象的构造
#### 字符串型对象的构造
```c
robj *createRawStringObject(const char *ptr, size_t len);
robj *createEmbeddedStringObject(const char *ptr, size_t len);
robj *createStringObject(const char *ptr, size_t len);
```


#### 数值型对象的构造
```c
robj *createStringObjectFromLongLongWithOptions(long long value, int valueobj);
robj *createStringObjectFromLongLong(long long value);
robj *createStringObjectFromLongLongForValue(long long value);
robj *createStringObjectFromLongDouble(long double value, int humanfriendly);
```

### Redis数值型字符串对象的编码与解码
```c
int isSdsRepresentableAsLongLong(sds s, long long *llval);
int isObjectRepresentableAsLongLong(robj *o, long long *llval);
int getDoubleFromObject(const robj *o, double *target);
int getDoubleFromObjectOrReply(client *c, robj *o, double *target, const char *msg);
int getLongDoubleFromObject(robj *o, long double *target);
int getLongDoubleFromObjectOrReply(client *c, robj *o, long double *target, const char *msg);
int getLongLongFromObject(robj *o, long long *target);
int getLongLongFromObjectOrReply(client *c, robj *o, long long *target, const char *msg);
int getLongFromObjectOrReply(client *c, robj *o, long *target, const char *msg);
```


### Redis字符串对象的比较
```c
int compareStringObjectsWithFlags(robj *a, robj *b, int flags);
int compareStringObjects(robj *a, robj *b);
int collateStringObjects(robj *a, robj *b);
int equalStringObjects(robj *a, robj *b);
```

### Redis字符串对象的其他操作
```c
robj *tryObjectEncoding(robj *o);
robj *getDecodedObject(robj *o);
robj *dupStringObject(const robj *o);
void trimStringObjectIfNeeded(robj *o);
size_t stringObjectLen(robj *o);
```

## Redis字符串对象的命令实现
### SET命令的实现
```c
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply);
void setCommand(client *c);
void setnxCommand(client *c);
void setexCommand(client *c);
void psetexCommand(client *c);
```

### GET命令的实现
```c
int getGenericCommand(client *c);
void getCommand(client *c);
```

### GETSET命令的实现
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
