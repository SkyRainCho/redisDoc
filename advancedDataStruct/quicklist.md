# Redis中快速连链表的实现

根据前文的分析，我了解到，基于经典双端链表的一些缺点，双端链表已经不在作为*LIST*对象的底层实现了。但是作为*LIST*的底层实现之一，压缩链表同样具有一定的劣势。压缩链表由于使用的是有一段连续的内存，这就意味着，执行插入操作导致内存大小发生变化时，便会引起一次内存的重新分配过程，同时还会执行一定量的数据移动。特别时对于压缩链表较长的情况下，大批量的数据移动势必会降低系统的性能。而在上述这个方面恰恰就是经典双端链表的优势，故此*Redis*设计实现了一种快速链表的数据结构，兼具经典双端链表与压缩链表的特点，实现了时间与空间上的一种折中。

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

### 快速链表节点的插入

### 快速链表节点的删除

### 快速链表节点的访问

### 快速链表的遍历









***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*