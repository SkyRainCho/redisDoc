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
* [紧凑列表](advancedDataStruct/listpack.md)
* [快速链表](advancedDataStruct/quicklist.md)
* [基数树](advancedDataStruct/rax.md)

## 对象类型
* [Redis对象](dataStruct/object.md)
* [字符串对象](dataStruct/string.md)
* [列表对象](dataStruct/list.md)
* [集合对象](dataStruct/set.md)
* [有序集合对象](dataStruct/zset.md)
* [散列对象](dataStruct/hash.md)

## 数据库操作

* [内存数据库基础操作](dataManager/db.md)
* [过期策略](dataManager/expire.md)
* [惰性删除策略](dataManager/lazyfree.md)
* [后台IO服务](dataManager/bio.md)
* [淘汰策略](dataManager/evict.md)
* [RDB数据备份](dataManager/rdb.md)
* [AOF数据备份](dataManager/aof.md)

## 网络IO接口
* [套接字接口](network/anet.md)
* [事件驱动](network/ae.md)
* [同步IO](network/syncio.md)
