# Redis中跳跃表的实现
跳跃表是一种能高效实现插入、删除、查找的内存数据结构，这些操作的期望时间复杂度都是`O(logN)`。与红黑树以及其他的二分查找树相比，跳跃表的优势在于实现简单。

## 跳跃表概述
在*5.0.8*版本的*Redis*源代码中，跳跃表实现没有单独的头文件件以及源文件，其数据结构的定义，以及接口函数的声明是*src/server.h*这个头文件中进行的，而接口函数的定义则是在*src/t_zset.c*文件之中进行的。

首先我们先了解一下跳跃表节点的定义方式：
```c
typedef struct zskiplistNode
{
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel
    {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;
```
在这个数据结构之中：
1. `zskiplistNode.ele`，这个字段用于存储这个节点所保存的字符串对象*key*。
2. `zskiplistNode.score`，这个字段用于存储这个节点*key*对应的分值。
3. `zskiplistNode.backward`，这个字段表示该节点的后退指针，使用这个字段，可以实现从跳跃表的尾部向头部的方向访问节点，这个指针只能指向其紧邻的节点，而不能跨越多个节点。
4. `zskiplistNode.level`，这个字段用于表示节点的层，节点可以包含多个层，每一层都包含一个指向其他节点的指针，程序可以通过这些层来加快对其他节点的访问，每一层都包含两个字段：
	1. `zskiplistLevel.forward`，这个字段是节点该层的前进指针，用于从表头向表尾方向访问节点，每个节点的指定层的前进指针指向下一个具有相同层数的节点。
	2. `zskiplistLevel.span`，这个字段是节点该层的跨度，跨度指的是，当前节点与节点该层的前进指针所指向的节点的距离。跨度数值越大，表明两个节点相距的越远；如果跨度为0，则表示该层的前进指针没有后续节点，而是指向一个`NULL`。

*Redis*每次创建一个跳跃表中的新节点的时候，会为这个节点生成随机的层数，这个层数的规律为：层数越高，对应的概率越小；层数越低，生成的概率越高。对于层数的上限，*Redis*给出的定义：
```c
#define ZSKIPLIST_MAXLEVEL 64
```

接下来，我们看一下跳跃表的数据结构定义：
```c
typedef struct zskiplist
{
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```
在这个数据结构之中，
1. `zskiplist.header`，这个字段用于指向跳跃表中的头节点，这个头部节点是一个特殊的节点，它不包含任何数据，但是这个节点的层数为`ZSKIPLIST_MAXLEVEL`，该节点的每一层的前进指针都适当地指向后续的某个节点或者空指针`NULL`。
2. `zskiplist.tail`，这个字段用于指向跳跃表的尾节点，这个节点与头节点不同，他是一个真实的存储数据的跳跃表节点，如果跳跃表为空的情况，那么这个指针为`NULL`。
3. `zskiplist.length`，这个字段用于存储跳跃表中节点的个数。
4. `zskiplist.level`，这个字段则是表示当前跳跃表中层数最高的节点的层数。

最后，我们在看一下，跳跃表里用于表示分值范围的两个数据结构
```c
typedef struct {
	double min, max;
	int minex, maxex;
} zrangespec;

typedef struct {
	sds min, max;
	int minex, maxex;
} zlexrangespec;
```
上述的两个数据结构分布用于表示按照浮点数排序的范围以及按照字典顺序排序的范围，通过`minex`以及`maxex`可以控制范围两端的开闭区间。

通过跳跃表的相关数据结构，可以看出来跳跃表本质上也是一种链表数据结构，基于这种数据结构，可以针对某个特定位置以`O(1)`的时间复杂度执行节点的插入与删除。对于经典的链表结构，即使表中的节点是有序的，我们依然无法针对这个链表执行二分搜索，只能通过遍历的形式，以`O(N)`的时间复杂度对某个节点进行搜索。但是由于在跳跃表中每个节点还有一组层数据`zskiplistLevel`，通过每层中的前进指针可以跨越多个节点进行访问。由于每个节点在生成时，其层数是遵从幂定规律生成的，也就是说每多一层，其概率就降低一半，这样通过节点的层，便可以近似的构成一个二叉树的结构，如下面的示例图所示：

## 跳跃表操作函数
### 跳跃表的创建与释放

这一部分主要涉及跳跃表节点以及跳跃表本身的创建与释放，在此之前，*Redis*首先定义了用于按照幂定规律随机生成层数：

```c
int zslRandomLevel(void) {
    int level = 1;
    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF)) level += 1;
    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

通过`zslRandomLevel`获得节点的随机层数`level`之后，我们可以根据给定的数据元素`ele`以及以及分值`score`，通`zslCreateNode`函数来创建一个跳跃表节点；而当一个给定的跳跃表节点不在被使用的时候，则会通过`zslFreeNode`来进行释放。

```c
zskiplistNode *zslCreateNode(int level, double score, sds ele);
void zslFreeNode(zskiplistNode *node);
```

上述三个函数都是定义在*src/t_zset.c*这个源文件之中的。



```c
zskiplist *zslCreate(void);
void zslFree(zskiplist *zsl);
```
用户创建与释放一个跳跃表，是通过`zslCreate`与`zslFree`两个函数来实现的。在创建跳跃表的时候，会有按照下面的逻辑，先为跳跃表创建一个特殊的`header`节点，这个节点具有`ZSKIPLIST_MAXLEVEL`层的`zskiplistLevel`的数据；同时`zslCreate`会初始化`header`各层的数据，具体内容如下面的代码片段所示：

```c
zskiplist *zslCreate(void) {
	...
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL, 0, NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
	...
}
```



### 跳跃表节点查找

### 跳跃表节点的插入与删除





跳跃表的接口函数为：

```c
zskiplist *zslCreate(void);
void zslFree(zskiplist *zsl);
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele);
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node);
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range);
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range);
unsigned long zslGetRank(zskiplist *zsl, double score, sds o);
int zslValueGteMin(double value, zrangespec *spec);
int zslValueLteMax(double value, zrangespec *spec);
void zslFreeLexRange(zlexrangespec *spec);
int zslParseLexRange(robj *min, robj *max, zlexrangespec *spec);
zskiplistNode *zslFirstInLexRange(zskiplist *zsl, zlexrangespec *range);
zskiplistNode *zslLastInLexRange(zskiplist *zsl, zlexrangespec *range);
int zslLexValueGteMin(sds value, zlexrangespec *spec);
int zslLexValueLteMax(sds value, zlexrangespec *spec);
```

跳跃表的底层接口函数
```c
zskiplistNode *zslCreateNode(int level, double score, sds ele);
void zslFreeNode(zskiplistNode *node);
int zslRandomLevel(void);
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update);
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore);
int zslIsInRange(zskiplist *zsl, zrangespec *range);
unsigned long zslDeleteRangeByScore(zskiplist *zsl, zrangespec *range, dict *dict);
unsigned long zslDeleteRangeByLex(zskiplist *zsl, zlexrangespec *range, dict *dict);
unsigned long zslDeleteRangeByRank(zskiplist *zsl, unsigned int start, unsigned int end, dict *dict);
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank);
static int zslParseRange(robj *min, robj *max, zrangespec *spec);
int zslParseLexRangeItem(robj *item, sds *dest, int *ex);
int sdscmplex(sds a, sds b);
int zslIsInLexRange(zskiplist *zsl, zlexrangespec *range);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
