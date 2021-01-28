# Redis中基数树的实现
基数树是一种有序的字典树，按照*key*的字典顺序进行排序，可以支持对*key*的快速定位、插入和删除操作。*Redis*自己实现一个基数树的主要目的是在性能与内存使用之间找到适当的平衡，同时可以满足许多不同需求的功能齐全的基数树的实现方式。

基数树可以被认为是前缀树的一种优化形式，首先我们来看一下基础的前缀树数据结构，对于前缀树，其具有三个特点：
1. 前缀树的根节点不包含任何字符，除了根节点之外每个节点只有一个字符。
2. 从根节点到某一个节点`x`路径上所有节点字符组成的字符串，便是`x`节点对应的字符串。
3. 对于一个节点，它的每一个子节点，其对应的字符都不相同。

基数树在组织形式上与前缀树有一定的差异，其节点不但可以表示单一字符，同时也可以表示字符串的公共子串。无论是单一字符，还是公共子串，其主旨的思想都是利用字符串的公共前缀来减少查询时间。而对于基数树来说，每一个叶子节点都是一个数据条目。

![前缀树与基数树](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%89%8D%E7%BC%80%E6%A0%91%E4%B8%8E%E5%9F%BA%E6%95%B0%E6%A0%91.JPG)

尽管用户对于基数树的实用性以及适用性越来越感兴趣，但是*Redis*的作者发现，实现一个健壮的基数树，尤其是具有较为灵活的迭代器、同时功能齐全的基数树是一件十分困难的事情。因为在处理节点的拆分与合并以及各种边缘情况时，非常容易出错。

## 基数树节点

### 基数树节点分类

对于基数树的节点来说，存在**普通节点**与**压缩节点**两种区分。
对于一个**普通节点**，如果其布局格式如下所示：

    +----+---+--------+--------+--------+--------+
    |HDR |xyz| x-ptr  | y-ptr  | z-ptr  |dataptr |
    +----+---+--------+--------+--------+--------+

上述这个**普通节点**表示这个基数树节点具有三个子节点，分别通过匹配字符`x`，`y`以及`z`来转移到对应的子节点，而这个节点与其子节点的关联是通过存储在`xyz`后面的三个子节点指针实现的。

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
这里需要为`raxNode`节点中的指针数据约定一下各自的名称，文章的后续将都会以此名称进行称呼：

1. **数据指针**，即基数树键空间之中，指向某一个*Key*对应的*Value*数据的指针。当`raxNode.iskey`为1的情况下，在`raxNode`节点数据的最后字段`dataptr`便会存储一个**数据指针**，用于指向当前节点所代表的*Key*对应的数据*Value*。
2. **子节点指针**，则是指代在当前基数树节点`raxNode`之中存储的，用于指向其子节点的指针。

这个数据结构中：

1. `raxNode.iskey`，这个字段表示当前节点是否是一个完整的*key*。如果节点是一个*key*，那么从基数树的根节点下降到当前节点路径上的字符所组成的字符串便是该基数树*key-value*空间中的一个*key*。需要注意的是，这个*key*是不包含当前节点的内容的，另外`raxNode`只有是*key*的时候，才会保存对应的*value*，并拥有**数据指针**`dataptr`。
2. `raxNode.isnull`，只有`raxNode.iskey`为1的时候，才会有**数据指针**，但是也会存在只有*Key*而没有*Value*的情况。`raxNode.isnull`标记这个节点是否为一个空节点，如果`raxNode.isnull`被设置为1，则该节点不需要为**数据指针**分配存储内存。
3. `raxNode.iscompr`，该字段用于标记当前节点是**普通节点**还是**压缩节点**。
4. `raxNode.size`，对于压缩节点，这个字段表示压缩数据的长度；如果这个节点是非压缩节点，则表示该节点的子节点的个数。
5. `raxNode.data`，这个字段用来存储这个基数树节点的数据载荷，前面介绍的关于基数树节点的内存分布，`raxNode`结构体的前四个字段都是属于**HDR**的内容，而子节点的分支字符，子节点的指针以及数据指针都是存储在`raxNode.data`这个字段之中的。

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

最后我们可以通过下面这两个宏，来获取指向存储在`raxNode.data`数据中第一个以及最后一个**子节点指针**的指针，注意这两个宏所返回的数据类型为`raxNode**`，因为`raxNode.data`中存储的是`raxNode*`，也就是指向`raxNode`的指针，而下面这两个宏返回的是指向`raxNode.data`中数据的指针，也就是指向 `raxNode`指针的指针，故此应该是`raxNode**`类型。
```c
#define raxNodeLastChildPtr(n)
#define raxNodeFirstChildPtr(n)
```

除了上述的四个宏定义之外，*Redis*还定义了一个用于分配初始化`raxNode`节点的函数：
```c
raxNode *raxNewNode(size_t children, int datafield);
```
在这个创建函数中，参数`children`表示这个待创建的`raxNode`节点具有的子节点个数，`datafield`则表示这个节点是否拥有一个存储**数据指针**的空间，这个函数调用会根据参数计算节点所需要的内存，并分配空间，并对其中的数据字段进行初始化，最终返回这个新的`raxNode`节点的指针。

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

### 基数树节点中子节点的插入与删除

```c
raxNode *raxAddChild(raxNode *n, unsigned char c, raxNode **childptr, raxNode ***parentlink);
```

`raxAddChild`这个函数会向给定的基数树节点`n`中按照字典顺序插入一个表示字符`c`的子节点，由于在向节点中插入新的子节点的操作可能会导致基数树节点的内存被重新分配，因此这个函数会返回新的`n`节点的指针。新创建的子节点的指针会通过`childptr`参数返回，这个节点在`raxNode.data`中存储的指针的位置会通过`parentlink`的参数进行返回。这里需要注意的是，待插入节点`n`不可以是压缩节点，必须是一个普通的节点。

例如，下图所显示的，一个给定的基数树普通节点，该节点具有四条分支子节点，分别用于匹配`abde`四个字符，

![基数树节点](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E8%8A%82%E7%82%B9.PNG)

当需要向这个节点之中插入一个表示匹配字符`c`的子节点：

![基数树节点插入](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E8%8A%82%E7%82%B9%E6%8F%92%E5%85%A5.PNG)

```c
raxNode *raxCompressNode(raxNode *n, unsigned char *s, size_t len, raxNode **child)
```

而上面这个`raxCompressNode`则用于将一个普通基数树节点`n`转化为一个压缩节点，这个待转化的节点`n`不能拥有任何**子节点指针**，函数会返回节点`n`的新指针。

与插入节点相反的，*Redis*从一个给定的基数树节点之中删除子节点，是通过下面这个`raxRemoveChild`函数来实现的。

```c
raxNode *raxRemoveChild(raxNode *parent, raxNode *child);
```

`raxRemoveChild`这个函数会从父节点`parent`中删除`child`子节点，并返回`parent`的新指针。与`raxAddChild`类似，也需要在删除完**子节点指针**之后，对`parent`节点的内存进行调整，并重新分配内存。

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

1. `rax.head`，存储基数树的根节点。
2. `rax.numele`，存储基数树中数据元素键的个数，也可以理解为`raxNode.iskey`字段为1的节点的个数。
3. `rax.numnodes`，存储基数树之中`raxNode`节点的个数。


出于节省空间的考虑，*Redis*没有为基数树节点设定一个指向父节点的指针。这样带来一个问题便是无法通过一个基数树节点回溯它的父节点，因此在对基数树执行遍历的时候，*Redis*设计了一个栈结构`raxStack`以深度优先的方式存储遍历过程中从根节点到当前节点路径上的节点指针，用以辅助对基数树的遍历。
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

此外，*Redis*还为基数树实现了一个深度优先的递归释放节点的算法，用于释放基数树中所有分配的节点`raxNode`，最后释放整个基数树`rax`结构。

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
1. 这个函数返回*key*中能在基数树中匹配到的最大前缀的长度。这也就意味着，如果返回值不等于*key*的`len`，那么说明这是因为在查找过程中因为不匹配的字符，导致查找提前结束；不过即使返回的长度等于字符串的长度`len`时也不表示在基数树中存在这个长度为`len`的字符串`s`所构成的*Key*，有可能这个字符串`s`是树中某个*Key*的前缀。
2. 如果`stopnode`这个参数没有传`NULL`，那么搜索停止的节点将会通过这个参数返回给调用者。这个停止节点可以是搜索成功所停止的节点；也可以是匹配失败提前终止的节点。这个特性，对于向基数树中插入新节点的操作来说是很重要。
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
3. 如果匹配的终止节点不是一个包含*key*节点，那么认为查找失败。可以认为这种情况下，待搜索的*key*是基数树中*key*空间的某一个*key*的子串。

### 基数树的插入操作

在基数树中最为核心也是最为复杂的操作便是节点的插入。对于这种基数树的插入操作，其基础的逻辑是调用查找函数找到一个合适的位置，然后将新的*Key*添加到基数树之中。

为了实现基数树的插入操作，*Redis*定义了一个基数树插入操作的底层函数接口：

```c
int raxGenericInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old, int overwrite);
```

这个底层函数接口会向基数树`rax`之中，插入一对*Key-Value*，其中这个*Key*是长度为`len`的字符串`s`；而*Value*则是`data`指针所指向的数据。如果待插入的*Key*不存在于基数树之中，那么这对*Key-Value*将会被插入，同时函数返回1。如果*Key*已经存在，并且`overwrite`参数为1时，旧的*Value*将会被覆盖，同时函数返回0。

`raxGenericInsert`这个函数首先回调用`raxLowWalk`底层搜索函数找到合适的插入位置，然后在执行后续的插入逻辑。下面我们将根据查找之后的不同情况来介绍其对应的插入逻辑。

我们定义变量`i`为函数`raxLowWalk`的返回结果，也就是字符串`s`在基数树之中可以匹配到最大前缀。首先我们先来看一种最为简单的插入情况，`raxLowWalk`函数在基数树中完全匹配了字符串`s`，同时匹配的终止节点是一个普通节点，或者虽然终止在一个压缩节点，但是字符串匹配的结果没有落在压缩节点内（也就是`raxLowWalk`的`splitpos`结果为0）。对于这种情况，我们只需要简单地将参数`data`作为**数据指针**添加这个终止节点之中便可以完成基数树的插入过程。例如下图所示，向基数树之中插入一个`XYZ`的*Key*以及对应的*Value*。

![简单的基数树插入过程](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E7%AE%80%E5%8D%95%E7%9A%84%E5%9F%BA%E6%95%B0%E6%A0%91%E6%8F%92%E5%85%A5%E8%BF%87%E7%A8%8B.JPG)

上面这种插入情况是整个插入逻辑之中一个较为简单的情况，接下来我们来看看最为复杂的插入情况。如果`raxLowWalk`函数返回的最大匹配前缀恰好落在了一个压缩节点的中间，也就是传出的`splitpos`结果不为0的情况，那么我们需要对这个压缩节点先进行拆分后，才能继续进行插入过程。为了描述拆分压缩节点的各种情况，我们先假设一个基数树的局部，在这个基数树中存在一个`[XXX]ANNIBALE`的*Key*，其中`ANNIBALE`存储在一个压缩节点`a`之中；另外存在一个`[XXX]ANNIBALESCO`的*Key*，而`SCO`存储在另外一个压缩节点`b`之中，这个基数树可以用下面的视图所表示：

![基数树的局部](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E7%9A%84%E5%B1%80%E9%83%A8.JPG)

现在我们可以将压缩节点的插入过程划分成5种情况来看待：

1. 如果我们向这个局部的基数树之中插入一个字符串内容为`ANNIENTARE`的*Key*。对于这种情况，相当于`i != len && splitpos != 0`，即`raxLowWalk`的匹配前缀长度于字符串长度`len`不相等，同时匹配终止在**压缩节点**之中的位置`splitpos`不为0。![基数树压缩节点插入情况1](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E5%8E%8B%E7%BC%A9%E8%8A%82%E7%82%B9%E6%8F%92%E5%85%A5%E6%83%85%E5%86%B51.JPG)
2. 向这个局部的基数树之中插入一个字符串内容为`ANNIBALI`的*Key*。这种上情况与上述的情况有一些类似，区别在于情况1中，排除`ANNI`这个公共前缀以及转移的字符`E`以后，剩下的`NTARE`可以构成一个**压缩节点**；而这种情况里排除公共前缀以及转移字符之后，没有多余的字符，这里将会插入一个**普通节点**。![基数树压缩节点插入情况2](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E5%8E%8B%E7%BC%A9%E8%8A%82%E7%82%B9%E6%8F%92%E5%85%A5%E6%83%85%E5%86%B52.JPG)
3. 向这个局部的基数树之中插入一个字符串内容为`AGO`的*Key*。这种情况相当于`i != len && i == 1 && splitpos != 0`。这种情况与上面的情况1类似，但是情况1中匹配的终止节点依然保留为**压缩节点**；而这种情况里，匹配的终止节点会从**压缩节点**转化为**普通节点**。![基数树压缩节点插入情况3](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E5%8E%8B%E7%BC%A9%E8%8A%82%E7%82%B9%E6%8F%92%E5%85%A5%E6%83%85%E5%86%B53.JPG)
4. 向这个局部的基数树之中插入一个字符串内容为`CIAO`的*Key*。这种情况可以理解为`i == 0 && splitpos == 0`，即待插入的*Key*完全不匹配终止节点中的字符串，这意味着需要这次插入不但需要修改终止节点，包括终止节点的父节点也需要被修改。![基数树压缩节点插入情况4](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E5%8E%8B%E7%BC%A9%E8%8A%82%E7%82%B9%E6%8F%92%E5%85%A5%E6%83%85%E5%86%B54.JPG)
5. 向这个局部的基数树之中插入一个字符串内容为`ANNI`的*Key*。这种情况可以理解为`i == len && splitpos != 0`，即待插入的字符串`ANNI`在基数树之中可以完全匹配，但是匹配终止落在**压缩节点**的中间。![基数树压缩节点插入情况5](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E5%9F%BA%E6%95%B0%E6%A0%91%E5%8E%8B%E7%BC%A9%E8%8A%82%E7%82%B9%E6%8F%92%E5%85%A5%E6%83%85%E5%86%B55.JPG)

通过上面这个`raxGenericInsert`这个函数，*Redis*实现了两个对外的插入接口：

```c
int raxInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old);
int raxTryInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old);
```

`raxInsert`会向基数树之中插入一对*Key-Value*，如果*Key*以及存在于基数树之中的话，旧的*Value*会被更新；而`raxTryInsert`函数，在*Key*存在是，则不会执行后续的更新插入过程。

### 基数树的删除操作

```c
int raxRemove(rax *rax, unsigned char *s, size_t len, void **old);
```
这个函数的作用是从给定的基数树`rax`中删除由`s`和`len`所指定的*key*，如果*key*被找到并且被删除，那么函数返回1，否则函数会返回0。当*key*被找到并被删除，同时`old`参数没有传`NULL`，那么这个*key*对应的*value*会通过`old`参数返回给调用者。

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

1. `raxIterator.flags`，表示这个迭代器的标记，用来标记迭代器当前所处于的状态：
   1. `RAX_ITER_JUST_SEEKED`，表示当前迭代器刚刚完成`raxSeek`定位，在第一次遍历返回当前元素时，会清理掉这个状态。
   2. `RAX_ITER_EOF`，表示迭代器已经完成迭代。
   3. `RAX_ITER_SAFE`，表示这是一个安全迭代器，可以在遍历的过程中执行一些操作，但是速度会更慢一些。
2. `raxIterator.rt`，这是一个基数树指针，记录当前迭代器关联的基数树。
3. `raxIterator.key`，用于存储迭代器当前遍历到的*Key*的字符串。
4. `raxIterator.data`，存储当前迭代器遍历到的*Key*对应的*Value*。
5. `raxIterator.key_len`，记录当前*Key*的长度。
6. `raxIterator.key_max`，记录当前可以容纳的*Key*字符串的最大长度。
7. `raxIterator.key_static_string`，这是一个静态数组，初始使用这个数组来存储*Key*的字符串，只有当字符串长度超过`RAX_ITER_STATIC_LEN`的时候，才会使用动态分配的内存来存储*Key*的字符串。
8. `raxIterator.node`，在非安全遍历下，记录迭代器当前遍历的基数树节点。
9. `raxIterator.stack`，在非安全遍历下，记录从根节点到当前节点的遍历路径。
10. `raxItertor.node_cb`，遍历节点的回调函数，通常被设置为`NULL`。

### 迭代器基础操作

```c
void raxStart(raxIterator *it, rax *rt);
void raxStop(raxIterator *it);
void raxEOF(raxIterator *it);
```

上述三个基数树迭代器的基础操作中：

1. `raxStart`，用于将迭代器与一个基数树进行关联。
2. `raxStop`，当不在需要迭代器时，这个函数会释放`raxIterator.key`被动态分配的内存，以及释放`raxIterator.stack`这个栈结构。
3. `raxEOF`，通过`raxIterator.flags`标记中是否存在`RAX_ITER_EOF`来判断迭代是否已经结束。

不过这里需要注意的是，`raxStart`初始化后的迭代器不可以立即进行迭代遍历，因为初始的迭代器便携带`RAX_ITER_EOF`标记。如果对这个迭代器进行迭代，那么迭代会立即结束。通过`raxStart`初始化后的迭代器只有在**SEEK**操作之后，才可以进行常规意义上的迭代遍历操作。

```c
int raxIteratorAddChars(raxIterator *it, unsigned char *s, size_t len);
void raxIteratorDelChars(raxIterator *it, size_t count);
```

上面这两个函数用于操作存储在`raxIterator.key`字段之中*Key*对应的字符串：

1. `raxIteratorAddChars`，用于将一个长度为`len`的子字符串`s`追加到`raxIterator.key`之中，这里如果`raxIterator.key`中字符串的长度没有超过`RAX_ITER_STATIC_LEN`，那么会使用`raxIterator.key_static_string`这个静态数组进行存储，否则便会为字符串动态分配内存。
2. `raxIteratorDelChars`，这个函数用于从迭代器*Key*字符串`raxIterator.key`的最右侧开始移除`count`个字符。

*Redis*基数树中的*Key*都是字符串，因此*Redis*提供了一个字符串与*Key*进行比较的函数：

```c
int raxCompare(raxIterator *iter, const char *op, unsigned char *key, size_t key_len)
```

这个函数可以比较一个迭代器所代表的*Key*的内容`raxIterator.key`与一个长度为`key_len`的字符串`key`进行比较。如果比较结果为真，则函数返回1；否则返回0。通过参数`op`可以给出比较的类型，可用的比较类型包括`==`，`>=`，`>`，`<=`，`<`这五种比较方式。

### 迭代器定位操作

由于在*Redis*的基数树中，*Key*是按照字典的升序顺序进行排列的，那么下面`raxSeekGreatest`这个函数便是用于将一个给定的迭代器定位到其所关联的基数树中的最大的那个*Key*所对应的节点。

```c
int raxSeekGreatest(raxIterator *it);
```

在了解完上面`raxSeekGreatest`这个特殊的定位函数之后，我们来看一下*Redis*为基数树迭代器实现的一个通用的定位接口：

```c
int raxSeek(raxIterator *it, const char *op, unsigned char *ele, size_t len);
```

这个函数可以根据比较类型`op`来为迭代器在基数树中定位不用的元素，例如下面这段代码：

```c
raxSeek(&iter,">=",(unsigned char*)"foo",3);
```

会将迭代器重新定位到基数树中第一个大于等于`foo`的*Key*，这里面的操作类型`op`除了可以是`raxCompare`函数中的`==`，`>=`，`>`，`<=`，`<`这五种比较操作类型之外，还可以是`^`以及`$`分别用于定位基数树中最小的以及最大的*Key*。

### 迭代器遍历操作

```c
int raxNext(raxIterator *it);
int raxPrev(raxIterator *it);
```

当我们通过`raxSeek`函数，对一个基数树迭代器进行初始化之后，便可以应用`raxNext`以及`raxPrev`函数对基数树进行迭代遍历。当迭代到达结束时，上述两个函数会返回0；否则函数返回1。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
