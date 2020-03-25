# Redis中的C语言动态Strings库

这个库主要是用来描述Redis之中的五个基本类型之一的String，
Redis使用 `typedef char *sds;` 来描述这个动态String，
其在内存中的分布格式为一个StringHeader以及在StringHeader后面
一段连续的动态内存，而`sds`则是指向StringHeader后面的连续内存的第一个字节。

## Strings的头部信息

sds的头部信息主要包含了sds被分配的缓存大小以及已经使用的缓存的大小，
在Redis中定义了五种sds的头部信息：
* sdshdr5，定义了类型SDS_TYPE_5
* sdshdr8，定义了类型SDS_TYPE_8
* sdshdr16，定义了类型SDS_TYPE_16
* sdshdr32，定义了类型SDS_TYPE_32
* sdshdr64，定义了类型SDS_TYPE_64
其中sdshdr5从来不被使用，其余的头部信息都是按照如下格式（以sdshdr32为例）进行定义的：
```c
struct __attribute__ ((__packed__)) sdshdr32
{
    uint32_t        len;    //已经使用缓存的长度
    uint32_t        alloc;  //包含header以及结尾null结束符在内分配的缓存的总长度
    unsigned char   flags;  //低三位保存header类型信息，SDS_TYPE_32
    char            buf[];  //动态分配的缓存
};
```

## Strings的通用底层操作

在头文件之中定义了若干个宏以及静态函数用于实现对于sds的基础操作。
```c
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
```
给定一个sds数据，使用`SDS_HDR`来获取其对应的`sdshdr`的头指针。

```c
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
```
给定一个sds数据，声明一个`sdshdr`指针变量`sh`，并将这个sds对应的`sdshdr`头指正赋值给这个`sh`变量。

```c
static inline size_t sdslen(const sds s);
```
给定一个sds数据，获取其已使用缓存的长度，具体的实现方式为：
1. 结合sdshdr的定义，以及sds在内存中的分布结构，通过`s[-1]`来获取header中的flags数据。
2. 根据flags计算出其对应的是什么类型的shshdr。
3. 调用宏`SDS_HDR`获取到对应的header的指针，进而获得`len`字段数据。

```c
static inline size_t sdsalloc(const sds s);
```
给定一个sds数据，获取其分配缓存的总长度。

```c
static inline size_t sdsavail(const sds s);
```
给定一个sds数据，获取其缓存可用的长度，可以理解为`sdsavail(s) == sdsalloc(s) - sdslen(s)`。

```c
static inline void sdssetlen(sds s, size_t newlen);
```
给定一个sds数据，以及一个新的长度newlen，将sds的header中的len字段设置为newlen。

```c
static inline void sdsinclen(sds s, size_t inc);
```
给定一个sds数据，以及一个需要增加的长度inc，将sds的header中的len字段增加inc。

```c
static inline void sdssetalloc(const sds s, size_t newlen);
```
给定一个sds数据，以及一个新的长度newlen，将sds的header中的alloc字段设置为newlen。

## 构造与释放Strings的操作函数
```c
static inline int sdsHdrSzie(char type);
static inline char sdsReqType(size_t string_size);
```
上述两个在sds.c头文件中定义的两个静态函数，分别用于计算一个特定HeaderType的header的长度，
以及根据一个string_size的长度，返回合适的HeaderType。

```c
sds sdsnewlen(const void *init, size_t initlen);
sds sdsempty(void);
sds sdsnew(const char* init);
sds sdsdup(const sds s);
void sdsfree(sds s);
```

其中`sdsnewlen`函数，是这一系列函数的基础，其作用是，给定一段初始化内存的头指针init，
以及初始长度initlen，构建一个sds数据。这个函数会根据你需要初始化数据的长度initlen通过`sdsReqType`来选择
所使用的HeaderType，8位，16位，32位还是64位，使用`s_malloc`调用，为其分配长度为`headerSize + initlen + 1`
的缓存，同时初始话Header中的type,len,alloc字段，并将init所指向的数据调用memcpy拷贝到sds的缓冲区中，
同时以`\0`作为结束标记(null-termined)。

后续的三个接口都是通过调用sdsnewlen来完成相关功能的。
`sdsempty`函数用来创建一个空的sds数据
`sdsnew`函数可以从一个null-terminated的C风格字符串中创建一个sds数据
`sdsdup`函数可以通过一个给定的sds数据，复制出一个新的sds数据并返回

最后一个借口`sdsfree`函数通过调用s_free接口来释放一个给定的sds数据，
需要注意的是，所释放的内容包括sds头指针，以及其前面的Header数据。


## 用于调整Strings长度信息的操作函数