# Redis中跳跃表的实现
跳跃表是一种基于链表的，同时可以实现平均情况下*O(logN)*时间复杂度搜索的数据结构。在*redis*之中，
跳跃表作为有序集合的底层实现之一。

## 跳跃表概述
在*5.0.8*版本的*Redis*源代码中，跳跃表实现没有单独的头文件件以及源文件，
其数据结构的定义，以及接口函数的声明是*src/server.h*这个头文件中进行的，
而接口函数的定义则是在*src/t_zset.c*文件之中进行的。
跳跃表的数据结构为：
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

typedef struct zskiplist
{
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

跳跃表的接口函数为：
```c
zskiplist *zslCreate(void);
void zslFree(zskiplist *zsl);
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele);
int zslDelete(zskiplist *zsl, double score, zskiplistNode **node);
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range);
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range);
unsigned long zslGetRank(zskiplist *zsl, double score, sds o);
zskiplistNode *zslFirstInLexRange(zskiplist *zsl, zrangespec *range);
zskiplistNode *zslLastInLexRange(zskiplist *zsl, zrangespec *range);
```
***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*