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

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*