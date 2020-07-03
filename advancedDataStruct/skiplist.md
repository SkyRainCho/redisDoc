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
![跳跃表](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E8%B7%B3%E8%B7%83%E8%A1%A8.PNG)

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

跳跃表的查找是整个跳跃表操作的关键，由于跳跃表本质上是一个有序的链表数据结构，除了正常的查找操作之外，其他的基础操作都是建立在查找的基础之上的，在向表中插入节点的时候，需要首先根据新节点的分值，在链表中找到何时的位置，才能执行插入操作；同理，当需要执行删除操作的时候，也要先按照分值，找到对应的节点，才可以执行对节点的删除。

虽然*Redis*没有为跳跃表定义一个通用的查找逻辑，但是在跳跃表的插入与删除函数实现中都存在一个跳跃表节点的范式：

```c
zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
int i;
x = zsl->header;
for (i = zsl->level - 1; i >= 0; i--)
{
    while (x->level[i].forward &&
           (x->level[i].forward->score ||
            (x->level[i].forward->score == score &&
             sdscmp->(x->level[i].forward->ele, ele) < 0)))
    {
        x = x->level[i].forward;
    }
    update[i] = x;
}
```

这个范式的基础逻辑为：

1. 创建一个跳跃表节点的数组`update`，这个数组用于保存整个搜索所经历的路径，定义一个节点的指针`x`，初始指向跳跃表的头部，用于在遍历过程表示已查找到的最接近目标节点的节点。
2. 从跳跃表的头部开始向后，从整个跳跃表的最高层向下，在每层去查找最接近目标节点的那个节点。
3. 如果在某一层，通过在该层对节点的跳跃的遍历找到了更接近目标节点的节点，那么更新`x`指针。
4. 完成跳跃表每一层的遍历后，无论`x`指针有没有被更新，都将`x`存储在`update`对应的层中，用于记录该层最接近目标节点的节点。
5. 按照上述步骤完成了从跳跃表最高层向下的遍历之后，`update[0]`中存储的便是整个跳跃表中最接近目标节点的节点，那么`update[0]->forward`所指向的节点，应该就是最终查找的终点，通过对这个最终节点与目标节点的比较可以确定在这个跳跃表中是否能查找到目标节点。

应用上述的逻辑范式，*Redis*给出了一个函数，用于在给定的跳跃表`zsl`中查找`score`与`ele`所对应的节点排名（从1开始），如果对应节点在跳跃表中不存在，那么会返回0：

```c
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele);
```

同时跳跃表还支持以`O(logN)`的时间复杂度来获取表中的第`N`个节点，与经典链表通过一个接一个的遍历方式来获取第`N`个节点的方式不同，跳跃表借助了节点的层数，实现了对节点的快速定位：

```c
zskiplistNode *zslGetElementByRank(zskiplist *zsl, unsigned long rank)
{
    ...
    x = zsl->header;
    for (i = zsl->level - 1; i >= 0; i--) {
    	while (x->level[i].forward && (traversed + x->level[i].span) <= rank) {
        	traversed += x->level[i].span;
            x = x->level[i].forward;
        }
    }
    return NULL;
}
```

### 跳跃表节点的插入、删除与更新

首先我们来看插入操作，对于跳跃表插入新的节点，*Redis*只给出了一个函数定义：

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele);
```

通过`zslInsert`这个接口，会向指定的跳跃表中插入一个新的节点，调用者需要在调用之前保证，这个新的节点并不存在与给定的跳跃表中，这个插入操作的基础逻辑为：

1. 通过`score`分值以及`ele`，结合前面所描述的查找范式，为新节点查找一个合适的待插入位置。
2. 调用`zslRandomLevel`函数来随机生成一个节点的层数，如果这个层数超过了当前跳跃表的最大层数`zskiplist.level`，那么更新跳跃表的层数信息。
3. 通过`zslCreateNode`创建一个新的节点，并将这个节点插入到待插入位置，同时更新相邻节点的层列表。

通过前一节的内容，我们知道在查找范式中`update`数组中的每一层保存的是跳跃表在该层上小于给定节点分值与数据，但同时是最接近的节点，因此在插入操作更新相邻节点层列表的时候，实际上便是针对`update`数组中每一层的数据进行操作的。

对于跳跃表节点的删除操作，*Redis*给出了三个较为通用的删除接口：

```c
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update);
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node);
unsigned long zslDeleteRangeByRank(zskiplist *zsl, unsigned int start, unsigned int end, dict *dict);
```

这里`zslDeleteNode`是一个较为底层的删除接口，它的含义是，给定一个跳跃表`zsl`，以及通过查找范式所找到的待删除节点`x`和查找过程确定的待删除节点所对应的`update`s数组数据，将`x`节点从跳跃表`zsl`中删除，同时更新`update`数组中的层信息。因为这个一个底层接口，因此不会释放被删除的节点`x`，对于`x`的处理由上一层调用来完成。

函数`zslDelete`是一个通用的删除单一节点的接口，用于从跳跃表中删除匹配`score`以及`ele`的节点，如果这个节点找到并被删除，那么函数会返回1，否则函数返回0。同时如果提供了`node`参数，那么被删除的数据将通过`node`的参数返回，否则被删除的数据将调用`zslFreeNode`来释放。

`zslDelete`这个函数的逻辑便是通过查找范式，找到对应的节点以及`update`数组数据，然后调用`zslDeleteNode`函数将节点删除。

而`zslDeleteRangeByRank`则是一个多个节点的删除接口，它会删除从`start`开始到`end`之间所有的元素，这里的`start`以及`end`都是从1开始的索引值，由于跳跃表是为了实现有序集合的，在这种实现了*Redis*还会使用一个哈希表来实现集合的辅助功能，因为这个函数在参数列表中还有一个哈希表指针`dict`，用于在从跳跃表中删除节点的同时，从哈希表中将对应的节点也一并删除。

基于前面的插入与删除操作，*Redis*给出了一个用于更新节点分值的接口：

```c
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore);
```

这个函数会将跳跃表中`ele`节点的分值从`curscore`更新到`newscore`，这个函数的基础流程为：

1. 通过查找范式来找到这个目标节点。
2. 调用`zslDeleteNode`将原始的节点从跳跃表之中删除。
3. 调用`zslInsert`将节点的值`x->ele`与新的分值`newscore`重新插入到跳跃表中。

### 跳跃表的数值范围操作

回顾一下前面所给出的跳跃表的数值范围数据结构的定义：

```c
typedef struct {
	double min, max;
	int minex, maxex;
} zrangespec;
```

*Redis*基于上述的数值范围数据结构`zrangespec`定义了两个函数用于判断一个数值是否满足`zrangespec`的下限，以及满足`zrangespec`的上限。

```c
int zslValueGteMin(double value, zrangespec *spec);
int zslValueLteMax(double value, zrangespec *spec);
```

*Redis*首先定义了一个函数，用来判断，是否跳跃表中的一部分恰好落在`zrangespec`的范围内：

```c
int zslIsInRange(zskiplist *zsl, zrangespec *range);
```

`zslIsInRange`函数基于下面两个比较进行判断：

1. 跳跃表的最大值不小于`zrangespec`的下限
2. 跳跃表的最小值不大于`zrangespec`的上限

满足上述两个条件，`zslIsInRange`函数返回1，否则函数返回0。

接下来，*Redis*又定义了两个函数，用于查找跳跃表落在给定区间范围内的第一个节点以及最后一个节点：

```c
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range);
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range);
```

上述两个函数会在找到符合调节的节点时，返回对应节点的指针；如果这样的节点不存在，那么会就会返回空指针`NULL`。

在定义了跳跃表基于范围的查找操作之后，*Redis*又定义了基于分值范围的删除操作：

```c
unsigned long zslDeleteRangeByScore(zskiplist *zsl, zrangespec *range, dict *dict);
```

这个函数与`zslDeleteRangeByRank`类似，这个函数将用于删除跳跃表中落在`zrangespec`分值范围内的所有节点。

### 跳跃表的字典范围操作

除了上述的基于分值浮点数的范围操作，*Redis*也为跳跃表提供了一系列的基于字典顺序的范围操作，首先我们在回顾一下字典顺序范围的数据结构：

```c
typedef struct {
	sds min, max;
	int minex, maxex;
} zlexrangespec;
```

与浮点数范围操作类似，*Redis*也提供了两个函数结构用于判断一个给定的字符串是否满足`zlexrangespec`的上限与下限：

```c
int zslLexValueGteMin(sds value, zlexrangespec *spec);
int zslLexValueLteMax(sds value, zlexrangespec *spec);
```

由于`zlexrangespec`中`min`与`max`都是`sds`数据，因此需要一个函数在释放`zlexrangespec`时，将对应的`sds`数据也一同释放，因此*Redis*定义了下面这个`zslFreeLexRange`函数用于释放指定的`zlexrangespec`数据：

```c
void zslFreeLexRange(zlexrangespec *spec);
```

*Redis*为跳跃表基于字典顺序的范围操作提供了一套与基于分值范围的操作相类似的操作接口，当然基于字典顺序的操作时对`zskiplistNode.ele`字段进行比较的。这一系列接口包括：

```c
int zslIsInLexRange(zskiplist *zsl, zlexrangespec *range);
```

`zslIsInLexRange`用于判断跳跃表中存在一部分节点恰好落在`zlexrangespec`所给定的范围内。

```c
zskiplistNode *zslFirstInLexRange(zskiplist *zsl, zlexrangespec *range);
zskiplistNode *zslLastInLexRange(zskiplist *zsl, zlexrangespec *range);
```

而上述的两个函数则别分用于获取落在`zlexrangespec`范围之内的第一个节点与最后一个节点。

```c
unsigned long zslDeleteRangeByLex(zskiplist *zsl, zlexrangespec *range, dict *dict);

```

而`zslDeleteRangeByLex`函数则是会从跳跃表中删除落在`zlexrangespec`范围内的所有节点。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
