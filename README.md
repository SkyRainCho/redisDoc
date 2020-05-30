# redisDoc
本系列通过分析*Redis*的*5.0.8*版本的源代码，来了解*Redis*的内部实现细节。

## 基础数据类型

* [双端链表实现](baseDataStruct/adlist.md)
* [动态字符串实现](baseDataStruct/sds.md)
* [哈希表实现](baseDataStruct/dict.md)

## 高级数据类型

* [压缩链表](advancedDataStruct/ziplist.md)
* [压缩字典](advancedDataStruct/zipmap.md)
* [整数集合](advancedDataStruct/intset.md)
* [紧凑链表](advancedDataStruct/listpack.md)
* [快速链表](advancedDataStruct/quicklist.md)
* [基数树](advancedDataStruct/rax.md)

## 对象类型
* [Redis对象](dataStruct/object.md)
* [字符串对象](dataStruct/string.md)
* [列表对象](dataStruct/list.md)
* [集合对象](dataStruct/set.md)
* [有序集合对象](dataStruct/zset.md)
* [散列对象](dataStruct/hash.md)