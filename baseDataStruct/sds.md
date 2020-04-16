# Redis中的C语言动态Strings库

在*src/sds.h*中定义了*Redis*中的动态*String*类型，这意味着，
使用者仅仅需要调用接口*API*就可以向*String*加入数据，而不需要关心扩容的问题。
*Redis*使用 `typedef char *sds;` 来描述这个动态*String*，
其在内存中的分布格式为一个*StringHeader*以及在*StringHeader*后面
一段连续的动态内存，而`sds`则是指向*StringHeader*后面的连续内存的第一个字节。
其在内存中问分布情况可以入下图所示：

![sds内存分布](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/sds%E5%86%85%E5%AD%98%E5%88%86%E5%B8%83.PNG)

***
## Strings的头部信息

`sds`的头部信息主要包含了`sds`被分配的缓存大小以及已经使用的缓存的大小，
根据所需要的分配缓存的大小，在*Redis*中定义了五种`sds`的头部信息：
* `sdshdr5`，定义了类型`SDS_TYPE_5`
* `sdshdr8`，定义了类型`SDS_TYPE_8`
* `sdshdr16`，定义了类型`SDS_TYPE_16`
* `sdshdr32`，定义了类型`SDS_TYPE_32`
* `sdshdr64`，定义了类型`SDS_TYPE_64`

上述的五种`sdshdr`分别表示最大可以分配缓存的大小，
其中`sdshdr5`表示最大可以分配`1 << 5`大小的缓存，而`sdshdr8`表示最大可以分配`1 << 8`大小的缓存。

除了`sdshdr5`之外，其余的头部信息都是按照如下格式（以`sdshdr32`为例）进行定义的：
```c
struct __attribute__ ((__packed__)) sdshdr32
{
    uint32_t        len;    //已经使用缓存的长度
    uint32_t        alloc;  //包含header以及结尾null结束符在内分配的缓存的总长度
    unsigned char   flags;  //低三位保存header类型信息，SDS_TYPE_32
    char            buf[];  //动态分配的缓存
};
```
而`sdshdr5`头部的格式则是按照如下的方式进行定义的：
```c
struct __attribute__ ((__packed__)) sdshdr5
{
    unsigned char    flags;    //低三位保存header类型信息，高五位用于表示已经使用缓存的长度
    char             buf[];    //动态分配的缓存 
};
```
基于我们需要的*String*的长度，选择不同的`sdshdr`，这样可以达到节约空间的目的。
通过*Redis*之中关于`sdshdr`数据类型的定义，我们可以发现，无论是哪种`sdshdr`,
`sdshdr.buf`缓存字段之前，都是`sdshdr.flags`标记字段，在*Redis*之中，
我们实际使用的`sds`变量，其实是指向`sdshdr.buf`的指针，而整个*SDS*是一段连续分配的内存，
那么，如果我们通过`sds`向前偏移一个字节长度的话`sds[-1]`，一定是这个*SDS*数据的`sdshdr.flags`字段。
通过位运算，我们便可以知道该*SDS*数据所使用`sdshdr`的类型，进而通过指针偏移，
便可以获取到整个*SDS*的头部信息，后续很对对于*SDS*的基础操作都是通过该方式实现的。

***

## Strings的通用底层操作

在头文件之中定义了若干个宏以及静态函数用于实现对于`sds`的基础操作。

```c
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
```
给定一个`sds`数据，使用`SDS_HDR`来获取其对应的`sdshdr`的头指针。
其使用方法是通过*SDS*数据获取到对应的`sdshdr.flags`，进而得到*HeaderType*，通过调用`SDS_HDR`来获取
整个头部信息，例如：
```c
unsigned char flags = s[-1];
switch (flags * SDS_TYPE_MASK)
{
    ...

    case SDS_TYPE_8:
        SDS_HDR(8, s)->len = new_len;
        break;
    ...
}
```

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
给定一个sds数据，获取其分配缓存`sdshdr.buf`的总长度。

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
需要注意的是，在`sdsinclen`中，并不会检查增加的长度`inc`是否是合法的，
仅仅是将`inc`累加到`sdshdr.len`之中。这就需要调用者在调用`sdsinclen`接口之前，
自己检查长度是否合法。

```c
static inline void sdssetalloc(const sds s, size_t newlen);
```
给定一个sds数据，以及一个新的长度newlen，将sds的header中的alloc字段设置为newlen。

通过源码，我们可以发现，`SDS_TYPE_5`类型的`sds`数据与其他类型的`sds`数据不论在`sdshdr`结构，
还是基础操作接口的处理，均有很大的差异。*Redis*在2015年7月15日的提交中引入了这个新的`sds`类型，
作者自己给出的提交信息是：
> A new type, SDS_TYPE_5 is introduced having a one byte header with just the string length, without information about the available additional length at the end of the string.

结合后续其他操作接口对于`SDS_TYPE_5`类型的处理，我们可以认为，
这个类型的`sds`数据，主要用于存储长度不超过32个字节，并且不会重新分配缓存的数据。
对此，*Redis*的作者也给出了建议：
> Don't use TYPE 5 if strings are going to be reallocated, since it sucks not having a free space left field.

***
## 构造与释放Strings的操作函数
```c
static inline int sdsHdrSzie(char type);
static inline char sdsReqType(size_t string_size);
```
上述两个在*src/sds.c*头文件中定义的两个静态函数，分别用于返回一个特定*HeaderType*的头部结构体长度，
以及根据一个`string_size`的长度，返回合适的*HeaderType*。

```c
sds sdsnewlen(const void *init, size_t initlen);
sds sdsempty(void);
sds sdsnew(const char* init);
sds sdsdup(const sds s);
void sdsfree(sds s);
```

其中`sdsnewlen`函数，是这一系列函数的基础，其作用是，给定一段初始化内存的头指针`init`，以及初始长度`initlen`，构建一个`sds`数据。
这个函数会根据你需要初始化数据的长度`initlen`通过`sdsReqType`接口来选择所使用的*HeaderType*，8位，16位，32位还是64位，
使用`s_malloc`调用，为其分配长度为`headerSize + initlen + 1`的缓存，之所以需要多分配出一个字节的缓存，
是因为在*Redis*中的`sds`总是以`\0`作为结束标记的，因为需要为这个结束标记多分配出一个字节的缓存。
同时由于`sds`本质上是二进制安全的，这也就意味着在数据的中间也有可能会出现`\0`，故此这也是我们为什么在头部信息结构体中需要`sdshdr.len`字段的原因。
同时初始话Header中的type,len,alloc字段，并将init所指向的数据调用`memcpy`拷贝到sds的缓冲区中，
同时以`\0`作为结束标记(null-termined)。

后续的三个接口都是通过调用sdsnewlen来完成相关功能的：
* `sdsempty`函数用来创建一个空的sds数据。
* `sdsnew`函数可以从一个null-terminated的C风格字符串中创建一个sds数据。注意这个接口不是二进制安全的，因为其内部是使用`strlen`来计算传入数据长度的。
* `sdsdup`函数可以通过一个给定的sds数据，复制出一个新的sds数据并返回

最后一个接口`sdsfree`函数通过调用`s_free`接口来释放一个给定的`sds`数据，
需要注意的是，所释放的内容包括`sds`头指针，以及其前面的*Header*数据的整个缓存。

***
## 用于调整Strings长度信息的操作函数
```c
void sdsupdatelen(sds s);
```
通过对内部数据调用`strlen`来更新`sds`的长度，这个接口在`sds`缓存被手动改写的情况下很有用。
通常来说，这个接口用于缩短`sds`的`sdshdr.len`字段，但是这个接口不会对`sdshdr.buf`中的数据进行修改。

```c
void sdsclear(sds s);
```
用于清空一个`sds`数据的内容，与`sdsupdatelen`接口类似，这个函数不会释放或者修改已经存在的缓存。仅仅是将`sdshdr.len`长度字段清零，但是空间还在。

```c
sds sdsMakeRoomFor(sds s, size_t addlen);
```
`sdsMakeRoomFor`这个接口用于扩大一个给定`sds`数据的可用缓存空间，可以确保用户在调用这个接口之后，可以向缓存之中续写
`addlen`个字节的内容，但是这个操作不会改变已经使用的缓存的大小，也就是不会改变`sdslen`调用的结果。
其中几个细节点：
1. 如果当前`sds`的可用空间也就是`sdsavail`的大小大于`addlen`，那么该函数什么操作也不会执行。
2. 同时为了减少重复分配缓存所带来的系统开销，`sdsMakeRoomFor`接口总是会多分配出一些预留（最多1MB字节）的缓存：
```c
    newlen = (len+addlen)
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
```
3. 如果当前操作没有引起*Header*的升级，例如从8位*Header*升级到16位，那么会调用`s_realloc`接口为其增加缓存容量。
4. 扩容操作永远不会使用`SDS_TYPE_5`类型的*Header*，因为该类型的*Header*无法保存可用缓存的大小，这也就意味着，如果使用`SDS_TYPE_5`类型的`sds`，
那么每次进行*append*操作的时候，都会调用`sdsMakeRoomFor`来重新分配缓存。那么对于一个`SDS_TYPE_5`类型的`sds`，在调用过一次`sdsMakeRoomFor`之后，
至少会被升级的`SDS_TYPE_8`类型的`sds`。
5. 如果引发了*Header*的升级，那么会调用`s_malloc`接口来分配一个新的`sds`，将原始数据拷贝进去，返回新的`sds`指针,
这也就意味着，调用者无法保证作为参数传入的`sds`指针在调用结束后是否依然有效，
因此比如使用函数的返回值来执行后续操作`s = sdsMakeRoomFor(s, newlen);`。

对于接口`sdsMakeRoomFor`，*Redis*的作者给出的建议是：
> Don't call sdsMakeRoomFor() when obviously not needed.

也就是说，当我们能确保`sds`中有足够的多余缓存时，那么就不要调用该接口。

```c
sds sdsRemoveFreeSpace(sds s);
```
`sdsRemoveFreeSpace`这个接口的用途是收缩sds的缓存大小，通过释放多余的可用空间，使之刚好保存`sdslen`大小的数据。
其中的细节点：
1. 如果收缩导致*Header*的降级，那么调用`s_malloc`接口重新分配一个新的`sds`，拷贝数据后，返回新的`sds`。
2. 如果收缩没有导致*Header*的降级，那么直接调用`s_realloc`接口调整缓存大小，实现缓存释放。

```c
size_t sdsAllocSize(sds s);
```
`sdsAllocSize`这个接口用与返回分配给指定`sds`数据的内存的总大小。
其中包含:
1. `sds`指针前的`Header`的大小
2. 缓存中已使用数据的大小
3. 可用空间的大小
4. 结束符`\0`的大小

这个接口与`sdsalloc`的区别是，`sdsalloc`返回的是`sdshdr.buf`分配的缓存的大小。

```c
void* sdsAllocPtr(sds s);
```
`sdsAllocPtr`这个接口返回一个`sds`数据直接被分配的头指针，也就是*Header*的指针。
```c
void sdsIncrLen(sds s, ssize_t incr);
```
可以理解为可以给指定的`sds`的`sdshdr.len`增加`incr`的长度，同时会导致`sdsavail`的可用空间减少，
该接口只负责处理`sds`数据的长度，而不会改动其内容。
基本应用场景:
1. 调用`sdsMakeRoomFor`函数为sds扩容
2. 向sds的缓存之中写入数据
3. 调用`sdsIncrLen`函数，调整写入数据之后的sdslen长度
```c
    oldlen = sdslen(s);
    s = sdsMakeRoomFor(s, BUFFER_SIZE);
    nread = read(fd, s+oldlen, BUFFER_SIZE);
    /* ... check for nread <= 0 and handle it ... */
    sdsIncrLen(s, nread);
```
这个接口与`sdsinclen`类似，但是该接口更多的是提供给用户调用，其内部增加了长度校验机制，
这就需要我们在调用前通过`sdsMakeRoomFor`接口来确保可用缓存空间，
或者手动检查`incr`的大小，不能超过`sdsavail`的大小，否则会触发断言机制。

***
## 用于处理Strings内容的操作函数
```c
sds sdsgrowzero(sds s, size_t len);
```
`sdsgrowzero`内部通过调用`sdsMakeRoomFor`将`sds`的缓存区增加`sdshdr.len`的长度，
同时将新增的缓冲区初始化为0。

```c
sds sdscatlen(sds s, const void *t, size_t len);
sds sdscat(sds s, const char *t);
sds sdscatsds(sds s, const sds t);
```
上面这三个接口函数都是用来执行`sds`数据的连接操作。
`sdscatlen`会向`sds`数据中扩展一段二进制安全的数据，这个上述三个函数的基础，
其余两个函数都是通过调用`sdscatlen`来实现的。
其内部会通过调用`sdsMakeRoomFor`尝试将`sds`的缓冲区进行扩展。
而函数`sdscat`会向`sds`中连接一个C风格字符串数据，`sdscatsds`函数则会执行两个`sds`数据的连接操作。
值得注意的是，上面三个函数均未对*target*数据执行释放操作，调用者需要根据自己的需求对*target*数据执行响应的操作。

```c
sds sdsjoin(char **argv, int argc, char *sep);
sds sdsjoinsds(sds *argv, int argc, const char *sep, size_t seplen);
```
上述两个函数与`sdscat`系列接口类似，是执行一组C风格字符串或者`sds`数据的连接操作，
同时会以`sep`作为分隔符。

```c
sds sdscatrepr(sds s, const char *p, size_t len);
```


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
将一个整形数字写入一个字符串。我们已知，在*Redis*之中，没有专门的数值类型，整形数字也是需要使用*String*类型来保存的，
而这个函数主要用于，将一个整形数字转化为*Redis*自定义的数据类型：
```c
robj *createStringObjectFromLongLongWithOptions(long long value, int valueobj) {
    robj *o;
    ...
    o = createObject(OBJ_STRING,sdsfromlonglong(value));  
    ...
}
```
上述代码便是用过调用`sdsfromlonglong`接口将一个整数转化为*Redis*的字符串对象的。

```c
sds sdscatvprintf(sds s, const char *fmt, va_list ap);
sds sdscatprintf(sds s, const char *fmt, ...);
sds sdscatfmt(sds s, char const* fmt, ...);
```
类似使用`printf`的调用的方式，将一个格式化C风格字符串添加到sds的结尾
需要注意的是，作为参数传入的s，在函数执行成功后已不能保证有效性，
应使用`s = sdscatprintf(s. "1");`这样的方式进行操作。
另外`sdscatfmt`函数是是一个与`sdscatprintf`类似的接口，但是其执行的速度要更快
原因在与其没依赖与*libc*库所提供的`sprintf()`族函数，而这一组操作通常是很慢的。
但是其只支持特定的几种通配符：
| 通配符 | 含义 |
|:------|:-----|
|`%s`|C风格字符串|
|`%S`|SDS字符串|
|`%i`|signed int 有符号整数|
|`%I`|64位有符号整数|
|`%u`|unsigned int 无符号整数|
|`%U`|64位无符号整数|
|`%%`|用于打印%符号|

```c
sds sdstrim(sds s, const char *cset);
```
接口`sdstrim`给定一个C风格的字符集合`cset`，以及一个特定的`sds`数据，移除这个`sds`中前后属于这个`cset`的字符，
并返回更新后的`sds`数据。需要注意的是：
```c
    sp = start = s;
    ep = end = s+sdslen(s)-1;
    while(sp <= end && strchr(cset, *sp)) sp++;
    while(ep > sp && strchr(cset, *ep)) ep--;
```
通过代码我们可以了解到该接口的移除操作在遇到第一个不属于`cset`的字符时结束。
其中一些代码示例：
```c
void loadServerConfigFromString(char *config) {
    ...
    lines[i] = sdstrim(lines[i]," \t\r\n");
    ...
}
```
在*Redis*中该接口的主要用途是用于移除空格、换行符、制表符等空字符。

```c
void sdsrange(sds s, sszie_t start, ssize_t end);
```
`sdsrange`给定一个特定的`sds`数据，以及一个*range*的`[start, end]`，将这个`sds`数据缩小成符合这个*range*的子串。
*range*的范围可以是负值，其含义可以参考*Python*中对于列表的切片操作。

```c
void sdstolower(sds s);
void sdstoupper(ssd s);
```
上述两个操作是给定一个`sds`数据，对于其内部的每一个字符调用`tolower`或者`toupper`操作。

```c
sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen);
```
`sdsmapchars`接口用于对一个`sds`数据执行字符替换，`from`为匹配串，`to`为替换串，
```c
    sds s = sdsnew("hello");
    s = sdsmapchars(s, "ho", "01", 2);
    printf("%c\n", s);
    //output 0ell1
```

```c
int sdscmp(const sds s1, const sds s2);
```
`sdscmp`接口内部通过调用`memcmp`来比较两个`sds`数据的大小：
* 如果`s1 > s2`，返回一个正数。
* 如果`s1 < s2`，返回一个负数。
* 如果两个`sds`的二进制数据完全相同，那么这个函数返回0。

```c
sds *sdssplitlen(const char *s, ssize_t len, const char *sep, int seplen, int *count);
void sdsfreesplitres(sds *tokens, int count);
sds *sdssplitargs(const char *line, int *argc);
```


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*