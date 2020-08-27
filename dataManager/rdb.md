# Redis数据持久化RDB策略
众所周知*Redis*是一款内存数据库，所有的数据都被存储在内存之中，然而如果数据仅仅被存储在内存中的话，那么一旦服务器进程出现停机，那么所有的数据都将丢失，因此*Redis*需要支持数据的持久化，将内存之中的数据存储在磁盘之中。当*Redis*进程启动时，会从磁盘之中将数据恢复到内存之中。

**RDB**持久化是*Redis*支持的一种持久化策略，*Redis*会将服务器的状态信息以及所有数据库中的数据序列化到磁盘的文件之中，相当于保存了服务器数据的一个快照。在*Redis*启动时，会从文件中反序列化恢复数据到内存之中。

*Redis*可以通过主动或者被动的方式来创建**RDB**文件：
1. 主动方式，通过客户端数据的命令**SAVE**以及**BGSAVE**，来启动**RDB**持久化。
    1. 如果客户端执行**SAVE**，那么*Redis*会同步地生成**RDB**文件，此时*Redis*的主线程会被阻塞起来，直到**RDB**文件生成完成，*Redis*都不会响应客户端的命令请求。
    2. 如果客户端执行**BGSAVE**，那么*Redis*会异步地生成**RDB**文件，此时*Redis*会主动创建一个子进程用于生成**RDB**文件，而*Redis*此时依然可以处理客户端的其他命令请求。
2. 被动方式，*Redis*检测一段时间内修改次数累积到一定的阈值时，会启动异步生成**RDB**文件的逻辑，*Redis*会通过下面这个结构体来描述这个自动备份的阈值：
    ```c
    struct saveparam {
        time_t seconds;
        int changes;
    };
    ```
这表示，如果*Redis*在`saveparam.seconds`秒数内累积了`saveparam.changes`次变化的化，便会启动备份。*Redis*可以保有多个这样的阈值，存储在服务器全局数据之中：
```c
struct redisServer {
    ...
    struct saveparam *saveparams;
    int saveparamslen;
};
```
多个阈值只要有一个满足，那么便会触发生成**RDB**文件的逻辑。

```c
typedef struct rdbSaveInfo {
    int repl_stream_db;
    int repl_id_is_set;
    int repl_id[CONFIG_RUN_ID_SIZE+1];
    long long repl_offset;
} rdbSaveInfo;
```
## RDB基础数据的读写

## RDB对象数据的读写
```c
int rdbSaveObjectType(rio *rdb, robj *o);
int rdbLoadObjectType(rio *rdb);
```
上述两个函数用于处理对象类型在`rio`对象中的读写操作，**RDB**使用的类型被定义在*src/rdb.h*头文件中：
1. `#define RDB_TYPE_STRING 0`，这个定义对应字符串类型对象
2. `#define RDB_TYPE_LIST   1`，对应列表对象，但是现在已经不在继续使用
3. `#define RDB_TYPE_SET    2`，对应使用`OBJ_ENCODING_HT`编码方式的集合对象
4. `#define RDB_TYPE_ZSET   3`，对应有序集合对象，但是现在已经不在继续使用
5. `#define RDB_TYPE_HASH   4`，对应使用`OBJ_ENCODING_HT`编码方式的散列对象
6. `#define RDB_TYPE_ZSET_2 5`，对应使用`OBJ_ENCODING_SKIPLIST`编码方式的有序集合对象
7. `#define RDB_TYPE_MODULE 6`，现在已经不在继续使用
8. `#define RDB_TYPE_MODULE_2`，对应使用*Module*对象类型
9. `#define RDB_TYPE_HASH_ZIPMAP    9`，现在已经不在继续使用，
10. `#define RDB_TYPE_LIST_ZIPLIST  10`，现在已经不在继续使用，
11. `#define RDB_TYPE_SET_INTSET    11`，对应使用`OBJ_ENCODING_INTSET`编码方式的集合对象
12. `#define RDB_TYPE_ZSET_ZIPLIST  12`，对应使用`OBJ_ENCODING_ZIPLIST`编码方式的有序集合对象
13. `#define RDB_TYPE_HASH_ZIPLIST  13`，对应使用`OBJ_ENCODING_ZIPLIST`编码方式的散列对象
14. `#define RDB_TYPE_LIST_QUICKLIST 14`，对应使用`OBJ_ENCODING_QUICKLIST`编码方式的列表对象
15. `#define RDB_TYPE_STREAM_LISTPACKS 15`，对应使用*Stream*对象类型

## RDB的存储策略
*Redis*定义了两种**RDB**的存储方式：
1. 同步生成**RDB**的方式，`int rdbSave(char *filename, rdbSaveInfo *rsi);`，这种方式会阻塞*Redis*的主线程。
2. 异步生成**RDB**的方式，`int rdbSaveBackground(char *filename, rdbSaveInfo *rsi)`，这种方式*Redis*会启动一个子进程来执行**RDB**文件的存储，主线程不会被阻塞。

## RDB操作码的读写
*Redis*定义了若干操作对应的操作码，通过`rdbSaveType`以及`rdbLoadType`接口进行存取：
1. `#define RDB_OPCODE_MODULE_AUX       247`
2. `#define RDB_OPCODE_IDLE             248`
3. `#define RDB_OPCODE_FREQ             249`
4. `#define RDB_OPCODE_AUX              250`
5. `#define RDB_OPCODE_RESIZEDB         251`
6. `#define RDB_OPCODE_EXPIRETIME_MS    252`
7. `#define RDB_OPCODE_EXPIRETIME       253`
8. `#define RDB_OPCODE_SELECTDB         254`
9. `#define RDB_OPCODE_EOF              255`

```c
int rdbSaveType(rio *rdb, unsigned char type);
int rdbLoadType(rio *rdb);
```

## RDB时间的读写
```c
int rdbSaveTime(rio *rdb, time_t t);
time_t rdbLoadTime(rio *rdb);
```
上面这两个接口用于秒数时间戳的读写，但是上述的两个接口只用于老的数据库中使用`RDB_OPCODE_EXPIRETIME`标记的数据，而新版本的则是使用`RDB_OPCODE_EXPIRETIME_MS`，用于村塾毫秒级的时间戳，同时使用下面的两个接口进行读写：
```c
int rdbSaveMillisecondTime(rio *rdb, long long t);
long long rdbLoadMillisecondTime(rio *rdb, int rdbver);
```
上述这几个函数都是使用`rio`原生的`rioRead`以及`rioWrite`读写操作，将数据直接写入文件，或从文件之中读取数据。

## RDB数据长度以及整数数据信息的读写
```c
int rdbSaveLen(rio *rdb, uint64_t len);
int rdbEncodeInteger(long long value, unsigned char *enc);
uint64_t rdbLoadLen(rio *rdb, int *isencoded);
int rdbLoadLenByRef(rio *rdb, int *isencoded, uint64_t *lenptr);
```
如果全部使用32位来存储数值上比较小的长度数值，那么将会浪费大量的存储空间，因此*Redis*在将长度数据序列化到文件中时，使用一种压缩的格式进行存储，序列化的第一个字节的最高两位比特，用于区分长度信息的类型。
对于长度信息的分类，*Redis*在*src/rdb.h*头文件中给出了几个定义：
1. `#define RDB_6BITLEN 0`，如果长度大小不超过63个字节，也就是`2^7-1`，那么长度信息会被序列化为1个字节长度的数据，其格式为`00|XXXXXX`，最高位`00`用于标记其类型，第六位用于存储长度信息。
2. `#define RDB_14BITLEN 1`，如果长度信息大于63个字节，但是不超过16383字节，那么长度信息会被序列化为2个字节长度的数据，其格式为`01|XXXXXX XXXXXXXX`，高两位`01`用于标记类型，剩余的14位用于存储长度数据。
3. `#define RDB_32BITLEN 0x80`，如果长度信息超过16383字节，但是可以使用32位来表述，那么长度信息将会被序列化为5个字节长度的数据，其格式为`10|000000 [32 bits]`，第一个字节用于标记类型，后面4个字节用于存储长度数据。
4. `#define RDB_64BITLEN 0x81`，如果长度信息需要64位才可以表示，那么会被序列化位9个字节的数据，其格式位`10|000001 [64 bits]`。
5. `#define RDB_ENCVAL 3`，对于这个标记也就是第一个字节的高两位为`11`的时候，表明这其中存储了一个编码的整数数据：
    1. `#define RDB_ENC_INT8 0`
    2. `#define RDB_ENC_INT16 1`
    3. `#define RDB_ENC_INT16 1`

`rdbSaveLen`便是按照上述的规则，将一个长度信息`len`写入`rdb`对应的文件之中。`rdbEncodeInteger`函数则是按照`RDB_ENCVAL`格式编码一个整数数据到`enc`缓存中。
`rdbLoadLenByRef`以及`rdbLoadLen`这两个函数则是用于从文件之中读取一个长度信息。

## RDB保存操作
```c
int rdbSaveObjectType(rio *rdb, robj *o);
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi);
int rdbSaveToSlavesSockets(rdbSaveInfo *rsi);
int rdbSave(char *filename, rdbSaveInfo *rsi);
ssize_t rdbSaveObject(rio *rdb, robj *o, robj *key);
size_t rdbSavedObjectLen(robj *o);
void backgroundSaveDoneHandler(int exitcode, int bysignal);
int rdbSaveKeyValuePair(rio *rdb, robj *key, robj *val, long long expiretime);
ssize_t rdbSaveSingleModuleAux(rio *rdb, int when, moduleType *mt);
ssize_t rdbSaveStringObject(rio *rdb, robj *obj);
ssize_t rdbSaveRawString(rio *rdb, unsigned char *s, size_t len);
int rdbSaveBinaryDoubleValue(rio *rdb, double val);
int rdbSaveBinaryFloatValue(rio *rdb, float val);
rdbSaveInfo *rdbPopulateSaveInfo(rdbSaveInfo *rsi);
```

## RDB加载操作
```c
int rdbLoadObjectType(rio *rdb);
int rdbLoad(char *filename, rdbSaveInfo *rsi);
robj *rdbLoadObject(int type, rio *rdb, robj *key);
robj *rdbLoadStringObject(rio *rdb);
void *rdbGenericLoadStringObject(rio *rdb, int flags, size_t *lenptr);
int rdbLoadBinaryDoubleValue(rio *rdb, double *val);
int rdbLoadBinaryFloatValue(rio *rdb, float *val);
int rdbLoadRio(rio *rdb, rdbSaveInfo *rsi, int loading_aof);
```


























































****
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
