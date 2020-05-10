# Redis中整数集合的实现
*Redis*在*src/intset.h*文件中，定义了一个整数集合的数据结构，用于保存若干整形数字，所有被存储的数字以升序的排列顺序，
存储在一段连续的内存之中，同时在插入与删除的过程中会保持整个集合的有序性。
可以在提供内存使用率的情况下，应用二分查找提高查找的效率。
对于长度较小，同时只存储整数的*Redis*集合对象，会使用整数集合来作为其底层的实现方式；
而既存储整数类型数据又存储字符串类型数据的集合对象，*Redis*则会使用`dict`作为底层的实现。

## 数据结构定义
```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;

#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
以上便是整数集合数据类型的定义，所有数据都被存储在由`intset.contents`所指向的动态分配的内存之中；
而`intset.length`则表示存储的整数的个数。

`intset.encoding`用于表示整个整数集合的编码方式，通过在`src/intset.c`中给出的定义，整数集合分别可以使用
16位有符号整数编码，32位有符号整数编码以及64位有符号整数编码。值得一提的是，这个编码方式是对于整数集合而言的，
如果一个整数集合的编码方式是`INTSET_ENC_INT16`，那么这个集合中`contents`里所存储的所有整数都应是16位有符号整数可以表示的。

在*src/intset.c*源文件中，定义了两个静态函数用于处理整数集合的编码相关内容：
```c
static uint8_t _intsetValueEncoding(int64_t v);
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc);
```
其中`_intsetValueEncoding`会通知传入的`v`的数值来返回其对应的编码方式；
而`_intsetGetEncoded`则会根据给定的编码方式`enc`从整数集合`is`中解码出`pos`位置上的整数。

## 底层基础操作

```c
static intset *intsetResize(intset *is, uint32_t len);
```

`intsetResize`用于为整数集合重新设置大小，调用这个函数会为整数集合重新分配空间使之可以存储`len`个`intset.encoding`个数据。



```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos);
```
基于整数集合中存储数据的有序性，因此我们可以基于这个前提，在整数集合上应用二分查找算法，
以*O(logN)*的时间复杂度对一个给定的整数进行查找。
`intsetSearch`，会在`is`中以二分查找算法查找`value`，没有找到这个给定的`value`，
那么函数会返回0，否则的话，则返回1，在*Redis*中则是给出了一个二分查找的经典模式：

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos)
{
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;
    
    ...
    
    while(max > min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is, mid);
        if (value > cur){
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }
    
    if (value == cur) {
        ...
        return 1;
    } else {
        ...
        return 0;
    }
}
```
同时如果传入了`pos`指针的话，那么在查找成功返回1的时候，`pos`会被赋值为`value`的索引位置；
而当查询失败返回0时，`pos`则会被赋值为参数`value`在这个有序整数集合中*应该*处于的合适位置。
而这个底层函数对于整个整数集合来说是一个十分重要的函数，除了正常的查找功能之外，
当我们需要想其中插入一个新的整数时，我们可以调用这个函数来确定其合适的插入位置，
以确保，在插入前后，该整数集合的有序性不会受到影响。



```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value);
```

这个函数`intsetUpgradeAndAdd` 用于将向整数集合 `is` 中插入一个超过这个整数集合编码的数值`value` ，同时将这个整数集合中所有的存储元素均提升到`value` 对应的编码方式，同时将这个`value` 数值插入到这个整数集合的指定位置。

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value)
{
    ...
    int prepend = value < 0 ? 1: 0;
    ...
    while(length--)
        _intsetSet(is, length+prepend, _intsetGetEncoded(is, length, curenc));
    ...
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is, intrev32ifbe(is->length), value);
    ...
}
```

对上述的代码段，我们发现需要被插入的`value`要么被插入到整数集合的起始位置，要么被插入到整数集合的结尾。这是因为通过这个函数插入的`value`一定是会导致整数集合编码方式的提升，这也就意味着这个`value`要么大于整数集合中的所有整数，要么小于整数集合中的所有整数。例如，当前的整数集合的编码方式是`INTSET_ENC_INT16`，而新插入的`value`其对应的编码方式`INTSET_ENC_INT32`，那么这个`value`的数值一定是会超过`INTSET_ENC_INT16`所能表示的数据范围，也就是大于或者小于这个整数集合中当前所有的元素。



```c
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to);
```

这个函数接口`insetMoveTail`用于移动整数集合中的元素，将从`from`开始的所有元素，通过调用`memmove`函数向前或者向后移动到`to`这个位置，这个函数主要用于向整数集合中插入元素或者从整数集合中删除元素时，对`intset.contents`中元素进行移动，以保持其连续性。

## 用户接口函数

```c
intset *intsetNew(void);
```

该接口用于创建一个新的整数集合`intset`，初始化的整数集合都是使用`INTSET_ENC_INT16`作为编码方式，后续随着新数据的插入，可以通过调用`intsetUpgradeAndAdd`函数来升级整数集合的编码方式。



```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success);
```

该接口用于将一个整数`value`插入了一个整数集合之中，如果`value`已经存在于整数集合之中，那么插入会失败，这个失败的结果会通过`success`进行返回，成功为1，失败为0。其中插入的流程为：

1. 获取`value`对应的编码类型，如果`value`的编码类型大于目标整数集合的编码类型，那么会调用`intsetUpgradeAndAdd`对目标整数集合进行升级。
2. 如果不需要升级，那么会调用`intsetSearch`在整数集合中搜索`value`，以期望获得合适的插入位置`pos`。
3. 如果`value`已经存在于整数集合中的话，那么插入失败。
4. 否则通过调用`intsetMoveTail`以及`_intsetSet`两个接口，将`value`插入到整数集合之中的合适位置。



```c
intset *intsetRemove(intset *is, int64_t value, int *success);
```

该接口用于从整数集合中删除给定的`value`元素，当`value`不在整数集合中的情况下，会用过`success`返回删除失败。其基本的删除过程为：

1. 获取`value`对应的编码类型，如果这个编码类型大于目标整数集合的编码类型，那么这个`value`一定不会存在于整数集合之中，因此可以直接返回失败。
2. 调用`intsetSearch`来获取这个`value`在整数集合中的位置，如果搜索之后，不存在于这个整数集合之中的话，则返回删除失败。
3. 根据搜索到的位置，调用`intsetMoveTail`以及`intsetResize`接口将`value`从整数集合之中删除。



```c
uint8_t intsetFind(intset *is, int64_t value);
int64_t intsetRandom(intset *is);
uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value);
uint32_t intsetLen(const intset *is);
size_t intsetBlobLen(intset *is);
```

上述五个函数是整数集合的一些简单的用户接口：

1. `intsetFind`用于判断`value`是否存在于给定的整数集合之中。
2. `intsetRandom`用于从整数集合中随机返回一个元素。
3. `intsetGet`用于获取整数集合中索引为`pos`的元素。
4. `intsetLen`用于获取整数集合的元素数量。
5. `intsetBlobLen`用于获取整数集合占用的内存字节数。



通过分析整数集合的源代码，可以发现整数集合数据类型兼具了内存使用效率以及搜索时间效率两个方面的考量。通过编码类型`intset.encoding`以及连续存储内存的使用，在一定程度上提高了内存使用效率，不过这里整数集合也存在一个小的瑕疵，***编码类型一旦升级便不会下降***这在某些情况下会造成内存的浪费。但是如果想要在删除元素的同时检测并实现***编码类型降级***这需要设计更为负责的数据结构或者复杂的算法，这样势必会增加整个整数集合的复杂性，笔者猜测*Redis*的作者基于上面的这些考虑，在这个方面选择了一定的妥协。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*