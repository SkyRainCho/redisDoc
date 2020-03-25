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
static inline size_t sdsavail(const sds s);
```

```c
static inline void sdssetlen(sds s, size_t newlen);
```

```c
static inline void sdsinclen(sds s, size_t inc);
```

```c
static inline size_t sdsalloc(const sds s);
```

```c
static inline void sdssetalloc(const sds s, size_t newlen);
```

## 构造与释放Strings的操作函数

## 用于调整Strings长度信息的操作函数