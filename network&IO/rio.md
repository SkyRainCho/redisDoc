# Redis通用面向流的IO接口
在*src/rio.h*以及*src/rio.c*这两文件中，提供了一组面向流的IO操作的抽象接口，通过这个抽象接口，我么们可以针对内存`sds`缓冲区、`FILE`文件以及*Socket*文件描述符上执行IO操作。对于*rio*的一个典型的应用便是在*src/rdb.c*中通过*rio*实现文件与缓存中*RDB*格式数据的读写。

在*src/rio.h*头文件中，给出*rio*对象的定义：
```c
struct _rio {
    size_t (*read)(struct _rio *, void *buf, size_t len);
    size_t (*write)(struct _rio *, const void *buf, size_t len);
    off_t (*tell)(struct _rio *);
    int (*flush)(struct _rio *);
    void (*update_cksum)(struct _rio *, const void *buf, size_t len);

    uint64_t cksum;
    size_t processed_bytes;
    size_t max_processing_chunk;
    union {
        struct {
            sds ptr;
            off_t pos;
        } buffer;

        struct {
            FILE *fp;
            off_t buffered;
            off_t autosync;
        } file;

        struct {
            int *fds;
            int *state;
            int numfds;
            off_t pos;
            sds buf;
        } fdset;
    } io;
};
typedef struct _rio rio;
```

在这个`rio`结构体中首先定义了五个函数指针成员，用于描述`rio`对象对外通用的方法：
1. `rio.read`，`rio`对象的通用读接口，用于从`rio`中读取`len`字节数据写入缓冲区`buf`中。
2. `rio.write`，`rio`对象的通用写接口，用于向`rio`写入`len`字节的数据。
3. `rio.tell`，用于获取`rio`对象内部偏移。
4. `rio.flush`，用于刷新`rio`对象数据，例如将数据写入文件中。
5. `rio.update_cksum`，用于在`rio`对象由于读或写操作导致变化后更新校验和。

除了上面五个函数指针外，`rio`还定义了三个成员变量：
1. `rio.cksum`，`rio`对象的校验和。
2. `rio.processed_bytes`，`rio`对象累计被读出或者写入的字节数。
3. `rio.max_processing_chunk`，`rio`对象单次最大读写的数据长度。

最后*Redis*还定义了了一个联合体`rio.io`用于描述核心IO操作的数据：
1. `rio.io.buffer`，用于处理针对内存中缓存的输入输出操作：
    1. `rio.io.buffer.ptr`，一个动态`sds`数据，用于维护内存缓存。
    1. `rio.io.buffer.pos`，当前处理到的偏移量，读出或者写入的偏移。
1. `rio.io.file`，用于处理针对标准文件指针的输入输出操作，主要应用场景为进行**RDB**存储以及**RDB**数据加载：
    1. `rio.io.file.fp`，`rio`对象对应的文件指针。
    1. `rio.io.file.buffered`，用于记录已经调用`fwrite`向文件中所写入的数据的大小，这个字段会在每次文件缓冲数据写入磁盘后被清零。
    1. `rio.io.file.autosync`，定义了当前`rio`对象缓冲文件写数据的上限，当`rio.io.file.buffered`超过这个上限，则会调用`fflush`以及`fdatasync`两个函数将文件写入磁盘。
1. `rio.io.fdset`，用于处理多个文件描述符的输出操作，主要应用场景为在**主从模式**下，**Master**实例将**RDB**文件推送给所有的**Slave**实例：
    1. `rio.io.fdset.fds`，用于存储*socket*文件描述符集合。
    1. `rio.io.fdset.state`，用于维护对应文件描述符的状态。
    1. `rio.io.fdset.numfds`，存储文件描述符集合中文件描述符的个数。
    1. `rio.io.fdset.buf`，一个`sds`内存缓存，用于存储向*socket*广播的数据。
    1. `rio.io.fdset.pos`，内存缓存偏移。

在*src/rio.h*文件中，定义了用于初始化与释放`rio`对象的四个接口：
```c
void rioInitWithFile(rio *r, FILE *fp);
void rioInitWithBuffer(rio *r, sds s);
void rioInitWithFdset(rio *r, int *fds, int numfds);
void rioFreeFdset(rio *r);
```

同时*Redis*还定义了四个通用接口，负责调用`rio`对象中对应的函数指针，实现对应的逻辑：
```c
static inline size_t rioWrite(rio *r, const void *buf, size_t len);
static inline size_t rioRead(rio *r, const void *buf, size_t len);
static inline off_t rioTell(rio *r);
static inline int rioFlush(rio *r);
```
上述四个接口分别封装了对于`rio`对应四个函数指针的调用，以`rioWrite`为例：
```c
static inline size_t rioWrite(rio *r, const void *buf, size_t len) {
    while (len) {
        size_t bytes_to_write = (r->max_processing_chunk && r->max_processing_chunk < len) ? r->max_processing_chunk : len;
        if (r->update_cksum) r->update_cksum(r,buf,bytes_to_write);
        if (r->write(r,buf,bytes_to_write) == 0)
            return 0;
        buf = (char*)buf + bytes_to_write;
        len -= bytes_to_write;
        r->processed_bytes += bytes_to_write;
    }
    return 1;
}
```
这个函数会向`rio`对象之中写入`len`字节长度的数据，循环调用`rio.write`函数，将数据以单次或者多次的形式写入`rio`对象中。如果`rio`设置了`update_cksum`函数指针，那么在每次写入数据的时候会调用`rio.update_cksum`来更新数据的校验和。同时会更新`rio.processed_bytes`数据。

如果说*Redis*通过`rioWrite`以及`rioRead`提供了一个对于`rio`对象通用的读写接口，那么下面的这几个函数则是通过调用`rioWrite`这个接口来实现了一组更加定制化的输出操作的功能：
```c
size_t rioWriteBulkCount(rio *r, char prefix, long count);
size_t rioWriteBulkString(rio *r, const char *buf, size_t len);
size_t rioWriteBulkLongLong(rio *r, long long l);
size_t rioWriteBulkDouble(rio *r, double d);
int rioWriteBulkObject(rio *r, struct redisObject *obj);
```
以上这几个函数用于将数据格式化地写入到`rio`对象之中，这其中`rioWriteBulkCount`是一个辅助函数，用于将一个前缀字符`prefix`以及一个字节长度数据`count`写入`rio`对象之中，同时在写入这些数据之后，添加辅助的`\r\n`来与下一组数据进行区分。那么执行该函数之后，会调用`rioWrite`函数向`rio`对象中写入如下的数据：
    \*<count>\r\n
这其中`*`表示`prefix`前缀字符，`<count>`则是转化为字符串形式的整形数`count`。

而函数`rioWriteBulkString`则是向`rio`对象中写入一段二进制安全的字符串数据，其格式为：
    $<len>\r\n<payload>\r\n
这里写入`rio`对象的数据被`\r\n`划分为两个部分，首先便是字符串的长度信息，显然其是通过调用`rioWriteBulkCount(r, '$', len)`，将长度数据写入`rio`之中的；接下来便是将这段二进制安全的字符串的具体内容写入`rio`对象中，同样的会议`\r\n`作为结尾的分隔标记。

而函数`rioWriteBulkLongLong`以及`rioWriteBulkDouble`则是用于将一个整形数以及浮点数写入`rio`对象中，这两个函数的逻辑都是将数字转化为字符串的形式，然后调用`rioWriteBulkString`执行的写入逻辑。


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*