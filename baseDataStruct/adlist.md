# Redis中双端链表实现

## 双端链表基础数据结构
在*src/adlist.h*头文件中，定义*Redis*双端链表的基础数据结构，
包括链表节点，链表本体以及链表迭代器的数据结构
```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```
其中`listNode.value`字段用于保存指向真实数据内存的指针。

```c
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

```c
typedef struct listIter {
    listNode *next;
    int direction;
} listIter;
```


## 宏定义的基础操作

|宏定义|含义|
|-----|---|
|`#define listLength(l) ((l)->len)`|给定一个双端链表，返回其链表的长度|
|`#define listFirst(l) ((l)->head)`|给定一个双端链表，返回链表的头节点|
|`#define listLast(l) ((l)->tail)`|给定一个双端链表，返回链表的尾节点|
|`#define listPrevNode(n) ((n)->prev)`|给定一个节点，返回其前驱节点指针|
|`#define listNextNode(n) ((n)->next)`|给定一个节点，返回其后继节点指针|
|`#define listNodeValue(n) ((n)->value)`|返回节点的`value`节点指针|
|`#define listSetDupMethod(l,m) ((l)->dup = (m))`|为链表设定节点拷贝方法|
|`#define listSetFreeMethod(l,m) ((l)->free = (m))`|为链表设定节点释放方法|
|`#define listSetMatchMethod(l,m) ((l)->match = (m))`|为链表设定比较方法|
|`#define listGetDupMethod(l) ((l)->dup)`|返回链表拷贝方法的函数指针|
|`#define listGetFree(l) ((l)->free)`|返回链表释放方法的函数指针|
|`#define listGetMatchMethod(l) ((l)->match)`|返回链表比较方法的函数指针|

## 双端链表操作接口API

### 链表的创建、释放
```c
list *listCreate(void);
```
这个函数会调用`zmalloc`函数，动态分配一个双端链表的结构体，如果分配失败，该函数会返回`NULL`指针；
如果分配成功，那么会对`list.head`，`list.tail`，`list.len`字段进行初始化，成功后返回双端链表对象的指针。
但是需要注意的是，`list.dup`，`list.free`，`list.match`这三个字段默认设置为`NULL`，
如果有特殊需要设置的情况下，可以后续调用`listSetDupMethod`，`listSetFreeMethod`，`listGetDupMethod`手动设置。

```c
void listEmpty(list *list);
```
这个函数会从链表中删除并释放所有的节点，如果这个链表定义了`list.free`的回调函数，
那么在释放节点之前，会对节点`listNode.value`调用`list.free`进行处理；
这个函数并不会释放这个链表本身。

```c
void listRelease(list *list);
```
这个函数通过调用`listEmpty`清空链表的元素，同时调用`zfree`来释放链表对象的内存。

### 链表节点的增加、删除与查找
```c
list *listAddNodeHead(list *list, void *value);
list *listAddNodeTail(list *list, void *value)；
```
这两个函数会向链表头部或者尾部插入一个节点，这个节点将以给定的`value`指针多为节点的`listNode.value`。

```c
list *listInsertNode(list *list, listNode *old_node, void *value, int after);
```
向链表的指定位置插入一个新的节点，这个指定的位置由`old_node`节点标记，如果`after`为1，
则表示新的节点插入到`old_node`节点后面，否则插入到该节点的前面。

```c
void listDelNode(list *list, listNode *node);
```
从链表`list`中，删除给定的`node`节点。

```c
listNode *listSearchKey(list *list, void *key);
```
从`list`链表中搜索`listNode.value`*等于*`key`的节点，这里的*等于*的含义在于：
1. 如果`list`链表中设置了`list.match`接口，那么通过`list.match`来判断是否*等于*。
2. 如果没有定义`list.match`，那么直接比较二者是否指向同一内存。
```c
    if (list->match) {
        if (list->match(node->value, key)) {
            return node;
        }
    } else {
        if (key == node->value) {
            return node;
        }
    }
```

```c
listNode *listIndex(list *list, long index);
```
这个函数用于对链表进行索引操作，这个索引值既可以是*zero-based*的正向索引，
也可以使从`-1`开始的一个反向索引，执行结束后，返回索引对应的节点的指针:
```c
listNode *listIndex(list *list, long index) {
    listNode *n;

    if (index < 0) {
        index = (-index)-1;
        n = list->tail;
        while(index-- && n) n = n->prev;
    } else {
        n = list->head;
        while(index-- && n) n = n->next;
    }
    return n;
}
```

### 链表迭代器
```c
listIter *listGetIterator(list *list, int direction);
void listReleaseIterator(listIter *iter);
listNode *listNext(listIter *iter);
```
其中`listGetIterator`可以根据传入的方向参数，生成一个正向或者反向迭代器，
方向参数的定义在*src/adlist.h*头文件中：
```c
#define AL_START_HEAD 0        //用于生成正向迭代器
#define AL_START_TAIL 1        //用于生成反向迭代器
```
而`listReleaseIterator`则用于释放一个迭代器；`listNext`用于对迭代器执行迭代操作，
返回迭代器当前所指向的节点指针，同时根据方向，将迭代器移动到下一个操作。

上述生成链表迭代器的操作所产生的迭代器是一个在堆空间上动态分配的迭代器，
这个迭代器可以通过指针，在整个程序的生命周期有效，直到调用`listReleaseIterator`将其释放。
而当我们仅仅需要在一个函数内，将链表迭代器作为局部变量使用的话，我们可以使用下面两个函数，
来对栈上的局部迭代器进行初始化，同时系统会在函数调用结束后，自动释放该局部迭代器：
```c
void listRewind(list *list, listIter *li);
void listRewindTail(list *list, listIter *li);
```
这两个函数分别用于初始化一个正向迭代器以及以及反向迭代器。其具体的用于可以如下所示：
```c
    listIter iter;
    listNode *node;
    listRewind(l, &iter);
    while((node = listNext(&iter)) != NULL) {
        //TODO:
    }
```

### 链表特殊操作
```c
list *listDup(list *orig);
```
该函数用于拷贝整个链表，如果通过`listSetDupMethod`为链表设置了*Dup*方法，
那么将执行一次深度拷贝工作，也就是使用`list.dup`函数来对节点进行构造；
否则的话，将执行一次浅拷贝，即将原始节点的`value`指针，赋值给新节点，也就是两个节点均引用同一个*value*。

```c
void listRotate(list *list);
```
这个函数用于对链表执行翻转操作，也就是将链表的尾节点转移到链表的头部，循环调用该链表，
可以实现翻转链表的操作。

```c
void listJoin(list *l, list *o);
```
这个函数用于对两个链表执行连接操作，也就是将`o`链表中元素按照顺序连接到`l`链表的后面。