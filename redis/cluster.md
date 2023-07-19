# Redis集群



## 集群数据结构

| 数据结构                | 含义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `clusterNode`           | 代表了集群之中每一个分片实例的抽象，每个**Redis**进群分片实例之中都通过`clusterNode`存储了其所能感知到的集群中所有其他分片信息。 |
| `clusterLink` | 代表了当前分片实例与某一个`clusterNode`所代表的的其他分片的网络连接。 |
| `clusterNodeFailReport` | 当集群中的某一个节点出现故障的时候，这个数据结构代表其他分片对该节点的故障报告。 |
| `clusterState`          | 分片的状态信息，存储了当前**Redis**集群分片实例运行时的状态信息。 |
| `clusterMsg`            | 用于表示集群之中各个分片之间通信消息的抽象。这个数据结构之中除了携带消息数据之外，还携带了发送消息的分片的一些基础状态信息。 |
| `clusterMsgData`        | 用于表示集群之中各个分片之间通信消息具体消息数据的抽象，这个`union`结构，可以分别表示分片间通信的五种消息类型。 |
| `clusterMsgDataGossip`  | `clusterMsgData`的五种消息类型之一，用于通知交换彼此状态信息。 |
| `clusterMsgDataFail`    | `clusterMsgData`的五种消息类型之一，用于通知某个分片发生故障处于下线状态。 |
| `clusterMsgDataPublish` | `clusterMsgData`的五种消息类型之一，用于向集群之中的分片广播消息用。 |
| `clusterMsgDataUpdate`  | `clusterMsgData`的五种消息类型之一，用于更新某个分片的配置纪元、槽位归属的信息。 |
| `clusterMsgDataModule`  | `clusterMsgData`的五种消息类型之一，用于处理模块数据。 |

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
   6. `CLUSTER_NODE_HANDSHAKE`，这个状态表示当前分片实例与对应节点对应分片还处于
   7. `CLUSTER_NODE_NOADDR`，这个状态表示当前分片实例还不知道这个节点对应分片的地址信息。
   8. `CLUSTER_NODE_MEET`
   9. `CLUSTER_NODE_MIGRATE_TO`
   10. `CLUSTER_NODE_NOFAILOVER`
4. `clusterNode.configEpoch`，存储当前分片实例所获取到的对应这个节点的分片的配置纪元。
5. `clusterNode.slots`，以掩码的形式存储这个节点对应的分片实例所负责的**slot**。
6. `clusterNode.numslots`，记录这个节点对应的分片实例负责的**slot**数量。
7. `clusterNode.numslaves`，如果这个节点对应的分片实例是一个主服务器，那么这个字段记录了这个主服务分片关联的从服务器数量。
8. `clusterNode.slaves`，如果这个节点对应的分片实例是一个主服务器，那么这个指针数组则存储其对应的所有从服务器分片对应的`clusterNode`节点的指针，数组的长度就是`clusterNode.numslaves`定义的长度。
9. `clusterNode.slaveof`，如果这个节点对应的分片实例是一个从服务器，那么这个指针指向该从服务器对应的主服务器的`clusterNode`节点。
10. `clusterNode.ping_sent`，记录当前集群分片实例向该节点对应的分片发送**PING**命令的时间戳。
11. `clusterNode.pong_received`，记录当前集群分片实例从`clusterNode`节点对应的分片中接收**PONG**返回的时间戳。
12. `clusterNode.fail_time`，记录该`clusterNode`节点被设置成`CLUSTER_NODE_FAIL`状态时的时间戳。
13. `clusterNode.voted_time`
14. `clusterNode.repl_offset_time`
15. `clusterNode.orphaned_offset`
16. `clusterNode.repl_offset`
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

1. `clusterState.myself`，这个集群节点指针用于指向当前的分片实例服务器所对应的`clusterNode`集群节点。
2. `clusterState.state`，这个字段用于标记当前的分片实例服务器的集群状态。
3. `clusterState.size`，这个字段记录了在当前的集群之中主服务器实例的数量。
4. `clusterState.nodes`，这个哈希表用于记录当前分片实例服务器之中所能感知的集群节点，以节点名字`clusterNode.name`为**Key**，以节点指针`clusterNode*`为**Value**。
5. `clusterState.nodes_black_list`，这个哈希表则用于记录短期内不能再重新加入到集群之中的分片节点。
6. `clusterState.migrating_slots_to`，如果当前分片实例服务器正在将某个指定的`slot`数据转移给其他的分片实例时，那么`migrating_slots_to`数组之中这个对应的位置则会存储这次数据移动的目标`clusterNode`节点指针。
7. `clusterState.importing_slots_from`，如果当前分片实例正在从其他分片实例之中导入指定的`slot`数据时，这个数组对应的位置则会存储这个`slot`原所在的分片对应的`clusterNode`节点指针。
8. `clusterState.slots`，这个数组存储了集群之中每一个**slot**所属的`clusterNode`节点指针。
9. `clusterState.slots_keys_count`，这个数组存储了集群之中每一个**slot**之中对应的**Key**的数量。
10. `clusterState.slots_to_keys`，
11. 



### 初始化

这里也需要解释一下*nodes.conf*这个配置文件在**Redis**集群之中的作用。这个文件之中最重要的配置便是当前集群之中所知的所有的实例配置信息。

```c
int clusterLoadConfig(char *filename);
int clusterSaveConfig(int do_fsync);
void clusterSaveConfigOrDie(int do_fsync);
int clusterLockConfig(char *filename);
void clusterUpdateMyselfFlags(void);
void clusterInit(void);
void clusterReset(int hard);
```

`clusterLockConfig`这个函数

`clusterLoadConfig`这个函数用于从集群配置文件之中加载当前分片对应的配置信息，这个配置文件默认的文件名被定义在*src/server.h*头文件之中：

```c
#define CONFIG_DEFAULT_CLUSTER_CONFIG_FILE "nodes.conf"
```

文件之中最为重要的的配置数据便是集群之中节点的配置信息，`clusterLoadConfig`函数会遍历配置文件之中每一个分片实例的配置信息，使用该信息构造分片对应的`clusterNode`数据，每一行配置信息首先会标记分片实例的唯一标识，网络连接信息，以及对应的标记信息，这其中包括：

| 标记       | 含义                                 | 对应Flags类型             |
| ---------- | ------------------------------------ | ------------------------- |
| myself     | 该行为启动实例自身的配置信息         | `CLUSTER_NODE_MYSELF`     |
| master     | 该行为一个主分片实例的配置信息       | `CLUSTER_NODE_MASTER`     |
| slave      | 该行为一个从分片实例的配置信息       | `CLUSTER_NODE_SLAVE`      |
| fail?      | 表示该分片处于主观下线的状态         | `CLUSTER_NODE_PFAIL`      |
| fail       | 表示该分片处于客观下线的状态         | `CLUSTER_NODE_FAIL`       |
| handshake  | 表示                                 | `CLUSTER_NODE_HANDSHAKE`  |
| noaddr     | 表示当前还暂时不知道该分片的地址信息 | `CLUSTER_NODE_NOADDR`     |
| nofailover |                                      | `CLUSTER_NODE_NOFAILOVER` |
| noflags    |                                      |                           |



`clusterLockConfig`这个函数会调用`flock`系统调用，对集群配置文件尝试添加文件锁，之所以这么做，是因为我们经常需要在有一个新版集群配置数据的时候原地更新*nodes.conf*这个配置文件，重新打开这个文件，并对这个文件进行原地写入。如果给定的配置文件不存在，那么**Redis**则会首先创建一个空的文件出来，并对该文件调用`flock`系统调用进行加锁操作，这个加锁操作是以`LOCK_EX|LOCK_NB`的策略进行的，也就是会独占这个配置文件的锁，同时当有其他进程已经锁住该文件的时候，进程不会阻塞在这里等待该锁被释放。

函数`clusterInit`用于执行**Redis**集群当前的分片服务器的初始化流程，会在**Redis**服务器启动初始化的`initServer`接口之中被调用：

```c
void initServer(void)
{
  ...
  if (server.cluster_enabled) clusterInit();
  ...
}
```

在这个函数之中，会对`serverState.cluster`这个`clusterState`数据结构进行初始化。

### 集群连接接口

```c
clusterLink *createClusterLink(clusterNode *node) ;
void freeClusterLink(clusterLink *link);
void clusterAcceptHandler(aeEventLoop *el, int fd, void *privdata, int mask);

```



### 键空间映射

```c
unsigned int keyHashSlot(char *key, int keylen);
```



### 集群节点接口

```c
clusterNode *createClusterNode(char *nodename, int flags);
int clusterNodeAddFailureReport(clusterNode *failing, clusterNode *sender);
void clusterNodeCleanupFailureReports(clusterNode *node);
int clusterNodeDelFailureReport(clusterNode *node, clusterNode *sender);
int clusterNodeFailureReportsCount(clusterNode *node);d
int clusterNodeRemoveSlave(clusterNode *master, clusterNode *slave);
int clusterNodeAddSlave(clusterNode *master, clusterNode *slave);
int clusterCountNonFailingSlaves(clusterNode *n);
void freeClusterNode(clusterNode *n);
int clusterAddNode(clusterNode *node);
void clusterDelNode(clusterNode *delnode);
clusterNode *clusterLookupNode(const char *name);
void clusterRenameNode(clusterNode *node, char *newname);
```


### 集群配置纪元接口

```c
uint64_t clusterGetMaxEpoch(void);
int clusterBumpConfigEpochWithoutConsensus(void);
void clusterHandleConfigEpochCollision(clusterNode *sender);
```



### 集群节点黑名单接口

```c
void clusterBlacklistCleanup(void);
void clusterBlacklistAddNode(clusterNode *node);
int clusterBlacklistExists(char *nodeid);
```


### 集群消息交换

```c
void clusterWriteHandler(aeEventLoop *el, int fd, void *privdata, int mask);
void clusterReadHandler(aeEventLoop *el, int fd, void *privdata, int mask);
void clusterSendMessage(clusterLink *link, unsigned char *msg, size_t msglen);
void clusterBroadcastMessage(void *buf, size_t len);
void clusterBuildMessageHdr(clusterMsg *hdr, int type);
void markNodeAsFailingIfNeeded(clusterNode *node);
void clearNodeFailureIfNeeded(clusterNode *node);
int clusterHandshakeInProgress(char *ip, int port, int cport);
int clusterStartHandshake(char *ip, int port, int cport);
void clusterProcessGossipSection(clusterMsg *hdr, clusterLink *link);
void nodeIp2String(char *buf, clusterLink *link, char *announced_ip);
int nodeUpdateAddressIfNeeded(clusterNode *node, clusterLink *link, clusterMsg *hdr);
void clusterSetNodeAsMaster(clusterNode *n);
void clusterUpdateSlotsConfigWith(clusterNode *sender, uint64_t senderConfigEpoch, unsigned char *slots);
int clusterProcessPacket(clusterLink *link);
void handleLinkIOError(clusterLink *link);
int clusterNodeIsInGossipSection(clusterMsg *hdr, int count, clusterNode *n);
void clusterSetGossipEntry(clusterMsg *hdr, int i, clusterNode *n);
void clusterSendPing(clusterLink *link, int type);
void clusterBroadcastPong(int target);
void clusterSendPublish(clusterLink *link, robj *channel, robj *message);
void clusterSendFail(char *nodename);
void clusterSendUpdate(clusterLink *link, clusterNode *node);
```

### 集群订阅发布支持

```c
void clusterPropagatePublish(robj *channel, robj *message);
```

### 集群中从节点的专用函数

```c
void clusterRequestFailoverAuth(void);
void clusterSendFailoverAuth(clusterNode *node);
void clusterSendMFStart(clusterNode *node);
void clusterSendFailoverAuthIfNeeded(clusterNode *node, clusterMsg *request);
int clusterGetSlaveRank(void);
void clusterLogCantFailover(int reason);
void clusterFailoverReplaceYourMaster(void);
void clusterHandleSlaveFailover(void);
```


### 集群从服务器迁移

```c
void clusterHandleSlaveMigration(int max_slaves);
```

### 集群手动故障迁移

```c
void resetManualFailover(void);
void manualFailoverCheckTimeout(void);
void clusterHandleManualFailover(void);
```


### 集群定时任务

```c
void clusterCron(void);
void clusterBeforeSleep(void);
void clusterDoBeforeSleep(int flags);
```

### 集群槽位管理

```c
int bitmapTestBit(unsigned char *bitmap, int pos);
void bitmapSetBit(unsigned char *bitmap, int pos);
void bitmapClearBit(unsigned char *bitmap, int pos);
int clusterMastersHaveSlaves(void);
int clusterNodeSetSlotBit(clusterNode *n, int slot);
int clusterNodeClearSlotBit(clusterNode *n, int slot);
int clusterNodeGetSlotBit(clusterNode *n, int slot);
int clusterAddSlot(clusterNode *n, int slot);
int clusterDelSlot(int slot);
int clusterDelNodeSlots(clusterNode *node);
void clusterCloseAllSlots(void);
```

### 集群状态评估接口

```c
void clusterUpdateState(void);
int verifyClusterConfigWithData(void);
```

### 集群从节点操作

```c
void clusterSetMaster(clusterNode *n);
```

### 集群命令

```c
void clusterCommand(client *c);
```








































***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*