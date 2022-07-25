# Redis集群



## 集群数据结构

| 数据结构                | 含义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `clusterNode`           | 代表了集群之中每一个分片实例的抽象，每个**Redis**进群分片实例之中都通过`clusterNode`存储了其所能感知到的集群中所有其他分片信息。 |
| `clusterLink` | 代表了当前分片实例与某一个`clusterNode`所代表的的其他分片的网络连接。 |
| `clusterNodeFailReport` | |
| `clusterState`          | 分片的状态信息，存储了当前**Redis**集群分片实例运行时的状态信息。 |
| `clusterMsg`            | 用于表示集群之中各个分片之间通信消息的抽象。这个数据结构之中除了携带消息数据之外，还携带了发送消息的分片的一些基础状态信息。 |
| `clusterMsgData`        | 用于表示集群之中各个分片之间通信消息具体消息数据的抽象，这个`union`结构，可以分别表示分片间通信的五种消息类型。 |
| `clusterMsgDataGossip`  |                                                              |
| `clusterMsgDataFail`    |                                                              |
| `clusterMsgDataPublish` |                                                              |
| `clusterMsgDataUpdate`  |                                                              |
| `clusterMsgDataModule`  |                                                              |

此外在**Redis**服务器全局状态数据`redisServer`之中也存储着集群的相关数据：

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
};
```

首先我们看一下代表分片实例抽象的数据类型`clusterNode`：

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
   6. `CLUSTER_NODE_HANDSHAKE`
   7. `CLUSTER_NODE_NOADDR`
   8. `CLUSTER_NODE_MEET`
   9. `CLUSTER_NODE_MIGRATE_TO`
   10. `CLUSTER_NODE_NOFAILOVER`
4. `clusterNode.configEpoch`，存储当前分片实例所获取到的对应这个节点的分片的配置纪元。
5. `clusterNode.slots`，以掩码的形式存储这个节点对应的分片实例所负责的**slot**。
6. `clusterNode.numslots`，记录这个节点对应的分片实例负责的**slot**数量。
7. `clusterNode.numslaves`，如果这个节点对应的分片实例是一个主服务器，那么这个字段记录了这个主服务分片关联的从服务器数量。
8. `clusterNode.slaves`，如果这个节点对应的分片实例
9. `clusterNode.slaveof`
10. `clusterNode.ping_sent`
11. `clusterNode.pong_received`
12. `clusterNode.fail_time`
13. `clusterNode.voted_time`
14. `clusterNode.repl_offset_time`
15. `clusterNode.orphaned_offset`
16. `clusterNode.repl_offset`
17. `clusterNode.ip`
18. `clusterNode.cport`
19. `clusterNode.link`
20. `clusterNode.fail_reports`

接下来我们看一下`clusterState`这个数据结构的定义：

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

1. `clusterState.myself`
2. 









































***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*