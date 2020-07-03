# Redis中基数树的实现
基数树是一种有序的字典树，按照*key*的字典顺序进行排序，可以支持对*key*的快速定位、插入和删除操作。*Redis*实现自身的基数树的主要目标是在性能与内存使用之间找到适当的平衡，同时可以满足许多不同需求的功能齐全的基数树的实现方式。

基数树可以被认为是前缀树的一种优化形式，对于前缀树，其具有三个特点：
1. 前缀树的根节点不包含任何字符，除了根节点之外每个节点只有一个字符。
2. 从根节点到某一个节点`x`路径上所有节点字符组成的字符串，便是`x`节点对应的字符串。
3. 一个节点对应的每一个子节点，其对应的字符都不相同。

基数树在组织形式上与前缀树有一定的差异，其节点不知可以表示单一字符，而且可以表示字符串的公共子串。无论是单一字符，还是公共子串，其主旨的思想都是利用字符串的公共前缀来减少查询时间。而对于基数树来说，每一个叶子节点都是一个数据条目。

尽管用户对于基数树的实用性以及适用性越来越感兴趣，但是*Redis*的作者发现，实现一个健壮的基数树，尤其是具有较为灵活的迭代器同时功能齐全的基数树是一件十分困难的事情。因为在处理节点的拆分与合并以及各种边缘情况时，非常容易出错。

## 基数树基础数据结构

对于基数树的节点来说存在普通节点与压缩节点两种区分。
对于一个普通节点，如果其布局格式如下所示：

    +----+---+--------+--------+--------+--------+
	|HDR |xyz| x-ptr  | y-ptr  | z-ptr  |dataptr |
	+----+---+--------+--------+--------+--------+

这表示这个基数树节点具有三个子节点，分别通过匹配字符`x`，`y`以及`z`来转移的对应的子节点，而这个节点与其子节点的关联是通过存储在`xyz`后面的三个指针实现的。

而压缩节点，其在内存之中的分布格式则如下所示：

    +----+---+--------+
	|HDR |ABC|chld-ptr|
	+----+---+--------+

如何理解这种压缩节点呢，可以考虑如下的场景，在基数树的某一个局部位置，从某一个节点开始均只有一个子节点，也就意味着在这个基数树的局部，树形结构已经退化成一个链表结构。如果依然按照普通基数树节点一样为这个链表中的每一个节点都创建节点的话，一来会浪费内容。二来在节点到其字节点的匹配跳转过程中，可能会因为缓存失效而导致系统的性能降低。
既然在基数树局部这个链表结构里，节点转移的路由是唯一的，那么可以将链表之中的节点整合成为一个压缩节点，这样既可以节约内存的使用，同时还可以提高遍历访问的效率。

节点`HDR`的数据数据的定义为：
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
1. `raxNode.iskey`，这个字段表示当前节点是否是一个完整的*key*，可以理解为如果当前界限是一个*key*，那么从基数树的根节点下降到当前节点路径上的字符所组成的字符串便是该基数树*key-value*空间中的一个*key*，主要注意的是，这个*key*是不包含当前节点的内容的，另外`raxNode`只有是*key*的时候，才会保存对应的*value*。
2. `raxNode.isnull`，该字段表示当前节点是否为`NULL`，应用这个字段便可以不用分配内存存储`NULL`指针，只使用这个标记标明这个节点为`NULL`。
3. `raxNode.iscompr`，该字段用于标记当前节点是普通节点还是压缩节点。
4. `raxNode.size`，对于压缩节点，这个字段表示压缩数据的长度；如果这个节点是非压缩节点，则表示该节点的子节点的个数。

针对这个基数树的节点数据结构，*Redis*在*src/rax.c*源文件之中提供了几个宏定义用于实现关于`raxNode`的基础操作。
```c
#define raxPadding(nodesize)
```
通过前面对于`raxNode`节点描述，我们可以知道，`raxNode`前面的`HDR`数据固定占用4个字节的空间，存储在`raxNode.data`中的各个指针数据固定占用4个字节或者8个字节的空间，而节点中存储字符的个数是变长的，处于对齐的考虑，需要在`raxNode.data`中补充几个字节用于数据对齐，`raxPadding`便是用于计算需要几个空字节来进行对齐。

接下来我们便可以通过下面这个宏，来获取一个给定的`raxNode`实际占用的内存大小。
```c
#define raxNodeCurrentLength(n)
```

最后我们可以通过下面这两个宏，来获取指向存储在`raxNode.data`数据中第一个以及最后一个子节点指针的指针，注意这两个宏所返回的数据类型为`raxNode**`。
```c
#define raxNodeLastChildPtr(n)
#define raxNodeFirstChildPtr(n)
```

除了上述的四个宏定义之外，*Redis*还定义了一个用于分配初始化`raxNode`节点的函数：
```c
raxNode *raxNewNode(size_t children, int datafield);
```
该函数调用中，参数`children`表示其具有的子节点的个数，`datafield`则表示这个节点是否拥有一个存储数据指针的空间，这个函数调用会根据参数计算节点所需要的内存，并分配空间，并对其中的数据字段进行初始化，最终返回这个新节点的指针。

如果一个已经存在的基数树节点，其`raxNode.data`数据之中不包含数据指针，当我们希望为其分配一个数据指针时，可以调用下面这个函数来进行操作：
```c
raxNode *raxReallocForData(raxNode *n, void *data);
```
不过需要注意的是，`raxReallocForData`这个函数仅仅为数据指针分配了新的空间，但是没有执行数据的赋值。那么当需要将数据指针写入`raxNode.data`数据之中的话，则需要通过下面这个函数来执行：
```c
void raxSetData(raxNode *n, void *data);
```

最后，我们可以通过下面这个函数来获取`raxNode`节点之中的数据指针：
```c
void *raxGetData(raxNode *n);
```
当给定节点`raxNode.isnull`字段为空的时候，那么表示当前节点不存在数据指针，`raxGetData`会返回一个空值。


处于节省空间的考虑，基数树节点没有一个指向父节点的指针，因此在对基数树执行遍历的时候，*Redis*设计了一个栈结构`raxStack`来完成辅助基数树的遍历。
```c
#define RAX_STACK_STATIC_ITEMS 32
typedef struct raxStack {
	void **stack;
	size_t items, maxitems;
	void *static_item[RAX_STACK_STATIC_ITEMS];
	int oom;
} raxStack;
```
这个栈结构在初始时，会有一个深度为`RAX_STACK_STATIC_ITEMS`的静态数组栈空间`static_item`，`stack`指针对指向这个数组栈空间，当栈中元素的个数超过`RAX_STACK_STATIC_ITEMS`时，会动态分配新的内存作为栈的存储空间，`oom`字段用于标记当前这个栈结构是使用静态数据作为栈空间还是使用动态分配内存作为栈空间。

在*src/rax.c*源文件中，定义了用于操作`raxStack`的函数接口：
|函数接口|用途|
|--------|----|
|`void raxStackInit(raxStack *ts)`|这个接口用于初始一个给定的`raxStack`|
|`int raxStackPush(raxStack *ts, void *ptr)`|用于向`raxStack`中压入一个指针|
|`void *raxStackPop(raxStack *ts)`|用于从`raxStack`中弹出一个指针|
|`void *raxStackPeek(raxStack *ts)`|用于从`raxStack`中获取栈顶的指针，但是不弹出|


## 基数树的操作函数
*Redis*会使用：
```c
rax *raxNew(void);
```
来创建一个新的基数树，并返回基数树的指针。

### 节点的查找
```c
void *raxFind(rax *rax, unsigned char *s, size_t len);
```
`raxFind`这个函数用于从基数树中查找一个由参数`s`以及`len`标记的*key*，如果找到，那么返回这个*key*所对应的*value*，如果在这个基数树中不存在队形的*key*，那么函数会返回一个特殊的标记值`raxNotFound`，这个值被定义为：
```c
void *raxNotFound = (void *)"rax-not-found-pointer";
```
这个查找函数本身比较简单，它是通过`raxLowWalk`这个底层还是来实现查找的基础逻辑。`raxLowWalk`这个函数的原型为：
```c
static inline size_t raxLowWalk(rax *rax, unsigned char *s, size_t len,
								raxNode **stopnode, raxNode ***plink,
								int *splitpos, raxStack *ts);
```
这个函数用于实现从给定的基数树`rax`中查找`s`以及`len`所对应的*key*，
1. 这个函数返回*key*中能在基数树中匹配到的最大前缀的长度，这也就意味着，如果返回值不等于*key*的`len`，那么说明在查找过程中因为不匹配的字符，导致查找提前结束。
2. 如果`stopnode`这个参数没有传`NULL`，那么搜索停止的节点将会通过这个参数返回给调用者。这个停止节点可以是搜索成功所停止的节点；也可以是匹配失败提前终止的节点，这个特性，对于向基数树中插入节点的操作很重要。
3. 如果`plink`这个参数没有传`NULL`，则会通过这个参数返回`stopnode`节点的父节点。
4. 最后，如果匹配搜索操作在一个压缩节点停止的话，根据前面讲解的压缩节点的含义，其`raxNpde.data`中存储的字符按照一个子串而非分支的情况参与匹配，那么匹配很有可能会在子串内部的某一个位置停止，对于这种情况，`splitpos`参数会将这个子串内部的停止位置返回给调用者。
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


### 节点的插入
```c
int raxInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old);
int raxTryInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old);
```
上述两个函数用于实现向基数树中插入一对*key-value*，其中*key*通过参数`s`以及`len`描述，而*value*则是通过`data`参数来表示，两个函数的区别在于如果*key*已经存在于基数树之中，那么`raxInsert`函数会覆盖*key*所对应的*value*；而`raxTryInsert`函数则不会覆盖已有的*key-value*。对于基数树的插入，*Redis*分成了五种情况进行了讨论：


### 节点的删除
```c
int raxRemove(rax *rax, unsigned char *s, size_t len, void **old);
```


```c
void raxRecursiveFree(rax *rax, raxNode *n, void (*free_callback)(void *));
void raxFreeWithCallback(rax *rax, void (*free_callback)(void *));
void raxFree(rax *rax);
```

### 树的遍历
```c
void raxStart(raxIterator *it, rax *rt);
```

```c
int raxIteratorNextStep(raxIterator *it, int noup);
int raxSeekGreatest(raxIterator *it);
int raxIteratorPrevStep(raxIterator *it, int noup);
```

```c
int raxSeek(raxIterator *it, const char *op, unsigned char *ele, size_t len);
int raxNext(raxIterator *it);
int raxPrev(raxIterator *it);
int raxRandomWalk(raxIterator *it, size_t steps);
int raxCompare(raxIterator *iter, const char *op, unsigned char *key, size_t key_len);
void raxStop(raxIterator *it);
int raxEOF(raxIterator *it);
uint64_t raxSize(rax *rax);
```


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
