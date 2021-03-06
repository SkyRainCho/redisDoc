

# Redis中压缩链表的实现

前面我们已经介绍了*Redis*中所实现的经典双端链表了，而这种链表存在一个问题，便是在存储小数据的时候，
内存使用效率过低，例如当一个链表节点中只保存一个字节的`unsigned char`数据时，我们需要为这个节点保存24个字节的额外数据，
其中包含`listNode.prev`指针，`listNode.next`指针，以及指向具体数据的`listNode.value`指针，
同时对于链表节点所占用内存的反复申请与释放，也容易导致内存碎片的产生。
为了解决经典双端链表在保存小数据而导致内存效率过低的问题，*Redis*设了一套压缩链表的数据数据结构*ziplist*来对这种场景下的链表应用进行优化。

同时在本文对应的*5.0.8*版本的*Redis*源码中，压缩链表将作为列表对象（也就是使用*LPUSH*或者*LPOP*命令操作的对象）的底层实现之一，
而经典的双端链表已经不再作为链表对象的底层实现方式。另外在*Redis*中，压缩链表同时也是哈希对象的底层实现之一，而且根据笔者现在看到内容，
初始的哈希对象都是采用压缩链表作为底层实现的，只有当哈希对象中*key-value*对的数量超过`OBJ_HASH_MAX_ZIPLIST_ENTRIES`时，
*Redis*才会将压缩链表实现的哈希对象转化为由*src/dict.h*中所描述的`dict`类型作为底层实现。
而悲剧的一点是，前一篇文章中讲解的*zipmap*数据结构，貌似在*Redis*中已经不再被使用，不过能从其中体会作者的设计思路，也未尝不是一件坏事。

## 压缩链表概述
压缩链表是一种专门为了提升内存使用效率而设计的，经过特殊编码的双端链表数据结构。
既可以用来保存整形数值，也可以用来保存字符串数值，为了节约内存，同时也是体现压缩之含义，
当保存一个整形数值时，压缩链表会使用一个真正的整形数来保存，而不是使用字符串的形式来存储。
这一点很容易理解，一个整数可以根据其数值的大小使用1个字节，2个字节，4个字节或者8个字节来表示，
如果使用字符串的形式来存储的话，其所需的字节数大小一定不小于使用整形数所需的字节数。

压缩链表允许在链表两端以 *O(1)* 的时间复杂度执行 *Pop* 或者 *Push* 操作，当然这只是一种理想状态下的情况，
由于压缩链表实际上是内存中一段连续分配的内存，因此这些操作需要对压缩链表所使用的内存进行重新分配，
所以其真实的时间复杂度是和链表所使用的内存大小相关的。

### 压缩链表的内存分布

压缩链表与经典双端链表最大的区别在于，双端链表的节点是分散在内存中并不是连续的，压缩链表中所有的数据都是存储在一段连续的内存之中的，
![压缩链表内存分布](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%8E%8B%E7%BC%A9%E9%93%BE%E8%A1%A8%E5%86%85%E5%AD%98%E5%88%86%E5%B8%83.PNG)

上图便是*Redis*中一个压缩链表在内存之中的分布情况，本质上，压缩链表并没有一个专门的数据结构用于描述它，所有的相关操作都是通过指针与解码出来的偏移进行的。
在分配给压缩链表的内存中，有几个字段用于描述压缩链表的长度信息与结束标记，其中
- `<zlbytes>`，该字段固定是一个四字节的无符号整数，用于表示整个压缩链表所占用内存的长度（以字节为单位），这个长度数值是包含这个`<zlbytes>`本身的。
- `<ztail>`，该字段固定是一个四字节的无符号整数，用于表示在链表中最后一个节点的偏移字节量，借助这个字段，我们不需要遍历整个链表便可以在链表尾部执行*Pop*操作。
- `<zllen>`，该字段固定是一个两个字节的无符号整数，用于表示链表中节点的个数。但是该字段最多只能表示`2^16-2`个节点个数；超过这个数量，也就是该字段被设置为`2^16-1`时，
意味着我们需要遍历整个链表，才可以获取链表中节点真实的数量。
- `<entry>`，该字段表示链表中的一个节点，同一个链表中的节点，其长度大概率是不同的，因此需要特殊的方式来获取节点的长度，具体的内容会在下一个部分详细介绍。
- `<zlend>`，该字段可以被认为是一个特殊的`<entry>`节点，用作压缩链表的结束标记，只有一个字节，存储着`0xFF`，一旦我们遍历到这个特殊的标记，便意味着我们完成了对这个压缩链表的遍历。

### 链表结点的内存分布

在压缩链表中每一个节点数据在其前缀都包含着两个对于节点而言重要的数据信息，
- `<prevlen>`，这个字段标记了该节点的前序节点的长度，以便我们可以向链表的头部反向遍历链表，这个字段的编码格式有两种：
- `<encoding>`，这个字段用于表示该节点使用的编码方式，具体是按照整形数进行编码还是按照字符串进行编码，
当节点使用的是字符串编码时，该字段还会指明字符串数据的字节长度。

这些字段中所存储的数据都是采用小端字节序进行存储的，哪怕代码是在大端字节序的系统上进行编译的，*Redis*对小端字节序的相关操作定义在*src/endianconv.h*这个头文件中。
> 大端字节序：高位字节在前，低位字节在后。
>
> 小端字节序：低位字节在前，高位字节在后。
>
> 大端字节序是符合大部分人从左到右的阅读顺序的，当然对于类似犹太人从右到左书写希伯来文的习惯来说，可能小端字节序更符合他们的阅读顺序。

抽象一些来说，这两个字段在一定意义上起到了经典双端链表中`prev`以及`next`指针的作用。通过解码一个给定节点的这两个字段中存储的数据，
我们可以指针移动的方式，实现对链表正向以及反向的遍历。

通常一个压缩链表节点包含三段数据，除了上面的两个标记字段之外，还有数据内容本身，

但是如果数据是一个很小的整数（0到12）的情况下，节点只需要两个字段，
其中`<encoding>`字段本身就在表示长度与编码信息之外，额外存储这个数据信息。

对于`<prevlen>`字段，需要注意下面两点：
1. 当前节点的前序节点的长度为0到253个字节时，`<prevlen>`字段只需要一个8位的无符号整数，便可以编码对应的长度。
2. 当前节点的前序节点的长度大于等于254个字节时，`<prevlen>`字段则需要5个字节，其中第一个字节会被设置成`0xFE`也就是254，
这是一个特殊标记，用于表明，这里保存了一个比较大的数值，需要继续读取后续的4个字节以解码出前序节点的长度。
而为什么不选择`0xFF`作为特殊标记的原因在于，`0xFF`是整个链表结束的标记。

对于`<encoding>`字段，根据第一个字节的前两个比特，可以确定是当前的节点是使用整形数编码还是按照字符串进行编码的：
1. 当前两个比特都为1的时候，也就是出现`[11XXXXXX]`这样的情况时，表明当前节点是使用整形数进行编码的，
    - `[11000000]`，表明节点的数据是按照16位整数进行编码的，需要额外的2个字节作为`<entry-data>`来存储数据。
    - `[11010000]`，表明节点的数据是按照32位整数进行编码的，需要额外的4个字节作为`<entry-data>`来存储数据。
    - `[11100000]`，表明节点的数据是按照64位整数进行编码的，需要额外的8个字节作为`<entry-data>`来存储数据。
    - `[11110000]`，表明节点的数据是按照24位整数进行编码的，需要额外的3个字节作为`<entry-data>`来存储数据。
    - `[11111110]`，表明节点的数据是按照8位整数进行编码的，需要额外的1个字节作为`<entry-data>`来存储数据。
    - `[1111XXXX]`，这种方式表明是使用`<encoding>`字段直接来存储数据的小整数，也就是使用`<encoding>`字节的后4个比特来表示。
    前面已经给出了，这样的小整数的取值范围在0到12之间，但是实际存储在`<encoding>`中的数值范围是1到13，
    这是由于，如果用0进行编码的话，其二进制的表示为`[11110000]`，这样会与24位整数编码的`<encoding>`字段冲突；
    而采用加1的方式来避免字段冲突的话，也就意味着最大可以使用的数值为`[11111101]`，也就是13（实际数值为12），
    因为`[11111110]`已经被用作8位整数编码的标记。
2. 当`<encoding>`字段首字节的前两个比特不是`11`的情况时，则表明该节点的数据是以字符串的形式进行编码的。
    - `[00PPPPPP]`，`<encoding>`字段占用1个字节，可以用来表示长度不超过63个字节的字符串，
    也就是使用`<encoding>`剩余的6个比特来表示字符串长度。
    - `[01PPPPPP|QQQQQQQQ]`，`<encoding>`字段会占用2个字节，使用首字节剩余的6比特以及额外的1个字节，
    可以表示长度不超过16383个字节的字符串。
    - `[10000000|PPPPPPPP|RRRRRRRR|SSSSSSSS|TTTTTTTT]`，当字符串的长度超过16383个字节时，就应该使用这种`<encoding>`的格式，
    使用额外的4个字节，最多可以表示长度不超过`2^32-1`的字符串。

假设一个压缩链表中保存着2和5这两个元素，那么其在内存中的分布为：

![具体的压缩链表](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%85%B7%E4%BD%93%E7%9A%84%E5%8E%8B%E7%BC%A9%E9%93%BE%E8%A1%A8.PNG)

在这个压缩链表的内存分布中，
1. `<zlbytes>`按照小字节序解码为`0x0F`，表示这个压缩链表占用15个字节的内存。
2. `<zltail>`按照小字节序解码为`0x0C`，表示最后一个节点距离压缩链表的起点的偏移为12个字节，通过这个字段相当于指明了经典链表中的`list.tail`指针。
3. `<zllen>`按照小字节序解码为`0x02`，表示在这个压缩链表中存在两个节点，通过这个字段在一定程度上相当于指明了经典链表中的`list.len`字段（一定程度是因为压缩链表能记录的节点个数是有限的）。
4. `<entry>`由于压缩链表的前3个字段都是固定长度，那么给定压缩链表的指针，向后偏移10个字节，便是链表中指向第一个节点的指针，相当于指明了经典链表中的`list.head`指针。
    - 第一个节点，作为第一个节点，`<prevlen>`为0，`<encoding>`字段用二进制的形式表示为`[11110011]`，我们可以确定这是一个使用`<encoding>`本身存储小整数数据的节点，解码后的数值为2。
    - 第二个节点，前一个节点的长度为2个字节，因此`<prevlen>`存储的数值为`0x02`，`<encoding>`字段和第一个节点是使用相同的编码方式，解码后的数值为5.
5. `<zlend>`作为压缩链表的结束标记。

由于压缩链表中的每一个节点都是保存在内存中的一段连续的二进制数据，相关的操作需要涉及到大量的指针操作，并不是非常直观，
因此*Redis*的作者专门设计了一个结构体，用于描述一个链表节点：
```c
typedef struct zlentry {
    unsigned int prevrawlensize;    //记录<prevlen>字段本身的字节长度
    unsigned int prevrawlen;        //记录保存在<prevlen>字段中前序节点的长度
    unsigned int lensize;           //记录<encoding>字段本身所占用的字节长度
    unsigned int len;               //记录当前entry中data数据所占用的长度，可以为0，表示存储在<encoding>中的小整数
    unsigned int headersize;        //记录当前entry中header的长度，等于prevrawlensize + lensize
    unsigned char encoding;         //记录当前entry中数据的编码方式
    unsigned char *p;               //保存指向当前entry起始位置的指针，而这个指针指向的是当前节点的<prevlen>字段
} zlentry;
```
同时*Redis*还给出了一个函数接口
```c
void zipEntry(unsigned char *p, zlentry *e);
```
通过这个函数，我们从指针`p`所指向的节点内存中解码出`zlentry`数据，需要注意的是，`zlentry`并非是压缩链表节点的实际存储方式，
它的作用仅是在函数调用中可以方便直观的获取节点的相关长度信息。

## 压缩链表的基础操作
在*src/ziplist.c*源文件中，作者给出不同编码类型的定义，可以对应上一部分的`<encoding>`编码类型：
```c
#define ZIP_STR_MASK 0xc0
#define ZIP_INT_MASK 0x30
#define ZIP_STR_06B (0 << 6)
#define ZIP_STR_14B (1 << 6)
#define ZIP_STR_32B (2 << 6)
#define ZIP_INT_16B (0xc0 | 0<<4)
#define ZIP_INT_32B (0xc0 | 1<<4)
#define ZIP_INT_64B (0xc0 | 2<<4)
#define ZIP_INT_24B (0xc0 | 3<<4)
#define ZIP_INT_8B 0xfe
```
通过宏定义：
```c
#define ZIP_IS_STR(enc) (((enc) & ZIP_STR_MASK) < ZIP_STR_MASK)
```
我们可以判断一个节点的是否是按照字符串的方式进行编码的。

### 基础宏定义

作者在源文件中，给出了若干宏定义，用于处理压缩链表的简单操作
|宏|含义|
|--|---|
|`#define ZIPLIST_BYTES(zl)`|获取压缩链表的字节数(小字节序)|
|`#define ZIPLIST_TAIL_OFFSET(zl)`|获取最后一个节点在压缩链表内的偏移(小字节序)|
|`#define ZIPLIST_LENGTH(zl)`|从`<zllen>`字段获取压缩链表节点的数量|
|`#define ZIPLIST_ENTRY_HEAD(zl)`|给点一个压缩链表，将指针向后偏移固定的6个字节，返回指向第一个节点的指针|
|`#define ZIPLIST_ENTRY_TAIL(zl)`|给定一个压缩链表，返回指向最后一个节点的指针|
|`#define ZIPLIST_ENTRY_END(zl)`|给定一个压缩链表，返回指向最后一个结束标记`<zlend>`的指针|
|`#define ZIPLIST_INCR_LENGTH(zl,incr)`|给定一个压缩链表，向`<zllen>`字段中增加`incr`的数量|

### 压缩链表长度操作

```c
unsigned int zipIntSize(unsigned char encoding);
```
这个接口用于获取一个给定的整数编码格式所需的额外的`<entry-data>`存储空间，例如`ZIP_INT_8B`编码方式，会返回1个字节的空间去存储节点的整数值；而处于`ZIP_INT_IMM_MIN`以及`ZIP_INT_IMM_MAX`之间的`encoding`由于是使用`encoding`本身来存储数据的，因此会返回0。



```c
int zipTryEncoding(unsigned char *entry, unsigned int entrylen, long long *v, unsigned char *encoding);
```

这个函数用于判断`entry`指针以及`entrylen`所描述的内存中保存的字符串是否是整形数。如果是整形数，那么会通过`v`返回这个整型数值，并通过`encoding`返回这个整数对应的编码，同时函数返回1；如果这段内存中的字符串无法被转化整数，那么这个函数会返回0，表示这个是一个字符串形式的数据。



```c
unsigned int zipStorePrevEntryLength(unsigned char *p, unsigned int len);
unsigned int zipStoreEntryEncoding(unsigned char *p, unsigned char encoding, unsigned int rawlen);
```
上面这两个接口函数功能比较类似，分别是用来处理`<prevlen>`字段数据以及`<encoding>`字段数据的函数。

其中`zipStorePrevEntryLength`函数主要用于给定`len`长度数据，计算其编码压缩链表元素`<prevlen>`字段所需的长度，并通过该函数返回；如果传入了`p`指针，那么会将数据编码到`p`指针指向的内存中。

而`zipStoreEntryEncoding`这个函数主要有两个用途：

1. 不传入指针`p`，那么这个函数会返回给定的`encoding`参数以及`rawlen`参数所需的的`<encoding>`字段的长度。
2. 传入指针`p`，那么这个函数会将`encoding`以及`rawlen`数据存在到`p`指针所指向的内存中。

需要注意的细节是，如果传入指针`p`参数，那么`p`应该指向的是压缩链表节点中的`<encoding>`字段。

```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen)
{
    ...
    reqlen += zipStorePrevEntryLength(NULL, prevlen);
    reqlen += zipStoreEntryEncoding(NULL, encoding, slen);
    ...
    zl = ziplistResize(zl, curlen+reqlen+nextdiff);
    ...
    
    p += zipStorePrevEntryLength(p, prevlen);
    p += zipStoreEntryEncoding(p, encoding, slen);
    ...
    
}
```

如上述代码段所示，上述的两个函数通常会搭配在一起使用，用于初始化压缩链表节点的头部数据，一般情况会先进行一次`p`为`NULL`的调用，以获取待操作数据的所对应的头部节点数据的长度，然后在进行一次传入有效指针`p`的调用，将相关数据写入压缩链表的节点中。



```c
int zipStorePrevEntryLengthLarge(unsigned char *p, unsigned int len);
```

这个函数与`zipStrorePrevEntryLength`函数类似，专门用于处理长度超过253个字节的`<prevlen>`数据的处理。



```c
#define ZIP_ENTRY_ENCODING(ptr, encoding)
#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len)
```

`ZIP_ENTRY_ENCODING`这个宏定义用于从`<encoding>`对应的`ptr`指针中，解码出这个对应的压缩链表元素所使用的编码方式。

`ZIP_DECODE_LENGTH`这个宏定义用于从一个`<encoding>`字段的起始指针`ptr`中，解码出这个对应的压缩链表元素所使用的编码方式`encoding`，`<encoding>`字段中表示后续数据长度的字节的长度`lensize`，以及这个对应节点元素的数据占用的内存长度`len`。



```c
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize)
#define ZIP_DECODE_PREVLEN(ptr, prevlensize, prevlen)
```
`ZIP_DECODE_PREVLENSIZE`这个宏定义用于从`<prevlen>`字段对应的指针`ptr`中，解码出`<prevlen>`字段所需要的长度，1个字节或者5个字节。`ZIP_DECODE_PREVLEN`这个宏定义用于从`<prevlen>`字段对应的指针`ptr`中，解码出字段长度，以及其中存储的长度数值。



```c
int zipPrevLebByteDiff(unsigned char *p, unsigned int len);
```
这个函数用于计算一个压缩链表节点的`<prevlen>`中的长度值与`len`的长度差值。

```c
unsigned int zipRawEntryLength(unsigned char *p);
```
这个函数会通过调用`ZIP_DECODE_PREVLENSIZE`以及`ZIP_DECODE_LENGTH`来计算一个压缩链表节点的占用内存的长度。



### 压缩链表底层的插入、删除与连锁更新等操作


```c
void zipSaveInteger(unsigned char *p, int64_t value, unsigned char encoding);
int64_t zipLoadInteger(unsigned char *p, unsigned char encoding);
```
函数`zipSaveInteger`用于根据给定的编码方式`encoding`，将`value`数值存储在`p`指针的位置中。
而函数`zipLoadInteger`则是根据给定的编码方式`encoding`，从`p`指针指向的内存中，解码出相应的整数数值。
这两个函数中的`p`指针都是指向`<entry>`节点中的`<entry-data>`字段，而非是`<entry>`的起始位置。



```c
unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p);
```
上面这个函数`__ziplistCascadeUpdate`便是处理压缩链表连锁更新的函数，而导致连锁更新的原因在于压缩链表中元素`<entry>`中的`<prevlen>`字段是变长的，当前元素的前一个元素的长度如果小于等于253个字节，那么`<prevlen>`字段只需要1个字节，否则就需要5个字节的存储空间。这也就意味着，压缩链表中一个元素`A`的内容所发生的变化，有可能会对`A`的后续元素`B`的内容产生影响，而`B`元素的变化则有可能对后续`C`元素的内容产生影响，以此类推，这便是产生连锁更新的原因。

理想化的情况，对压缩链表中`A`节点的修改所导致的元素长度变化，恰好没有超越其后续元素`B`的`<prevlen>`字段所能表达的范围，那么变不会发生连锁更新的情况。

而最坏的情况，则是对压缩链表中`A`元素的变化，会导致这个变化一直传递到链表的结尾，例如在链表中存在`A->B->C-D->E`这5个元素，每个元素的长度都在250到253之间，这样每个元素节点的`<prevlen>`字段只要1个字节，就可以表示前序元素的长度。当我们将第一个元素`A`的长度扩展到253以上时，会导致后续的`B`节点`<prevlen>`字段被扩展成5个字节，这样`B`节点的长度也超过了253个字节，这种变化会导致`C`节点被更新，进而又会波及到`D`以及`E`两个节点。



```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen);
unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num);
```
1. 函数`__ziplistInsert`用于实现压缩链表的底层插入操作，其中`s`指针以及`slen`用于标记待插入的字符串数据，如果这个字符串数据可以被转化整数形式，该接口会用按照整数进行插入，否则的话，会按照字符串的形式进行插入，而`p`则是标记了插入的位置。
2. 函数`__ziplistDelete`用于处理压缩链表的底层删除操作，其中`p`为被删除的起始节点指针，`num`则表示需要被删除的节点的个数。

上述两个函数都会调用`ziplistResize`对整个压缩列表所占用的内存进行扩容或者缩容，因此所传入的`zl`参数可能会在调用结束后失效，应该使用函数返回的指针作为更新后的压缩链表的参数，而当执行完插入或者删除操作之后，会调用`__ziplistCascadeUpdate`来对`p`位置操作后的后续节点尝试进行连锁更新，刷新后续节点的`<prevlen>`字段。



## 压缩链表的用户接口

### 初始化与长度获取

```c
unsigned char *ziplistNew(void);
unsigned char *ziplistResize(unsigned char *zl, unsigned int len);
unsigned int ziplistLen(unsigned char *zl);
size_t ziplistBlobLen(unsigned char *zl);
```

1. `ziplistNew`函数用于返回一个空的，只包含`<zlbytes>`、`<zltail>`、`<zllen>`以及`<zlend>`字段的数据，从它的代码实现中，我们也是可以窥探到其在内存中的分布格式：
```c
unsigned char *ziplistNew(void)
{
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

2. `ziplistResize`用于在更新压缩链表的大小，这个函数用于在压缩链表执行插入、删除以及连锁更新的时候，通过调用`zrealloc`去更新压缩链表占用的内存的大小，并将更新后的内存大小写入`<zlbytes>`字段中。

3. 函数`ziplistLen`函数则是会获取给定压缩列表中节点的个数，该函数首先会判断`<zllen>`字段中存储的节点个数，如果其存储的个数小于`UINT16_MAX`，那么这便是实际的个数值；否则会对整个链表进行遍历，以获取其中节点的真实个数：
```c
unsigned int ziplistLen(unsigned char *zl)
{
    unsigned int len = 0;
    ...
    unsigned char *p = zl+ZIPLIST_HEADER_SIZE;
    while (*p != ZIP_END)
    {
        p += zipRawEntryLength(p);
        len ++;
    }
    ...
    return len;
}
```
这里代码给出对于压缩链表节点遍历的一种范式，通过`zipRawEntryLength`来计算指针`p`对应的节点长度，将指针向后移动相应的长度，便实现了移动到下一个节点的操作。

4. `ziplistBlobLen`函数则是通过解码`<zlbytes>`中的数据获取整个压缩链表所占用的内存大小。

### 插入与删除

```c
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where);
unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slend);
```

这两个函数主要用于对压缩链表执行插入操作，其底层都是使用`__ziplistInsert`函数实现的插入操作，其中：

1. `ziplistPush`用于实现向压缩链表的两端进行插入操作。`where`参数用于标记是向链表的头部还是尾部进行插入，在*src/ziplist.h*头文件中定义了`ZIPLIST_HEAD`以及`ZIPLIST_TAIL`分别用于表示链表头部和链表尾部。
2. `ziplistInsert`用于实现向压缩链表的指定位置进行插入操作。



```c
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p);
unsigned char *ziplistDeleteRange(unsigned char *zl, int index, unsigned int num);
```

上面两个函数都是用于执行从压缩链表中删除节点的操作，底层都是使用`__ziplistDelete`函数来实现实际的删除操作：

1. `ziplistDelete`函数用于删除给定的单一节点，这个函数会用过`p`指针来指定需要被删除的节点，并返回删除节点后新的链表指针，同时会原地更新`p`指针，通过这种实现方式，可以实现在删除节点的同时，遍历整个链表：

   ```c
   while (ziplist(p, &entry, &elen, &value))
   {
     ...
     zl = ziplistDelete(zl, &p);
     p = ziplistPrev(zl, p)
     ...
   }
   ```

   

2. `ziplistDeleteRange`函数用于从压缩链表的给定索引`index`开始，删除`num`个节点，并返回删除之后新的压缩链表指针。

### 遍历与查找获取

```c
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p);
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p);
```

上述两个函数是压缩链表中对节点实现正向与反向遍历的方法，相当于使用`listNode.prev`以及`listNode.next`对链表执行遍历：

1. `ziplistNext`用于返回压缩链表`zl`中`p`节点的下一个节点的指针，通过`p += zipRawEntryLength(p)`来实现指针的移动。

2. `ziplistPrev`用于返回压缩链表`zl`中`p`节点的前一个节点的指针，其实现细节为：

   * `p`节点为`ZIP_END`节点，那么使用`ZIPLIST_ENTRY_TAIL`来返回最后一个节点的指针。
   * 如果`p`节点为压缩链表中的第一个节点，那么该函数返回`NULL`。
   * `p`节点处于链表中，那么通过`ZIP_DECODE_PREVLEN`来获取`<prevlen>`存储的长度信息，将`p`指针向前移动，便可以找到其前一个节点。

   

```c
unsigned char *ziplistIndex(unsigned char *zl, int index);
unsigned int ziplistGet(unsigned char *p, unsigned char **sval, unsigned int *slen, long long *lval);
unsigned char *ziplistFind(unsigned char *zl, unsigned char *vstr, unsigned int vlen, unsigned int skip)
unsigned int ziplistCompare(unsigned char *p, unsigned char *s, unsigned int slen);
```

上述的四个接口函数的作用是从压缩列表之中获取或这个查找某一个元素的，

1. `ziplistIndex`接口从压缩链表中返回索引为`index`的节点的指针，该函数既支持正数的`index`索引，用于从前向后遍历；同时也支持负数的`index`索引，用于从后向前遍历。
2. `ziplistGet`函数用于从`p`指针所指向的链表节点中解析出其存储的数据，如果该节点的数据是按照字符串进行编码的话，那么会通过`sval`以及`slen`字段进行返回；如果是按照整数方式进行编码的话，那么会通过`lval`数值进行返回。
3. `ziplistFind`用于从`p`指针对应的节点开始，以`skip`为跳数，来查找是否存在所存储的数据等于`vstr`的节点。如果找到，那么会返回这个节点的指针，否则会返回`NULL`。如果这个`skip`传入0的话，那么意味着按照顺序从`p`节点开始依次向后查找，不会跳过任何一个节点。
4. `ziplistCompare`函数则会用于比较`p`指针对应的压缩链表节点中的数据是否等于`sstr`以及`slen`参数描述的数据。在执行比较操作时，会根据`p`节点的编码形式，如果是字符串编码，那么直接调用`memcmp`系统函数进行比较；如果是整数编码，则会对对应的数据进行解码，然后执行整数比较的操作。最终的结果如果相同，那么函数返回1，否则函数返回0。

### 其他操作

```c
unsigned char *ziplistMerge(unsigned char **first, unsigned char **second);
```

`ziplistMerge`函数用于合并两个链表，将`second`链表中的内容会被连接到`first`链表的内容之后，合并成功后，该函数会返回指向新的链表的指针。其中**较长**的链表将会被重新分配内存以存储所有的节点元素，这个**较长**的链表`target`既可以是`first`也可以是`second`，而另外一个链表的内容在合并之后，将会被释放掉。

```c
if (first_len >= second_len)
{
  target = *first;
  target_bytes = first_bytes;
  source = *second;
  source_bytes = second_bytes;
  append = 1;
}
else
{
  target = *second;
  target_bytes = second_bytes;
  source = *first;
  source_bytes = first_bytes;
  append = 0;
}
```



1. 如果`first`链表是**较长**链表`target`的话，那么会通过`memcpy`调用，将**较短**链表的链表节点拷贝到`target`链表中。

   ```c
   memcpy(target+target_bytes-ZIPLIST_END_SZIE, source+ZIPLIST_HEADER_SIZE, 
         source_bytes - ZIPLIST_HEADER_SIZE);
   ```

2. 如果`second`链表是**较长**链表`target`的话，那么首先会调用`memmove`，将`second`链表中的元素移动到链表尾部，以便为`first`链表中的节点预留空间；然后调用`memcpy`函数，将`first`链表中的节点拷贝到新的链表中。

   ```c
   memmove(target+source_bytes-ZIPLIST_END_SIZE, target+ZIPLIST_HEADER_SIZE,
          target_bytes - ZIPLIST_HEADER_SIZE);
   memcpy(target, source, source_bytes - ZIPLIST_END_SIZE);
   ```

最后由于执行了两个链表的连接操作，那么需要对连接处节点调用`__ziplistCascadeUpdate`的连锁更新，以维护`<prevlen>`字段数据的正确性。

```c
target = __ziplistCascadeUpdate(target, target+first_offset);
```



***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*