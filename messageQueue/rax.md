# Redis中基数树的实现
基数树是一种有序的字典树，按照*key*的字典顺序进行排序，可以支持对*key*的快速定位、插入和删除操作。*Redis*实现自身的基数树的主要目标是在性能与内存使用之间找到适当的平衡，同时可以满足许多不同需求的功能齐全的基数树的实现方式。

基数树可以被认为是前缀树的一种优化形式，对于前缀树，其具有三个特点：
1. 前缀树的根节点不包含任何字符，除了根节点之外每个节点只有一个字符。
2. 从根节点到某一个节点`x`路径上所有节点字符组成的字符串，便是`x`节点对应的字符串。
3. 一个节点对应的每一个子节点，其对应的字符都不相同。

基数树在组织形式上与前缀树有一定的差异，其节点不只可以表示单一字符，而且可以表示字符串的公共子串。无论是单一字符，还是公共子串，其主旨的思想都是利用字符串的公共前缀来减少查询时间。而对于基数树来说，每一个叶子节点都是一个数据条目。

![前缀数与基数树]()

尽管用户对于基数树的实用性以及适用性越来越感兴趣，但是*Redis*的作者发现，实现一个健壮的基数树，尤其是具有较为灵活的迭代器同时功能齐全的基数树是一件十分困难的事情。因为在处理节点的拆分与合并以及各种边缘情况时，非常容易出错。

## 基数树节点

### 基数树节点分类

对于基数树的节点来说存在**普通节点**与**压缩节点**两种区分。
对于一个**普通节点**，如果其布局格式如下所示：

    +----+---+--------+--------+--------+--------+
    |HDR |xyz| x-ptr  | y-ptr  | z-ptr  |dataptr |
    +----+---+--------+--------+--------+--------+

这表示这个基数树节点具有三个子节点，分别通过匹配字符`x`，`y`以及`z`来转移的对应的子节点，而这个节点与其子节点的关联是通过存储在`xyz`后面的三个子节点指针实现的。

而**压缩节点**，其在内存之中的分布格式则如下所示：

    +----+---+---------+
    |HDR |ABC|child-ptr|
    +----+---+---------+

如何理解这种**压缩节点**呢，可以考虑如下的场景，在基数树的某一个局部位置，从某一个节点开始到另外一个节点结束中间的每个节点均只有一个子节点，也就意味着在这个基数树的局部，树形结构已经退化成一个链表结构。如果依然按照普通基数树节点一样为这个链表中的每一个节点都创建一个真实的基数树节点的话，一来会浪费内存；二来在节点到其字节点的匹配跳转过程中，可能会因为缓存失效而导致系统的性能降低。既然在基数树局部这个链表结构里，节点转移的路由是唯一的，那么可以将链表之中的节点整合成为一个压缩节点，这样既可以节约内存的使用，同时还可以提高遍历访问的效率。

### 基数树节点数据结构

*Redis*为基数树节点定义的数据数据为：

```c
typedef struct raxNode {
	uint32_t iskey:1;
	uint32_t isnull:1;
	uint32_t iscompr:1;
	uint32_t size:29;
	unsigned char data[];
} raxNode;
```
这个数据结构中：
1. `raxNode.iskey`，这个字段表示当前节点是否是一个完整的*key*。可以理解为节点是一个*key*，那么从基数树的根节点下降到当前节点路径上的字符所组成的字符串便是该基数树*key-value*空间中的一个*key*，需要注意的是，这个*key*是不包含当前节点的内容的，另外`raxNode`只有是*key*的时候，才会保存对应的*value*。
2. `raxNode.isnull`，该字段表示当前节点是否为`NULL`，应用这个字段便可以不用分配内存存储`NULL`指针，只使用这个标记标明这个节点为`NULL`。
3. `raxNode.iscompr`，该字段用于标记当前节点是**普通节点**还是**压缩节点**。
4. `raxNode.size`，对于压缩节点，这个字段表示压缩数据的长度；如果这个节点是非压缩节点，则表示该节点的子节点的个数。
5. `raxNode.data`，则是存储这个基数树节点的数据载荷，前面介绍的关于基数树节点的内存分布，`raxNode`结构体的前四个字段都是属于**HDR**的内容，而子节点的分支，子节点的指针以及数据指针都是存储在`raxNode.data`这个字段之中的。

这里需要为`raxNode`节点中的指针数据约定一下各自的名称，后文将都会以此名称进行称呼：

1. **数据指针**，即基数树键空间之中，对应只想某一个*Key*的*Value*数据的指针。当`raxNode.iskey`为1的情况下，在`raxNode`节点数据的最后便会存储一个**数据指针**，用于指向当前节点所代表的*Key*对应的数据*Value*。
2. **子节点指针**，则是指代在当前基数树节点`raxNode`之中存储的，用于指向其子节点的指针。

### 基数树节点的基础操作逻辑

针对这个基数树的节点数据结构，*Redis*在*src/rax.c*源文件之中提供了几个宏定义用于实现关于`raxNode`的基础操作。

```c
#define raxPadding(nodesize)
```
通过前面对于`raxNode`节点描述，我们可以知道，`raxNode`前面的`HDR`数据固定占用4个字节的空间，存储在`raxNode.data`中的各个指针数据固定占用4个字节或者8个字节的空间，而节点中存储字符的个数是变长的，处于对齐的考虑，需要在`raxNode.data`中补充几个字节用于数据对齐，`raxPadding`便是用于计算需要几个空字节来进行对齐。

接下来我们便可以通过下面这个宏，来获取一个给定的`raxNode`实际占用的内存大小。
```c
#define raxNodeCurrentLength(n)
```

最后我们可以通过下面这两个宏，来获取指向存储在`raxNode.data`数据中第一个以及最后一个**子节点指针**的指针，注意这两个宏所返回的数据类型为`raxNode**`。
```c
#define raxNodeLastChildPtr(n)
#define raxNodeFirstChildPtr(n)
```

除了上述的四个宏定义之外，*Redis*还定义了一个用于分配初始化`raxNode`节点的函数：
```c
raxNode *raxNewNode(size_t children, int datafield);
```
在这个创建函数中，参数`children`表示这个待创建的`raxNode`节点具有的子节点个数，`datafield`则表示这个节点是否拥有一个存储**数据指针**的空间，这个函数调用会根据参数计算节点所需要的内存，并分配空间，并对其中的数据字段进行初始化，最终返回这个新节点的指针。

如果一个已经存在的基数树节点，其`raxNode.data`数据之中不包含**数据指针**，当我们希望为其分配一个**数据指针**时，可以调用下面这个函数来进行操作：
```c
raxNode *raxReallocForData(raxNode *n, void *data);
```
不过需要注意的是，`raxReallocForData`这个函数仅仅为数据指针分配了新的空间，并没有执行数据的赋值。如果需要将**数据指针**写入`raxNode.data`数据之中的话，则需要通过下面这个函数来执行：
```c
void raxSetData(raxNode *n, void *data);
```

此外，我们可以通过下面这个函数来获取`raxNode`节点之中的**数据指针**：

```c
void *raxGetData(raxNode *n);
```
当给定节点`raxNode.isnull`字段为空的时候，那么表示当前节点不存在数据指针，`raxGetData`会返回一个空值。

### 基数树节点中子节点的插入

```c
raxNode *raxAddChild(raxNode *n, unsigned char c, raxNode **childptr, raxNode ***parentlink);
```

`raxAddChild`这个函数会向给定的基数树节点`n`中插入一个表示字符`c`的子节点，由于在向节点中插入新的子节点可能会导致基数树节点被重新分配内存，因此这个函数会返回新的`n`节点的指针，新创建的子节点会通过`childptr`参数返回，这个节点在`raxNode.data`中存储的指针的位置会通过`parentlink`的参数进行返回。这里需要注意的是，待插入节点`n`不可以是压缩节点，必须是一个普通的节点。

例如，下图所显示的，一个给定的基数树普通节点，该节点具有四条分支子节点，分别用于匹配`abde`四个字符，

![基数树节点](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E8%8A%82%E7%82%B9.PNG)

当需要向这个节点之中插入一个表示匹配字符`c`的子节点：

![基数树节点插入](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E8%8A%82%E7%82%B9%E6%8F%92%E5%85%A5.PNG)

```c
raxNode *raxCompressNode(raxNode *n, unsigned char *s, size_t len, raxNode **child)
```

而上面这个`raxCompressNode`则用于将一个普通基数树节点`n`转化为一个压缩节点，这个待转化的节点`n`不能拥有任何**子节点指针**，函数会返回节点`n`的新指针。

### 基数树节点中子节点的删除

*Redis*从一个给定的基数树节点之中删除子节点，是通过下面这个`raxRemoveChild`函数来实现的。

```c
raxNode *raxRemoveChild(raxNode *parent, raxNode *child);
```

`raxRemoveChild`这个函数会从父节点`parent`中删除`child`子节点，并返回`parent`的新指针。与`raxAddChild`类似，也需要在删除完**子节点指针**之后，对`parent`节点的内存进行调整，并重新分配内存。我们可以使用下面这个图示来表示一个子节点的删除过程。

![基数树节点的删除]()

## 基数树的实现逻辑

### 基数树的数据结构

*Redis*会使用`raxNode`来创建一个基数树结构：

```c
typedef struct rax {
    raxNode *head;
    uint64_t numele;
    uint64_t numnodes;
} rax;
```

在这个基数树结构体之中：

1. `rax.head`，记录基数树的根节点。
2. `rax.numele`，存储基数树中数据元素的个数，也可以理解为`raxNode.iskey`字段为1的节点的个数。
3. `rax.numnodes`，存储基数树之中`raxNode`节点的个数。


处于节省空间的考虑，*Redis*没有为基数树节点设定一个指向父节点的指针。这样带来一个问题便是无法通过一个基数树节点回溯它的父节点，因此在对基数树执行遍历的时候，*Redis*设计了一个栈结构`raxStack`以深度优先的方式存储遍历过程中从根节点到当前节点路径上的节点指针，用以辅助对基数树的遍历。
```c
#define RAX_STACK_STATIC_ITEMS 32
typedef struct raxStack {
	void **stack;
	size_t items, maxitems;
	void *static_item[RAX_STACK_STATIC_ITEMS];
	int oom;
} raxStack;
```
这个栈结构在初始时，会有一个深度为`RAX_STACK_STATIC_ITEMS`的静态数组栈空间`raxStack.static_item`，`raxStack.stack`指针对指向这个数组栈空间，当栈中元素的个数超过`RAX_STACK_STATIC_ITEMS`时，会动态分配新的内存作为栈的存储空间，`raxStack.oom`字段用于标记当前这个栈结构是使用静态数据作为栈空间还是使用动态分配内存作为栈空间。而`raxStack.items`以及`raxStack.maxitems`分别用于存储这个栈中当前元素的个数以及栈空间的最大值。

在*src/rax.c*源文件中，定义了用于操作`raxStack`的函数接口：
|函数接口|用途|
|--------|----|
|`void raxStackInit(raxStack *ts)`|这个接口用于初始一个给定的`raxStack`|
|`int raxStackPush(raxStack *ts, void *ptr)`|用于向`raxStack`中压入一个指针|
|`void *raxStackPop(raxStack *ts)`|用于从`raxStack`中弹出一个指针|
|`void *raxStackPeek(raxStack *ts)`|用于从`raxStack`中获取栈顶的指针，但是不弹出|

### 基数树基础操作

```c
rax *raxNew(void)
{
	rax *rax = rax_malloc(sizeof(*rax));
    rax->numele = 0;
    rax->numnodes = 1;
    rax->head = raxNewNode(0,0);
    ...
}
```
*Redis*会使用`raxNew`来创建一个新的基数树，并返回基数树的指针；在这个新的基数树会通过`raxNewNode`来创建一个空的基数树节点`raxNode`用作为基数树的根节点。

```c
uint64_t raxSize(rax *rax);
```

同时*Redis*会使用上面`raxSize`这个函数，通过返回`rax.numele`来获取基数树之中数据元素的个数。

此外，*Redis*还为基数树实现了一个深度优先的递归释放节点，用于释放基数树中所有分配的节点`raxNode`，最后释放整个基数树`rax`结构。

```c
void raxRecursiveFree(rax *rax, raxNode *n, void (*free_callback)(void*));
void raxFreeWithCallback(rax *rax, void (*free_callback)(void*));
void raxFree(rax *rax);
```

这三个函数之中`raxRecursiveFree`是一个辅助的递归函数，用于递归地释放基数树节点。如果给出了`free_callback`回调函数参数，那么当*Key*节点在被释放前会调用`free_callback`接口来处理对应的*Value*被释放前的回调逻辑。另外两个释放函数都是通过`raxRecursiveFree`来实现的。我们来看看这个函数的实现细节，也可以重温一下，如何以深度优先的方式对一个树节点进行递归的遍历：

```c
void raxRecursiveFree(rax *rax, raxNode *n, void(*free_callback)(void*))
{
    int numchildren = n->iscompr ? 1 : n-size;
    raxNode **cp = raxNodeLastChildPtr(n);
    while(numchildren--)
    {
        raxNode *child;
        memcpy(&child, cp, sizeof(child));
        raxRecursiveFree(rax, child, free_callback);
        cp --;
    }
    if (free_callback && n->iskey && !n->isnull)
        free_callback(raxGetData(n));
    rax_free(n);
    rax->numnodes--;
}
```



### 基数树的查找操作

```c
void *raxFind(rax *rax, unsigned char *s, size_t len);
```
`raxFind`这个函数用于从基数树中查找一个长度为`len`的字符串`s`所对应的*key*，如果找到，那么返回这个*key*所对应的*value*，如果在这个基数树中不存在队形的*key*，那么函数会返回一个特殊的标记值`raxNotFound`，这个值被定义为：

```c
void *raxNotFound = (void *)"rax-not-found-pointer";
```
这个查找函数本身比较简单，它是通过`raxLowWalk`这个底层还是来实现查找的基础逻辑。`raxLowWalk`这个函数的原型为：
```c
static inline size_t raxLowWalk(rax *rax, unsigned char *s, size_t len, raxNode **stopnode, raxNode ***plink, int *splitpos, raxStack *ts);
```
这个函数用于实现从给定的基数树`rax`中查找`len`长的字符串`s`所对应的*key*，
1. 这个函数返回*key*中能在基数树中匹配到的最大前缀的长度。这也就意味着，如果返回值不等于*key*的`len`，那么说明在查找过程中因为不匹配的字符，导致查找提前结束；不过即使返回的长度等于字符串的长度`len`时也不表示在基数树中存在长度为`len`的字符串`s`的*Key*，有可能这个字符串`s`是树中某个*Key*的前缀。
2. 如果`stopnode`这个参数没有传`NULL`，那么搜索停止的节点将会通过这个参数返回给调用者。这个停止节点可以是搜索成功所停止的节点；也可以是匹配失败提前终止的节点。这个特性，对于向基数树中插入节点的操作很重要。
3. 如果`plink`这个参数没有传`NULL`，则会通过这个参数返回`stopnode`节点的父节点。
4. 最后，如果匹配搜索操作在一个压缩节点停止的话，根据前面讲解的压缩节点的含义，其`raxNode.data`中存储的字符按照一个子串而非分支的情况参与匹配，那么匹配很有可能会在子串内部的某一个位置停止，对于这种情况，`splitpos`参数会将这个子串内部的停止位置返回给调用者。
5. 如果在搜索过程中传入了`ts`参数，那么在搜索匹配过程中的所遍历过的节点都会被压入到这个`raxStack`数据机构之中。

对于`raxFind`函数，在执行完`raxLowWalk`底层操作之后，会通过如下的方式来判断是否查找成功：
```c
if (i != len || (h->iscompr && splitpos != 0) || !h->iskey)
	return raxNotFound;
```
也就是，
1. 如果匹配搜索提早结束，那么认为查找失败。
2. 如果匹配的终止节点是一个压缩节点，同时`splitpos`不为0，那么认为查找失败。
3. 如果匹配的终止节点不是一个包含*key*节点，那么认为查找失败。可以认为这种情况下，待搜所的*key*是基数树中*key*空间的某一个*key*的子串。

### 基数树的插入操作

在基数树中最为核心也是最为复杂的操作便是节点的插入，对于这种搜索树的插入操作，同时是调用查找函数找到一个合适的位置，然后将新的*Key*添加到基数树之中。

为了实现基数树的插入操作，*Redis*定义了一个基数树插入操作的底层函数接口：

```c
int raxGenericInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old, int overwrite);
```

这个底层函数接口会向基数树`rax`之中，插入一对*Key-Value*，其中这个*Key*是长度为`len`的字符串`s`；而*Value*则是`data`指针所指向的数据。如果待插入的*Key*不存在于基数树之中，那么这对*Key-Value*将会被插入，同时函数返回1。如果*Key*已经存在，并且`overwrite`参数为1时，旧的*Value*将会被覆盖，同时函数返回0。

`raxGenericInsert`这个函数首先回调用`raxLowWalk`底层搜索函数找到合适的插入位置，然后在执行后续的插入逻辑。下面我们将根据查找之后的不同情况来介绍其对应的插入逻辑。

我们定义变量`i`为函数`raxLowWalk`的返回结果，也就是字符串s在基数树之中可以匹配到最大前缀。首先我们先来看一种最为简单的插入情况，`raxLowWalk`函数在基数树中完全匹配了字符串`s`，同时匹配的终止节点是一个普通节点或者虽然终止在一个压缩节点，但是字符串匹配的结果没有落在压缩节点内（也就是`raxLowWalk`的`splitpos`结果为0）。对于这种情况，我们只需要简单地将参数`data`作为**数据指针**添加这个终止节点之中便可以完成基数树的插入过程。

![简单的基数树插入过程]()

如果`raxLowWalk`函数返回的最大匹配前缀恰好落在了一个压缩节点的中间，那么我们需要对这个压缩节点先进行拆分，才能继续进行插入过程。为了描述拆分压缩节点的各种情况，我们先假设一个基数树的局部，在这个基数树中存在一个`[XXX]ANNIBALE`的*Key*，其中`ANNIBALE`存储在一个压缩节点`a`之中；另外存在一个`[XXX]ANNIBALESCO`的*Key*，而`SCO`存储在另外一个压缩节点`b`之中，这个基数树可以用下面的视图所表示：

![基数树的局部]()

现在我们可以将压缩节点的插入过程划分成5种情况来看待：

### 基数树的删除操作

```c
int raxRemove(rax *rax, unsigned char *s, size_t len, void **old);
```
这个函数的作用是从给定的基数树`rax`中删除由`s`和`len`所指定的*key*，如果*key*被找到并且被删除，那么函数返回1，否则函数会返回0。当*key*被找到并被删除，同时`old`参数没有传`NULL`，那么这个*key*对应的*value*会通过`old`参数返回给调用者。


```c
void raxRecursiveFree(rax *rax, raxNode *n, void (*free_callback)(void *));
void raxFreeWithCallback(rax *rax, void (*free_callback)(void *));
void raxFree(rax *rax);
```

## 基数树的迭代器遍历

### 迭代器的数据结构

*Redis*定义了一个迭代器数据结构，用于辅助实现对于基数树的遍历：

```c
typedef struct raxIterator {
    int flags;
    rax *rt;
    unsigned char *key;
    void *data;
    size_t key_len;
    size_t key_max;
    unsigned char key_static_string[RAX_ITER_STATIC_LEN];
    raxNode *node;
    raxStack stack;
    raxNodeCallback node_cb;
} raxIterator;
```

在这个迭代器数据结构之中：

1. `raxIterator.flags`
2. `raxIterator.rt`
3. `raxIterator.key`
4. `raxIterator.data`
5. `raxIterator.key_len`
6. `raxIterator.key_max`
7. `raxIterator.key_static_string`
8. `raxIterator.node`
9. `raxIterator.stack`
10. `raxItertor.node_cb`

### 迭代器基础操作

*Redis*会使用下面`raxStart`以及`raxStop`这两个函数来初始化一个

```c
void raxStart(raxIterator *it, rax *rt);
void raxStop(raxIterator *it);
```

```c
int raxIteratorNextStep(raxIterator *it, int noup);
int raxSeekGreatest(raxIterator *it);
int raxIteratorPrevStep(raxIterator *it, int noup);
```

### 迭代器定位操作

```c
int raxSeek(raxIterator *it, const char *op, unsigned char *ele, size_t len);
int raxNext(raxIterator *it);
int raxPrev(raxIterator *it);
int raxRandomWalk(raxIterator *it, size_t steps);
int raxCompare(raxIterator *iter, const char *op, unsigned char *key, size_t key_len);
int raxEOF(raxIterator *it);

```

### 迭代器操作流程

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
