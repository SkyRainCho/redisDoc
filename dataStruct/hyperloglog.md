# Redis中的基数统计HyperLogLog

## 何为基数统计
所谓基数统计，就是指从一组数据集中获取不重复出现的数据的个数，例如一组数据`[1, 2, 1, 0, 3, 5, 2]`，那么对于这组数据的基数统计结果便是5。基数统计在当下有很多应用的场景，例如统计一个网络游戏每天的登录账号的数量，一个账号一天之内可能会进行多次登录；又或者统计某个网站每天的访问IP地址来源，同样也会有同一个IP多次访问的情况。

对于C++程序员来说，遇到上述问题第一个会想到的便是使用**STL**标准库中的`std::set`或者`std::unordered_set`这两种容器进行数据统计，遍历原始数据集，将其中的每一条数据都插入容器之中，完成遍历之后容器的大小即为基数统计结果。

然而C++标准库中这种通用的方式存在一个隐患，对于数据集较小的情况下这种基于红黑树或者哈希表的方式较为适用同时得到的结果也一定是准确的。然而如果对于数据集很大的情况呢，比如10T大小的数据量同时重复数据比例很低的情况呢？如果依然采用类似**STL**标准库中的这种通用解决方案，那么使用`std::set`以及`std::unordered_set`存储数据，那么这些容器也会相应的占用大约10T左右的内存，这还仅仅是对一项数据进行基数统计，就要消耗如此多的内存。如果针对多个数据项进行基数统计，那么在数据集大小非常大的情况下，这种通用的方案显然是不能接受的。

考虑另外一个问题，对于拥有1亿条数据的数据集，假设其中不重复的数据有20000000条。如果我们给出一种方法，通过这种方法最后得出的基数统计的结果为19999914条，虽然不是准确的结果，但是是一个误差很小的结果，那么这种方法是否可以接收呢？进一步来说，如果这种虽然结果不准确但是误差很小的解决方案，同时还具有很快的运行效率，例如线性时间复杂度；并且消耗较少的内存，例如常量级的空间复杂度，那么这种方案是否可以接收呢？

**HyperLogLog**算法便是这样一种基于概率的基数统计算法，这个算法可以接收大小最大为`2^63`的数据集合，同时仅需要占用12K的内存，并且可以将基数统计结果的误差控制在0.86%以内。

*Redis*服务器也实现了**HyperLogLog**算法用于对数据进行基数统计，本文将介绍如何使用*Redis*中的**HyperLogLog**基数统计功能，以及介绍这个算法是如何实现的。不过本文将不会介绍这个算法在数学上的证明过程，如果有兴趣的同学可以自行搜索**HyperLogLog**算法的数学证明。

## Redis中基数统计功能的概述

*Redis*通过**PFADD**、**PFCOUNT**、**PFMERGE**这三个命令来实现基于**HyperLogLog**算法的。

### PFADD命令

    PFADD key element [element ...]

在*Redis*之中，每一个基数统计可以被认为是一个**HyperLogLog**对象，都会被存储在数据库的键空间之中，可以被持久化，拥有自己的键；通过**PFADD**命令，我们可以将指定的元素数据加入到给定`key`的**HyperLogLog**对象之中。如果`key`不存在于数据库键空间之中，则会向数据库中插入一个**HyperLogLog**对象。

### PFCOUNT命令

    PFCOUNT key [key ...]

**PFCOUNT**这个命令用于获取一个**HyperLogLog**对象截止当前的基数统计结果；如果参数给出了多个`key`，*Redis*则会将这些`key`指向的**HyperLogLog**对应的数据集合并成一个临时的数据集，并返回这个临时数据集的基数统计结果，这个临时数据集不会真正地存储在数据库键空间之中。

### PFMERGE命令

    PFMERGE destkey sourcekey [sourcekey ...]

**PFMERGE**这个命令，可以用于数据集的合并，会将多个`sourcekey`对应的数据集进行合并，并将合并结果存储于`destkey`这个键中。

## Redis中基数统计的实现细节

### Redis进行基数统计的流程
*Redis*没有为**HyperLogLog**对象单独实现一个数据结构，而是使用一段连续分配的内存来存储**HyperLogLog**的相关数据。同时将这段连续的内存用字符串对象类型进行包装存储在数据库的键空间之中。*Redis*实现基数统计的逻辑为：
1. 对于每一个**HyperLogLog**，*Redis*为其分配16384个寄存器，每个寄存器具有6比特大小。
1. 对于待统计数据集中的每个数据元素，通过哈希函数计算出64位的数据元素哈希值。
1. 根据64位哈希值的低14位，将这个对应元素分配到对应的**HyperLogLog**寄存器之中。
1. 剩余50位的哈希值`h`，我们定义`f(h)`返回从最低位向最高位遍历，第一次出现1的位置。例如`h=10100100000`，那么`f(h)`的值为6。
1. 每个**HyperLogLog**寄存器中，存储配分配到这个寄存器的元素哈希值对应的`f(h)`的最大值。
1. 通过收集16384个寄存器中存储的数据，通过某种计算方法，就可以获得了数据集的基数统计结果。

至于在步骤2中是使用什么样的哈希算法，步骤4中为什么要计算第一次出现1的位置，以及步骤6中最终结果是如何计算出来的，大家可以自行搜索HyperLogLog算法相关的内容。


### Redis中HyperLogLog数据结构体的定义
*Redis*为了描述这个**HyperLogLog**对象定义了一个对应的数据结构：

```c
struct hllhdr {
	char magic[4];
    uint8_t encoding;
    uint8_t notused[3];
    uint8_t card[8];
    uint8_t registers[];
};
```

在这个数据结构之中：

1. `hllhdr.magic`，用于存储字符串`HYLL`标记这是**HyperLogLog**对象。
2. `hllhdr.encoding`，记录**HyperLogLog**对象的编码类型：
   1. `HLL_DENSE`，表示这是一个**稠密**的**HyperLogLog**对象，拥有较多的数据元素，大部分的寄存器之中都拥有数据。
   2. `HLL_SPARSE`，表示这是一个**稀疏**的**HyperLogLog**对象，拥有较少的数据元素，大部分的寄存器之中都是0，没有存储数据。
3. `hllhdr.notused`，这三个字节的数据被预留暂时没有被使用，需要被设置为0。
4. `hllhdr.card`，用于缓存**HyperLogLog**对象基数统计的结果，在向**HyperLogLog**对象中插入元素时，如果新的数据元素导致某个寄存器被更新，那么这个缓存结果将会失效，当**PFCOUNT**铭记获取统计结果时，会从通过遍历所有的寄存器来重新计算统计结果；否则会继续使用这个字段存储的统计结果。
5. `hllhdr.registers`，**HyperLogLog**中被分配的12KB大小的16384个寄存器。

### Redis中HyperLogLog的代码实现
```c
uint64_t MurmurHash64A(const void * key, int len, unsigned int seed);
```
上面这个`MurmurHash64A`函数用于计算一段数据的哈希值，会返回一个64比特的哈希值。用于处理前面基数统计流程中步骤2中计算数据元素的64位哈希值。

```c
int hllPatLen(unsigned char *ele, size_t elesize, long *regp);
```
这个`hllPatLen`函数，用于将一个数据元素加入**HyperLogLog**之中，会调用`MurmurHash64A`来计算数据元素64位哈希值，通过`regp`这个参数会返回数据元素究竟落在哪个**HyperLogLog**寄存器之中；而函数的返回值前面基数统计流程中步骤4中，哈希值中从最低位开始第一次出现1的位置`f(h)`。

接下来，我们以**稠密**的**HyperLogLog**为例，讲述一下后续的代码内容。
由于程序中，内存的分配以及访问都是按照字节来进行的，也就是以8比特为单位的，但是**HyperLogLog**中的每个寄存器都是6比特大小的，有的寄存器是落在一个字节内的，有的寄存器则是跨越两个字节的。因此*Redis*定义了两个宏，分别用于从指定的寄存器之中获取存储的值，以及向指定的寄存器之中存储数据：
```c
#define HLL_DENSE_GET_REGISTER(target, p, regnum)
#define HLL_DENSE_SET_REGISTER(p, regnum, val)
```

当有一个新的元素被加入到**HyperLogLog**时，在通过`hllPatLen`计算出数据元素应该落在哪个寄存器之中以及对应哈希值`h`的`f(h)`之后，需要判断这个`f(h)`是否可以被更新到寄存器之中。
```c
int hllDenseSet(uint8_t *registers, long index, uint8_t count)
{
    uint8_t oldcount;
    HLL_DENSE_GET_REGISTER(oldcount, registers, index);
    if (count > oldcount)
    {
        HLL_DENSE_SET_REGISTER(registers, index, count);
        return 1;
    }
    else
    {
        return 0;
    }
}
```

而**HyperLogLog**的插入接口便是通过上面`hllPatLen`以及`hllDenseSet`这两个函数来实现的:
```c
int hllDenseAdd(uint8_t *register, unsigned char *ele, size_t elesize)
{
    long index;
    uint8_t count = hllPatLen(ele,elesize,&index);
    return hllDenseSet(registers, index, count);
}
```

那么当向**HyperLogLog**对象之中插入一定量的数据元素之后，我我们便可以获取基于当前已插入数据的基数统计结果。首先我们来看一下用于计算统计结果的一个辅助函数，收集16384个寄存器中存储数据的频数：
```c
void hllDenseRegHisto(uint8_t *registers, int* reghisto)
{
    int j;
    for (j = 0; j < HLL_REGISTERS; j++>)
    {
        unsigned long reg;
        HLL_DENSE_GET_REGISTER(reg, register, j);
        reghisto[reg]++;
    }
}
```
通过上面收集的寄存器数据频数，通过计算便可以获得基数统计的结果：
```c
uint64_t hllCount(struct hllhdr *hdr, int *invalid)
{
    double m = HLL_REGISTERS;
    double E;
    int j;
    int reghisto[64] = {0};

    hllDenseRegHisto(hdr->registers, reghisto);

    double z = m * hllTau((m-reghisto[HLL_Q+1])/(double)m);
    for (j = HLL_Q; j >= 1; --j)
    {
        z += reghisto[j];
        z *= 0.5;
    }
    z += m * hllSigma(reghisto[0]/(double)m);
    E = llroundl(HLL_ALPHA_INF*m*m.z);
    return (uint64_t) E;
}
```

最后我们看一下合并两个**HyperLogLog**对象的逻辑，假设我们有一个**HyperLogLog**对象`hf1`，其寄存器之中的数据为`[a1, a2, ..., a16384]`，同时有另外一个对象`hf2`，其寄存器之中的数据为`[b1, b2, ..., b16384]`，那么合并后的新的**HyperLogLog**对象，它的寄存器中的内容将以如下形式来表示`[max(a1,b1), max(a2, b2), ..., max(a16384, b16384)]`。通过计算这个新的**HyperLogLog**对象的基数统计结果，便是`hf1`以及`hf2`这两对象对代表的数据集的交集之后的基数统计结果。相关的代码为：
```c
int hllMerge(uint8_t *max, robj *hll)
{
    struct hllhdr *hdr = hll->ptr;
    int i;
    uint8_t val;
    for (i = 0; i < HLL_REGISTERS; i++)
    {
        HLL_DENSE_GET_REGISTER(val, hdr->registers, i);
        if (val > max[i]) max[i] = val;
    }
}
```

以上便是*Redis*中**HyperLogLog**内容的一个简要介绍，当数据很少时，*Redis*会采用稀疏的数据结构，出于节约内容的考虑，此时不会足量地分配16384个寄存器，而是会对寄存器进行一个压缩，不过这里的实现细节较为复杂，但是其本质上有稠密的数据结构是一致的，因此处于通俗易懂的考虑，本只对稠密数据结构进行了介绍，如果有兴趣的同学可以自行查阅*Redis*源代码，了解实现细节。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*