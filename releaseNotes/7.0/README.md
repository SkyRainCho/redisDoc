# Redis 7.0 发布说明

## Redis 7.0 版本介绍

Redis 7.0 包含了几个新的面向用户的功能、显著的性能优化和许多其他改进。
它还包括可能破坏与旧版本向后兼容性的更改。我们敦促用户在升级之前仔细阅读发行说明。

特别是，用户应该注意以下变更：

1. Redis 7 将 AOF 存储为文件夹中的多个文件；请参阅下面的多部分 AOF。
2. Redis 7 使用新的版本 10 格式存储 RDB 文件，该格式与旧版本不兼容。
3. 在加载旧的 RDB 格式时，Redis 7 会即时将 ziplist 编码的键转换为 listpacks。
转换适用于从磁盘加载文件或从 Redis 主节点复制，这将略微增加加载时间。
4. 请参阅下面提到的关于破坏性变更的部分。


以下是与 6.2.6 版本相比，此版本的全面变更列表。
每项变更都包含了添加它的 PR 编号，
因此你可以在 https://github.com/redis/redis/pull/ 获取更多详细信息。

### 新特性

- Redis 函数：一种使用服务器端脚本扩展 Redis 的新方法 (#8693) 
参见 https://redis.io/topics/functions-intro
- ACL：细粒度的基于键的权限，允许用户支持带选择器的多组命令规则 (#9974) 
参见 https://redis.io/topics/acl#key-permissions 和 https://redis.io/topics/acl#selectors
- 集群：分片（节点特定）的发布/订阅支持 (#8621) 
参见 https://redis.io/topics/pubsub#sharded-pubsub
- 在大多数上下文中对子命令进行一流处理（影响 ACL 类别、INFO commandstats 等）(#9504, #10147)
- 命令元数据和文档 (#10104) 参见 https://redis.io/commands/command-docs, https://redis.io/topics/command-tips
- 命令键规范：为客户端提供更好的方式来定位键参数及其读/写目的 (#8324, #10122, #10167) 
参见 https://redis.io/topics/key-specs
- 多部分 AOF 机制，避免 AOF 重写开销 (#9788)
- 集群：支持主机名，不仅限于 IP 地址 (#9530)
- 改进网络缓冲区消耗的内存管理，并提供当总内存超过限制时断开客户端连接的选项 (#8687)
- 集群：断开集群总线连接的机制，防止不受控制的缓冲区增长 (#9774)
- AOF：时间戳注释和支持时间点恢复 (#9326)
- Lua：在 EVAL 脚本中支持函数标志 (#10126) 
参见 https://redis.io/topics/eval-intro#eval-flags
- Lua：支持 RESP3 回复中的 Verbatim 和 Big-Number 类型 (#9202)
- Lua：通过 redis.REDIS_VERSION, redis.REDIS_VERSION_NUM 获取 Redis 版本 (#10066)