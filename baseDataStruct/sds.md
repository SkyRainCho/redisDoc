# Redis中的C语言动态Strings库

这个库主要是用来描述*Redis*之中的五个基本类型之一的*String*，
*Redis*使用 `typedef char *sds;` 来描述这个动态*String*，
其在内存中的分布格式为一个*StringHeader*以及在*StringHeader*后面
一段连续的动态内存，而`sds`则是指向*StringHeader*后面的连续内存的第一个字节。

## Strings的头部信息

`sds`的头部信息主要包含了`sds`被分配的缓存大小以及已经使用的缓存的大小，
在*Redis*中定义了五种`sds`的头部信息：
* `sdshdr5`，定义了类型`SDS_TYPE_5`
* `sdshdr8`，定义了类型`SDS_TYPE_8`
* `sdshdr16`，定义了类型`SDS_TYPE_16`
* `sdshdr32`，定义了类型`SDS_TYPE_32`
* `sdshdr64`，定义了类型`SDS_TYPE_64`

其中`sdshdr5`从来不被使用，其余的头部信息都是按照如下格式（以`sdshdr32`为例）进行定义的：
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

在头文件之中定义了若干个宏以及静态函数用于实现对于`sds`的基础操作。
```c
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
```
给定一个`sds`数据，使用`SDS_HDR`来获取其对应的`sdshdr`的头指针。

```c
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
```
给定一个`sds`数据，声明一个`sdshdr`指针变量`sh`，并将这个sds对应的`sdshdr`头指正赋值给这个`sh`变量。

```c
static inline size_t sdslen(const sds s);
```
给定一个`sds`数据，获取其已使用缓存的长度，具体的实现方式为：
1. 结合`sdshdr`的定义，以及`sds`在内存中的分布结构，通过`s[-1]`来获取*StringHeader*中的`flags`数据。
2. 根据`flags`计算出其对应的是什么类型的`shshdr`。
3. 调用宏`SDS_HDR`获取到对应的*StringHeader*的指针，进而获得`len`字段数据。

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
的缓存，同时初始话Header中的type,len,alloc字段，并将init所指向的数据调用`memcpy`拷贝到sds的缓冲区中，
同时以`\0`作为结束标记(null-termined)。

后续的三个接口都是通过调用sdsnewlen来完成相关功能的。
* `sdsempty`函数用来创建一个空的sds数据
* `sdsnew`函数可以从一个null-terminated的C风格字符串中创建一个sds数据
* `sdsdup`函数可以通过一个给定的sds数据，复制出一个新的sds数据并返回

最后一个借口`sdsfree`函数通过调用s_free接口来释放一个给定的sds数据，
需要注意的是，所释放的内容包括sds头指针，以及其前面的Header数据。


## 用于调整Strings长度信息的操作函数
```c
void sdsupdatelen(sds s);
```
通过对内部数据调用strlen来更新sds的长度，这个接口在sds缓存被手动改写的情况下很有用

```c
void sdsclear(sds s);
```
用于清空一个sds数据的内容，但是不会释放已经存在的缓存。可以理解为内容和长度清零，但是空间还在。

```c
sds sdsMakeRoomFor(sds s, size_t addlen);
```
这个接口用于增大一个给定sds数据的可用空间，可以让确保玩家在调用改接口之后，可以向缓存之中续写
addlen个字节的内容，但是这个操作不会改变已经使用的缓存的大小，也就是不会改变sdslen调用的结果。
其中几个细节点：
1. 如果当前sds的可用空间也就是sdsavail的大小大于addlen，那么该函数什么操作也不会执行。
2. 如果当前操作没有引起Header的升级，例如从8位Header升级到16位，那么会调用s_realloc接口为其增加缓存容量。
3. 如果引发了Header的升级，那么会调用s_malloc接口来分配一个新的sds，将原始数据拷贝进去，返回新的sds数据。

```c
sds sdsRemoveFreeSpace(sds s);
```
这个接口的用途是收缩sds的缓存大小，使之刚好保存sdslen大小的数据，而没有多余的可用空间
其中的细节点：
1. 如果收缩导致Header的降级，那么调用s_malloc接口重新分配一个新的sds，拷贝数据后，返回新的sds
2. 如果收缩没有导致Header的降级，那么直接调用s_realloc接口调整缓存大小，实现容量收缩

```c
size_t sdsAllocSize(sds s);
```
这个接口用与返回分配给指定sds数据的内存的总大小，
其中包含:
1. s指针前的Header的大小
2. string数据的大小
3. 可用空间的大小
4. 强制结束符`\0`的大小

```c
void* sdsAllocPtr(sds s);
```
这个接口返回一个sds数据直接被分配的头指针，也就是Header首个字节的指针。

```c
void sdsIncrLen(sds s, ssize_t incr);
```
可以理解为可以给指定的sds的len增加incr的长度，同时会导致sdsavail的可用空间减少
基本应用场景:
1. 调用`sdsMakeRoomFor`函数为sds扩容
2. 向sds的缓存之中写入数据
3. 调用`sdsIncrLen`函数，调整写入数据之后的sdslen长度
同时需要注意incr的大小，不能超过sdsavail的大小，否则会触发断言机制

```c
sds sdsgrowzero(sds s, size_t len);
```
内部通过调用`sdsMakeRoomFor`将sds的缓存区增加len的长度，同时将新增的缓冲区初始化为0。

## 用于处理Strings内容的操作函数

```c
sds sdscatlen(sds s, const void *t, size_t len);
```
向sds数据中扩展一段二进制安全的数据，t为这段数据的指针，len为需要扩展的长度，
其内部会通过调用`sdsMakeRoomFor`尝试将sds的缓冲区进行扩展。

```c
sds sdscat(sds s, const char *t);
```
通过内部调用`sdscatlen`函数来实现向sds中扩展一个C风格字符串。

```c
sds sdscpylen(sds s, const char *t, size_t len);
sds sdscpy(sds s, const chart *t);
```
实现了向sds数据中复制数据的功能，与memcpy接口类似，会覆盖已有的数据。
上述两个函数分别实现了二进制安全数据的拷贝以及C风格字符串的拷贝。

```c
int sdsll2str(char *s, long long value);
int sdsull2str(char *s, unsigned long long value);
```
上述两个函数分别用于将有符号和无符号的long long整数转化为字符串，同时返回字符串长度。
注意其内部有一个自己实现的字符串翻转算法。

```c
sds sdsfromlonglong(long long value);
```
从一个long long整形数构建一个sds数据，类似调用`int sprintf(char *str, const char *format, ...)`
将一个整形数字写入一个字符串。

```c
sds sdscatvprintf(sds s, const char *fmt, va_list ap);
sds sdscatprintf(sds s, const char *fmt, ...);
sds sdscatfmt(sds s, char const* fmt, ...);
```
类似使用`printf`的调用的方式，将一个格式化C风格字符串添加到sds的结尾
需要注意的是，作为参数传入的s，在函数执行成功后已不能保证有效性，
应使用`s = sdscatprintf(s. "1");`这样的方式进行操作。
另外`sdscatfmt`函数是是一个与`sdscatprintf`类似的接口，但是其执行的速度要更快
原因在与其没依赖与libc库所提供的`sprintf()`族函数，而这一组操作通常是很慢的。
但是其只支持特定的几种通配符：
| 通配符 | 含义 |
|:------|:-----|
|`%s`|C风格字符串|
|`%S`|SDS字符串|
|`%i`|signed int 有符号整数|
|`%I`|64位有符号整数|
|`%u`|unsigned int 无符号整数|
|`%U`|64位无符号整数|
|`%%`||

```c
sds sdstrim(sds s, const char *cset);
```
给定一个C风格的字符集合`cset`，以及一个特定的`sds`数据，去除这个`sds`中前后属于这个`cset`的字符，
并返回更新后的`sds`数据。需要注意的是，这个移除操作，在遇到第一个不属于`cset`的字符时结束

```c
void sdsrange(sds s, sszie_t start, ssize_t end);
```
给定一个特定的`sds`数据，以及一个*range*的`[start, end]`，将这个`sds`数据缩小成符合这个*range*的子串。
*range*的范围可以是负值，其含义可以参考*Python*中对于列表的切片操作。

```c
void sdstolower(sds s);
void sdstoupper(ssd s);
```
上述两个操作是给定一个`sds`数据，对于其内部的每一个字符调用`tolower`或者`toupper`操作。

```c
int sdscmp(const sds s1, const sds s2);
```
内部通过调用`memcmp`来比较两个`sds`数据的大小：
* 如果`s1 > s2`，返回一个正数。
* 如果`s1 < s2`，返回一个负数。
* 如果两个`sds`的二进制数据完全相同，那么这个函数返回0。

