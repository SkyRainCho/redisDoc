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

如果在
***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*