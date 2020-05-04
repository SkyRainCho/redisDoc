# Redis中优化内存的字符串map数据结构

在*src/zipmap.h*头文件中，*Redis*定义了一个*zipmap*的数据结构，这个数据结构同时提供了一种
从字符串到字符串的映射机制，同时实现了*O(n)*时间复杂度的数据查找，并且具有很高的内存使用效率。

鉴于在很多的时候，用户使用*Redis*的哈希表数据类型，仅仅是存储少料数据构成的对象，在这种情况下
会使用*zipmap*数据结构来表示哈希数据对象，当存储数据的数量达到一个给定的阈值时，
便会喜欢成哈希表数据结构来存储数据，这种设计思想，在对于内存使用的方面，被证明时一种有效方法。

## zipmap在内存中的分布
*zipmap*在内存中是使用一段连续分配的内存进行维护的，通过特殊的标记字节来标识整个*zipmap*的长度，
*key-value*数据的长度等等。

以`foo => bar, hello => world`映射为例：

![zipmap内存分布](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/zipmap%E5%86%85%E5%AD%98%E5%88%86%E5%B8%83.PNG)

其中第一个字节`<zmlen>`表示这个*zipmap*的`size`，由于只有一个字节，因此最多可以表示的长度为253，
如果当一个*zipmap*的`size`大于等于254的时候，就需要遍历整个*zipmap*才可以知道它的`size`。

`<len>`字段则表示后面跟随的*key*或者*value*字符串的长度，当字符串的长度在0到253之间时，
`<len>`字段只需要一个字节来存储；否则需要5个字节来保存，其中第一个字节以`0xFE`来标记使用5个字节来存储字符串长度，
后续的4个字节以主机字节序来表示字符串的长度；如果出现了`0xFF`的字节，这是一个结束标记，
表明到达了*zipmap*连续分配内存的结尾。

`<free>`字段则表示了在字符串后的未使用的字节的数量，这通常时由于修改*key*对应的*value*的内容而产生的，
例如将`foo => bar`中修改`bar`为`b`，那么就会产生两个字节的未使用内存。这个`<free>`字段是使用一个字节来表示的，
如果在更新*zipmap*之后，出现无法使用一个字节来表示未使用数据长度的情况的话，那么会重新分配*zipmap*，以确保内存的使用效率。

## zipmap基础操作接口

```c
static unsigned int zipmapDecodeLength(unsigned char *p);
static unsigned int zipmapEncodeLength(unsigned char *p, unsigned int len);
static unsigned long zipmapRequiredLength(unsigned int klen, unsigned int vlen);
static unsigned int zipmapRawKeyLength(unsigned char *p);
static unsigned int zipmapRawValueLength(unsigned char *p);
static unsigned int zipmapRawEntryLengthe(unsigned char *p);
static unsigned char *zipmapResize(unsigned char *zm, unsigned char len);
static unsigned char *zipmapLookupRaw(unsigned char *p, unsigned char *key, unsigned int klen, unsigned int *totlen);
```

接口`zipmapDecodeLength`用以从一个指向*zipmap*中的一个元素的指针中解码出后续字串的长度，
```c
static unsigned int zipmapDecodeLength(unsigned char *p) {
    unsigned int len = *p;

    if (len < ZIPMAP_BIGLEN) return len;
    memcpy(&len,p+1,sizeof(unsigned int));
    memrev32ifbe(&len);
    return len;
}
```
由于在第一部分我们已经了解到，`<len>`根据保存数据的长度会被设置为1个字节或者5个字节，这里首先会解码出第一个字节保存的数据，
如果小于`ZIPMAP_BIGLEN`，那么就是使用单字节模式，直接返回解码出的数据就可以。如果时长度大于等于`ZIPMAP_BIGLEN`，
那么就是使用5字节的形式，`p`指针后面的四个字节数据才是真正的长度数据。

接口`zipmapEncodeLength`则是`zipmapDecodeLength`的反向操作，用于将一个长度数值`len`编码到`p`指针对应的内存之中，
如果`p`指针为空值，那么会返回这个长度数值对应所需编码的字节数。

接口`zipmapRequiredLength`则是用于计算给定*key*的长度`klen`，以及*value*的长度`vlen`，那么这对*key-value*数据在*zipmap*
这段连续内存中所需占用的内存大小，需要在`klen`以及`vlen`的基础上加上两个`<len>`标记以及一个`<free>`标记的大小。

接口`zipmapRawKeyLength`用来返回一个*key*的总大小，包含`<len>`字段的长度以及*key*本身的长度。
接口`zipmapRawValueLength`用来返回一个*value*的总大小，包含`<len>`字段，`<free>`字段长度，以及*key*本身的长度。
接口`zipmapRawEntryLength`会调用`zipmapRawKeyLength`以及`zipmapRawValueLength`来返回一个*key-value*的总长度。

接口`zipmapResize`用于对*zipmap*进行重新内存分配。

接口`zipmapLookupRaw`用于从一个*zipmap*之中查找一个给定的`key`，如果没有找到，那么会返回一个`NULL`指针；
在返回`NULL`指针的情况下，如果我们传入`totlen`指针，那么这个参数会被设置成这个*zipmap*的总长度。
由于本质上*zipmap*是一段被分配出连续内存，因此对于`key`的查找，是使用顺序查找的方式进行的。


## zipmap用户操作接口

```c
unsigned char *zipmapNew(void);
unsigned char *zipmapSet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char *val, unsigned int vlen, int *update);
unsigned char *zipmapDel(unsigned char *zm, unsigned char *key, unsigned int klen, int *deleted);
unsigned char *zipmapRewind(unsigned char *zm);
unsigned char *zipmapNext(unsigned char *zm, unsigned char **key, unsigned int *klen, unsigned char **value, unsigned int *value);
int zipmapGet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char **value, unsigned int *vlen);
int zipmapExists(unsigned char *zm, unsigned char *key, unsigned int klen);
unsigned int zipmapLen(unsigned char *zm);
size_t zipmapBlobLen(unsigned char *zm);
```

接口`zipmapNew`用来创建一个空的*zipmap*，这个空的*zipmap*初始只占用了两个自己的空间，第一个字节表示`<zmlen>`，第二个字节表示的是一个结束标记。
接口`zipmapSet`用于设定一个给定的*zipmap*中，指定`key`对应的`val`值，如果这个`key`并不存在，那么会在*zipmap*的结尾分配出一块内存，用于创建这个新的*key-value*。
```c
unsigned char *zipmapSet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char *val, unsigned int vlen, int *update) 
{
    unsigned int zmlen, offset;
    unsigned int freelen, reqlen = zipmapRequiredLength(klen,vlen);
    unsigned int empty, vempty;
    unsigned char *p;

    freelen = reqlen;
    if (update) *update = 0;
    p = zipmapLookupRaw(zm,key,klen,&zmlen);
    if (p == NULL) {
        zm = zipmapResize(zm, zmlen+reqlen);
        p = zm+zmlen-1;
        zmlen = zmlen+reqlen;

        if (zm[0] < ZIPMAP_BIGLEN) zm[0]++;
    } else {
        if (update) *update = 1;
        freelen = zipmapRawEntryLength(p);
        if (freelen < reqlen) {
            offset = p-zm;
            zm = zipmapResize(zm, zmlen-freelen+reqlen);
            p = zm+offset;

            memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));
            zmlen = zmlen-freelen+reqlen;
            freelen = reqlen;
        }
    }

    empty = freelen-reqlen;
    if (empty >= ZIPMAP_VALUE_MAX_FREE) {
        offset = p-zm;
        memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));
        zmlen -= empty;
        zm = zipmapResize(zm, zmlen);
        p = zm+offset;
        vempty = 0;
    } else {
        vempty = empty;
    }

    p += zipmapEncodeLength(p,klen);
    memcpy(p,key,klen);
    p += klen;

    p += zipmapEncodeLength(p,vlen);
    *p++ = vempty;
    memcpy(p,val,vlen);
    return zm;
}
```
通过源代码，我们可以总结出`zipmapSet`的实现细节：
1. 通过调用`zipmapRequiredLength`接口来获取这个*key-value*所需的内存大小`reqlen`。
2. 调用`zipmapLookupRaw`接口来判断这个`key`是否存在于这个*zipmap*之中。
3. 如果`key`不存在，那么意味着需要多分配出一块内存来存储新创建的*key-value*，调用接口`zipmapResize`为*zipmap*多分配出`reqlen`大小的内存。
4. 如果`key`存在，那么会调用`zipmapRawEntryLength`来获取原始*key-value*的长度`freelen`：
    - 如果`freelen`的大小小于`reqlen`，那么意味着原始*key-value*所使用的内存无法保存新的*key-value*，
    因此需要调用`zipmapResize`来为*zipmap*多分配出`reqlen-freelen`大小的内存，同时调用`memmove`将后续的*key-value*向后移动`reqlen-freelen`字节，
    用于为新的*key-value*预留出足够的空间。
    - 当`freelen`大于`reqlen`时，如果减少的内存大小小于254个字节，那么这个长度数据会被存储在这个*key-value*对应的`<free>`字节中，
    如果减小的内存已经超过了253个字节，那么需要调用`zipmapResize`为*zipmap*重新分配大小，以达到节约内存的目的，调整后这个*key-value*数据将不会有预留的内存空间，
    同时`<free>`字节将会被设置为0。
5. 通过指针的移动以及`memcpy`函数，将`key`以及`value`拷贝到对应的内存之中。


接口`zipmapDel`用于从给定的*zipmap*中删除指定的`key`，如何`deleted`参数没有被设置成`NULL`的话，那么将要被删除的`key`是否存在会通过这个这个参数返回。
其基本的逻辑是：
1. 通过`zipmapLookupRaw`找到对应*key-value*的指针，
2. 通过`zipmapRawEntryLength`接口获取其占用内存的大小，
3. 通过`memmove`将后续数据向前移动，
4. 调用`zipmapResize`来释放空间。

这里有一点需要注意的是：
```c
unsigned char *zipmapDel(unsigned char *zm, unsigned char *key, unsigned int klen, int *deleted) {
    ...
    /* Decrease zipmap length */
    if (zm[0] < ZIPMAP_BIGLEN) zm[0]--;
    ...
}
```
我们注意到，只有`<zmlen>`保存的长度在`ZIPMAP_BIGLEN`以下时，调用`zipmapDel`导致*zipmap*长度的变化才会体现到这个字段上，
而如果一个*zipmap*的`<zmlen>`超过了`ZIPMAP_BIGLEN`，那么即使多次调用`zipmapDel`使这个*zipmap*实际的数值下降到`ZIPMAP_BIGLEN`以下，
这个数值的变化也不会体现到`<zmlen>`字段上。

`zipmapRewind`与`zipmapNext`这两个接口通常是搭配在一起使用的，其中`zipmapRewind`接口用来跳过`<zmlen>`字节，返回指向第一个*key-value*内存的指针，
而`zipmapNext`则是根据传入的*key-value*指针，获取其`key`以及`value`，并返回下一个*key-value*的指针，两个接口一起使用，可以实现对*zipmap*的遍历，例如：
```c
unsigned char *i = zipmapRewind(zipmap);
while ((i = zipmapNext(i, &key, &klen, &val, &vlen)) != NULL)
{
    //do something with key and value
}
```

接口`zipmapGet`用于从*zipmap*中获取指定`key`对应的`value`，相当于查找操作，在内部通过调用`zipmapLookupRaw`来实现，如果`key`存在，则返回1，否则返回0。
`zipmapExists`接口则是通过调用`zipmapLookupRaw`来判断一个给定的`key`是否存在于*zipmap*中。

接口`zipmapBlobLen`用于返回一个给定*zipmap*占用的字节总大小，而`zipmapLen`则是会返回*zipmap*中*key-value*的数量，如果在`<zmlen>`字节中保存的数值小于`ZIPMAP_BIGLEN`，
那么直接返回`<zmlen>`中的数值，否则就需要遍历整个*zipmap*才可以确定，另外需要注意的是，在`zipmapDel`调用中，不会更新超过`ZIPMAP_BIGLEN`的`<zmlen>`，而在调用`zipmapLen`时，
如果出现`<zmlen>`大于`ZIPMAP_BIGLEN`，而实际遍历结果小于`ZIPMAP_BIGLEN`时，会将实际的结果回写到`<zmlen>`字节中。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*