# Redis集群

## 概述

虽然单机的**Redis**服务器可以提供一个较高的并发访问服务，集合前面所介绍的**主从复制**机制，从节点同步复制主节点的数据，并通过读写命令的分离，从节点分担部分用户发来的只读命令，可以进一步地提升**Redis**服务器的并发访问量。但是这种机制却无法对写命令的并发进行扩展，因此**Redis**的作者设计了**Redis**集群的架构，集群整合了多组**Redis**单机服务器或者**主从复制**服务器群，集群中的每台服务器只负责处理整个键空间的一个子集。用户执行的写命令按照其操作的**Key**的不同被分散到不同的子服务器上，这样一来便可对整个**Redis**的执行写命令的并发访问量进行水平扩容，这样便可以极大地提高服务器处理数据的并发能力。

```bash
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

使用上述最简单的一个`redis.conf`启动配置文件便能够以集群的方式启动一个**Redis**服务器。集群的管理员可以在某一个集群服务器上通过执行**CLUSTER MEET**命令将另外一个以集群方式启动的服务器加入集群。可以将一个单机的服务器加入集群作为一个单独的**Master**分配实例；也可以将一个**主从复制**的服务器群加入到集群之中，这里只需要将**Master**节点加入到集群，对应的**Slave**节点会自动的被集群感知并加入到集群之中。为了提高集群的可用性，在搭建集群的时候，集群通常是由多个**主从复制**的服务器群构成了，其中的**Master**节点主要提供对外的数据查询服务，而**Slave**节点的主要功能是复制**Master**节点，并在**Master**节点出现故障的时候顶替其继续对外提供数据查询服务。 

每个用户操作的**Key**都可以根据哈希算法计算出哈希值，并可以将这个哈希值映射到`[0, 16383]`的范围内，这其中每一个映射的索引都被称为一个哈希槽`slot`。集群之中的每一个**Master**实例都负责维护一部分哈希槽`slot`数据，属于同一个哈希槽`slot`的**Key**都被存储在负责维护该哈希槽`slot`的**Master**实例上。同时集群之中的每一个分片实例都会定期将自己感知到的集群配置信息存储到磁盘上的文件里，这个文件就是前面在`redis.conf`配置文件之中配置的`cluster-config-file`这个配置项，这个配置项给定了一个**集群配置文件**，其中包含了集群之中各个节点的配置信息，用于在集群分片节点重启的时候，能够通过**集群配置文件**快速地恢复集群的配置状态。

每一个**Key**都对应于一个哈希槽`slot`，每个哈希槽`slot`都被分配给集群之中的某一个分配实例进行维护。这意味着不同的**Key**有可能存在于集群之中的不同节点上，那么相应地用户在集群上通过查询命令操作数据时，也会相比单机模式下受到更多的限制。同时前面我们在介绍**Redis**键空间的逻辑时，曾经介绍过在**Redis**之中实际上时有`[0,15]`这16个数据库，而在集群模式下，所有的数据都被存储在0号数据库之中。

如果用户使用客户端在集群中的某个分片实例上执行命令操作某个**Key**时，这个**Key**有可能并不在接收命令的分片实例上，在这种情况下集群会向用户返回`MOVED`错误，在这个错误信息之中会返回给用户该**Key**所在的分片实例信息，用户应该根据`MOVED`错误之中携带的地址信息，向新的分片实例重新发起请求。如果用户使用的是支持集群模式的客户端，那么客户端会自动处理`MOVED`错误，将请求重新转发到正确的分片实例上。

同样的，如果用户在一条查询命令或者Lua脚本之中操作多个**Key**的时候，也需要保证这些**Key**都属于同一个哈希槽`slot`，否则将无法执行。对于这种情况需要用户在业务层面上保证所操作的多个**Key**都从属于同一个哈希槽`slot`，**Redis**对此也提供了相应的支持，用户在设置**Key**的名字时，可以将其中部分内容使用`{}`包裹起来，这样**Redis**仅仅会对`{}`之中的内容计算其映射的哈希槽`slot`，如此一来，用户便可以在业务层面使用一定的规则将某些**Key**强制映射到同一哈希槽`slot`内进而满足业务处理的需求。

当集群之中的某些分片实例维护了过多的数据的话，集群管理员可以通过**CLUSTER SETSLOTS**命令将某一个分片实例上负责维护某些哈希槽`slot`重新分配给集群之中的其他分片实例进行维护，同时也会将源实例上隶属于这些哈希槽`slot`的键值数据调用**MIGRATE**命令迁移到目标实例上，用以调整与平衡集群之中各个分片实例之间的负载。

与前面用户执行查询命令时，有可能会遇到返回`MOVED`的情况类似。用户执行查询命令与集群管理员迁移数据重新分配哈希槽`slot`的时机会有交叉，这就导致了当我们在某个分片实例上执行关于某个**Key**的查询命令时，这个**Key**对应的数据有可能已经被迁移到另外一个分片实例上，这时集群会向用户返回一个`ASK`错误并携带有对应目标分片实例的地址信息。用户在接收到这个`ASK`错误的情况下也应该像处理`MOVED`错误一样，重新转发查询命令到目标分片集群。

集群之中的各个分片实例之间通过各自的一个特殊端口彼此连接，用于通信与交换信息，这个特殊的端口被称之为**集群总线端口**。各个分片实例可以通过消息将自己的状态信息广播更新给集群之中的其他节点，同时也可以消息的**KeepAlive**机制来确定集群中的分片实例是否处于处于正常工作的状态。分片实例还会在消息之中携带其所能感知的其他节点状态的**Gossip**消息，之所以称之为**Gossip**消息，这就好像人们之前私下传播关于别人的八卦消息一样。基于这个机制，只需要与集群之中的一个分片实例相连便可以通过收到的**Gossip**消息自动地发现集群之中的其他分片实例。同时通过**Gossip**消息，分片实例也可以将其自己感知到的其他节点的故障信息广播给所有的整个集群，用以判断一个疑似的故障节点是否真的出现故障。

虽然单机模式的服务器实例也可以加入到集群之中，但是这里面会存在一定的安全隐患，一旦这台单机的服务器实例出现故障导致下线，那么由这个服务器维护的所有`slot`将无法对外提供服务器。一个**Master**实例没有对应的**Slave**节点，或者其对应的**Slave**节点都处于下线状态的话，我们便可以称之为处于**孤立**状态，一个集群之中如果存在处于**孤立**状态的**Master**实例，这将是一种非常危险的状态。因此在通常情况下，集群应该由多组**主从复制**的服务器群来构成，一旦某台**Master**分片实例发生故障，那么集群会从其对应的**Slave**实例之中选择一台将其提升为新的**Master**实例，以保证集群的可用性。前面我们介绍的哨兵模式之中，可以部署多台哨兵服务器来监控整个**主从复制**的服务器群，一旦**Master**节点出现故障，哨兵服务器便会发起故障转移的流程，选择一个**Slave**节点提升为新的**Master**节点。

在集群之中，参与组成集群的所有**Master**分片实例承担了哨兵模式之中哨兵服务器的作用，一旦检测到某一个**Master**实例因为故障下线，那么其他的**Master**实例便会通过投票选举的方式从该故障**Master**实例的**Slave**节点之中选择一个成为新的**Master**实例，以保证整个集群的可用性。

以上便是对**Redis**集群的一个简要的概述，下我们从代码层面详细地介绍一下**Redis**集群的实现细节。

## 集群数据结构

首先我们来看一下在*src/cluster.h*以及*src/cluster.c*这两个文件之中定义了若干个集群相关的数据结构，通过介绍这些数据结构，我们可以更好地了解后续的业务逻辑的代码。

| 数据结构                | 含义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `clusterNode`         | 代表了集群之中每一个分片实例的抽象，每个**Redis**进群分片实例之中都通过`clusterNode`存储了其所能感知到的集群中所有其他分片信息。 |
| `clusterLink` | 代表了当前分片实例与某一个`clusterNode`所代表的的其他分片的网络连接。 |
| `clusterNodeFailReport` | 当集群中的某一个节点出现故障的时候，这个数据结构代表其他分片对该节点的故障报告。 |
| `clusterState`          | 集群的状态信息，存储了当前**Redis**集群分片实例运行时的状态信息。 |
| `clusterMsg`            | 用于表示集群之中各个分片之间通信消息的抽象。这个数据结构之中除了携带消息数据之外，还携带了发送消息的分片的一些基础状态信息。 |
| `clusterMsgData`        | 用于表示集群之中各个分片之间通信消息具体消息数据的抽象，这个`union`结构，可以分别表示分片间通信的五种消息类型。 |
| `clusterMsgDataGossip`  | `clusterMsgData`的五种消息类型之一，用于通知交换彼此状态信息。 |
| `clusterMsgDataFail`    | `clusterMsgData`的五种消息类型之一，用于通知某个分片发生故障处于下线状态。 |
| `clusterMsgDataPublish` | `clusterMsgData`的五种消息类型之一，用于向集群之中的分片广播消息用。 |
| `clusterMsgDataUpdate`  | `clusterMsgData`的五种消息类型之一，用于更新某个分片的配置纪元、槽位归属的信息。 |
| `clusterMsgDataModule`  | `clusterMsgData`的五种消息类型之一，用于处理模块数据。 |

### 集群配置相关数据结构

首先我们来看一下**Redis**服务器全局状态数据`redisServer`之中所定义的集群的相关数据，而这部分数据主要是集群的配置信息，而上面表格之中提到的`clusterState`数据结构里面主要存储的是集群在运行时的状态信息：

```c
struct redisServer
{
	...
	int cluster_enabled;
	mstime_t cluster_node_timeout;
	char *cluster_configfile;
	struct clusterState *cluster;
	int cluster_migration_barrier;
	int cluster_slave_validity_factor;
	int cluster_require_full_coverage;
	int cluster_slave_no_failover;
  	char *cluster_announce_ip;
  	int cluster_announce_port;
  	int cluster_announce_bus_port;
  	int cluster_module_flags;
    ...
};
```

这其中部分数据自动是通过在集群**Redis**实例启动时通过读取配置文件进行初始化的，一个典型的供集群分片实例使用的配置文件的通常形式为：

```bash
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-migration-barrier 1
cluster-require-full-coverage yes
cluster-replica-no-failover no
cluster-announce-ip 10.1.1.5
cluster-announce-port 6379
cluster-announce-bus-port 6380
```

这个配置文件之中除了配置处理客户端查询请求的端口之外，还拥有若干集群相关的配置项：

1. `cluster-enabled`，表明该**Redis**实例按照集群的模式进行启动，对应`redisServer.cluster_enabled`字段。
2. `cluster-config-file`，配置了该集群**Redis**实例对应集群配置文件，这个文件与手动配置的`redis.conf`配置文件不同，通常是由集群**Redis**实例创建与更新，用于记录集群之中各个分片实例的配置信息，对应`redisServer.cluster_node_timeout`字段。
3. `cluster-node-timeout`，设置了集群中的分片超时时间，如果一个节点超过这个配置的毫秒时间内无法访问，便被视为超时失败状态，对应`redisServer.cluster_node_timeout`字段。
4. `cluster-migration-barrier`，设置了`master`分片想要保持在集群之中的连接时所需要的最小的`slave`副本数量，对应`redisServer.cluster_migration_barrier`字段。
5. `cluster-require-full-coverage`，默认是`yes`，如果任何一个分片节点没有覆盖一定比例的键空间，那么整个集群将停止写入操作，对应`redisServer.cluster_require_full_coverage`。
6. `cluster-replica-no-failover`，默认是`no`，但是如果被设置为`yes`，表示集群会禁止`slave`分片节点在`master`分片节点出现故障时进行故障转移；不过即使在这种情况下，依然可以强制进行手动故障转移，对应`redisServer.cluster_replica_no_failover`字段。
7. `cluster-announce-id`、`cluster-announce-port`、`cluster-announce-bus-port`这三个配置项可以为使用该配置文件的集群分片实例设置静态网络配置，使每个节点都知道自己的公共网络地址，以便其他节点能够正确地映射这个节点的地址信息，适用于因特殊网络环境导致节点地址发现失败的情况，并非必须配置的配置项，分别对应`redisServer.cluster_announce_ip`，`redisServer.cluster_announce_port `，`redisServer.cluster_announce_bus_port`三个字段。

### 分片节点相关数据结构

接下来我们看一下代表分片实例抽象的数据类型`clusterNode`：

```c
typedef struct clusterNode {
    mstime_t ctime;
    char name[CLUSTER_NAMELEN];
    int flags;
    uint64_t configEpoch;
    unsigned char slots[CLUSTER_SLOTS/8];
    int numslots;
    int numslaves;
    struct clusterNode **slaves;
    struct clusterNode *slaveof;
    mstime_t ping_sent;
    mstime_t pong_received;
    mstime_t fail_time;
    mstime_t voted_time;
    mstime_t repl_offset_time;
    mstime_t orphaned_time;
    long long repl_offset;
    char ip[NET_IP_STR_LEN];
    int port;
    int cport;
    clusterLink *link;
    list *fail_reports;
} clusterNode;
```

在这个数据结构之中：

1. `clusterNode.ctime`，存储这个节点对象被创建的时间。
2. `clusterNode.name`，存储这个节点对应的分片实例的名字。
3. `clusterNode.flags`，记录节点对应的分片实例的状态，节点的状态以掩码的形式定义如下：
   1. `CLUSTER_NODE_MASTER`，这个状态表示这个节点对应的分片实例是一个主服务器。
   2. `CLUSTER_NODE_SLAVE`，这个状态表示这个节点对应的分片实例是一个从服务器。
   3. `CLUSTER_NODE_PFAIL`，这个状态表示这个节点对应的分片实例被当前实例认为是下线状态，可以对应于哨兵模式下的**主观下线**。
   4. `CLUSTER_NODE_FAIL`，这个状态表示这个节点对应的分片实例被集群中的过半分片实例判定为下线，可以对应于哨兵模式下的**客观下线**。
   5. `CLUSTER_NODE_MYSELF`，这个状态表示这个节点对应的分片实例就是当前分片实例自己。
   6. `CLUSTER_NODE_HANDSHAKE`，这个状态表示当前分片实例与对应节点对应分片还处于建立连接的握手状态。
   7. `CLUSTER_NODE_NOADDR`，这个状态表示当前分片实例还不知道这个节点对应分片的地址信息。
4. `clusterNode.configEpoch`，存储当前分片实例所获取到的对应这个节点的分片的配置纪元。
5. `clusterNode.slots`，以掩码的形式存储这个`clusterNode`节点对应的分片实例所负责的**slot**。
6. `clusterNode.numslots`，记录这个`clusterNode`节点对应的分片实例负责的**slot**数量。
7. `clusterNode.numslaves`，如果这个节点对应的分片实例是一个主服务器，那么这个字段记录了这个主服务分片关联的从服务器数量。
8. `clusterNode.slaves`，如果这个节点对应的分片实例是一个主服务器，那么这个指针数组则存储其对应的所有从服务器分片对应的`clusterNode`节点的指针，数组的长度就是`clusterNode.numslaves`定义的长度。
9. `clusterNode.slaveof`，如果这个节点对应的分片实例是一个从服务器，那么这个指针指向该从服务器对应的主服务器的`clusterNode`节点。
10. `clusterNode.ping_sent`，记录当前集群分片实例向该节点对应的分片发送**PING**命令的时间戳。
11. `clusterNode.pong_received`，记录当前集群分片实例从`clusterNode`节点对应的分片中接收**PONG**返回的时间戳。
12. `clusterNode.fail_time`，记录该`clusterNode`节点被设置成`CLUSTER_NODE_FAIL`状态时的时间戳。
13. `clusterNode.voted_time`，仅针对**Master**节点有效，在故障转移时，记录当前实例为该故障**Master**节点的某一个**Slave**节点的投票时间戳；在每一轮投票之中，只能为一个**Slave**节点进行投票。
15. `clusterNode.orphaned_offset`，仅针对**Master**节点，记录该节点成为**孤立**状态的时间戳，所谓**孤立**状态指定就是这个**Master**节点没有对应的**Slave**节点或者所有的**Slave**节点全部故障下线。
16. `clusterNode.repl_offset`，仅针对**Slave**节点，记录了该节点针对其对应的**Master**节点的复制偏移量。
17. `clusterNode.ip`，这个字段存储了该节点对应的分片实例的**IP**地址。
18. `clusterNode.port`，这个字段存储了该节点对应的分片实例监听的用于处理客户端查询的端口号。
19. `clusterNode.cport`，这个字段存储了该节点对应的分片实例监听的用于处理分片实例间交互的端口号。
20. `clusterNode.link`，存储当前分片实例与该节点的分片实例的网络连接的抽象。
21. `clusterNode.fail_reports`，这个双端链表之中存储了类型为`clusterNodeFailReport`数据节点，当这个节点出现故障时，这个字段存储了当前分片实例所接收到的其他分片实例对于该节点的错误报告。

现在我们介绍一下`clusterLink`这个数据结构的定义：

```c
typedef struct clusterLink {
    mstime_t ctime;             /* Link creation time */
    int fd;                     /* TCP socket file descriptor */
    sds sndbuf;                 /* Packet send buffer */
    sds rcvbuf;                 /* Packet reception buffer */
    struct clusterNode *node;   /* Node related to this link if any, or NULL */
} clusterLink;
```

在这个数据结构之中：

1. `clusterLink.ctime`，这个字段记录该条网络连接的创建时间戳。
2. `clusterLink.fd`，存储这个网络连接对应的文件描述符。
3. `clusterLink.sndbuf`，这条网络连接对应的应用层发送缓冲区。
4. `clusterLink.rcvbuf`，这条网络连接对应的应用层接收缓冲区。
5. `clusterLink.node`，这条网络连接对应连接的分片实例对应的`clusterNode`节点。

我们看一下`clusterNodeFailReport`这个数据结构的定义：

```c
typedef struct clusterNodeFailReport {
    struct clusterNode *node;
    mstime_t time;
} clusterNodeFailReport;
```

在这个数据结构之中，

1. `clusterNodeFailReport.node`，该`clusterNode`节点指针指向了该条错误报告是哪个分片实例所报告的。
2. `clusterNodeFailReport.time`，则是该条错误报告的时间戳。

### 集群状态数据结构

最后我们看一下代表集群运行状态的`clusterState`这个数据结构的定义：

```c
typedef struct clusterState {
    clusterNode *myself;
    uint64_t currentEpoch;
    int state;
    int size;
    dict *nodes;
    dict *nodes_black_list;
    clusterNode *migrating_slots_to[CLUSTER_SLOTS];
    clusterNode *importing_slots_from[CLUSTER_SLOTS];
    clusterNode *slots[CLUSTER_SLOTS];
    uint64_t slots_keys_count[CLUSTER_SLOTS];
    rax *slots_to_keys;
    mstime_t failover_auth_time;
    int failover_auth_count;
    int failover_auth_sent;
    int failover_auth_rank;
    uint64_t failover_auth_epoch;
    int cant_failover_reason;
    mstime_t mf_end;
    clusterNode *mf_slave;
    long long mf_master_offset;
    int mf_can_start;
    uint64_t lastVoteEpoch;
    int todo_before_sleep;
    long long stats_bus_messages_sent[CLUSTERMSG_TYPE_COUNT];
    long long stats_bus_messages_received[CLUSTERMSG_TYPE_COUNT];
    long long stats_pfail_nodes;
} clusterState;
```

在这个数据结构之中：

1. `clusterState.myself`，这个集群节点指针用于指向当前的分片实例服务器所对应的`clusterNode`集群节点。
2. `clusterState.state`，这个字段用于标记当前的分片实例服务器的集群状态。
3. `clusterState.size`，这个字段记录了在当前的集群之中主服务器实例的数量。
4. `clusterState.nodes`，这个哈希表用于记录当前分片实例服务器之中所能感知的集群节点，以节点名字`clusterNode.name`为**Key**，以节点指针`clusterNode*`为**Value**。
5. `clusterState.nodes_black_list`，这个哈希表则用于记录短期内不能再重新加入到集群之中的分片节点。
6. `clusterState.migrating_slots_to`，如果当前分片实例服务器正在将某个指定的`slot`数据转移给其他的分片实例时，那么`migrating_slots_to`数组之中这个对应的位置则会存储这次数据移动的目标`clusterNode`节点指针。
7. `clusterState.importing_slots_from`，如果当前分片实例正在从其他分片实例之中导入指定的`slot`数据时，这个数组对应的位置则会存储这个`slot`原所在的分片对应的`clusterNode`节点指针。
8. `clusterState.slots`，这是一个指针数组，其中存储了相应索引的`slot`所属的分片实例所对应的`clusterNode`对象指针。
9. `clusterState.slots_keys_count`，这个数组存储了集群之中每一个**slot**之中对应的**Key**的数量。
10. `clusterState.slots_to_keys`，是一个基数树，其中存储了`slot`与对应归属于这个`slot`的`key`的映射关系。
11. `clusterState.stats_bus_messages_sent`，按照消息类型统计当前实例节点中各类总线消息的发送数量。
12. `clusterState.stats_bus_messages_received`，按照消息类型统计当前实例节点中各类纵向消息的接收数量。
13. `clusterState.failover_auth_time`，上一次或者下一次选举的时间戳。
14. `clusterState.failover_auth_count`，存储在故障转移中当前已经收到的投票数。
15. `clusterState.failover_auth_sent`，存储在故障转移中当前已经发出的投票数。
16. `clusterState.failover_auth_rank`，故障转移中当前**Slave**节点在复制其**Master**节点的所有**Slave**节点之中的排名。
17. `clusterState.failover_auth_epoch`，存储故障转移当前的选举纪元。
18. `clusterState.cant_failover_reason`，记录了当前**Slave**实例没能开启故障转移的原因。
19. `clusterState.lastVoteEpoch`，用于记录在故障转移当前的**Master**节点最近一次投票的纪元号。

## 集群配置与初始化以及节点操作

### 集群配置文件

这里也需要解释一下*nodes.conf*这个配置文件在**Redis**集群之中的作用。这个配置文件并不是一个用户可以编辑的文件，而是每次集群信息发生变换时，由**Redis**进程自动保存的文件。这个文件之中存储了集群之中已知的各个分片节点的连接信息、状态信息等重要信息。这样当集群之中的**Redis**进程启动后便可以通过这个配置文件快速加载整个集群的信息。这个配置文件默认的文件名被定义在*src/server.h*头文件之中：

```c
#define CONFIG_DEFAULT_CLUSTER_CONFIG_FILE "nodes.conf"
```

当然我们也可以在*redis.conf*启动配置文件之中指定对应的集群配置文件的文件名。

首先我们来看一下一个典型的*nodes.conf*配置文件的内容：

``` bash
617f92606a9204d52a1cc12f35cdc4f05fede2c4 127.0.0.1:30003@40003 master - 0 1690185189137 3 connected 6554-9829
428fdf546d6c486425552ad3df00503407b1219b 127.0.0.1:30010@40010 slave 617f92606a9204d52a1cc12f35cdc4f05fede2c4 0 1690185189137 10 connected
418f85d8aef01c177baca2dcd88f431602bab251 127.0.0.1:30004@40004 master - 0 1690185189137 4 connected 9830-13106
76fda0b35614df76eb2904b1c797b155764d7adb 127.0.0.1:30007@40007 slave 5eb10f1300857a0363f802cc756e455533ddb78b 0 1690185189137 7 connected
8883e968f680b2debd4291b65dac2786d68ed7d1 127.0.0.1:30001@40001 myself,master - 0 1690185189000 1 connected 0-3276
230ffd9704360afb01d9225469cfedb26caf1001 127.0.0.1:30008@40008 slave 418f85d8aef01c177baca2dcd88f431602bab251 0 1690185189137 8 connected
46b42185c5a887797ffb41d26bdc2ce5ff823bc5 127.0.0.1:30006@40006 slave 8883e968f680b2debd4291b65dac2786d68ed7d1 0 1690185189036 6 connected
8e98d6a447ad07f6e898d9b3a93d062cd9c07643 127.0.0.1:30009@40009 slave 7a4110a155651eb31e8263eba7da0e9ca4a34e8e 0 1690185189538 9 connected
5eb10f1300857a0363f802cc756e455533ddb78b 127.0.0.1:30002@40002 master - 0 1690185189137 2 connected 3277-6553
7a4110a155651eb31e8263eba7da0e9ca4a34e8e 127.0.0.1:30005@40005 master - 0 1690185189137 5 connected 13107-16383
vars currentEpoch 10 lastVoteEpoch 0
```

这个集群配置文件主要由两个部分组成：

1. 分片实例配置信息。
2. 集群纪元信息。

#### 集群配置文件的加载

```c
int clusterLoadConfig(char *filename);
```

`clusterLoadConfig`这个函数用于从集群配置文件之中加载集群分片对应的配置信息，文件之中最为重要的的配置数据便是集群之中各个节点的配置信息，`clusterLoadConfig`函数会遍历配置文件之中每一个分片实例的配置信息，使用这些信息创建分片实例对应的`clusterNode`集群节点数据对象，每一行配置信息中会包含如下的数据：

1. 首先是标记分片实例的唯一标识，这个数据对应`clusterNode.name`字段之中的数据。
2. 网络连接信息，这里面包含了三个连接数据，以`ip:port@busport`的形式组成的，这其中`port`端口用来处理分片节点处理客户端的查询命令，`busport`端口被称为**集群总线端口**，这是一个节点对节点的通信通道，使用二进制协议，由于带宽和处理时间短，它更适合在分片节点之间交换信息。
3. 分片节点的标记信息，用于表示这个节点对应的状态，这些标记信息包括：
   1. `myself`，对应`CLUSTER_NODE_MYSELF`，表明这个分片节点就是当前启动的实例自己。
   2. `master`，对应`CLUSTER_NODE_MASTER`，表明这个分片是一个`master`节点。
   3. `slave`，对应`CLUSTER_NODE_SLAVE`，表明这个分片是一个`slave`节点。
   4. `fail?`，对应`CLUSTER_NODE_PFAIL`，表明这个分片实例在当前实例的角度上处于主观下线的状态。
   5. `fail`，对应`CLUSTER_NODE_FAIL`，表明这个分片实例处于客观下线的状态。
   6. `handshake`，对应`CLUSTER_NODE_HANDSHAKE`，表明和这个分片实例处于建立连接的握手阶段。
   7. `noaddr`，对应`CLUSTER_NODE_NOADDR`，表明当前还不知道这个分片实例的地址信息。
4. 对于一个`slave`分片节点，这里会存储其对应的**master**节点的唯一标识符，对应`clusterNode.slaveof`字段；如果这是一个`master`这里则是会使用一个`-`字符进行占位。
5. 记录这个分片发送**PING**命令的时间戳，对应于`clusterNode.ping_sent`字段。
6. 记录这个分片接受到**PONG**命令的时间戳，对应于`clusterNode.pong_received`字段。
7. 记录这个分片的配置纪元，对应`clusterNode.configEpoch`字段。
8. 记录这个分片是否处于连接状态，有`connected`以及`disconnected`两种状态。
9. 如果这个分片是一个`master`节点，那么最后会以一个闭区间的形式存储这个分片负责维护的`slot`段，例如`13107-16383`表示这个分片之中存储着属于从`13107`到`16383`这些`slot`的**Key**。

在这个集群配置文件之中，除了记录分片节点的配置数值之外，还通过行首的`vars`关键字，定义了一个特殊的配置行，在这里面存储了集群的当前纪元，对应于`clusterState.currentEpoch`字段；以及上一次的投票纪元，对应于`clusterState.lastVoteEpoch`字段。

上述的这些集群配置信息都会通过`clusterLoadConfig`这个函数接口进行解析，并加载到**Redis**的内存之中。

#### 集群配置文件的保存

与加载数据对应的，**Redis**之中还给出了从内存之中`clusterNode`数据生成配置文件的接口函数，首先便是通过给定的`clusterNode`节点信息生成集群配置文件之中分片实例配置信息的函数接口：

```c
sds clusterGenNodeDescription(clusterNode *node);
sds clusterGenNodesDescription(int filter);
```

这其中`clusterGenNodeDescription`函数用于从给定的`clusterNode`分片节点数据之中生成集群配置文件之中的节点配置信息；而函数`clusterGenNodesDescription`则会遍历系统的`clusterState.nodes`这个哈希表，依次为表中的每一个分片节点调用`clusterGenNodeDescription`函数，生成整个集群的分片节点配置信息，同时这个函数也可以根据传入的`filter`参数过滤掉不符合条件的分片节点。

这样我们便通过下面的两个函数将当前的集群状态存储在集群配置文件之中：

```c
int clusterSaveConfig(int do_fsync);
void clusterSaveConfigOrDie(int do_fsync);
```

保存的集群配置文件的时机，可以是手动通过命令**CLUSTER SAVECONFIG**这个命令来手动触发保存集群配置文件；也可以是系统调用`clusterSaveConfigOrDie`来自动生成集群配置文件，自动生成的实际包括：

1. 在集群新的**Redis**示例启动时会在`initServer`函数之宗调用`clusterInit`来初始化集群示例所需的相关数据，如果本地不存在集群配置文件，那么会立即调用`clusterSaveConfigOrDie`函数来生成一个默认的集群配置文件。
2. 当集群之中的各个分片在进行数据交互的时候，如果当前实例是一个`master`实例，同时收到另外一个`master`分片实例的数据时并且两者的配置纪元`clusterNode.configEpoch`相同的时候，会调用`clusterSaveConfigOrDie`自动生成集群配置文件。
3. 当集群之中的某一个`master`分片实例出现故障进行故障转移时，被提升为新的`master`的对应的`slave`分片会立即调用`clusterSaveConfigOrDie`来将新的集群信息固化到集群配置文件之中。
4. 当前集群实例感知的集群配置信息发生变化的时候，会在`clusterState.todo_before_sleep`字段之中添加上`CLUSTER_TODO_SAVE_CONFIG`标记，并在**Redis**准备退出事件循环前，通过调用`clusterBeforeSleep`函数来将变化的信息存储到集群配置文件之中。
5. 由于在集群配置文件之中存储了每个`master`分片实例负责的`slot`的范围，在一个集群**Redis**分片实例重启后，如果发现实际负责`slot`范围与本地集群配置文件不符的话，那么会在`verifyClusterConfigWithData`校验函数之中，通过`clusterSaveConfigOrDie`来更新本地集群配置文件。

```c
int clusterLockConfig(char *filename);
```

`clusterLockConfig`这个函数会调用`flock`系统调用，对集群配置文件尝试添加文件锁，之所以这么做，是因为我们经常需要在有一个新版集群配置数据的时候原地更新*nodes.conf*这个配置文件，重新打开这个文件，并对这个文件进行原地写入。如果给定的配置文件不存在，那么**Redis**则会首先创建一个空的文件出来，并对该文件调用`flock`系统调用进行加锁操作，这个加锁操作是以`LOCK_EX|LOCK_NB`的策略进行的，也就是会独占这个配置文件的锁，同时当有其他进程已经锁住该文件的时候，进程不会阻塞在这里等待该锁被释放。

### 集群的启动与初始化

函数`clusterInit`用于执行**Redis**集群当前的分片服务器的初始化流程，会在**Redis**服务器启动初始化的`initServer`接口之中被调用：

```c
void initServer(void)
{
  ...
  if (server.cluster_enabled) clusterInit();
  ...
}
```

在这个函数之中，会执行如下的初始化逻辑：

1. 会对`serverState.cluster`这个`clusterState`数据结构进行初始化。
2. 通过`clusterLockConfig`锁定本地集群配置文件，并通过`clusterLoadConfig`来加载集群配置文件；如果本地集群配置文件不存在，那么会通过`clusterSaveConfigOrDie`来生成初始默认的集群配置文件。
3. 调用`listenToPort`函数接口建立对集群总线端口的监听，并调用`aeCreateFileEvent`函数将其注册到事件循环之中，设置对应的事件处理函数函数`clusterAcceptHandler`，这个事件处理函数将会负责接受其他分片实例的连接请求。
4. 初始化从`slot`到键之间映射的基数树数据结构`clusterState.slots_to_keys`，以及存储每个`slot`中键数量的数组`clusterState.slots_keys_count`。
5. 设置对应实例自己的`clusterNode`上的地址信息。

与上面初始化集群信息的`clusterInit`函数相对的，是重置集群的函数`clusterReset`，这个函数会通过执行命令的方式被手动触发：

```bash
CLUSTER RESET [HARD | SOFT]
```

这个函数用于重置当前的**Redis**集群实例，将执行如下的逻辑：

1. 如果当前分片实例是一个`slave`实例，那么通过`clusterSetNodeAsMaster`将其转化为一个`master`实例。
2. 通过`clusterCloseAllSlots`清理关闭当前正在移入以及移出的`slot`数据，也就是清理`clusterState.migrating_slots_to`以及`clusterState,importing_slots_from`两个数组。
3. 所有的`slot`调用`clusterDelSlot`函数来清理所有的`slot`分配信息。
4. 释放`clusterState.nodes`之中除了`clusterState.myself`指定的自己`clusterNode`之外的所有其他的`clusterNode`。
5. 如果选择的是**HARD**重置，那么会清理`clusterState.currentEpoch`以及`clusterState.lastVoteEpoch`等数据，并为该节点重新分配新的唯一标识。
6. 最后会将上述的修改更新到本地的集群配置文件。

### 集群连接接口

前面我们已经介绍集群**Redis**实例会在启动是在`clusterInit`函数之中建立对于集群总线端口的监听，并注册事件处理函数`clusterAcceptHandler`用来接收其他集群实例的连接请求，下面我们详细介绍一下集群实例处理网络连接的相关逻辑。

```c
clusterLink *createClusterLink(clusterNode *node) ;
void freeClusterLink(clusterLink *link);
void clusterAcceptHandler(aeEventLoop *el, int fd, void *privdata, int mask);
void clusterCron(void)；
```

这里`createClusterLink`以及`freeClusterLink`两个函数分别用于创建以及释放一个用于表示节点之间总线连接的数据结构`clusterLink`。

```c
void clusterAcceptHandler(aeEventLoop *el, int fd, void *privdata, int mask)
{
    clusterLink *link;
    ...
	while(max--) {
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        anetNonBlock(NULL,cfd);
        anetEnableTcpNoDelay(NULL,cfd);
        link = createClusterLink(NULL);
        link->fd = cfd;
        aeCreateFileEvent(server.el,cfd,AE_READABLE,clusterReadHandler,link);
    }
}
```

而函数`clusterAcceptHandler`的用途是会在每次事件循环之中最多接收1000条其他集群分片实例建立网络连接的请求，并为每个新的集群总线连接注册可读事件处理函数`clusterReadHandler`用来处理当前的集群分片实例与其他分片实例之间的交互。当然这里我们看到在`clusterAcceptHandler`函数之中**Redis**只是接收了网络连接请求，并为之创建了一个`clusterLink`对象，但是需要注意的是此时并没有将其与对应的`clusterNode`节点对象建立联系。

在另外一个集群心跳接口`clusterCron`之中也会调用`createClusterLink`函数来创建`clusterLink`对象：

```c
void clusterCron(void)
{
    dictIterator *di;
    ...
    di = dictGetSafeIterator(server.cluster->nodes);
    while((de = dictNext(di)) != NULL) {
        clusterNode *node = dictGetVal(de);
        if (node->flags & (CLUSTER_NODE_MYSELF|CLUSTER_NODE_NOADDR)) continue;
        if (node->link == NULL) {
            int fd;
            clusterLink *link;
            fd = anetTcpNonBlockBindConnect(server.neterr, node->ip, node->cport, NET_FIRST_BIND_ADDR);
            link = createClusterLink(node);
            link->fd = fd;
            aeCreateFileEvent(server.el, link->fd, AE_READABLE, clusterReadHandler, link);
            ...
            node->flags &= ~CLUSTER_NODE_MEET;
        }
    }
    ...
}
```

如果说在`clusterAcceptHandler`函数之中是**Redis**被动等待其他集群分片实例向当前分片实例发起连接的话，那么`clusterCron`心跳函数便是处理实例主动向其他分片节点发起连接的情况，在使用`anetTcpNonBlockBindConnect`向对应分片节点发起连接后，便会创建一个`clusterLink`对象，并将其与`clusterNode`节点对象进行关联。

### 集群节点接口

集群实例中的每一个节点对象`clusterNode`都代表了对于集群之中的每个分片实例的抽象，这些节点对象都是存储在`clusterState.nodes`哈希表中，并以节点唯一标识符作为哈希表的键，以节点的`clusterNode`指针作为哈希表的值。

```c
clusterNode *createClusterNode(char *nodename, int flags);
void freeClusterNode(clusterNode *n);
int clusterAddNode(clusterNode *node);
void clusterDelNode(clusterNode *delnode);
clusterNode *clusterLookupNode(const char *name);
```

这五个基础的节点操作函数的作用分别为：

1. `createClusterNode`，分配并初始化构造一个`clusterNode`对象。
2. `freeClusterNode`，释放一个给定的`clusterNode`对象。
3. `clusterAddNode`，将一个给定的`clusterNode`对象对应的键值对加入到当前实例的`clusterState.nodes`哈希表之中。
4. `clusterDelNode`，从当前实例`clusterState.nodes`这个哈希表之中移除给定的`clusterNode`的键值对。
5. `clusterLookupNode`，根据给定的节点唯一标识符来从实例`clusterState.nodes`这个哈希表之中查找对应的`clusterNode`节点指针。

```c
clusterNode *createClusterNode(char *nodename, int flags);
int clusterStartHandshake(char *ip, int port, int cport);
```

我们现在着重看一下`createClusterNode`这个函数接口，按照给定的节点唯一标识符以及`flags`信息来创建一个`clusterNode`节点数据，**Redis**会在如下的时机上调用`createClusterNode`来创建`clusterNode`：

1. 在**Redis**启动加载集群配置文件时，从文件之中的分片节点配置信息之中调用`createClusterNode`创建`flags`为`0`的对应的`clusterNode`对象。
2. 当管理使用**CLUSTER MEET**这个集群命令向集群中添加新的节点时，或者在集群分片通过**Gossip**消息交互数据时发现有新的节点加入集群时，都会通过`clusterStartHandshake`函数开启当前分片节点同新节点的连接**握手**流程，在这个函数之中同样会调用`createClusterNode`接口来为新的节点创建`flags`标记为`CLUSTER_NODE_HANDSHAKE|CLUSTER_NODE_MEET`的`clusterNode`对象。

但是上面这两种创建`clusterNode`对象的逻辑，都只是创建出了对应的节点对象，但是此时都没有建立与节点之间的网络连接，也就是`clusterNode.link`保存的`clusterLink`指针此时都是为`NULL`，集群会在前面介绍的`clusterCron`心跳函数之中，遍历所有没有建立网络连接的`clusterNode`对象，也就是`clusterNode.link`指针为`NULL`的节点对象，根据其网络配置信息，发起与对应节点的建立连接的请求。

如果说上述两种创建节点`clusterNode`对象的方式属于主动创建节点对象，并主动发起建立网络连接请求的话，**Redis**还提供了被动接收网络连接并为其创建`clusterNode`对象的方式。在上述步骤`2`之中，当用户在实例**A**上通过执行**CLUSTER MEET**命令尝试将实例**B**加入到集群之中时，会在实例**A**的心跳函数之中调用`anetTcpNonBlockBindConnect`函数尝试向**B**实例发起网络连接，并发送一条**MEET**消息向自己的配置信息告知实例**B**。**B**实例接收到**MEET**消息之后，会根据消息之中的配置信息，为实例**A**创建一个`clusterNode`对象：

```c
int clusterProcessPacket(clusterLink *link) {
    clusterMsg *hdr = (clusterMsg*) link->rcvbuf;
    uint16_t type = ntohs(hdr->type);
    clusterNode *sender;
    sender = clusterLookupNode(hdr->sender);
    ...
        if (!sender && type == CLUSTERMSG_TYPE_MEET) {
            clusterNode *node;
            node = createClusterNode(NULL,CLUSTER_NODE_HANDSHAKE);
            ...
            clusterAddNode(node);
            clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG);
        }
}
```

通过代码的阅读，我们可以发现一个有趣的地方，虽然**TCP**连接是双工的网络连接，也就是在一条网络连接上，既可以发送数据，又可以接收数据，但是在集群之中两个分片实例之间实际上是存在着两条网络连接的，也就是每个`clusterNode`节点对象是对应两个`clusterLink`连接对象，这其中有一个`clusterLink`对象通过`clusterLink.node`字段以及`clusterNode.link`字段与`clusterNode`相关联的，这个`clusterLink`连接对象是在前面介绍的`clusterCron`函数之中，由当前实例主动发起创建的，并关联到对应的`clusterNode`节点对象上的。

而另外一个`clusterLink`连接对象则是在`clusterAcceptHandler`函数之中分片实例被动接受其他实例的连接请求而创建的，这个连接对象并不会与`clusterNode`对象相关联，也就是说对于这个连接对象`clusterLink.node`的指针字段为空。从上面的代码片段我们也可以看到，我们从这个`clusterLink`连接对象上接收到的数据都是`clusterMsg`形式的消息，在这个消息之中，给出了其对应的节点的唯一标识符`clusterMsg.sender`，通过这个字段上的唯一标识符便可以通过`clusterLookupNode`函数查找到对应的`clusterNode`，这样便可以在处理接收到的消息数据时，与对应的`clusterNode`进行关联。

接下来我们再来看另外一个操作节点对象的接口：

```c
void clusterRenameNode(clusterNode *node, char *newname);
```

顾名思义，这个函数是用来更新节点对象的唯一标识符的，同时这个函数也会将存储在`clusterState.nodes`这个哈希表之中的对应节点的键值对进行更新。之所以需要对节点对象`clusterNode`的唯一标识符进行更新，是因为在如下的情况下，我们并不知道节点对象对应的分片实例的唯一标识符：

1. 当我们在实例**A**上，使用**CLUSTER MEET**命令将一个实例**B**加入到集群时，会立即为**B**实例创建一个对应的`clusterNode`节点对象，但是此时实例**A**并不能事先预知实例**B**的唯一标识符，因此预先会给`clusterNode`对象预先分配一个临时的唯一标识符。
2. 还是以上述的示例，结合前面的介绍，分片实例**A**主动连接分片实例**B**后会向其发送一个`CLUSTERMSG_TYPE_MEET`消息，而分片实例**B**在接收到这个`CLUSTERMSG_TYPE_MEET`消息之后，会为分片实例**A**创建一个对应的`clusterNode`节点对象，这里依然时为这个节点对象分配一个临时的唯一标识符。

上述两种情况，在分片节点完成**握手**建立连接之后，双方也就知道了彼此的唯一标识符，接下来便可以使用`clusterRenameNode`函数接口更新对应节点对象的唯一标识符：

```c
int clusterProcessPacket(clusterLink *link) {
    ...
    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG || type == CLUSTERMSG_TYPE_MEET)
    {
        if (link->node) {
            ...
            if (nodeInHandshake(link->node)) {
                ...
                clusterRenameNode(link->node, hdr->sender);
                link->node->flags &= ~CLUSTER_NODE_HANDSHAKE;
            }
        }
    }
```

我们在来看另外一个函数：

```c
int clusterHandshakeInProgress(char *ip, int port, int cport);
```

这个函数用于遍历当前的实例的节点列表`clusterState.nodes`，检查是否存在与参数之中给定的地址信息匹配的处于握手状态的`node`。这个函数主要是用于在与一个给定的集群节点地址进行握手`clusterStartHandshake`之前检查是否重复建立连接。

```c
int clusterStartHandshake(char *ip, int port, int cport) {
    ...
    if (clusterHandshakeInProgress(norm_ip,port,cport)) {
        errno = EAGAIN;
        return 0;
    }
    ...
}
```



### 集群故障报告

接下来我们看一下关于故障报告有关的代码。

前面我们在简介`clusterNode`数据结构的时候，介绍了`clusterNode.fail_reports`这个存储故障报告的双端链表，其中链表之中的每一个节点都是一个针对这个`clusterNode`所代表的集群分片实例的故障报告，这个故障报告的数据结构为：

```c
typedef struct clusterNodeFailReport {
    struct clusterNode *node;
    mstime_t time;
} clusterNodeFailReport;
```

这其中`clusterNodeFailReport.node`表示是集群之中哪个分片实例上报的这个故障的报告，而`clusterNodeFailReport.time`则是上报故障报告的时间戳。

```c
int clusterNodeAddFailureReport(clusterNode *failing, clusterNode *sender);
```

如果在集群之中有实例**A**、实例**B**、实例**C**，如果**B**实例发生故障被**C**实例感知到，并通过**Gossip**消息告知实例**A**，那么便会在**A**实例上调用`clusterNodeAddFailureReport`这个接口，将**B**实例对应的`clusetrNode`作为`failing`，将**C**实例对应的`clusterNode`作为`sender`，创建一个`clusterNodeFailReport`故障报告，将其加入到**B**实例对应的`clusterNode.fail_reports`链表之中。

处理上面这个用于添加故障报告的函数结构之外，在*src/cluster.c*源文件之中还给出了三个用于操作故障报告的函数接口。

```c
void clusterNodeCleanupFailureReports(clusterNode *node);
int clusterNodeDelFailureReport(clusterNode *node, clusterNode *sender);
int clusterNodeFailureReportsCount(clusterNode *node);
```

这里函数分别为：

1. `clusterNodeCleanupFailureReports`，用于清理给定的`clusterNode`节点上过于陈旧的故障报告。
2. `clusterNodeDelFailureReport`，用于从`node`这个`clusterNode`节点上删除由`sender`上报的故障报告。
3. `clusterNodeFailureReportsCount`，用于获取`node`这个`clusterNode`节点上的故障报告的数量。

在了解了关于集群故障报告的逻辑之后，我们来看一下，**Redis**集群是以何种逻辑将分片实例标记为故障状态的。

```c
void markNodeAsFailingIfNeeded(clusterNode *node);
```

这个函数用于检查一个给定的节点`clutserNode`是否应该被标记成客观故障状态`CLUSTER_NODE_FAIL`，这里需要检查两个条件：

1. 这个节点`node`必须被标记为主观故障状态`CLUSTER_NODE_PFAIL`。
2. 有过半的集群节点上报了关于这个节点`node`的故障报告，也就是调用`clusterNodeFailureReportsCount`函数获取到的故障报告数量超过了当前节点数量`clusterState.size`的一半。

如果给定节点`node`满足上面的两个条件，便会取消掉这个节点上的`CLUSTER_NODE_PFAIL`，并设置上`CLUSTER_NODE_FAIL`标记，同时更新节点的故障时间`clusterNode.fail_time`。这个函数会在当前节点处理**Gossip**消息时，如果处理到一个携带着`CLUSTER_NODE_FAIL|CLUSTER_NODE_PFAIL`标记的**Gossip**消息时，会调用`markNodeAsFailingIfNeeded`来进行前面的检查。

```c
void clearNodeFailureIfNeeded(clusterNode *node);
```

与前面这个`markNodeAsFailingIfNeeded`这个为节点`node`设置故障标记的函数相对应的，**Redis**集群还提供了`clearNodeFailureIfNeeded`这个函数用于在给定节点`node`能够再次访问的时候，取消这个节点的故障标记`CLUSTER_NODE_FAIL`。这个函数会在当前实例节点收到故障节点的**PONG**消息时被调用，其清理的逻辑为：

1. 如果这个`node`节点是一个**SLAVE**节点，或者没有关联的`slot`，那么便会取消掉这个节点的`CLUSTER_NODE_FAIL`的标记。
2. 如果这个`node`节点是一个**MASTER**节点，并且已经处于故障状态很长时间，同时依然有关联的`slot`，那么也会取消掉这个节点的`CLUSTER_NODE_FAIL`标记。

### 集群黑名单

现在我们现在看一下集群的黑名单的相关操作逻辑：

```c
void clusterBlacklistCleanup(void);
void clusterBlacklistAddNode(clusterNode *node);
int clusterBlacklistExists(char *nodeid);
```

所谓集群黑名单对应集群状态数据结构之中的`clusterState.nodes_black_list`字典字段，而**Redis**设计这种黑名单机制是一种确保在一定时间内不重新添加具有给定节点ID的给定节点的方法。之所以需要这样的机制，是为了确保在将一个分片实例从集群之中完全删除的过程中，这个集群节点不会被错误的重新添加到集群中。例如我们在实例**A**上执行命令，将实例**B**从集群之中移除，但是集群**A**有可能在后续马上收到其他集群实例携带着实例**B**的**Gossip**消息，这是就需要集群黑名单机制来防止实例**B**被重新添加回集群之中。这个黑名单是使用`clusterNode.name`这个唯一标识符作为**Key**，黑名单的有效期是60秒，使用`CLUSTER_BLACKLIST_TTL`定义。

1. `clusterBlacklistCleanup`，用于清理`clusterState.nodes_black_list`之中超过`CLUSTER_BLACKLIST_TTL`有效期的节点信息。
2. `clusterBlacklistAddNode`，用于将`node`这个节点对象加入到`clusterState.nodes_black_list`黑名单之中。
3. `clusterBlacklistExists`，用于查找`nodeid`这个唯一标识符对应的节点是否在`clusterState.nodes_black_list`黑名单之中。

### 集群纪元接口

正如前面所介绍的，在**Redis**集群之中有两个集群纪元概念，分别是节点的配置纪元`clusterNode.configEpoch`以及集群的当前纪元`clusterState.currentEpoch`。这里所谓的纪元，是用来标记每一个事件版本，例如故障转移，这样可以帮助我们理解集群之中事件发生的顺序，也能保证集群状态的一致性；另外当集群的状态发生变化的时候，也可以根据纪元来解决冲突这在分布式系统之中是尤为重要的概念。

在集群之中，如果一个分片实例上的集群当前纪元`clusterState.currentEpoch`发生变化，或者某个节点的配置纪元`clusterNode.configEpoch`发生变化，会通过消息的形式通知到集群之中的其他分片实例，用以实现数据的更新。

```c
uint64_t clusterGetMaxEpoch(void);
int clusterBumpConfigEpochWithoutConsensus(void);
void clusterHandleConfigEpochCollision(clusterNode *sender);
```

函数`clusterGetMaxEpoch`用于从所有节点对象的配置纪元`clusterNode.configEpoch`以及集群状态中的当前纪元`clusterState.currentEpoch`之中的最大值。

函数`clusterBumpConfigEpochWithoutConsensus`这个函数从名字上可以理解为“在没有共识的情况下集群配置纪元”，这个函数会在当前的实例`myself->configEpoch`为0，或者不是集群中的最高的纪元值，那么会执行如下的流程：

1. 将实例记录的当前纪元`clusterState.currentEpoch`执行加1操作。
2. 将系统的当前纪元`clusterState.currentEpoch`赋值给当前节点对象`myself->configEpoch`的配置纪元上。
3. 设置`CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_FSYNC_CONFIG`标记，在发送带有新的配置数据包之前，将纪元配置更新到磁盘的配置文件上。

如果当前实例对应的节点`clusterNode`是一个**Master**节点，并且从另外一个**Master**节点收到一个与自己相同的配置纪元`clusterNode.configEpoch`时，会调用函数`clusterHandleConfigEpochCollision`。这个函数的处理逻辑，如果`sender`的唯一标识符在字典顺序上大于当前节点`myself`的唯一标识符，那么执行配置纪元冲突解决逻辑，更新集群系统当前配置`clusterState.currentEpoch`执行加1操作， 并将之更新到当前节点的配置纪元`clusterNode.configEpoch`字段上，并将配置信息立即存储到磁盘上。

## 哈希槽slot操作

如前面概述之中所记载的，用户存储的键会通过算法映射到一个哈希槽`slot`之中。一个哈希槽`slot`对应了若干个用户的键值对数据，并且归属于集群之中的某一个**Master**实例进行存储与管理。那么在本段内容中，将对集群哈希槽`slot`的相关业务逻辑进行一个详细的梳理。

### 键空间映射

```c
unsigned int keyHashSlot(char *key, int keylen);
```

这个函数用于计算一个给定的`key`属于`[0,16383]`区间内的哪个`slot`索引。这里有一个规则，如果`key`的字符串之中有`{}`标记，那么会按照`{}`之中的字符串来计算其对应的`slot`索引。使用这个规则，我们可以强制某些`key`被分配到同一个`slot`之中，这样我们便可以在针对多个`key`来执行事务操作或者调用脚本操作。

通过前面的介绍，我们知道在`clusterState`这个数据结构之中还定义了两个用于统计`slot`以及`key`之间映射关系以及数量信息的数据字段`clusterState.slots_to_keys`以及`clusterState.slots_keys_count`。当我们通过客户端的查询命令向**Redis**的键空间添加或者删除一个`key`的时候，会向相应的数据更新到`clusterState.slots_to_keys`以及`clusterState.slots_keys_count`这两个统计数据之中。

```c
void slotToKeyAdd(robj *key);
void slotToKeyDel(robj *key);
void slotToKeyUpdateKey(robj *key, int add);
```

上述三个函数定义在*src/db.c*文件之中，用于更新`slot`统计数据，以添加一个`key`为例：

```c
void dbAdd(redisDb *db, robj *key, robj *val) {
    ...
    if (server.cluster_enabled) slotToKeyAdd(key);
}
```

当我们通过调用`dbAdd`接口向**Redis**数据库键空间之中添加一个`key`的时候，如果当前**Redis**进程是集群模式，那么会调用`slotToKeyAdd`将`key`加入`slot`的统计之中；而从统计数据之中移除一个`key`的时候，与添加操作类似，通过调用`slotTokeyDel`来达成移除的目的。而这两个函数本质上都是通过`slotToKeyUpdateKey`来实现的，下面我们可以看一下这个函数的实现细节：

```c
void slotToKeyUpdateKey(robj *key, int add)
{
    unsigned int hashslot = keyHashSlot(key->ptr,sdslen(key->ptr));
    unsigned char buf[64];
    unsigned char *indexed = buf;
    size_t keylen = sdslen(key->ptr);
    server.cluster->slots_keys_count[hashslot] += add ? 1 : -1;
    ...
    indexed[0] = (hashslot >> 8) & 0xff;
    indexed[1] = hashslot & 0xff;
    memcpy(indexed+2,key->ptr,keylen);
	if (add) {
		raxInsert(server.cluster->slots_to_keys,indexed,keylen+2,NULL,NULL);
	} else {
		raxRemove(server.cluster->slots_to_keys,indexed,keylen+2,NULL);
	}
    ...
}
```

这里我们可以看到，当我们试图更新统计数据的时候，首先会更新`clusterState.slots_keys_count`这个统计计数；然后会使用这个`key`对应的`slot`以及这个`key`本身组成一个类似形如`slot:key`的基数树索引，并将这个索引插入`clusterState.slots_to_keys`或者从中删除。如此一来便可以利用基数树这种前缀树的特定，快速定位给定`slot`下的`key`。

```c
unsigned int getKeysInSlot(unsigned int hashslot, robj **keys, unsigned int count);
```

使用上面这个`getKyesInSlot`函数，便可以从`clusterState.slots_to_keys`这个基数树之中获取到给定`slot`下所属的`key`。

### Slot数据操作

在集群启动初始化之后，新的分片实例是没有关联任何哈希槽`slot`数据的，这也就意味这个此时是无法对外提个查询服务的，这就需要集群系统管理员将一部分的哈希槽`slot`数据分配给指定的的分配实例，具体的操作可以通过下面这个命令进行`slot`分配：

```bash
CLUSTER ADDSLOTS slot [slot ...]
```

这条命令会将指定的`slot`分配给执行该命令的集群分片节点。不过需要注意的是这个待分配的`slot`必须在该集群分片实例上没有被分配给其他节点，也就是`clusterState.slots`数组之中对应位置的节点指针必须为空，否则分配失败。在一个节点上分配好一个`slot`后，后续节点便会通过**PING**、**PONG**以及**UPDATE**消息将这个信息通知给集群中其他所有的节点，完成`slot`配置信息在整个集群内的更新。这就意味着一旦一个`slot`通过**ADDSLOTS**命令被分配出去以后便不能通过这个命令分配给其他节点。

与之相反的，我们通过下面的命令将一个`slot`从一个节点上移除：

```bash
CLUSTER DELSLOTS slot [slot ...]
```

以上是在集群初始化阶段对于集群之中`slot`的分配操作，接下来我们可以讨论一下在集群运行过程当中的`slot`操作，在这种操作下，某一个`slot`会从一个节点转移分配给另外一个节点，而这个`slot`下的所有的键值数据也会从源节点转移给目标节点。不过这类操作通常是需要有集群管理源或者专门处理迁移任务的脚本或工具来执行。其基本流程为：

1. 向需要进行迁移的目标节点发送命令`CLUSTER SETSLOT slot IMPORTING node-id`，告知其将要从唯一标识符为`node-id`的源节点之中将`slot`下所有数据迁移过来。这一步操作会将`clusterState.importing_slots_from[slot]`的指针位赋值给`node-id`对应的源节点指针。
2. 向`slot`当前所属的源节点发送命令`CLUSTER SETSLOT slot MIGRATING node-id`，告知其将要把属于自己管理的`slot`下的所有数据迁移到`node-id`所指定的节点之中。这一步操作会将`clusterState.migrating_slots_to[slot]`的指针位赋值给`node-id`对应的目标节点指针。
3. 向源节点发送命令`CLUSTER GETKEYSINSLOT slot count`，获取源节点下给定的`slot`下数量为`count`的键。
4. 对于步骤3之中返回的每一个键，向源节点发送命令`MIGRATE host port key 0 timeout`以原子的方式将键迁移到目标节点。
5. 在重复完步骤3、4将`slot`下所有的键从源节点全部迁移给目标节点之后，向迁移的源节点以及目标节点发送命令`CLUSTER SETSLOT slot NODE node-id`，明确声明将这个`slot`指派给`node-id`所指定的目标节点。
6. 另外也可以将向集群之中除了源节点以及目标节点之外的其他**Master**节点发送命令`CLUSTER SETSLOT slot NODE node-id`，当然这一步骤不是必须要发送的，因为源节点以及目标节点会通过自己发送的**PING**消息将`slot`所有权的变化更新到整个集群之中。

我们首先看一下`CLUSTER GETKEYSINSLOT`这个命令，这个命令便是通过前面介绍的`getKeysInSlot`这个函数从`clusterState.slots_to_keys`这个基数树种查找到满足条件前缀的键的集合，并返回给命令的发起者。

下面我们来介绍一个在迁移数据时使用的一个重要的命令**MIGRATE**，不过在介绍**MIGRATE**命令之前，我们现在看一下**Redis**代码之中提供的**迁移缓存套接字**机制，

```c
typedef struct migrateCachedSocket {
    int fd;
    long last_dbid;
    time_t last_use_time;
} migrateCachedSocket;

struct redisServer {
    ...
    dict *migrate_cached_sockets;
    ...
};
```

当我们在源分片实例上执行**MIGRATE**命令向目标实例分片上迁移数据的时候，实际上是以源分片实例作为用户`client`向目标分片实例发起**RESTORE-ASKING**命令转移数据，这就需要源分片实例作为一个客户端`client`向目标分片实例发起网络连接。为了防止频繁发起网络连接或者永久保持网络连接所带来的开销，**Redis**集群设计了**迁移缓存套接字**的机制，缓存了最近正在使用的从源分片实例到目标分片实例的网络连接，并定期将不活跃的连接进行释放。需要注意的时，这里说的连接与前面介绍的集群分片实例之间通过集群总线端口建立的用于交互`clusterMsg`消息的连接不同，其本质上就是一个发起查询命令的用户与接收执行查询命令的**Redis**服务器之间的网络连接。在`migrateCachedSocket`这个数据结构之中：

1. `migrateCachedSocket.fd`，用于记录这个网络连接的套接字文件描述符。
2. `migrateCachedSocket.last_dbid`，用于缓存上一次在这个网络连接上发起的查询命令对应的键空间编号。
3. `migrateCachedSocket.last_use_time`，缓存上一次在这个网络连接上的查询命令的发起时间，用于定期释放不活跃的网络连接。

以上所有的`migrateCachedSocket`缓存对象都存储在`redisServer.migrate_cached_sockets`这个哈希表之中，以该链接对应的网络地址作为键。

另外还提供了若干操作`migrateCacheSocket`的函数接口：

```c
migrateCachedSocket* migrateGetSocket(client *c, robj *host, robj *port, long timeout);
void migrateCloseSocket(robj *host, robj *port);
void migrateCloseTimedoutSockets(void);
```

`migrateGetSocket`这个函数会在`redisServer.migrate_cached_sockets`这个哈希表之中搜索给定网络地址信息所对应的`migrateCachedSocket`对象，如果不存在那么则会向这个地址信息发起连接，用返回的套接字文件描述符创建一个新的`migrateCachedSocket`对象，并将其加入到`redisServer.migrate_cached_sockets`哈希表之中进行缓存。

调用`migrateCloseSocket`这个函数则会关闭对应的网络连接，并释放`migrateCloseSocket`缓存对象，而在系统的心跳函数之中，也会定期调用`migrateCloseTimedoutSockets`来遍历`redisServer.migrate_cached_sockets`，将不活跃的连接对象关闭并删除。

下面我们来正式看一下**MIGRATE**迁移命令的实现细节，先看一下这个这个迁移命令的两种格式：

```b
MIGRATE host port key dbid timeout [COPY | REPLACE | AUTH password]
```

使用这个格式的命令可以将一个给定的`key`迁移到`host`以及`port`标记的另一个**Redis**上的编号为`dbid`的键空间之中，由于在集群之中都使用0号键空间，因此在集群`slot`迁移的过程之中，这里应该赋值为0；因为迁移操作是一个阻塞的同步操作，因此还要用过`timeout`设定一个阻塞操作的超时时间；如果目标**Redis**需要密码认证的话，可以通过`AUTH password`来设置认证密码；另外`COPY`以及`REPLACE`两个参数用于标记迁移的行为，`COPY`标记会保留在源**Redis**上的数据，`REPLACE`标记会在目标**Redis**上也存在相同数据的时候，覆盖目标**Redis**上的数据，这两个标记是可选，默认的情况下的迁移行为是如果目标**Redis**上存在相同数据会导致迁移失败，如果迁移成功会在源**Redis**上将对应的数据删除。

```bash
MIGRATE host port "" dbid timeout [COPY | REPLACE | AUTH password] KEYS key1 key2 ... keyN
```

而上面这种格式则会将多个给定的键数据迁移到另外一个**Redis**上。

现在我们来看看这个命令的具体实现：

```c
void migrateCommand(client *c);
```

这个函数会执行如下的逻辑，对数据进行迁移：

1. 从查询命令之中，解析出是否携带有`COPY`、`REPLACE`以及`AUTH`标记，并解析出相关的参数。
2. 调用键空间的`lookupKeyRead`接口，收集到命令参数之中所指定的一个或者多个键值对。
3. 调用`migrateGetSocket`接口，获取与`host`以及`port`所关联的`migrateCachedSocket`套接字缓存对象。
4. 调用`rioInitWithBuffer`函数创建并初始化一个用于存储发送给目标分片实例的查询命令的**I/O**对象`cmd`。
5. 如果携带有`AUTH`标记，那么首先向`cmd`命令之中添加一条**AUTH**命令。
6. 如果操作的键空间编号`dbid`与`migrateCachedSocket.last_dbid`所缓存的编号不一致，那么向`cmd`之中追加一条**SELECT**命令。
7. 对于收集到的每一个键值对，都向`cmd`之中追加一个**RESTORE-ASKING**命令，这个命令包含的参数有：
   1. `key`，需要被迁移的键。
   2. `ttl`，这个键的生命期。
   3. `serialized-value`，被迁移的键值对的序列化数据。
   4. `REPLACE`，标记被迁移的键如果在目标分片实例之中存在，是否会对已有数据进行覆盖。
8. 调用`syncWrite`同步**I/O**操作将`cmd`之中的命令发送给目标分片实例，这个发送过程是阻塞的，这也就意味着在**MIGRATE**命令结束之前，这个源分片实例将会被阻塞，无法处理其他用户的查询请求，这也就保证了数据的一致性。
9. 在完成了`cmd`命令数据的同步发送之后，会调用`syncReadLine`这个同步**I/O**接口来等待并接收目标分片实例发送回来的`cmd`命令数据的执行结果。针对**RESTORE-ASKING**命令的返回结果，如果**MIGRATE**命令没有携带`copy`标记，那么会将这个已经迁移出去的键从源分片实例之中删除。

与之对应的，在迁移的目标分片实例一侧，需要使用**RESTORE-ASKING**命令来接收数据，其实**Redis**还提供了另外一个与**RESTORE-ASKING**命令类似的**RESTORE**命令，我们先来看一下这两个命令在`redisCommandTable`之中的定义：

```c
struct redisCommand redisCommandTable[] = {
    {"restore",restoreCommand,-4,"wm",0,NULL,1,1,1,0,0},
    {"restore-asking",restoreCommand,-4,"wmk",0,NULL,1,1,1,0,0},
};
```

可以看出，两个命令都是通过`restoreCommand`来实现的，唯一不同的是命令标记里**RESTORE-ASKING**命令多了一个`'k'`标记，这个标记的含义是在**Redis**集群模式下执行这个命令时，会对该命令进行隐式的**ASKING**操作。如果待操作的键对应的`slot`如果被标记为`importing`状态，那么这个命令也会被接受并执行。

```bash
RESTORE-ASKING key ttl serialized-value [REPLACE] [ABSTTL] [IDLETIME seconds] [FREQ frequency]
```

以上是**RESTORE-ASKING**命令的格式用法。

```c
void restoreCommand(client *c);
```

`restoreCommand`这个函数在实现**RESTORE-ASKING**时，会将命令之中给定的`key`以及`serialized-value`反序列化出来的数据加入到键空间之中，并按照`ttl`设置这个键值对的生命期。如果这个`key`已经存在于这个键空间之中，并且命令没有携带`REPLACE`标记的话，那么会返回错误。

```c
clusterNode *getNodeByQuery(client *c, struct redisCommand *cmd, robj **argv, int argc, int *hashslot, int *error_code);
```

最后我们来看一下迁移`slot`的最后一个步骤，就是执行**CLUSTER SETSLOT NODE**命令的过程，我们先看一下这个命令的格式：

```bash
CLUSTER SETSLOT slot NODE node-id
```

这个命令的含义是将`slot`这个槽位分配给唯一标识符为`node-id`的节点，这个命令同样也是需要系统管理员或者脚本工具来发起执行，这个命令的处理逻辑是：

1. 如果这个给定的`slot`的所有者和接收命令的节点是同一个节点，这意味着这个`slot`要被分配到另外一个节点上，如果在当前接收命令的节点上调用`countKeysInSlot`检查该`slot`下依然有键值数据，那么说明依然有数据没有转移完，命令会返回错误，防止数据丢失。如果确认当前`slot`下没有数据，并且这个`slot`声明了`migrating`导出状态，那么会将这个状态清除。
2. 如果接收命令的节点与命令分配`slot`的目标节点是同一个节点，并且在该节点上对应`slot`被声明为`importing`导入状态，那么将会执行下面两个逻辑。
   1. 调用`clusterBumpConfigEpochWithoutConsensus`这个函数检查自身的节点配置纪元`myself->configEpoch`是否是当前最大的纪元，如果不是会更新该节点上的系统当前纪元`clusterState.currentEpoch`并将其赋值给自己的节点配置集团`myself->configEpoch`。
   2. 清除这个节点的`importing`导入状态。
3. 在这个节点上，将`slot`分配给`node-id`对应的节点进行管理。

在导入的目标节点调用`clusterBumpConfigEpochWithoutConsensus`后，该节点的配置纪元将成为为当前整个集群之中的最大值。结合前面在介绍集群`CLUSTERMSG_TYPE_UPDATE`消息时所描述的，在`slot`的所属节点发生冲突的情况下，以节点配置纪元`clusterNode.configEpoch`最大的一方为准。这样在目标节点将自己的配置纪元成为集群中的最大值之后，它便可以通过`CLUSTERMSG_TYPE_UPDATE`消息，将配置信息的更新同步到整个集群之中。

### 节点的定位

由于集群之中存在多个向用户提供查询服务的分片实例，因此通常情况下用户应该与集群之中的每一个分片实例建立连接。用户发起向一个分片实例发起查询命令，其最好的情况是该查询命令操作的键所对应的`slot`恰好在接收命令的分片实例上，那么便会直接处理数据，并将结果返回给用户。但是还有很多无法直接处理的情况：

1. 查询命令操作的键所对应的`slot`并不在接收命令的分片实例上。
2. 查询命令操作的键所对应的`slot`在接收命令的分片实例上处于`migrating`导出状态，但具体的键已经迁移到了导出的目标节点上。
3. 查询命令操作的键所对应的`slot`在接收命令的分片实例上处于`importing`导入状态，但具体的键还没有迁移过来。

针对上面的这些情况，**Redis**专门给出了一个函数接口进行处理：

```c
clusterNode *getNodeByQuery(client *c, struct redisCommand *cmd, robj **argv, int argc, int *hashslot, int *error_code);
```

`getNodeByQuery`这个函数会遍历命令之中操作的每一个键，获取所对应的`slot`以及该`slot`对应的`clusterNode`：

1. 如果键对应的`slot`没有绑定的`clusterNode`，那么函数会返回失败，`error_code`被设置错误码`CLUSTER_REDIR_DOWN_UNBOUND`。
2. 如果当集群的状态不是`CLUSTER_OK`的状态，那么函数也会返回失败，`error_code`会被设置为`CLUSTER_REDIR_DOWN_STATE`。
3. 如果查询命令操作多个键，这多个键对应多个不同的`slot`，那么函数会返回失败，`error_code`被设置错误码`CLUSTER_REDIR_CROSS_SLOT`。
4. 如果查询命令操作的键对应的`slot`，在当前接收命令的实例上处于`migrating`导出状态，并且对应的键在该实例上没有找到，也就是这个键已经被转移到了**MIGRATE**操作的目标节点，那么函数会返回`slot`导出的目标节点`server.cluster->migrating_slots_to[slot]`，并且将`error_code`设置为`CLUSTER_REDIR_ASK`。
5. 如果查询命令操作的键对应的`slot`，在当前接收命令的实例上处于`importing`导入状态，在这种情况下：
   1. 如果该命令没有携带`CMD_ASKING`标记，且没有通过**ASKING**命令将发起这个命令的客户端`client`设置上`CLIENT_ASKING`状态，那么会将`error_code`设置为`CLUSTER_REDIR_MOVED`。
   2. 如果当前`client`上设置了`CLIENT_ASKING`状态，或者执行的命令是带有`CMD_ASKING`标记的状态，如果命令涉及多个键并且并非所有的键都存在于接收命令的实例上，那么会将`error_code`设置为`CLUSTER_REDIR_UNSTABLE`，通知用户等待所有的数据稳定后，重新尝试。
6. 如果查询命令操作的键对应的`slot`，在当前接收查询命令的实例上没有处于`migrating`导出状态或者`importing`导入状态，并且这个`slot`对应的节点不是当前实例本身，那么这个函数会返回`slot`对应所属的节点指针，同时`error_code`被设置为`CLUSTER_REDIR_MOVED`。

| 重定向错误码定义                       | 错误码含义                             |
| -------------------------------------- | -------------------------------------- |
| `#define CLUSTER_REDIR_NONE 0`         | 表明节点可以处理请求。                 |
| `#define CLUSTER_REDIR_CROSS_SLOT 1`   | 请求之中包含`CROSSSLOT`的键。          |
| `#define CLUSTER_REDIR_UNSTABLE 2`     | `TRYAGAIN`需要重定向。                 |
| `#define CLUSTER_REDIR_ASK 3`          | `ASK`需要重定向。                      |
| `#define CLUSTER_REDIR_MOVED 4`        | `MOVED`需要重定向。                    |
| `#define CLUSTER_REDIR_DOWN_STATE 5`   | 集群已经关闭。                         |
| `#define CLUSTER_REDIR_DOWN_UNBOUND 6` | 集群已经关闭，`slot`没有绑定分片节点。 |

在集群实例执行客户端命令时会调用`getNodeByQuery`查询命令所属的节点`clusterNode`，如果没有找到或者查询到的节点不是当前实例自己`myself`。那么会调用`clusterRedirectClient`来通知用户执行重定位的逻辑：

```c
void clusterRedirectClient(client *c, clusterNode *n, int hashslot, int error_code);
```

根据设置为`CLUSTER_REDIR_*`宏中的某个值的`error_code`，向客户端发送正确的重定向代码。

如果使用`CLUSTER_REDIR_ASK`或`CLUSTER_REDIR_MOVED`错误代码，则节点`n`不应为`NULL`，而应为我们要在重定向中提及的节点。此外，应设置`hashslot`为导致重定向的哈希槽。

## 集群消息交换

前面在介绍集群节点互联的时候提到过，节点之间通过**集群总线端口**互相连接，形成一个网络结构，各个节点之间通过这个网络结构以单播或者广播的形式发送消息，用于同步自己的状态信息、报告节点的故障信息、转发**订阅/发布**消息、更新哈希槽`slot`分配信息。

### 消息数据结构

```c
union clusterMsgData {
    /* PING, MEET and PONG */
    struct {
        /* Array of N clusterMsgDataGossip structures */
        clusterMsgDataGossip gossip[1];
    } ping;

    /* FAIL */
    struct {
        clusterMsgDataFail about;
    } fail;

    /* PUBLISH */
    struct {
        clusterMsgDataPublish msg;
    } publish;

    /* UPDATE */
    struct {
        clusterMsgDataUpdate nodecfg;
    } update;

    /* MODULE */
    struct {
        clusterMsgModule msg;
    } module;
};

typedef struct {
    char sig[4];        /* Signature "RCmb" (Redis Cluster message bus). */
    uint32_t totlen;    /* Total length of this message */
    uint16_t ver;       /* Protocol version, currently set to 1. */
    uint16_t port;      /* TCP base port number. */
    uint16_t type;      /* Message type */
    uint16_t count;     /* Only used for some kind of messages. */
    uint64_t currentEpoch;  /* The epoch accordingly to the sending node. */
    uint64_t configEpoch;   /* The config epoch if it's a master, or the last
                               epoch advertised by its master if it is a
                               slave. */
    uint64_t offset;    /* Master replication offset if node is a master or
                           processed replication offset if node is a slave. */
    char sender[CLUSTER_NAMELEN]; /* Name of the sender node */
    unsigned char myslots[CLUSTER_SLOTS/8];
    char slaveof[CLUSTER_NAMELEN];
    char myip[NET_IP_STR_LEN];    /* Sender IP, if not all zeroed. */
    char notused1[34];  /* 34 bytes reserved for future usage. */
    uint16_t cport;      /* Sender TCP cluster bus port */
    uint16_t flags;      /* Sender node flags */
    unsigned char state; /* Cluster state from the POV of the sender */
    unsigned char mflags[3]; /* Message flags: CLUSTERMSG_FLAG[012]_... */
    union clusterMsgData data;
} clusterMsg;
```

上面是消息结构体格式的定义，每个`clusterMsg`都是一个节点之间交互的消息，在这个消息之中除了`clusterMsg.data`这个载荷数据之前的数据被称为`clusterMsg`的消息头。

```c
void clusterBuildMessageHdr(clusterMsg *hdr, int type);
```

上面这个函数`clusterBuildMessageHdr`用于构建消息的消息头数据，也就是会对`clusterMsg`之中除了`clusterMsg.data`之外的字段进行赋值，下面我们来介绍一下`clusterMsg`消息的消息头中的各个字段的含义：

1. `clusterMsg.sig`，这是固定四个字符的一个签名字段，存储字符串`RCmb`，意思是*Redis集群消息总线*。
2. `clusterMsg.totlen`，这个字段记录了整个消息的长度。
3. `clusterMsg.ver`，这个字段标记了消息的版本号，当前版本为1。
4. `clusterMsg.myip`、`clusterMsg.port`、`clusterMsg.cport`，这三个字段记录了发送该消息的实例节点的IP、端口以及集群总线端口信息。
5. `clusterMsg.sender`，记录了发送该消息的实例节点的唯一标识符，也就是`myself->name`中的数据。
6. `clusterMsg.type`，标记了该条消息的类型。
7. `clusterMsg.flags`，记录当前实例节点的状态标记，对应`myself->flags`，相当于将自己的标记状态告知其他集群之中的节点。
8. `clusterMsg.state`，记录了当前实例节点的集群状态信息，对应`clusterState.state`。
9. `clusterMsg.myslots`，如果当前实例节点是一个**Master**节点，那么这里存储分配给自己的`slot`信息；否则的话，则会存储自己复制的对应的**Master**节点负责的`slot`信息。
10. `clusterMsg.slaveof`，如果当前实例节点是一个**Slave**节点，那么这里存储了这个节点所复制的对应的**Master**节点的唯一标识符；如果这个节点本身就是一个**Master**，那么这个字段将会被置0。
11. `clusterMsg.configEpoch`，如果当前实例节点是一个**Master**节点，那么这个字段存储该节点自身的配置纪元，否侧存储其对应**Master**节点的配置纪元。
12. `clusterMsg.currentEpoch`，这个字段存储了节点的当前纪元`clusterState.currentEpoch`。

#### 消息的发送

##### 点对点发送

```c
void clusterSendMessage(clusterLink *link, unsigned char *msg, size_t msglen);
```

这个函数是一个集群的底层发送数据的接口，调用这个接口会将`msg`数据写入`clusterLink.sndbuf`输出缓冲区之中等待发送；同时会解析出这个消息的类型，并按照分类更新集群状态信息之中的发送信息统计字段`clusterState.stats_bus_messages_sent`之中。

##### 广播发送

```c
void clusterBroadcastMessage(void *buf, size_t len);
```

这是一个广播消息的接口，会遍历当前节点所有已知的节点数据`clusterState.nodes`，对除了是自己`CLUSTER_NODE_MYSELF`以及处于握手状态下`CLUSTER_NODE_HANDSHAKE`的所有节点，调用`clusterSendMessage`接口来发送消息。

#### 消息的类型

现有的消息类型为：

| 消息定义                                          | 消息含义                                                     |
| :------------------------------------------------ | :----------------------------------------------------------- |
| `#define CLUSTERMSG_TYPE_PING 0`                  | 这是一条**PING**消息，会携带**Gossip**数据。                 |
| `#define CLUSTERMSG_TYPE_PONG 1`                  | 这是一条**PONG**消息，是对**PING**消息的回复，会携带**Gossip**数据 |
| `#define CLUSTERMSG_TYPE_MEET 2`                  | 用于建立与其他节点之间网络连接的握手消息                     |
| `#define CLUSTERMSG_TYPE_FAIL 3`                  | 当将某个节点标记为客观下线之后，会通过这个类型的消息告知其他节点 |
| `#define CLUSTERMSG_TYPE_PUBLISH 4`               | 辅助实现集群**发布/订阅**功能消息，当一个分片收到用户的**发布**消息时，会通过这个类型的消息转发给其他分片实例，使其他分片连接的用户也能接收**发布/订阅**消息。 |
| `#define CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST 5` | 这个消息用于在故障迁移的过程之中，**Slave**节点用于向集群之中的其他**Master**节点请求选举投票。 |
| `#define CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 6`     | 这个消息用于在故障迁移的过程之中，给**Slave**节点投票，选举其为候选的主节点。 |
| `#define CLUSTERMSG_TYPE_UPDATE 7`                | 这个消息用于通知给定节点的唯一标识符、`slot`配置信息以及这个节点的配置纪元。用于告知其他节点，自己的`slot`配置信息发生变化。 |

以上的消息类型之中`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`和`CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`这两种涉及故障转移的消息类型只有消息头数据，其余的消息类型都会通过`clusterMsg`数据结构之中的`clusterMsgData`载荷字段来携带额外的数据。接下来我们按照不同的消息类型来分别介绍一下各类消息的详细内容。

### Ping,Pong,Meet消息

之所以将这三个类型的消息放在一起来介绍，是因为这三类消息都使用`clusterMsgDataGossip`数据结构作为数据载荷来携带了一些关于其他节点的**Gossip**消息。

#### Gossip消息

这个**Gossip**消息主要是描述了一个节点的基础信息，用于在集群节点发送消息的时候，相互交流自己所知的节点状态。因此我们先来看看`clusterMsgDataGossip`数据结构的定义：

```c
typedef struct {
    char nodename[CLUSTER_NAMELEN];
    uint32_t ping_sent;
    uint32_t pong_received;
    char ip[NET_IP_STR_LEN];  /* IP address last time it was seen */
    uint16_t port;              /* base port last time it was seen */
    uint16_t cport;             /* cluster port last time it was seen */
    uint16_t flags;             /* node->flags copy */
    uint32_t notused1;
} clusterMsgDataGossip;
```

在描述了**Gossip**消息的数据结构之后，**Redis**还给出了向`clusterMsg`之中添加一条**Gossip**的接口：

```c
void clusterSetGossipEntry(clusterMsg *hdr, int i, clusterNode *n);
```

这个函数用于将节点`n`的信息填充到`hdr`的**Gossip**信息之中`clusterMsg.data.ping.gossip`字段之中。这里我们介绍一下`clusterMsgDataGossip`中的各个字段的含义：

1. `clusterMsgDataGossip.nodename`，消息所对应的节点的唯一标识符`clusterNode.name`。
2. `clusterMsgDataGossip.ping_sent`，以秒记录的节点**PING**消息的发送时间戳`clusterNode.ping_sent`。
3. `clusterMsgDataGossip.pong_received`，以秒记录的节点**PONG**消息的接收时间戳`clusterNode.pong_received`。
4. `clusterMsgDataGossip.ip`、`clusterMsgDataGossip.port`、`clusterMsgDataGossip.cport`，消息所对应节点的地址信息。
5. `clusterMsgDataGossip.flags`，消息所对应节点的状态标识`clusterNode.flags`。

**Gossip**消息会跟随着`CLUSTERMSG_TYPE_PING`和`CLUSTERMSG_TYPE_PONG`以及`CLUSTERMSG_TYPE_MEET`三种类型的消息一起发送给其他的节点。

#### 消息的收发

```c
void clusterSendPing(clusterLink *link, int type);
```

`clusterSendPing`这个函数接口用于向指定的连接上发送`CLUSTERMSG_TYPE_MEET`、`CLUSTERMSG_TYPE_PING`以及`CLUSTERMSG_TYPE_PONG`这三种类型的消息，这个函数会在发送节点自身`myself`的状态信息之外，随机携带部分的该分片实例已知其他节点的**Gossip**消息，以及当前节点视角下处于主观故障`CLUSTER_NODE_PFAIL`的节点的**Gossip**消息。如果调用这个接口是发送`CLUSTERMSG_TYPE_PING`类型的消息的话，那么会更新对应节点上**PING**消息发送时间戳`link->node->ping_sent`。

当节点接收一个`CLUSTERMSG_TYPE_MEET`以及`CLUSTERMSG_TYPE_PING`消息时，会立即调用`clusterSendPing`接口发送一条`CLUSTERMSG_TYPE_PONG`消息作为答复：

```c
int clusterProcessPacket(clusterLink *link) {
    ...
	if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_MEET) {
        ...
        clusterSendPing(link,CLUSTERMSG_TYPE_PONG);
    }
    ...
}
```

当节点接收到一个`CLUSTERMSG_TYPE_PONG`消息时，则会按照下面代码的逻辑进行处理：

```c
int clusterProcessPacket(clusterLink *link) {
    ...
	if (link->node && type == CLUSTERMSG_TYPE_PONG) {
		link->node->pong_received = mstime();
		link->node->ping_sent = 0;
		if (nodeTimedOut(link->node)) {
			link->node->flags &= ~CLUSTER_NODE_PFAIL;
			clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE);
		} else if (nodeFailed(link->node)) {
			clearNodeFailureIfNeeded(link->node);
		}
    }
    ...
}
```

当一个分片实例在集群心跳`clusterCron`中遍历自己已知的集群节点`clusterState.nodes`，对于没有关联`clusterLink`连接对象的节点会主动向其发起连接，并根据该节点的标记状态，调用`clusterSendPing`接口向其发送`CLUSTERMSG_TYPE_MEET`或者`CLUSTERMSG_TYPE_PING`消息：

```c
void clusterCron(void) {
    ...
    while((de = dictNext(di)) != NULL) {
        clusterNode *node = dictGetVal(de);
        ...
        if (node->link == NULL) {
            fd = anetTcpNonBlockBindConnect(server.neterr, node->ip, node->cport, NET_FIRST_BIND_ADDR);
            link = createClusterLink(node);
            link->fd = fd;
            node->link = link;
            aeCreateFileEvent(server.el,link->fd,AE_READABLE, clusterReadHandler,link);
            clusterSendPing(link, node->flags & CLUSTER_NODE_MEET ? CLUSTERMSG_TYPE_MEET : CLUSTERMSG_TYPE_PING);
        }
    }
}
```

同样时在分片实例的集群心跳`clusterCron`中，每10次心跳便会随机选择若干个节点，并从中选择`clusterNode.pong_received`最小的那个节点，调用`clusterSendPing`接口向其发送一条`CLUSTERMSG_TYPE_PING`消息。

```c
void clusterCron(void) {
    ...
    if (!(iteration % 10)) {
        ...
        if (min_pong_node) {
            clusterSendPing(min_pong_node->link, CLUSTERMSG_TYPE_PING);
        }
    }
}
```

最后还是在分片实例的集群心跳`clusterCron`中，会遍历所有已知的节点`clusterState.nodes`，向满足条件的节点调用`clusterSendPing`函数发送`CLUSTERMSG_TYPE_PING`消息，发送的条件包含下面两种：

1. 从未向该节点发送过`CLUSTERMSG_TYPE_PING`并且长时间没有再次接收到该节点的`CLUSTERMSG_TYPE_PONG`消息，那么便会主动向其发送一条`CLUSTERMSG_TYPE_PING`消息。
2. 如果当前分片实例是一个**Master**节点，当前遍历节点是复制其数据的**Slave**节点并且请求手动故障转移，那么也会持续地向其发送`CLUSTERMSG_TYPE_PING`消息。

```c
void clusterCron(void) {
    while((de = dictNext(di)) != NULL) {
        clusterNode *node = dictGetVal(de);
        ...
        if (node->link && node->ping_sent == 0 &&
            (now - node->pong_received) > server.cluster_node_timeout/2)
        {
            clusterSendPing(node->link, CLUSTERMSG_TYPE_PING);
            continue;
        }
        if (server.cluster->mf_end && nodeIsMaster(myself) && 
            server.cluster->mf_slave == node && node->link)
        {
            clusterSendPing(node->link, CLUSTERMSG_TYPE_PING);
            continue;
        }
    }
}
```

除此之外，**Redis**还提供了一个广播`CLUSTERMSG_TYPE_PONG`消息的函数

```c
#define CLUSTER_BROADCAST_ALL 0
#define CLUSTER_BROADCAST_LOCAL_SLAVES 1
void clusterBroadcastPong(int target);
```

这个函数会通过区分`target`参数是`CLUSTER_BROADCAST_ALL`还是`CLUSTER_BROADCAST_LOCAL_SLAVES`来将`CLUSTERMSG_TYPE_PONG`消息广播给所有的已知节点或者仅仅广播给自己对应的所有**Slave**节点。

### Publish消息

下面我们介绍一下集群之中针对**发布/订阅**功能的支持，按照**发布/订阅**功能的设计机制，一个用户在某一个频道上发布的消息应该会被所有其他订阅该频道的用户所接收。将这种机制放置到集群环境之中，就需要考虑到这样一种情况，不同的用户可能连接在集群中的不同分片实例上，一个用户在一个分片实例上使用**发布/订阅**功能发布消息，而连接在其他分片实例上的用户相应地也要能接收到消息，这就需要集群将一个分片实例上接收到的消息转发给集群之中的其他分片上。

```c
typedef struct {
    uint32_t channel_len;
    uint32_t message_len;
    unsigned char bulk_data[8]; /* 8 bytes just as placeholder. */
} clusterMsgDataPublish;

void clusterSendPublish(clusterLink *link, robj *channel, robj *message);
void clusterPropagatePublish(robj *channel, robj *message);
```

我们回顾一下当用户执行**PUBLISH**命令时的实现代码：

```c
void publishCommand(client *c) {
    int receivers = pubsubPublishMessage(c->argv[1],c->argv[2]);
    if (server.cluster_enabled)
        clusterPropagatePublish(c->argv[1],c->argv[2]);
    else
        forceCommandPropagation(c,PROPAGATE_REPL);
    addReplyLongLong(c,receivers);
}
```

这里我们可以看到，对于普通的**Redis**服务器那么会选择`forceCommandPropagation`接口，如果当前是一个集群实例则会选择`clusterPropagatePublish`接口进入，而这个函数本质上是调用`clusterSendPublish`函数实现的。

函数`clusterSendPublish`，会首先调用`clusterBuildMessageHdr`来构建一个消息类型为`CLUSTERMSG_TYPE_PUBLISH`的消息`clusterMsg`，并将频道数据`channel`以及消息`message`嵌入到`clusterMsg.publish.msg`这个`clusterMsgDataPublish`数据结构之中，`channel`以及`message`都会被存储到`clusterMsgDataPublish.bulk_data`之中；最后可以通过`link`参数是否为空来实现点对点发送消息以及广播消息给所有其他节点。

当其他节点接受到`CLUSTERMSG_TYPE_PUBLISH`后，会从`clusterMsg.publish.msg.bulk_data`字段之中解析出`channel`以及`message`数据，然后调用`pubsubPublishMessage`函数将消息`message`广播给所以连接该分片实例节点并订阅了`channel`频道的用户：

```c
int clusterProcessPacket(clusterLink *link) {
    ...
    if (type == CLUSTERMSG_TYPE_PUBLISH) {
        robj *channel, *message;
        ...
        if (dictSize(server.pubsub_channels) || listLength(server.pubsub_patterns))
        {
            ...
            pubsubPublishMessage(channel,message);
            ...
        }
    }
}
```

### Fail消息

```c
typedef struct {
    char nodename[CLUSTER_NAMELEN];
} clusterMsgDataFail;

void clusterSendFail(char *nodename)
```

前面我们介绍了，集群节点在处理上报节点故障报告的**Gossip**消息时，会调用`markNodeAsFailingIfNeeded`函数来判断是否需要将对应的节点从主观故障`CLUSTER_NODE_PFAIL`切换成客观故障`CLUSTER_NODE_FAIL`的状态，一旦确定该故障节点被转换为`CLUSTER_NODE_FAIL`，那么便会将这个事件包装成一个`CLUSTERMSG_TYPE_FAIL`消息，将故障节点的唯一标识符存储在`clusterMsg.data.fail.about`这个`clusterMsgDataFail`字段之中，并通过`clusterSendFail`广播给所有的集群节点。

这个操作相当集群中某个节点一旦察觉导有一个节点出现故障，那么会立即将这个事件告知给集群之中的所有节点，其他的集群分片节点在接收到一个`CLUSTERMSG_TYPE_FAIL`消息时，也会消息中所指定的节点设置为客观故障状态：

```c
int clusterProcessPacket(clusterLink *link) {
    ...
    if (type == CLUSTERMSG_TYPE_FAIL) {
        clusterNode *failing;
        if (sender) {
            failing = clusterLookupNode(hdr->data.fail.about.nodename);
            if (failing && !(failing->flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_MYSELF)))
            {
                failing->flags |= CLUSTER_NODE_FAIL;
                failing->fail_time = mstime();
                failing->flags &= ~CLUSTER_NODE_PFAIL;
            }
        }
    }
}
```

1. 设置更新该节点接收**PONG**消息的时间戳`clusterNode.pong_received`，并将向该节点发送**PING**消息的时间戳`clusterNode.ping_sent`清零。
2. 如果该节点处于**主观下线**状态，那么将该故障状态清空，并将该状态的更新存入磁盘的配置文件中。
3. 如果该节点处于**客观下线**状态，那么同样清理该故障状态。

### Update消息

```c
typedef struct {
    uint64_t configEpoch; /* Config epoch of the specified instance. */
    char nodename[CLUSTER_NAMELEN]; /* Name of the slots owner. */
    unsigned char slots[CLUSTER_SLOTS/8]; /* Slots bitmap. */
} clusterMsgDataUpdate;

void clusterSendUpdate(clusterLink *link, clusterNode *node);
void clusterUpdateSlotsConfigWith(clusterNode *sender, uint64_t senderConfigEpoch, unsigned char *slots);
```

**Redis**集群会使用`clusterSendUpdate`函数构造`CLUSTERMSG_TYPE_UPDATE`类型的`clusterMsg`消息向`link`对应的分片实例发送`node`的配置更新信息，其中包括`node`节点的唯一标识符、`slot`配置、以及配置纪元信息，这些信息会被存储到`clusterMsg.data.update.nodecfg`这个`clusterMsgDataUpdate`结构体之中。

下面我们来看一下`clusterUpdateSlotsConfigWith`这个函数，当一个集群分片实例通过**PING**、**PONG**、**MEET**和**UPDATE**消息接收到其他**Master**实例的配置时会调用这个函数。从上面的消息之中，会接收到一个代表发送方的节点对象`sender`，这个节点的发送来配置纪元`senderConfigEpoch`以及在这个配置纪元来声明的`slots`集合。

`clusterUpdateSlotsConfigWith`这个函数会遍历`sender`发送来其所声明拥有的`slots`合集，会按照如下的逻辑进行更新:

1. 如果实例本地记录的对应`slot`没有分配负责的节点`clusterState.slots[j] == NULL`，那么便会向这个`slot`槽位分配给sender负责。
2. 如果实例本地记录的对应`slot`有负责的节点，但是该节点的配置纪元比`senderConfigEpoch`要小`clusterState.slots[j]->configEpoch < senderConfigEpoch`，也就意味着本地该`slot`的配置更旧，那么也会将该`slot`槽位分配给`sender`负责。

关于**Redis**集群是如何更新槽位信息的，我们可以看一下下面的代码片段：

```c
int clusterProcessPacket(clusterLink *link) {
    ...
    if (sender && !nodeInHandshake(sender)) {
        if (senderCurrentEpoch > server.cluster->currentEpoch)
            server.cluster->currentEpoch = senderCurrentEpoch;
        if (senderConfigEpoch > sender->configEpoch) {
            sender->configEpoch = senderConfigEpoch;
        }
        ...
    }
    ...
    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
        type == CLUSTERMSG_TYPE_MEET)
    {
        ...
        clusterNode *sender_master = NULL;
        int dirty_slots = 0;
        sender_master = nodeIsMaster(sender) ? sender : sender->slaveof;
        dirty_slots = memcmp(sender_master->slots, hdr->myslots,sizeof(hdr->myslots)) != 0;
        if (sender && nodeIsMaster(sender) && dirty_slots)
            clusterUpdateSlotsConfigWith(sender,senderConfigEpoch,hdr->myslots);
        if (sender && dirty_slots) {
            for (j = 0; j < CLUSTER_SLOTS; j++) {
                if (bitmapTestBit(hdr->myslots,j)) {
                    if (server.cluster->slots[j] == sender ||
                        server.cluster->slots[j] == NULL) continue;
                    if (server.cluster->slots[j]->configEpoch > senderConfigEpoch)
                    {
                        clusterSendUpdate(sender->link, server.cluster->slots[j]);
                        break;
                    }
                }
            }
        }
    }
}
```

前面我们通过对`clusterMsg`数据结构的介绍可以得知，每次消息的发送方在发送`clusterMsg`的时候，都会在其中携带有方法方自身的配置纪元以及发送方缓存的集群当前纪元，分别存储在`clusterMsg.configEpoch`以及`clusterMsg.currentEpoch`两个字段中。从上面的代码段中便可知，分配实例在接收到`sender`发来的`clusterMsg`消息后，首先会尝试将对应节点的配置纪元`sender->configEpoch`以及系统的当前纪元`clusterState.currentEpoch`更新到最新。接下来在**PING**、**PONG**以及**MEET**消息的处理逻辑之中，会判断当前分片实例中缓存的`slot`集合`sender_master->slots`与发送方在消息之中声明的其拥有的集合`clusterMsg->myslots`是否一致。如果不一致，也就说明集群配置信息发生了变化，需要进行更新。在这里不一致的情况可以被划分为两种：

1. 发送方拥有更新的配置纪元，而当前分片实例中缓存的配置纪元较旧，那么便会调用`clusterUpdateSlotsConfigWith`使用`clusterMsg.myslots`中所声明的`slot`集合来更新当前实例本地缓存的`slot`集合配置。
2. 发送方的配置纪元较旧，而当前实例中相应`slot`所属节点的配置纪元更新`clusterNode.configEpoch`更新，那么说明发送方的配置需要更新。在这种情况下就会如上面代码所示，通过调用`clusterSendUpdate`函数，将拥有更新配置纪元的`slot`所对应的节点配置信息发送回给消息的发送方`sender`。

作为**UPDATE**消息的接收方，当集群分片实例通过`clusterProcessPacket`函数接收到`CLUSTERMSG_TYPE_UPDATE`消息后，便会从`clusterMsg.data.update`字段之中解析出需要更新的`clusterNode`节点的唯一标识符与对应上报来的配置纪元，通过`clusterUpdatSlotsConfigWith`来尝试更新对应节点的配置信息。

## 集群定时任务

### 状态评估

```c
void clusterUpdateState(void);
```

首先我们来介绍一下集群状态评估函数`clusterUpdateState`，在`clusterState`结构体之中定义了一个集群状态`clusterState.state`，这个字段用于标记实例的集群状态，如果这个字段被设置为`CLUSTER_OK`，则说明实例的集群状态看起来是正常的，可以对外提供服务；反之如果被设置为`CLUSTER_FAIL`，则说明实例的集群状态是故障状态，无法对外提供服务。而`clusterUpdateState`这个函数就是用于评估设置`clusterState.state`这个状态字段的。 如果实例处于如下的两个状态之一，便会评估当前的集群状态为`CLUSTER_FAIL`：

1. 如果设置`cluster-require-full-coverage`，那么要求每一个哈希槽`slot`都分配集群节点`clusterNode`，并且该节点没有处于客观下线状态`CLUSTER_NODE_FAIL`，否则会将集群状态设置为`CLUSTER_FAIL`。
2. 遍历所有的集群节点`clusterState.nodes`，更新负责处理至少一个哈希槽`slot`的**Master**节点个数`clusterState.size`；并更新这些**Master**节点之中状态正常的可达**Master**节点的数量`reachable_masters`。如果可达的主节点数量`reachable_masters`小于集群主节点数量一半加一`(server.cluster->size / 2) + 1`的话，那么该实例的集群状态将被设置为`CLUSTER_FAIL`。

除此之外，都会认为当前集群状态为`CLUSTER_OK`，但是这里面还有一个额外的延迟重入机制，也就是实例的集群状态如果被认定为`CLUSTER_FAIL`之后，那么在`clusterUpdateState`函数被重新评估为`CLUSTER_OK`的时候，并不会立即将`clusterState.state`设置为`CLUSTER_OK`状态，而是要等待一个延迟时间后，再次调用`clusterUpdateState`才会更新`clusterState.state`状态数据为`CLUSTER_OK`的状态。

### 集群心跳

```c
void clusterCron(void);
```

在**Redis**的心跳函数之中，会每100次`tick`便触发一次`cluserCron`接口，这也就意味着，按照默认的配置，每一个**Redis**集群中的分片实例每秒中会触发10次心跳逻辑。

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
	...
    run_with_period(100) {
        if (server.cluster_enabled) clusterCron();
    }
    ...
}
```

##### 校验announce-ip数据

这部分代码逻辑会试图将`myself->ip`与`cluster-announce-ip`两项数据进行同步，`cluster-announce-ip`可以通过**CONFIG SET**命令在运行之中设置，通过定期在心跳函数之中进行检查将这个数据同步到`myself->ip`数据上。

##### 更新节点连接信息

这部分对检查是否有没有建立网络连接的集群节点`clusterNode`，如果存在这样的节点那么向其主动建立网络连接，同时这里会更新一些统计信息，这个信息可以用于在代码的其他部分做出更好的决策。

在心跳函数这部分代码之中，首先会遍历所有的已知节点列表，针对其中的每一个`clusterNode`节点对象执行如下逻辑：

1. 如果节点带有`CLUSTER_NODE_PFAIL`标记，那么更新`clusterState.stats_pfail_nodes`这个统计主观故障节点的计数。
2. 如果节点处于握手状态时间过长，那么按照超时处理，删除并释放这个节点。
3. 对于`clusterNode.link`为空的节点，如前面所介绍的，向对应节点主动发起连接，创建对应的`clusterLink`对象并于这个节点进行关联，同时发送一条**PING**或者**MEET**消息给节点对应的分片实例。

##### 定期向其他节点发送PING消息

每十次心跳也就是每1秒钟会触发一次发送**PING**消息的逻辑，也是如前面所介绍的，随机选取5个节点对象，从中选择最久没有收到对应分片实例**PONG**回复的节点，调用`clusterSendPing`函数向其发送一条**PING**消息。

##### 刷新节点状态

在这部分逻辑之中，心跳函数`clusterCron`会再次重新遍历所有的已知节点列表，检测是否需要将某些内容标记为失败状态。这部分逻辑将主要执行如下的逻辑：

1. 如果当前实例是一个**Slave**节点，那么会在遍历之中刷新孤立的**Master**节点的数量；同时还会更新各个**Master**节点的非故障的**Slave**节点数量的最大值；如果遍历到的节点是自己对应的**Master**节点，那么也会更新该节点的非故障**Slave**节点的数量。
2. 如果与一个节点的网络连接已经建立，当前实例已经向该节点对应的分片实例发送过**PING**消息（也就是`clusterNode.ping_send`不为0），但是没有收到该分片返回的**PONG**消息（`node->pong_received < node->ping_sent`），并且距离发送**PING**消息到现在已经超过了集群超时时间阈值的一半（`now - node->ping_sent > server.cluster_node_timeout/2`），那么就认为与这个节点对应的分片实例的网络连接出现问题，会调用`freeClusterLink`将这个连接关闭。
3. 对于一个节点，如果当前实例从该节点对应的分片实例接收过**PONG**消息，并且接收**PONG**的时间距离现在已经超过了集群超时时间阈值的一半（`(now - node->pong_received) > server.cluster_node_timeout/2`），但是还没有向其发送**PING**消息，那么会调用`clusterSendPing`函数向对应的实例立即发送一条**PING**消息。这个机制可以确保所有的节点都能被及时地**PING**到，防止延时过大。
4. 前面已经介绍过，当前实例在收到节点对应分片实例的**PONG**消息时，会清空节点的**PING**消息发送时间戳，因此如果`clusterNode.ping_sent`时间戳不为0的话，我们便可以使用这个时间戳与当前时间的差值来判断是否尝试加没有接收到**PING**消息返回的**PONG**（使用这个判断`now - node->ping_sent > server.cluster_node_timeout`），如果确认超时，那么将该节点设置为**主观下线**状态。

##### 开启从节点对主节点的复制

心跳函数会检查，如果当前的实例是一个**Slave**节点，其复制功能处于关闭状态，但是其对应的**Master**节点看起来是活跃的并且我们知道它的地址，那么会调用**主从复制**之中的`replicationSetMaster`来开启它的复制机制。

```c
void clusterCron(void)
{
    ...
    if (nodeIsSlave(myself) &&
        server.masterhost == NULL &&
        myself->slaveof &&
        nodeHasAddr(myself->slaveof))
    {
        replicationSetMaster(myself->slaveof->ip, myself->slaveof->port);
    }
    ...
}
```

##### 定期检测故障迁移

如果当前实例是一个**Slave**节点，那么会在心跳函数之中通过调用故障迁移的一系列接口定期检测与处理故障迁移的相关逻辑。

##### 刷新集群状态

如果实例的当前集群状态`clusterState.state`处于故障状态`CLUSTER_FAIL`，并且在前面的心跳逻辑之中状态信息发生了变化，那么会调用`clusterUpdateState`函数来尝试更新实例的当前集群状态。

### 返回Sleep前的回调

```c
void clusterDoBeforeSleep(int flags);
void clusterBeforeSleep(void);
```

在集群的`clusterState`结构体之中，定义了一个`clusterState.todo_before_sleep`字段，这个字段以掩码的形式规定了在集群返回休眠状态前需要需要执行操作，这些操作包括：

| 定义                                          | 操作                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| `#define CLUSTER_TODO_HANDLE_FAILOVER (1<<0)` | 需要调用`clusterHandleSlaveFailover`处理故障转移逻辑。       |
| `#define CLUSTER_TODO_UPDATE_STATE (1<<1)`    | 需要调用`clusterUpdateState`重新评估集群状态。               |
| `#define CLUSTER_TODO_SAVE_CONFIG (1<<2)`     | 需要调用`clusterSaveConfigOrDie`将集群配置信息更新到配置文件之中。 |
| `#define CLUSTER_TODO_FSYNC_CONFIG (1<<3)`    | 需要配合`CLUSTER_TODO_FSYNC_CONFIG`，更新的配置文件立即存储到磁盘中。 |

通过调用`clusterDoBeforeSleep`函数在某些操作之后，为`clusterState.todo_before_sleep`字段上设置对应的掩码。这样便可以在重新返回休眠状态去，通过调用`clusterBeforeSleep`函数，根据掩码来决定执行何种操作。

## 集群的主从复制

集群还提供了几个用于操作`clusterNode`节点之间主从复制关系的基础操作函数。

```c
int clusterNodeRemoveSlave(clusterNode *master, clusterNode *slave);
int clusterNodeAddSlave(clusterNode *master, clusterNode *slave);
int clusterCountNonFailingSlaves(clusterNode *n);
void clusterSetNodeAsMaster(clusterNode *n)；
```

上述前两个函数分别用于从`master`节点的`clusterNode.slaves`数组之中插入或者移除对应的`slave`节点的`clusterNode`节点指针，同时会更新`master`节点的`clusterNode.numslaves`计数，并根据这个`master`节点是否有`slave`从节点来更新`CLUSTER_NODE_MIGRATE_TO`标记到`master`节点的`clusterNode.flags`字段。

`clusterCountNonFailingSlaves`函数则是用于统计一个给定的节点`clusterNode`下的`clusterNode.slaves`之中，正常的从节点的数量。

函数`clusterSetNodeAsMaster`则是将参数所给定的`clusterNode`节点设置为**Master**节点，如果这个节点之前曾是某个其他节点的**Slave**节点，那么会调用`clusterNodeRemoveSlave`来移除主从复制关系。同时还会清理该节点的`CLUSTER_NODE_SLAVE`标记，并为该节点添加上`CLUSTER_NODE_MASTER`标记。

而将一个节点设置另外一个节点的**Slave**节点，则可以通过下面的函数来实现：

```c
void clusterSetMaster(clusterNode *n);
```

执行这个函数，会将当前实例`myself`设置为参数`n`所对应的目标节点的**Slave**节点，该函数会为当前实例设置上`CLUSTER_NODE_SLAVE`的标志，同时调用`clusterNodeRemoveSlave`以及`clusterNodeAddSlave`来调整节点之间的对应关系，并调用在前面介绍主从复制机制之中所介绍的`replicationSetMaster`函数，让当前实例`myself`开始复制目标节点。

当一个节点的主从复制状态发生变化的时候，它可以通过`clusterMsg`消息将这个变化广播给集群内群内的所有节点，在`clusterMsg`消息数据结构之中携带有消息发送者复制的**Master**节点的唯一标识符`clusterMsg.slaveof`。通过这个字段是否携带有数据，可以判断出消息的发送者当前的主从复制状态，进而更新消息接收者本地的集群配置信息。

```c
int clusterProcessPacket(clusterLink *link) {
    ...
	if (sender) {
		if (!memcmp(hdr->slaveof,CLUSTER_NODE_NULL_NAME,sizeof(hdr->slaveof)))
		{
			/* Node is a master. */
			clusterSetNodeAsMaster(sender);
		} else {
			/* Node is a slave. */
			clusterNode *master = clusterLookupNode(hdr->slaveof);
			if (nodeIsMaster(sender)) {
				/* Master turned into a slave! Reconfigure the node. */
				clusterDelNodeSlots(sender);
				sender->flags &= ~(CLUSTER_NODE_MASTER|CLUSTER_NODE_MIGRATE_TO);
				sender->flags |= CLUSTER_NODE_SLAVE;
				/* Update config and state. */
				clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE);
			}
			/* Master node changed for this slave? */
			if (master && sender->slaveof != master) {
				if (sender->slaveof)
					clusterNodeRemoveSlave(sender->slaveof,sender);
				clusterNodeAddSlave(master,sender);
				sender->slaveof = master;
				/* Update config. */
				clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG);
			}
		}
	}
    ...
}
```

上面的代码片段便是节点的主从复制状态是如何更新到集群之中的其他节点的：

1. 如果`clusterMsg`消息之中没有携带`clusterMsg.slaveof`数据，说明发送消息的节点`sender`是一个**Master**节点，那么便会调用`clusterSetNodeAsMaster`将`sender`尝试设置为**Master**节点。
2. 如果携带了相应的**Master**节点的唯一标识符，则会按照如下两种情况进行区分：
   1. 如果发送消息的节点`sender`曾经是一个**Master**节点，那么说明需要将其转换为一个**Slave**节点。需要调用`clusterDelNodeSlots`函数来清理分配给`sender`的`slot`槽位信息，并重置`sender->flags`上的标记信息。
   2. 如果发送消息的节点`sender`是一个**Slave**节点，但是它并不是复制的`clusterMsg.slaveof`所携带的唯一标识符对应的**Master**节点，这说明复制结构发生了变化，会调用相关函数结构进行调整。

## 集群故障迁移

### 评估排名

在介绍故障迁移之前，我们先来看一个评估**Slave**节点排名的函数接口：

```c
int clusterGetSlaveRank(void);
```

该函数会返回当前集群实例在其主从复制架构之中的排名，这个排名是由**Slave**节点的复制偏移量决定的，具有更大复制偏移量的**Slave**节点的排名越靠前，也就意味着这个**Slave**节点同步了更多的**Master**节点的数据，说明这个**Slave**节点是相对更新的。

排名值为0的**Slave**节点便是具有最大复制偏移量，这个排名值的用途用于在故障转移的过程之中添加一个延迟。排名越小的**Slave**节点会以更小的延迟向集群之中其他的**Master**节点请求投票，这个机制能够保证较新的**Slave**节点能更为容易地获得其他**Master**节点的投票。

### 迁移条件

```c
void clusterHandleSlaveFailover(void);
```

上面`clusterHandleSlaveFailover`这个函数有两个主要功能：

1. 检查迁移条件判断是否需要进行故障迁移，就如前面集群心跳逻辑的内容中所介绍的，**Slave**节点会在心跳函数`clusterCron`之中调用`clusterHandleSlaveFailover`这个函数来对当前的集群状态进行检测。
2. 检测并处故障迁移选举超时以及处理最终选举结果。

这里我们先了解几个关于故障迁移的时间变量：

1. 故障转移超时时间`auth_timeout`，是发起投票到等待回复的最大时间，通常这个时间是集群节点超时时间`server.cluster_node_timeout`的2倍。如前文所介绍的如果向某个节点发送**PING**消息之后，在`server.cluster_node_timeout`的时间内没有收到该节点返回的**PONG**消息，那么便会认为这个节点**主观下线**。
2. 故障转移重试时间`auth_retry_timeout`，是尝试再次获取选票之前等待的时间，这个时间是故障转移超时时间`auth_timeout`的2倍。
3. 故障迁移年龄`auth_age`，标识从开启一轮选举投票到现在所经历的时间。
4. 数据年龄`data_age`，标识当前实例与其对应的**Master**节点之间处于连接断开状态时间的长短。

在判断是否需要开启故障迁移流程时还必须满足一些前提条件：

1. 调用该函数的集群实例必须是一个**Slave**节点，且有复制的**Master**节点。
2. 所对应的**Master**节点要处于**客观下线**状态，或者当前是在手动故障转移。
3. 没有通过配置禁止故障转移，并且这不是手动故障转移。
4. 对应的**Master**节点至少负责处理若干哈希槽`slot`。

上述的4个条件必须全部满足才有可能进行故障迁移，否则`clusterHandleSlaveFailover`函数将中断执行。

在执行过上述前提条件后，**Redis**会评估该实例是否适合作为故障迁移的候选人开启一轮投票：

1. 会根据`data_age`来判断实例上的数据是否足够新，只有数据足够新的**Slave**节点才有资格开启故障迁移投票。

2. 根据`auth_age`以及迁移重试时间`auth_retry_time`来判断是否需要开启一轮新的投票：

   1. 初始化此轮选举之中当前实例发起投票请求的时间戳`clusterState.failover_auth_time`，这里会为其分配一个随机的延迟。
   2. 清零该实例收到的选票数量`clusterState.failover_auth_count`。
   3. 清空发起投票的标记`clusterState.failover_auth_sent`。
   4. 调用`clusterGetSlaveRank`函数刷新当前实例在同级**Slave**节点之中的排名`clusterState.failover_auth_rank`。
   5. 根据排名在为发起投票请求的时间戳加上一个相应的延迟。
   6. 向所有同级**Slave**节点广播一条**PONG**消息用于在请求投票前同步自己的状态数据。

   这样当前实例一旦开启了新一轮的选举投票，那么`clusterHandleSlaveFailover`会立即返回，并等待下次心跳中重入该函数并继续处理后续的投票请求逻辑。

3. 如果已经开启了一轮投票，但是当前实例还没有向集群中的节点发起投票请求，那么会刷新自己的排名`clusterState.failover_auth_rank`以及发起投票请求的时间戳`clusterState.failover_auth_time`。之所以需要重新刷新排名以及时间戳，是因为在重入`clusterHandleSlaveFailover`之前，当前实例可能通过消息更新了其他同级**Slave**节点的复制偏移量，这有可能会导致自己排名的变化，所以需要重新刷新。

4. 如果当前时间还没有到发起投票请求的时间`clusterState.failover_auth_time`，当前实例不会发起投票请求。这就如前面介绍的，集群希望具有更新数据的**Slave**节点率先发起投票请求，让其更容易被选举为故障迁移的目标节点。

5. 根据`auth_age`以及迁移超时时间`auth_timeout`来判断投票是否已经超时。

6. 在经过前面的判断后，如果没有请求过投票，也就是`clusterState.failover_auth_sent`为0的话，便会通过`clusterRequestFailoverAuth`函数来向集群之中的其它实例请求投票。

### 请求投票

```c
void clusterRequestFailoverAuth(void);
```

当集群之中的一个**Slave**节点发现自己的**Master**节点处于**客观下线**的状态，那么它会调用`clusterRequestFailoverAuth`来广播一条`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`消息，向集群之中的其他**Master**节点请求投票。

而集群之中的其他节点在接收到`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`这条投票请求消息时，会调用`clusterSendFailoverAuthIfNeeded`来判断是否需要投票给消息的发送者。

**Slave**节点在调用`clusterRequestFailoverAuth`前，会将自己的集群当前纪元`clusterState.currentEpoch`加一处理，同时发送其他`clusterMsg`消息的逻辑一样，当前纪元的变化以及该节点的配置纪元`clusterNode.configEpoch`也会通过`clusterMsg`消息通知给集群内的其他实例。

### 投票

````c
void clusterSendFailoverAuthIfNeeded(clusterNode *node, clusterMsg *request);
void clusterSendFailoverAuth(clusterNode *node);
````

能够处理投票请求的集群节点必须满足如下两个条件：

1. 必须是一个**Master**节点。
2. 至少负责一个哈希槽`slot`。

当一个**Master**节点收到了某个其他**Slave**节点的`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`消息时，会验证是否应该投票，以及有没有在投过票：

1. 消息发送者的当前纪元`requestCurrentEpoch`必须大于等于消息接收者的当前纪元`clusterState.currentEpoch`才有可能进行投票。如果不满足上述的条件说明消息接收者接收到的是一个过期的投票请求，这种情况下消息接收者会拒绝该投票请求。
2. 如果消息接收者的上次投票纪元`clusterState.lastVoteEpoch`等于当前纪元`clusterState.currentEpoch`，说明在这一轮投票之中，该消息接收者已经发起过投票，这种情况下消息接收者也会拒绝该投票请求。
3. 请求投票的消息发送者必须是一个**Slave**节点，并且其对应的**Master**节点在消息接收者的角度来看也是**客观下线**的状态，否则消息接收者会拒绝该投票请求。
4. 在2倍的集群节点超时时间内`server.cluster_node_timeout`，消息接收者只能对涉及同一个故障**Master**节点的故障迁移进行一次投票。
5. 请求投票的消息发送者所携带的配置纪元`requestConfigEpoch`不能小于其所宣称的哈希槽`slot`对应节点（也就是该**Slave**节点对应的**Master**节点）的配置纪元。

当消息接收者经过上面的判断后，便可以投票给发送`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`消息的请求者：

```c
void clusterSendFailoverAuthIfNeeded(clusterNode *node, clusterMsg *request)
{
    ...
	server.cluster->lastVoteEpoch = server.cluster->currentEpoch;
	node->slaveof->voted_time = mstime();
	clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_FSYNC_CONFIG);
	clusterSendFailoverAuth(node);
}
```

这里会将消息接收者的当前纪元`clusterState.currentEpoch`更新到上次投票纪元`clusterState.lastVoteEpoch`字段之中；并更新对于消息接收者对于故障**Master**节点的投票时间`node->slaveof->voted_time`；最后调用`clusterSendFailoverAuth`发送`CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`投票消息投给请求投票的发送者`node`。

### 更新投票结果

故障**Master**节点对应的**Slave**节点在通过`clusterRequestFailoverAuth`广播了投票请求之后，便会等待`CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`消息，来更细自己的投票计数：

```c
int clusterProcessPacket(clusterLink *link) {
    ...
    if (type == CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK) {
        if (nodeIsMaster(sender) && sender->numslots > 0 &&
            senderCurrentEpoch >= server.cluster->failover_auth_epoch)
        {
            server.cluster->failover_auth_count++;
            clusterDoBeforeSleep(CLUSTER_TODO_HANDLE_FAILOVER);
        }
    }
    ...
}
```

这里**Slave**节点在接收到有效的投票消息后，会更新自己的故障迁移投票计数`clusterState.failover_auth_count`，同时设置`CLUSTER_TODO_HANDLE_FAILOVER`标记，用于在返回休眠前调用`clusterHandleSlaveFailover`处理后续的故障迁移逻辑，之所以不立即调用该函数是为了尽能多地收集集群中其他分片实例的投票信息后再进行处理，防止逻辑空跑。

在`clusterHandleSlaveFailover`会验证当前收到的投票数是否达到要求的票数，如果达到要求就会调用`clusterFailoverReplaceYourMaster`将自己提升为**Master**节点，用以替换故障的分片实例。

```c
void clusterHandleSlaveFailover(void) {
    ...
    if (server.cluster->failover_auth_count >= needed_quorum) {
        clusterFailoverReplaceYourMaster();
    }
    ...
}
```

在`clusterFailoverReplaceYourMaster`这个函数之中，将执行如下的逻辑，用以最后进行故障转移：

1. 将当前实例从**Slave**节点转换为**Master**实例。
2. 将所有分配给故障**Master**的哈希槽`slot`全部分配给新的**Master**节点。
3. 更新集群状态`clusterState.state`并将集群配置更新保存到磁盘的配置文件上。
4. 向所有的集群节点广播一个**PONG**消息，用于将这个故障迁移的结果广播给整个集群。
5. 如果当前有一个正在进行的手动故障转移，那么充值清理这个手动故障迁移状态。

## 集群命令

### ASKING命令

这个命令的格式为：

```bash
ASKING
```

当集群用户在向一个节点发送查询命令后接收到一个`ASK`重定向时，会向目标节点发送ASKING命令，随后是被重定向的命令。这个操作用于给用户的客户端对象`client`添加一个`CLIENT_ASKING`标记，这个标记表示如果待操作的键对应的`slot`如果被标记为`importing`状态，那么这个命令也会被接受并执行。这通常由集群客户端自动完成。

### READONLY与READWRITE命令

这两个命令的格式为：

```bash
READONLY
READWRITE
```

**READONLY**命令用于将发送命令的用户设置为只读客户端；**READWRITE**命令则是相反的作用，用于将用户设置为可读写的客户端。

之所以需要这个命令，是因为在集群中，**Slave**分片实例的作用通常是为了提供对**Master**分片实例的数据备份，而对外提供查询命令服务的则是**Master**分片实例。如果用户在**Slave**节点上执行只读命令时，即使命令操作的**Key**对应的哈希槽`slot`属于这个节点所复制的**Master**节点，集群依然会返回`MOVED`错误给用户。

但是用户可以在**Slave**节点上通过**READONLY**命令强制告诉集群，用户能够接收读取可能过时的数据，并且不会执行写入的操作命令。而通过**READWRITE**命令则可以告知集群取消这个客户端的标记。

### DUMP命令

这个命令的格式为：

```bash
DUMP key
```

这个命令会将`key`对应的值以**Redis**特定的格式进行序列化，并将结果返回给用户。返回的数值可以使用**RESTORE**命令还原成**Redis**之中的键值对。

### MIGRATE与RESTORE命令

这两个命令的格式为：

```bash
MIGRATE host port <key | ""> destination-db timeout [COPY] [REPLACE] [AUTH password] [KEYS key [key ...]]
RESTORE key ttl serialized-value [REPLACE] [ABSTTL] [IDLETIME seconds] [FREQ frequency]
```

其中**MIGRATE**命令用于将指定的一个或者多个**Key**从接收命令的源服务器上迁移到地址为`host:port`的另外一台目标**Redis**服务器上。这个命令会在接收命令的源服务器上以**同步I/O**的方式向目标服务器发起**RESTORE**命令，将键值数据转移至目标服务。这个命令通常是由集群系统管理员执行，用于在重新分配哈希槽`slot`时使用。

### CLUSTER命令

```c
void clusterCommand(client *c);
```

现在我们来系统地介绍一下**CLUSTER**命令，用户无法单独使用这个命令，而是通过子命令来实现一系列的集群管理操作。而这些**CLUSTER**子命令逻辑都是通过上面这个`clusterCommand`函数来实现的。下面我们便依次介绍一下**CLUSTER**的系列子命令。

#### CLUSTER HELP

这个子命令用于向用户返回**CLUSTER**系列命令的详细信息。

#### CLUSTER MEET

命令的格式为：

```bash
CLUSTER MEET <ip> <port> [cport]
```

用于连接一个给定地址的节点并将其接入到集群之中，实例会从命令中解析出目标实例的`ip`地址、`port`端口以及`cport`集群总线端口，并调用`clusterStartHandshake`函数开启与这个目标实例的握手连接过程。

#### CLUSTER NODES

命令的格式为：

```bash
CLUSTER NODES
```

这个命令用于输出整个集群之中节点配置信息，这个子命令会调用`clusterGenNodesDescription`这个函数接口获取集群节点的配置信息，并将返回的字符串数据返回给用户，这个命令的行为与集群实例定期保存集群配置信息到磁盘配置文件的行为是类似，都是用过调用`clusterGenNodesDescription`函数来实现的。

#### CLUSTER MYID

命令的格式为：

```bash
CLUSTER MYID
```

这个命令用于获取接收命令的实例的唯一标识符，就是将`myself->name`的字段返回给用户。

#### CLUSTER SLOTS

命令的格式为：

```bash
CLUSTER SLOTS
```

这个命令向用户返回集群`slot`与集群分片实例之间映射的详细信息，这个命令是调用`clusterReplyMultiBulkSlots`函数来实现的。

```bash
> CLUSTER SLOTS
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 30001
      3) "09dbe9720cda62f7865eabc5fd8857c5d2678366"
   4) 1) "127.0.0.1"
      2) (integer) 30004
      3) "821d8ca00d7ccf931ed3ffc7e3db0599d2271abf"
2) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 30002
      3) "c9d93d9f2c0c524ff34cc11838c2003d8c29e013"
   4) 1) "127.0.0.1"
      2) (integer) 30005
      3) "faadb3eb99009de4ab72ad6b6ed87634c7ee410f"
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 30003
      3) "044ec91f325b7595e76dbcb18cc688b6a5b434a1"
   4) 1) "127.0.0.1"
      2) (integer) 30006
      3) "58e6e48d41228013e5d9c1c37c5060693925e97e"
```

以上便是一个**CLUSTER SLOTS**命令的返回结果，每一组返回数据包含：

1. `slot`区间的起始`slot`。
2. `slot`区间的结束`slot`。
3. 其所属的**Master**实例的信息：
   1. **Master**实例的`ip`地址。
   2. **Master**实例的`port`端口号。
   3. **Master**实例的唯一标识符。
4. 所属**Master**实例对应的**Slave**信息：
   1. **Slave**实例的`ip`地址。
   2. **Slave**实例的`port`端口号。
   3. **Slave**实例的唯一标识符。

该条子命令是通过调用`clusterReplyMultiBulkSlots`遍历`clusterState.nodes`集群节点列表来实现的。

#### CLUSTER ADDSLOTS & DELSLOTS

这两条子命令的格式为：

```bash
CLUSTER ADDSLOTS <slot> [slot] ...
CLUSTER DELSLOTS <slot> [slot] ...
```

这两条子命令分别用于将一个或者多个`slot`分配给命令接收的分片实例或将这些`slot`从接收命令的实例上移除。

这两个操作分别会调用`clusterAddSlot`以及`clusterDelSlot`接口来实现，同时会触发集群配置存盘的逻辑。

#### CLUSTER SETSLOTS

这个子命令用于在运行时对集群的`slot`进行重新分配，这个子命令也有若干个子命令：

```bash
CLUSTER SETSLOT <SLOT> MIGRATING <NODE ID>
CLUSTER SETSLOT <SLOT> IMPORTING <NODE ID>
CLUSTER SETSLOT <SLOT> STABLE
CLUSTER SETSLOT <SLOT> NODE <NODE ID>
```

##### CLUSTER SETSLOT MIGRATING

这个子命令用于在接收命令的实例上将`slot`设置为导出`migrating`的状态，会将`clusterState.migrating_slots_to[slot]`设置为`node-id`对应的目标节点。

##### CLUSTER SETSLOT IMPORTING

这个子命令用于在接收命令的实例上将`slot`设置为导入`importing`的状态，会将`clusterState.importing_slots_from[slot]`设置为`node-id`对应的源节点。

##### CLUSTER SETSLOT STABLE

这个子命令用于清理`slot`的导出`migrating`/导入`importing`状态，主要用于修复陷于错误状态的集群，这个子命令会清空`clusterState.migrating_slots_to[slot]`以及`clusterState.importing_slots_from[slot]`上的节点指针数据。

##### CLUSTER SETSLOT NODE

这个子命令如前面所介绍的，在**MIGRATING**过程的最后一步，在源分片实例以及目标分片实例上声明将`slot`这个槽位分配给目标实例。

#### CLUSTER BUMPEPOCH

这个命令的格式为：

```bash
CLUSTER BUMPEPOCH
```

这个子命令会调用`clusterBumpConfigEpochWithoutConsensus`来前置增加接收命令的分配实例的配置纪元`myself->configEpoch`，使其配置纪元成为整个集群之中的最大值。这个命令通常用于强制重新平衡配置纪元以及故障转移的场景。

####  CLUSTER INFO

这个命令的格式为：

```bash
CLUSTER INFO
```

这个命令返回关于集群状态的信息，该信息包括当前集群状态、已知节点数量、已分配的哈希槽等信息。通常这个命令会返回一系列的键-值对，分别描述了集群的各项参数和状态。

通常情况下，这个命令的返回数据如下所示：

```bash
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:2
cluster_stats_messages_sent:1483972
cluster_stats_messages_received:1483968
```

在这些返回的数据之中：

1. `cluster_state`，如果该节点能够接收查询命令，则节点的状态被标记为`ok`；如果有至少一个`slot`没有分配处理的节点，或者其关联的节点处于故障状态，则该状态被标记为`fail`。
2. `cluster_slots_assigned`，记录已经分配了处理节点`clusterNode`的`slot`槽位的数量，如果节点能够正常工作，这个数字应该是`16384`，也就是所有的`slot`都分配到了一个集群节点。
3. `cluster_slots_ok`，记录了映射到一个不在**客观下线**或**主观下线**状态的节点`clusterNode`的`slot`槽位的数量。
4. `cluster_slots_pfail`，记录了映射到处于**主观下线**状态的节点的`slot`槽位的数量。
5. `cluster_slots_fail`，记录映射到处于**客观下线**状态的节点的`slot`槽位数量。
6. `cluster_known_nodes`，集群之中已知节点`clusterNode`数量，包括了可能当前不是集群正式分片节点的那些处于`HANDSHAKE`状态的节点。
7. `cluster_size`，
8. `cluster_current_epoch`，记录集群的当前纪元`clusterState.currentEpoch`。
9. `cluster_my_epoch`，如果当前实例是一个**Master**实例，那么这里为自己的节点配置纪元`myself->configEpoch`；如果当前实例是一个**Slave**实例，那么这里为自己对应的**Master**节点的配置纪元`myself->slaveof->configEpoch`。
10. `cluster_stats_messages_sent`，记录当前实例通过集群总线发送的集群消息`clusterMsg`的数量。
11. `cluster_stats_messages_received`，记录当前实例通过集群总线接收的集群消息`clusterMsg`的数量。

#### CLUSTER SAVECONFIG

这个命令的格式为：

```bash
CLUSTER SAVECONFIG
```

这个子命令通过调用`clusterSaveConfig`函数强制让接收命令的分片实例将集群配置保存到磁盘上的配置文件中。这个命令是告诉节点需要将最新的集群配置保存到硬盘上，以便在节点重启之后重新加载。每次集群状态发生变化时，**Redis**会自动将集群配置持久化到磁盘上，但是也可以使用该命令强制配置的持久化。通常情况下，建议在集群配置发生变更之后手动执行该命令。

#### CLUSTER KEYSLOT

这个命令的格式为：

```bash
CLUSTER KEYSLOT <key>
```

这个子命令用于根据给定的键名计算出该键属于哪个哈希槽，通过调用`keyHashSlot`来计算给定的`key`所属的`slot`，并将其返回给用户。

#### CLUSTER COUNTKEYSINSLOT

这个命令的格式为：

```bash
CLUSTER COUNTKEYSINSLOT <slot>
```

这个子命令用于获取指定哈希槽上的键数量，通过调用`countKeysInSlot`这个函数接口来获取给定的`slot`上的键的数量。

#### CLUSTER GETKEYSINSLOT

这个命令的格式为：

```bash
CLUSTER GETKEYSINSLOT <slot> <count>
```

这个子命令用于获取给定的哈希槽`slot`上的数量不超过`count`的键的集合，通过调用`getKeysInSlot`函数来获取到这个键的集合。

#### CLUSTER FORGET

这个命令的格式为：

```bash
CLUSTER FORGET <NODE ID>
```

这个子命令用于从接收该命令的集群实例上将给定唯一标识符`node-id`的节点删除。换句话说，该命令将指定节点从接收到该命令的节点的节点表中删除。因为当一个节点是集群的一部分时，参与集群的所有其他节点都了解它，所以为了完全将节点从集群中移除，必须向所有剩余节点发送**CLUSTER FORGET**命令，无论它们是主节点还是从节点。

该命令禁止将接收命令的实例自己从集群之中移除；如果接收命令的实例是一个**Slave**节点，那么该命令也禁止其将自己对应的**Master**节点移除。

该命令会如前面所介绍的调用`clusterBlacklistAddNode`将待删除节点加入黑名单，防止通过其他节点发来的**Gossip**消息将该节点重新添加回去；然后会调用`clusterDelNode`将节点删除，并将集群配置更新刷新到文件之中。

#### CLUSTER REPLICATE

这个命令的格式为：

```bash
CLUSTER REPLICATE <NODE ID>
```

这个子命令将接收到命令的集群实例重新配置为指定唯一标识符`node-id`的**Master**节点的**Slave**节点。如果接收到该命令的实例是空的**Master**节点，则该命令会将节点角色从**Master**节点更改为**Slave**节点。

这里`node-id`所指定的目标节点必须要满足两个条件：

1. 这个目标节点不能是接收命令的分片实例自己。
2. 这个目标节点也不能是一个**Slave**节点。

如果接收命令的分片实例自身就是一个**Master**节点，那么需要保证这个实例是一个空的实例，才可以将其指定为其他节点的**Slave**节点。

这个命令最终会调用`clusterSetMaster`函数接口，将给定的`clusterNode`设置为自己的**Master**节点。

#### CLUSTER SLAVES

这个命令的格式为：

```bash
CLUSTER SLAVES <NODE ID>
```

这个子命令用于获取集群中指定唯一标识符`node-id`所标识的**Master**节点的所有**Slave**节点的信息。这个命令确保`node-id`对应的节点必须是一个**Master**节点。该命令会遍历指定节点的`clusterNode.slaves`列表，对其中的**Slave**节点依次调用`clusterGenNodeDescription`函数来生成状态信息，并返回给用户。

#### CLUSTER COUNT-FAILURE-REPORTS

这个命令的格式为：

```bash
CLUSTER COUNT-FAILURE-REPORTS <NODE ID>
```

这个命令用户获得上报关于指定唯一标识符`node-id`节点故障报告的数量，通过`clusterNodeFailureReportsCount`函数来实现。

#### CLUSTER SET-CONFIG-EPOCH

这个命令的格式为：

```bash
CLUSTER SET-CONFIG-EPOCH <epoch>
```

这个命令用于设置接收命令的分片实例的配置纪元`myself->configEpoch`。如果设置的配置纪元大于集群的当前纪元`clusterState.currentEpoch`，那么集群当前纪元也会被设置为相同的数值。


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*