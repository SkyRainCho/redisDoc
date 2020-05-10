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
}
```
同时如果传入了`pos`指针的话，那么在查找成功返回1的时候，`pos`会被赋值为`value`的索引位置；
而当查询失败返回0时，`pos`则会被赋值为参数`value`在这个有序整数集合中*应该*处于的合适位置。
而这个底层函数对于整个整数集合来说是一个十分重要的函数，除了正常的查找功能之外，
当我们需要想其中插入一个新的整数时，我们可以调用这个函数来确定其合适的插入位置，
以确保，在插入前后，该整数集合的有序性不会受到影响。

## 用户接口函数

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*