# Redis中快速链表的实现

根据前文的分析，我了解到，基于经典双端链表的一些缺点，双端链表已经不在作为*LIST*对象的底层实现了。但是作为*LIST*的底层实现之一，压缩链表同样具有一定的劣势。压缩链表由于使用的是有一段连续的内存，这就意味着，执行插入操作导致内存大小发生变化时，便会引起一次内存的重新分配过程，同时还会执行一定量的数据移动。特别时对于压缩链表较长的情况下，大批量的数据移动势必会降低系统的性能。而在上述这个方面恰恰就是经典双端链表的优势，故此*Redis*设计实现了一种快速链表的数据结构，兼具经典双端链表与压缩链表的特点，实现了时间与空间上的一种折衷。

相信了解*STL*的人对上述的描述会有一种似曾相识的感觉。没错，*STL*中的*deque*便是使用上述思想实现的一种既可以在队列两端快速执行插入与删除操作，同时可以根据下标对数据进行随机访问的数据结构。

## 快速链表概述

在*Redis*中，作者结合了经典双端链表以及压缩链表的特性，实现了快速链表的数据结构。简单来说，使用双端链表的形式描述整个快速链表，而每一个快速链表的节点都是一个压缩链表作为底层实现，可以存储若干个数据节点。同时基于链表数据结构通常对头部与尾部的访问最为频繁，而对链表中间的数据访问并不是特别频繁，因此出于节省空间的目的，会对中间节点底层压缩链表所使用的内存进行压缩，*Redis*使用**LZF**算法对内存进行压缩，相应的算法被定义在*src/lzf.h*，*src/lzfp.h*，*src/lzf_c.c*，*src/lzf_d.c*这四个文件中，具体的压缩算法不属于*Redis*代码的核心，因此不做过多的讲解，有兴趣的同学可以自行研究，这里只给出压缩与解压的接口*API*：

```c
unsigned int lzf_compress(const void *const in_data, unsigned int in_len, void *out_data, unsigned int out_len);
unsigned int lzf_decompress(const void *const in_data, unsigned int in_len, void *out_data, unsigned int out_len);
```

其中快速链表节点`quicklistNode`的数据结构如下所示：
```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;
    unsigned int count : 16;
    unsigned int encoding : 2;
    unsigned int container : 2;
    unsigned int recompress : 1;
    unsigned int attempted_commpress : 1;
    unsigned int extra : 10;
} quicklistNode;
```
在这个数据结构之中，
* `quicklistNode.prev`和`quicklistNode.next`这两节点作为双端链表的必要成员变量，用于指向当前节点的前序节点以及后续节点。
* `quicklistNode.zl`，这个成员变量用于指向存储数据的内存，内存中可以是一个压缩链表，也可以是一个经过*LZF*算法压缩过的数据。
* `quicklistNode.sz`，这个成员变量永远都会存储`quicklistNode`中压缩链表所占用的内存的长度，即使`quicklistNode.zl`中存储的是经过*LZF*算法压缩后的数据。
* `quicklistNode.count`，用于存储在这个节点中的压缩率链表里数据节点的个数。
* `quicklistNode.encoding`，用于表示这个节点中存储的数据是原始的压缩链表，还是经过*LZF*压缩过的数据，对于成员变量，*Redis*给出了两个定义，`QUICKLIST_NODE_ENCODING_RAW`用于表示当前`quicklistNode.zl`中存储的是原始的压缩链表；`QUICKLIST_NODE_ENCODING_LZF`用于表示当前`quicklistNode.zl`中存储的是经过*LZF*压缩的数据。
* `quicklistNode.container`，用于表示当前这个节点所使用的容器类型，*Redis*给出两个容器类型的定义，`QUICKLIST_NODE_CONTAINER_NONE`，这个定义在代码中并未使用；`QUICKLIST_NODE_CONTAINER_ZIPLIST`，这个定义便是当前节点是使用压缩链表进行数据存储的。
* `quicklistNode.recompress`，用于表示当前的节点是否处于临时被解压状态。

在本篇文章中，将会使用**链表节点**来表示`quicklistNode`，同时使用**数据节点**来表示存储在`quicklistNode`的压缩链表种的数据。

如果`quicklistNode.encoding`是`QUICKLIST_NODE_ENCODING_LZF`时，那么表示这个节点在`quicklistNode.zl`中存储的是一个经过*LZF*压缩的数据，对于压缩数据，*Redis*也定义了对应的数据类型：

```c
typedef struct quicklistLZF
{
    unsigned int sz;
    char compressed[];
} quicklistLZF;
```

这里`quicklistLZF.sz`存储的是压缩数据的大小，`quicklistLZF.compressed`则是存储压缩数据的具体内容。这里需要注意区别`quicklisLZF.sz`和`quicklistNode.sz`这两个字段，`quicklistNode.sz`之所以要永远记录原始压缩链表的大小，是为了在对`quicklistNode`节点进行某些操作时，可以不解压*LZF*数据，便可以确定原始数据的大小。

快速链表数据结构`quicklist`便是以双端链表的形式，组织维护所有的`quicklistNode`节点的，下面便是`quicklist`的数据结构：

```c
typedef struct quicklist
{
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;
    unsigned long len;
    int fill : 16;
    unsigned int compress : 16;
} quicklist;
```

* `quicklist.head`和`quicklist.tail`两个指针分别指向了快速链表的头节点与尾节点。
* `quicklist.count`用于标记这个快速链表中的数据节点的个数，`quicklist.len`则是标记了快速链表节点`quicklistNode`的个数，这两个字段的区别需要注意。
* `quicklist.fill`是快速链表的装载因子，对于这个装载因子，有两种情况：
  1. 如果装载因子为正数，那么表示每一个快速链表节点`quicklistNode`中存储数据节点的个数的上限，例如`quicklist.fill`被设置成10，那么意味着每个`quicklisNode`的压缩链表中最多只能存储10个数据节点。
  2. 如果装载因子为负数，则标记了每个`quicklistNode`中压缩链表的最大内存大小，也就是`quicklistNode.sz`字段的上限。
* `quicklist.compress`则是这个压缩链表的压缩深度，因为前面已经提到过双端链表的访问主要集中在链表的两端，因此快速链表使用这个字段来确定在链表两端不被压缩的链表节点`quicklistNode`的个数；另外通常下情况下，链表处于中间的`quicklistNode`都是使用的*LZF*算法压缩后的数据，只有在需要访问的时候，才会被临时解压出来。

同样的，为了实现对这个快速链表中**数据节点**的遍历，*Redis*设计了快速链表迭代器的数据结构：
```c
typedef struct quicklistIter
{
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi;
    long offset;
    int direction;
} quicklistIter;
```
在这个数据结构中
* `quicklistIter.quicklist`表示这个迭代器所绑定的快速链表。
* `quicklistIter.current`表示这个迭代器当前遍历到的**链表节点**。
* `quicklistIter.si`表示这个迭代器当前遍历的**数据节点**。
* `quicklistIter.offset`表示这个迭代器当前遍历的**数据节点**在**链表节点**之中的偏移。
* `quicklistIter.direction`表示这个迭代器的遍历方向，*Redis*在*src/quicklist.h*头文件中定义了迭代器的遍历方向，`AL_START_HEAD`为正向遍历，`AL_START_TAIL`为反向遍历。

与压缩链表中所定义的`zlentry`数据节点用以描述压缩链表中的节点类似，快速链表中也定义了类似的数据结构用来描述**数据节点**：
```c
typedef struct quicklistEntry
{
    const quicklist *quicklist;
    quicklistNode *node;
    unsigned char *zi;
    unsigned char *value;
    long long longval;
    unsigned int sz;
    int offset;
} quicklistEntry;
```
在这个数据结构中：
* `quicklistEntry.quicklist`指向这个**数据节点**所在的快速链表；`quicklistEntry.node`指向这个**数据节点**所在的**链表节点**；`quicklistEntry.zi`指向**数据节点**在**链表节点**之中的位置。
* 如果**数据节点**中存储的是字符串编码的数据，那么`quicklistEntry.value`和`quicklistEntry.sz`这两个字段用来标记这个字符串编码数据。
* 如果**数据节点**中存储的是整数编码的数据，那么会从对应的压缩链表中解码出这个整数数据，将之存储在`quicklistEntry.longval`字段中。
* `quiclistEntry.offset`的含义与`quicklistIter.offset`一致，都是表示当前**数据节点**在**链表节点**的压缩链表中的偏移。

## 快速链表操作接口

### 快速链表的创建、释放与初始化

```c
quicklist *quicklistCreate(void);
quicklist *quicklistCreateNode(void);
```

上述两个函数通过调用`zmalloc`函数分别为快速链表以及**链表节点**分配内存空间，同时会为快速链表以及**链表节点**的对应字段设置初始化默认的数值。



```c
void quicklistSetCompressDepth(quicklist *quicklist, int compress);
void quicklistSetFill(quicklist *quicklist, int fill);
void quicklistSetOpetions(quicklist *quicklist, int fill, int depth);
```

上述三个函数用于设定了快速链表的*压缩深度*以及*装载因子*。

* *压缩深度*其中必须为非负数，其数值上限被定义为`#define COMPRESS_MAX (1 << 16)`，同时*压缩深度*是可以被定义为0的，这表示，快速链表关闭了压缩功能，这个特性可以通过`#define quicklistAllowsCompression(_ql) ((_ql)->compress != 0)`来判断。
* **装载因子**可以为正数也可以为负数，当**装载因子**为正数时，表示**链表节点**中的压缩链表里存储**数据节点**的个数上限，其上限被定义为`#define FILL_MAX (1 << 15)`，因为在*压缩链表*之中，其不需要遍历便可以获取的数据长度数值上限为`2^16-2`，因此这个**装载因子**的数值上限不可以过大，否则在处理快速链表的时候会导致性能的问题；当**装载因子**为负数时，表示**链表节点**中的压缩链表所占用的内存大小，**装载因子**最低可以为`-5`，对应了5档内存的限制，定义在*src/quicklist.c*文件中：
  * `-1`，4096字节
  * `-2`，8192字节
  * `-3`，16384字节
  * `-4`，32768字节
  * `-5`，65536字节

而函数`quicklistSetOpetions`会一次性调用`quicklistSetCompressDepth`和`quicklistSetFill`来同时设定*快速链表*的*压缩深度*和*装载因子*。

```c
void quicklistRelease(quicklist *quicklist);
```

这个函数用于释放一个给定的*快速链表*，

```c
void quicklistRelease(quicklist *quicklist)
{
    ...
    len = quicklist->len;
    while (len--)
    {
        next = current->next;
        zfree(current->zl);
        ...
        zfree(current);
        ...
        cirrent = next;
    }
    zfree(quicklist);
}
```

这个释放过程是通过循环释放*快速链表*中的每一个**链表节点**的*压缩链表*以及**链表节点**自身，最后在释放整个*快速链表*实现的。

### 快速链表的压缩与解压

对于外部调用者来说，真正关系的是*快速链表*的插入与删除等操作，而并不需要关心**链表节点**的压缩与解压一类的操作，因此这一组操作都是定义在*src/quicklist.c*源文件中的静态函数。

#### 节点的压缩

```c
REDIS_STATIC int __quicklistCommpressNode(quicklistNode *node)
{
    if (node->sz > MIN_COMPRESS_BYTES)
    	return 0;
    quicklistLZF *lzf = zmalloc(sizeof(*lzf) + node->sz);
    if (((lzf->sz = lzf_compress(node->zl, node->sz, lzf->compressed,node->sz)) == 0) 
        || lzf->sz + MIN_COMPRESS_IMPROVE >= node->sz)
    {
  		zfree(lzf);
        return 0;
    }
    ...
    node->zl = (unsigned char *)lzf;
	...
    return 1;
}
```

上述这个函数是快速链表中**链表节点**压缩的基础，这里需要注意两个问题

1. 如果当前**链表节点**的压缩链表占用的内存大小小于`#define MIN_COMPRESS_BYTES 48`这里定义的48个字节时，便没有必要进行压缩。
2. 如果经过压缩后，数据的内存反而大于未压缩的数据，那么也没有必要进行压缩。

如果压缩失败，这个函数会返回0；如果压缩成功，那么在更新这个**链表节点**的相关字段之后，函数会返回1。

```c
#define quicklistCompressNode(_node)										\
	do {																	\
		if ((_node) && (_node)->encoding == QUICKLIST_NODE_ENCODING_RAW) {	\
			__quicklistCompressNode((_node));								\
		}																	\
	} while (0)
```

上面这个宏定义`quicklistCompressNode`通过调用`__quicklistCompressNode`尝试对一个**链表节点**进行压缩，在压缩前会检查被压缩的节点的`quicklistNode.encoding`字段是`QUICKLIST_NODE_ENCODING_RAW`编码。在后续的代码中，尝试对**链表节点**进行压缩，都会通过调用这个宏定义来实现。

#### 节点的解压

```c
REDIS_STATIC int __quicklistDecompressNode(quicklistNode *node);
#define quicklistDecompressNode(_node)
```

上面这个函数是快速链表中对**链表节点**进行解压别的基础，通过调用`lzf_decompress`来对将压缩数据解压成原始的压缩链表，同时更新`quicklistNode.encoding`字段。

而宏定义`quicklistDecompressNode`与`quicklistCompressNode`类似，会尝试判断`quicklistNode.encoding`这个编码字段，并通过调用`__quicklistDecompressNode`函数，对一个**链表节点**的底层压缩数据进行解压。

#### 链表的压缩与解压
由于*快速链表*中只有两端的部分**链表节点**是处于未压缩的状态，因此当我们需要操作一个位于*快速链表*中间的**链表节点**时，我们需要对该**链表节点**先调用`quicklistDecompressNode`进行解压，当完成对该节点的操作之后，再调用`quicklistCompressNode`对其进行压缩。如果对**链表节点**的操作在一次函数调用中就可以完成，那么在同一个堆栈中调用解压与压缩函数接口便可以完成相关的操作。但是在一些情况下，调用者无法在对**链表节点**进行解压之后立即对其进行重新压缩，压缩操作需要在另外的地方进行，因此*Redis*在定义压缩**链表节点**数据结构的时候，特意定义了`quicklistNode.recompress`这个字段用于处理上述这种情况，同时也定义了两个宏，用于处理这种无法原地重压缩的情况：

```c
#define quicklistDecompressNodeForUse(_node)
#define quicklistRecompressOnly(_ql, _node)
```
其中`quicklistDecompressNodeForUse`与`quicklistDecompressNode`的不同点之处在于，会将`quicklistNode.recompress`字段设置为1，表明其是一个临时被解压的**链表节点**，需要在后续的操作中调用`quicklistRecompressOnly`将其重新压缩。

而对于整个*快速链表*的压缩操作，*Redis*定义了一个链表压缩的基础操作：
```c
REDIS_STATIC void __quicklistCompress(const quicklist *quicklist, quicklistNode *node);
```
这个函数有两个用途：
1. 根据*快速链表*设定的压缩深度，对链表进行维护，也就是对链表两端的**链表节点**进行解压操作。
2. 如果传入的`node`节点不为空，那么判断这个节点是否处于压缩深度之内，如果该节点并非是处于链表两端的节点，那么对这个节点执行压缩操作。

### 快速链表的访问
前面我们已经知道*Redis*会使用`quicklistEntry`这个数据结构来表示链表中的一个**数据节点**。因此*Redis*给出了一个接口函数，通过一个给定的索引`index`来获取在*快速链表*中这个索引所对应的**数据节点**的`quicklistEntry`描述：
```c
int quicklistIndex(const quicklist *quicklist, const long long index, quicklistEntry *entry)
{
    quicklistNode *n;
    unsigned long long accum = 0;
    unsigned long long index;

    initEntry(entry);
    entry->quicklist = quicklist;

    ...

    while (likely(n))
    {
        if ((accum + n->count) > index)
        {
            break;
        }
        else
        {
            accum += n->count;
            n = forward ? n->next : next->prev;
        }
    }

    entry=->node = n;
    if (forward)
    {
        entry->offset = index - accum;
    }
    else
    {
        entry->offset = (-index) - 1 + accum;
    }

    quicklistDecompressNodeForUse(entry->node);
    entry->zi = ziplistIndex(entry->node->zl, entry->offset);
    ziplistGet(entry->zi, &entry->value, &entry->sz, &entry->longval);

    ...
}
```
这个函数传入的`index`的参数，可以是一个从0开始的整数，这表示从*快速链表*的头部开始向后查找定位；可以是从-1开始的负数，这表示从*快速链表*的尾部开始向前查找定位。**数据节点**的定位大致分为5步：
1. 声明辅助局部变量，
2. 初始化`quicklistEntry`
3. 通过，找到**数据节点**所在的**链表节点**
4. 根据`index`以及`accum`计算出该**数据节点**在**链表节点**之中的偏移
5. 调用`quicklistDecompressNodeForUse`解压出当前的**链表节点**，然后调用`ziplistIndex`以及`ziplistGet`从**链表节点**的压缩链表中解码出相关数据赋值给`quicklistEntry`。

这也就意味这，调用`quicklistIndex`后，对应的**链表节点**会被临时的解压出来，因此后续需要调用`quicklistCompress`函数尝试对该**链表节点**重新压缩。


```c
int quicklistReplaceAtIndex(quicklist *quicklist, long index, void *data, int sz);
```
上述这个函数，会使用`quicklistIndex`函数，将*快速链表*中`index`索引对应的**数据节点**中的数据使用`data`和`sz`进行替换。

### 快速链表的遍历

#### 快速链表迭代器
```c
quicklistIter *quicklistGetIterator(const quicklist *quicklist, int direction);
```
`quicklistGetIterator`这个函数通过给定迭代器方向`AL_START_HEAD`或者`AL_START_TAIL`，来创建一个指向*快速链表*头部或者尾部的迭代器。



```c
quicklistIter *quicklistGetIteratorAtIdx(const quicklist *quicklist, const int direction, const long long idx);
```
`quicklistGetIteratorAtIdx`这个函数与`quicklistGetIterator`类似，都是用来创建一个*快速链表*的迭代器。但是不同的地方在于，这个函数所返回的迭代器并不是指向*快速链表*的头部或者尾部，而是指向给定索引`idx`对应的**数据节点**的迭代器。



```c
void quicklistReleaseIterator(quicklistIter *iter);
```
而`quicklistReleaseIterator`这个函数的作用是释放一个给定的迭代器`iter`，如果迭代器指向了某一个**链表节点**，那么会调用`quicklistCompress`尝试对该节点进行压缩。



```c
int quicklistNext(quicklistIter *iter, quicklistEntry *entry);
```
该函数用于实现迭代器对*快速链表*的迭代过程，将迭代器`iter`沿迭代方向向后移动，同时将所指向的**数据节点**赋值给`quicklistEntry`，其基本的调用范式为：
```c
iter = quicklistGetIterator(quicklist,<direction>);
quicklistEntry entry;
while (quicklistNext(iter, &entry)) {
    if (entry.value)
        [[ use entry.value with entry.sz ]]
    else
        [[ use entry.longval ]]
}
```

这个函数在对迭代器进行迭代的时候，会根据迭代器的方向，选择调用`ziplistNext`或者`ziplistPrev`来对**链表节点**的底层压缩链表进行迭代，将迭代器底层指向**数据节点**的指针`quicklistIter.zi`沿着迭代器方向进行移动。如果`quicklistIter.zi`已经处于压缩链表的两端，那么在进行压缩链表迭代时，将会导致指针失效，对于这种情况，那么将迭代器所在的**链表节点**指针`quicklistIter.current`沿着迭代器方向移动到下一个**链表节点**，然后递归调用`quicklistNext`函数继续对迭代器进行迭代。

对于`quicklistNext`这个函数，需要注意的是，调用者在迭代器迭代过程中禁止执行对*快速链表*的插入操作，这样的操作可能会导致迭代器失效。但是*Redis*允许在迭代的过程中应用`quicklistDelEntry`函数来执行删除操作，对于这个函数的讲解会在后续进行介绍。

### 快速链表的插入

因为对于调用者来说，**链表节点**这一层对于他其实是透明，调用者主要进行的是**数据节点**的操作，而并不是太关心一个**数据节点**是以何种策略插入到哪个**链表节点**中，或者是从哪个**链表节点**中删除的。因此我们可以发现，在*src/quicklist.h*头文件中定义的通常是对于**数据节点**的操作函数，而对于**链表节点**的操作，则通常是定义在*src/quicklist.c*源文件中的静态函数。

#### 插入条件的判断
因为*快速链表*定义了装载因子`quicklist.fill`这个字段，用来对每个**链表节点**底层压缩链表的内存大小或者节点长度进行限制，因此在向某个**链表节点**中插入**数据节点**的时候，需要预先进行判断，新的**数据节点**是否可以进行插入。

如果*快速链表*的装载因子设置对于压缩链表内存大小的限制，那么*Redis*会通过下面这个函数来检查插入后新的内存大小是否满足装载因子的限制：
```c
REDIS_STATIC int _quicklistNodeSizeMeetsOptimizationRequirement(const size_t sz, const int fill);
```
这个函数会首先判断`fill`参数是否为负数，因为*快速链表*只有在装载因子为负数的情况下才会对压缩链表内存大小进行限制。如果`fill`参数合法，那么会从`optimization_level`数组中查询对应内存上限，然后判断新的内存大小是否符合要求。

那么对于插入条件的通用判断，*Redis*则是通过下面这个静态函数来实现的：
```c
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node, const int fill, const size_t sz);
```
给定一个待插入的**链表节点**，以及这个*快速链表*的装载因子和插入数据的内存大小，来判断插入操作是否可以执行，返回0，表示无法插入到给定的**链表节点**；返回1，则表示可以进行插入。整个判断过程分为三步：
1. 如果装载因子限制了压缩链表的内存的大小，那么通过`_quicklistNodeSizeMeetsOptimizationRequirement`函数检查内存大小是否符合要求。
2. 如果装载因子限制的是压缩链表的节点长度，那么对于这种情况，*Redis*也会定义其内存大小的限制`#define SIZE_SAFETY_LIMIT 8192`，也就是这种情况下，整个压缩链表的内存大小不能查过8K字节，否则就需要一个单独的**链表节点**来存储数据。
3. 最后会判断`node`节点中**数据节点**的个数是否满足装载因子对于节点个数的限制。

在操作*快速链表*的过程中，为了提高内存使用率，会尝试对两个相邻的**链表节点**进行合并，这种合并操作也应该符合*快速链表*中装载因子对**链表节点**的限制，因此*Redis*还定义个下面的函数，用于判断两个**链表节点**是否可以合并为1个：
```c
REDIS_STATIC int _quicklistNodeAllowMerge(const quicklistNode *a, const quicklistNode *b, const int fill);
```
这个函数的判断流程与`_quicklistNodeAllowInsert`类似，也一样会从三个方面进行判断。

#### 链表节点的插入
*Redis*定义了一个底层的**链表节点**的插入操作函数：
```c
REDIS_STATIC void __quicklistInsertNode(quicklist *quicklist, quicklistNode *old_node, quicklistNode *new_node, int after);
```
这个函数执行的是一个经典的双端链表插入操作，
1. 如果`after`为1，那么`new_node`会被插入到`old_node`节点的后面。
2. 如果`after`为0，那么`new_node`会被插入到`old_node`节点的前面。

此外需要注意的一点，待插入的**链表节点**`new_node`，应该是一个未被压缩的节点，如果被插入到*快速链表*的头部或者尾部的时候，便没有必要对该节点进行压缩，而插入到其他位置的时候，需要调用者在`__quicklistInsertNode`之后，手动调用压缩函数，对节点进行压缩。

通过对`__quicklistInsertNode`进行包装，*Redis*还定义了两个静态函数，用于实现先向某个已存在的**链表节点**前后进行插入新的**链表节点**的操作：
```c
REDIS_STATIC void _quicklistInsertNodeBefore(quicklist *quicklist, quicklistNode *old_node, quicklistNode *new_node)
{
    __quicklistInsertNode(quicklist, old_node, new_node, 0);
}

REDIS_STATIC void _quicklistInsertNodeAfter(quicklist *quicklist, quicklistNode *old_node, quicklistNode *new_node) 
{
    __quicklistInsertNode(quicklist, old_node, new_node, 1);
}
```

#### 链表节点的拆分与合并

*Redis*会使用静态函数`_quicklistZiplistMerge`来作为合并**链表节点**的操作基础，该函数会给定两个**链表节点**`a`和`b`，然后尝试将合并这两个**链表节点**的底层压缩链表。

```c
REDIS_STATIC quicklistNode *_quicklistZiplistMerge(quicklist *quicklist, quicklistNode *a, quicklistNode *b)
{
    ...
	quicklistDecompressNode(a);
    quicklistDecompressNode(b);
    if ((ziplistMerge(&a->zl, &b->zl))) {
    	quicklistNode *keep = NULL, *nokeep = NULL;
        ...
        __quicklistDelNode(quicklist, nokeep);
        quicklistCompress(quicklist, keep);
        return keep;
    }
    ...
}
```

这个合并操作的流程为：

1. 调用`quicklistDecompressNode`将两个**链表节点**进行解压。
2. 通过`ziplistMerge`合并两个**链表节点**的底层压缩链表。
3. 调用`__quicklistDelNode`将被合掉的**链表节点**从*快速链表*之中删除。
4. 最后将保留的节点通过`quicklistCompress`进行重新压缩。

不过我们发现，前面我们介绍了`_quicklistNodeAllowMerge`这个函数用于检查两个**链表节点**的合并是否可以满足装载因子的限制。但是`_quicklistZiplistMerge`这个函数看起来只是执行了**链表节点**的合并操作，而并没有对合并的合法性进行校验。因此*Redis*在`_quicklistZiplistMerge`的基础上，结合了合并合法性的校验函数`_quicklistNodeAllowMerge`实现了另外一个更加*高级*的**链表节点**合并操作的接口：
```c
REDIS_STATIC void _quicklistMergeNodes(quicklist *quicklist, quicklistNode *center);
```
这个函数会尝试对`center`节点以及其两侧各两个**链表节点**一共五个节点尝试合并成一个**链表节点**，其合并的顺序为：
1. `center->prev->prev`与`center->prev`
2. `center->next`与`center->next->next`
3. `center->prev`与`center`
4. `center`与`center->next`

上述的四步合并都是通过先调用`_quicklistNodeAllowMerge`检查合法性，然后调用`_quicklistZiplistMerge`将两个节点进行合并。

在实现了**链表节点**的合并操作之后，*Redis*也为我们提供了一个**链表节点**的拆分功能，这个特性在执行**数据节点**的插入时，是非常有用的。想象一个场景：
1. 当我们期望在一个给定的索引位置进行**数据节点**插入，
2. 而这个给定的索引恰恰对于**链表节点**底层压缩链表的中间，
3. 同时这次插入的结果会导致对应的**链表节点**将不满足*快速链表*对于装载因子的限制

对于这种情况，便需要对对应的**链表节点**执行拆分操作，将一个较大的节点拆分成两个更小的节点，以满足**数据节点**的插入。因此*Redis*定义了一个静态函数用于实现**链表节点**的拆分：
```c
REDIS_STATIC quicklistNode *_quicklistSplitNode(quicklistNode *node, int offset, int after);
```
这个函数会根据`offset`以及`after`来确定**链表节点**中低层压缩链表的拆分位置。如果拆分成功，那么这个函数会返回一个新的**链表节点**，其中保存了被拆分出来的**数据节点**，而剩下的**数据节点**则会被保留在`node`节点之中。对于**链表节点**的拆分策略，如下所示：
1. 如果`after`为1，原`node`节点中保留了`[0, offset]`这些**数据节点**，返回的新节点存储了`[offset+1, END]`这些**数据节点**。
2. 如果`after`为0，原`node`节点中保留了`[offset+1, END]`这些**数据节点**，返回的新节点存储了`[0, offset]`这些**数据节点**。

#### 数据节点的Push插入

*Redis*提供了两个接口函数，用于实现向*快速链表*的头部和尾部插入**数据节点**，这两个函数可以供调用者在外部进行调用：

```c
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz);
int quicklistPushTail(quicklist *quicklist, void *value, size_t sz);
```
上述两个函数实现方式基本上是一致的：
1. 通过调用前面提到的`_quicklistNodeAllowInsert`静态函数来判断，新数据是否可以插入到*快速链表*的头节点或者尾节点。
2. 如果可以插入，那么调用压缩链表的插入接口`ziplistPush`，将数据插入到头节点或者尾节点的压缩链表中，值得注意的是，因为不论当前的*快速链表*的压缩深度是多少，其头节点和尾节点一定是未压缩的状态，因此执行压缩链表插入的时候，不需要对**链表节点**进行解压缩操作。
3. 如果头尾节点因为达到了装载因子的限制而无法插入新元素的话，或者当前的*快读链表*为空的时候，那么*Redis*会按照如下的步骤执行插入流程：
    1. 调用`quicklistCreateNode`创建一个新的**链表节点**。
    2. 调用压缩链表的插入接口`ziplistPush`，将数据插入到新创建的**链表节点**的底层压缩链表之中。
    3. 调用`_quicklistInsertNodeBefore`或者`_quicklistInsertNodeAfter`函数，将新创建的**链表节点**插入到快速链表之中。

上述两个函数的插入过程一定不会失败，当新数据被插入到一个已存在的**链表节点**之中，那么该函数会返回0；如果新创建一个**链表节点**，那么函数返回1。

同时*Redis*还将上述的两个函数包装成一个通用的函数接口：

```c
void quicklistPush(quicklist *quicklist, void *value, const size_t sz, int where);
```

通过将参数`where`设置成`QUICKLIST_HEAD`或者`QUICKLIST_TAIL`来实现向*快速链表*的头部或者尾部进行**数据节点**的插入。

#### 数据节点的随机插入
前面介绍的**数据节点**在*快速链表*两端的**Push**插入操作是*快速链表*中较为常用的操作，因为是在链表的两端执行插入，因此不需要考虑**链表节点**的合并与拆分的细节，通过前面的描述，我们也发现当在链表两端执行插入操作时，如果两端的**链表节点**不能满足装载因子的限制，那么*Redis*会直接为新数据创建一个新的**链表节点**，将之插入到*快速链表*中。
但是对于将**数据节点**插入到指定位置索引的需求依然存在，因此*Redis*定义了一个向*快速链表*中的给定位置索引插入一个**数据节点**的函数：
```c
REDIS_STATIC void _quicklistInsert(quicklist *quicklist, quicklistEntry *entry, void *value, const size_t sz, int after);
```
这个函数向*快速链表*`quicklist`中的**数据节点**`entry`的前面或者后面根据`value`以及`sz`插入一个新的**数据节点**。在插入过程，*Redis*会执行如下的逻辑：
1. 定义了五个标记变量：
    |标记变量|变量含义|
    |-------|--------|
    |`full`|`entry`对应的**链表节点**是否已经达到装载因子的限制|
    |`at_tail`|新数据的插入位置是否位于`entry`对应**链表节点**底层压缩链表的尾部|
    |`at_head`|新数据的插入位置是否位于`entry`对应**链表节点**底层压缩链表的头部|
    |`full_next`|`entry`对应的**链表节点**的前序节点是否已满|
    |`full_prev`|`entry`对应的**链表节点**的后序节点是否已满|

2. 通过判断`quicklistEntry.offset`以及调用`_quicklistNodeAllowInsert`函数来为上述的5个标记赋值。
3. 根据前面定义的5个标记以及`after`参数，将插入过程划分为6种情况，根据情况的不同，执行不同的插入逻辑：
    * `!full && after`，插入到给定**数据节点**之后，对应的**链表节点**未满，那么直接调用`ziplistPush`或者`ziplistInsert`将新数据插入到当前**链表节点**的底层压缩链表中。
    * `!full && !after`，插入到给定**数据节点**之前，对应的**链表节点**未满，那么处理方式与上面一种情况类似，调用`ziplistInsert`将新数据插入到当前**链表节点**的底层压缩链表中。
    * `full && at_tail && node->next && !full_next && after`，插入对应**链表节点**的尾部，同时该节点已满，而其后序的**链表节点**未满，那么将新数据插入到其后续**链表节点**压缩链表的头部。
    * `full && at_head && node->prev && !full_prev && !after`，插入对应**链表节点**的头部，同时该节点已满，而其前序的**链表节点**未满，那么将数据插入到其前序**链表节点**压缩链表的尾部。
    * `full && ((at_tail && node->next && full_next && after) || (at_head && node->prev && full_prev && !after))`，在对应**链表节点**的头部或者尾部插入新的数据，但是该节点已满无法插入新数据，同时对应的前序或者后续节点也无法插入新数据，对于这种所有关联**链表节点**都无法进行插入的情况，只能为这个新的数据创建一个新的**链表节点**，调用`__quicklistInsertNode`将其插入到*快速链表*中。
    * 最后一种情况，插入的位置正好是对应**数据节点**的中间位置，而该节点已满，那么将调用`_quicklistSplitNode`函数，通过`quicklistEntry.offset`，以及`after`作为参数，将**链表节点**进行拆分，对拆分操作返回的新**链表节点**应用`ziplistPush`将数据插入其中，并将新**链表节点**插入*快速链表*，最后调用`_quicklistMergeNodes`尝试对**链表节点**进行合并。

基于这个`_quicklistInsert`函数，*Redis*定义了两个接口函数：
```c
void quicklistInsertBefore(quicklist *quicklist, quicklistEntry *entry, void *value, const size_t sz);
void quicklistInsertAfter(quicklist *quicklist, quicklistEntry *entry, void *value, const size_t sz);
```
这两个函数本质上是对`_quicklistInsert`简单的封装，分别用于将数据插入到一个给定的**数据节点**之前，或者插入到给定的**数据节点**之后。

#### 数据节点的批量插入

*Redis*为*快速链表*提供了三个函数，可以实现从一个压缩链表构造 一个*快速链表*，或者将压缩链表作为输入数据的集合，实现批量的**数据节点**插入。

```c
void quicklistAppendZiplist(quicklist *quicklist, unsigned char *zl);
```

这个函数通过给定的压缩链表参数`zl`创建一个**快速节点**，然后调用`_quicklistInsertNodeAfter`将这个新建的**链表节点**插入到*快速链表*的节点。

```c
quicklist *quicklistAppendValuesFromZiplist(quicklist *quicklist, unsigned char *zl);
```

这个函数也是将一个压缩链表`zl`中的数据插入到*快速链表*中，但是与`quicklistAppendZiplist`不同的是，`quicklistAppendValuesFromZiplist`不是将压缩链表作为一个整体插入到*快速链表*中的，而是遍历读取压缩链表中中的每一个节点，将其作为**数据节点**的输入，通过调用`quicklistPushTail`将**数据节点**插入到*快速链表*的尾部。

```c
quicklist *quicklistCreateFromZiplist(int fill, int compress, unsigned char *zl)
{
    return quicklistAppendValuesFromZiplist(quicklistNew(fill, compress), zl);
}
```

上述这个函数是通过压缩链表来创建一个*快速链表*，具体是通过函数`quicklistNew`创建一个*快速链表*，然后调用`quicklistAppendValuesFromZiplist`函数将压缩链表中的数据批量的插入到新创建的*快速链表*中。

### 快速链表的删除

#### 链表节点的删除
*Redis*为*快速链表*中对于**链表节点**的删除给一个基础的操作函数：
```c
REDIS_STATIC void __quicklistDelNode(quicklist *quicklist, quicklistNode *node)
{
    ...
    __quicklistCompress(quicklist, NULL);
    ...
}
```
此处对**链表节点**的删除，与**链表节点**的插入类似，应用经典的双端链表的删除方式。但是执行删除操作时，如果这个节点刚好处于*快速链表*的压缩深度之内，那么我们需要调用`__quicklistCompress(quicklist, NULL);`来对某一个处于压缩状态，但是因为删除了`node`节点而落入压缩深度范围内的节点进行解压。

#### 指定数据节点的删除
对于**数据节点**的删除操作，*Redis*同样定义了一个基础的操作函数：
```c
REDIS_STATIC int quicklistDelIndex(quicklist *quicklist, quicklistNode *node, unsigned char **p);
```
这个函数用于从给定的*快速链表*`quicklist`中，删除**链表节点**`node`里由`p`所指向的**数据节点**，这里有两点需要注意：
1. 因为这是一个内部调用的静态函数，因此需要在调用的外部，自行对`node`节点进行解压。
2. 如果`node`节点底层的压缩链表在调用`ziplistDelete`删除`p`之后，变成空的压缩链表，那么需要调用`__quicklistDelNode`将这个**链表节点**从链表中删去。

基于上面的静态函数，*Redis*定义了一个用户节点函数，用于从*快速链表*中删除一个由`quicklistEntry`描述的数据节点：
```c
void quicklistDelEntry(quicklistIter *iter, quicklistEntry *entry) {
    ...
    int deleted_node = quicklistDelIndex((quicklist *)entry->quicklist, entry->node, &entry->zi);
    iter->zi = NULL;
    ...
}
```
对于链表而言，除了在链表两端执行的**Pop**删除之外，其他对给定**数据节点**的删除大概率都是在遍历的过程中进行，因此`quicklistDelEntry`这个函数除了接受一个`quicklistEntry`变量作为参数之后，选择使用一个`quicklistIter`迭代器作为参数，而不是传入一个*快速链表*指针作为参数的。其常用的遍历删除范式为：
```c
while (quicklistNext(iter, &entry))
{
    if (quicklistCompare(entry.zi, data, size))
    {
        quicklistDelEntry(iter, &entry);
    }
    i++
}
```

那么*Redis*是如何保证迭代器在一边迭代一边删除的时候，不会失效的呢？我们可以在回头看一下`quicklistNext`函数的实现细节：

```c
int quicklistNext(quicklistIter *iter, quicklistEntry *entry)
{
    ...
    if (!iter->zi)
    {
        quicklistDecompressNodeForUse(iter->current);
        iter->zi = ziplistIndex(iter->current->zl, iter->offset);
    }
}
```

我们可以看到，*Redis*在执行`quicklistDelEntry`对**数据节点**进行删除操作后，会将`quicklistIter.zi`字段设置为`NULL`，

* 如果**链表节点**没有被删除的话，那么`quicklistIter.offset`不会变化，在下一轮迭代`quicklistNext`中，如果发现`quicklistIter.zi`为`NULL`的时候，会调用`ziplistIndex`根据`quicklistIter.offset`来将迭代器移动到下一个位置。
* 如果**链表节点**被删除的话，那么`quicklistDelEntry`会更新迭代器`quicklistIter.current`和`quicklistIter.offset`这两个字段，用于将迭代器移动到下一个**链表节点**，最后在`quicklistNext`调用中，会将迭代器移动到下一个位置。



#### 数据节点的Pop删除

*Redis*除了提供若干个在指定位置执行删除**数据节点**的函数之外，还提供了两个更加定制化的，用于从链表两端执行删除操作的函数。

```c
int quicklistPopCustom(quicklist *quicklist, int where, unsigned char **data, unsigned int *sz, long long *sval, void *(*saver)(unsigned char *data, unsigned int sz))；
```

这个函数是*快速链表*的*Pop*操作的基础，会从`quicklist`中弹出元素，将弹出的**数据节点**的数据通过函数指针`saver`存储在`data`之中，如果**数据节点**中的数据是一个整形数的话，那么会将数据存储在`sval`之中。如果弹出成功，函数返回1，否则的话，函数返回0。

除了上述这个基础*Pop*函数之外，*Redis*还基于这个函数，定义了一个默认的*Pop*函数：

```c
int quicklistPop(quicklist *quicklist, int where, unsigned char **data, unsigned int *sz, long long *slong);
```

这个函数本质上是对`quicklistPopCustom`的简单封装，但是这个函数只在测试代码中有调用，而在外部则没有对应调用的地方。

#### 数据节点的批量删除

```c
int quicklistDelRange(quicklist *quicklist, const long start, const long count);
```

这个函数会从*快速链表*中删除从`start`开始连续`count`个数的**数据节点**，需要注意的是，被删除的多个**数据节点**有可能会跨越多个**链表节点**。


### 快速链表的其他操作
```c
void quicklistRotate(quicklist *quicklist);
```

`quicklistRotate`用于实现对*快速链表*的一次反转操作，也就是将*快速链表*尾部的**数据节点**弹出，然后插入到*快速链表*的头部。



```c
quicklist *quicklistDup(quicklist *orig);
```

这个函数则用于拷贝一个给定的*快速链表* ，并返回新建的*快速链表的指针*。



```c
int quicklistCompare(unsigned char *p1, unsigned char *p2, int p2_len);
```

这个函数t通过调用`ziplistCompare`来判断一个底层**数据节点**的数据是否等于`p2`和`p2_len`所表示的数据。



***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*