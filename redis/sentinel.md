# Redis哨兵模式

如果说主从复制模式是*Redis*之中的一种高并发的解决方案的话，那么哨兵模式便是*Redis*实现的一种高可用解决方案。哨兵模式可以监视主从复制模式下各个实例的运行状态，当**Master**实例由于各种原因的故障而下线时，哨兵模式可以探测到这种掉线的情况，能够将某一个**Slave**实例提升为**Master**实例，并让其他的**Slave**实例都复制这个新的**Master**实例，让整个主从复制模式的*Redis*实例集群能够继续对外提供服务，以实现故障的自动迁移。

## 哨兵模式概述

*Redis*的哨兵模式可以提供监视、通知以及为客户端提供配置等功能。

- **监控**
  - 哨兵实例会不断检查**Master**实例以及**Slave**实例是否正常地进行工作。
- **通知**
  - 当哨兵实例监控的某个*Redis*实例出现故障时，可以通过API通知系统管理员以及其他计算机程序。
- **故障自动迁移**
  - 如果**Master**实例没有正常工作，那么哨兵实例可以启动故障迁移。在故障迁移的过程之中，**Slave**实例会被升级为**Master**实例，同时哨兵实例会强制其他**Slave**模式复制这个新的**Master**实例，并会通知使用这个*Redis*服务器的应用程序更新服务器的地址。
- **提供配置**
  - 哨兵实例可以充当客户端服务发现的来源，客户端直接连接到哨兵实例，并以轮询的方式来获取当前的*Redis*服务器**Master**实例的地址。

同时，哨兵模式本身便是一个分布式的系统。哨兵模式可以被部署为多个哨兵实例协同工作的集群，这种方式具有如下的优点：

- 只有当多个哨兵实例都认定给定的*Redis*实例掉线的时候，才会执行故障的检测与迁移流程。这样可以大大地降低了误报的可能。
- 即使个别的哨兵实例宕机，整个的哨兵模式依然可以正常地工作来帮助整个*Redis*系统应对故障。毕竟作为保证分布式系统高可用性的模块，如果自身就是一个单点，那么这就有一些搞笑了。

由哨兵模块，包括**Master**与**Slave**实例在内的*Redis*实例，以及连接在*Redis*实例或者哨兵模块上的客户端构成了一个具有特定属性的大型分布式系统。

### 运行哨兵实例

通过`redis-sentinel`我们可以启动一个哨兵实例，启动命令为：

```
redis-sentinel /path/to/sentinel.conf
```

除了这种启动方式之外，我们还可以通过`redis-server`这个*Redis*实例的可执行文件来启动一个哨兵实例，启动命令为：

```
redis-server /path/to/sentinel.conf --sentinel
```

而本质上，`redis-sentinel`以及`redis-server`这两个可执行文件是完全一样的两个文件，如果我们使用`md5sum`命令来检查这两个可执行文件的**MD5**值，我们便可以发现这两个文件是完全一模一样的。

### 配置哨兵实例

在*Redis*发行版的源代码之中，包含了一个名为`sentinel.conf`的配置文件，这个文件可以用于配置哨兵实例，一个典型的最小的配置文件可以入下面所示：

```
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-time
```

### Raft算法

之所以在这里介绍**Raft**算法，是因为在故障迁移的过程之中，当**主从**集群之中的**Master**服务器下线时，负责监控这些**Redis**服务器的**Sentinel**集群便会应用**Raft**算法，从集群中的所有**Sentinel**服务器之中选举出一个**Leader**服务器，在由这个**Leader**服务器对**Master**服务器进行故障迁移的操作。在介绍**Raft**算法之前，我们先来看一下提出**Raft**算法的缘由，也就是分布式系统之中著名的**拜占庭将军问题**。

#### 拜占庭将军问题

首先我们来看一下对于**拜占庭将军问题**的一个简要描述：

> 拜占庭将军问题是一个协议问题，拜占庭帝国军队的将军们必须全体一致的决定是否攻击某一支敌军。问题是这些将军在地理上是分隔开来的，并且将军中存在叛徒。叛徒可以任意行动以达到以下目标：
>
> 1. 欺骗某些将军采取进攻行动；
> 2. 促成一个不是所有将军都同意的决定，如当将军们不希望进攻时促成进攻行动；
> 3. 或者迷惑某些将军，使他们无法做出决定。
>
> 如果叛徒达到了这些目的之一，则任何攻击行动的结果都是注定要失败的，只有完全达成一致的努力才能获得胜利。
>
> 拜占庭假设是对现实世界的模型化，由于硬件错误、网络拥塞或断开以及遭到恶意攻击，计算机和网络可能出现不可预料的行为。
>
> ​																			引自百度百科

上述这个**拜占庭将军问题**是对于分布式系统之中最复杂、最严格的容错模型，然而实际问题之中，我们遇到更多的是分布式系统之中的某一台主机下线或者由于网络问题数据无法送达。基于这个前提，便得到了一个简化版本的**拜占庭将军问题**。在这个简化的问题之中：

1. 将军之中没有叛徒
2. 将军之间传递消息的信使所传递的消息都是可靠的
3. 信使有可能被杀导致消息无法被送达

而**Raft**算法便是一种解决这个简化**拜占庭将军问题**的共识算法，该算法具有运行效率高、理解容易的特点。这里我们简单描述一下**Raft**算法是如何解决**拜占庭将军问题**的。**Raft**算法的解决方案是从所有的大将军之中选举出一个领导人，后续所有的决定都由这个领导人来做。

假设有若干名将军，那么这个从将军之中选举领导人的过程为：

1. 为每一个将军设置一个随机时间的倒计时计时器
2. 当某一个将军的倒计时结束后，首先投票选择自己作为领导人，然后向其他所有的将军发送一条投票请求，请求其他将军选择自己作为领带人
3. 其他将军收到投票请求后，如果还没有选自己作为领导人，那么便同意对方的投票请求，并告知对方；如果已经选择了自己作为领导人，或者已经同意了其他人的投票请求，那么便拒绝该投票请求，并告知对方
4. 如果某一个将军收到了超过半数的投票，那么这个将军便会成为领导人；如果所有将军都没有收到过半数的投票，那么在下一轮倒计时结束之后，会重启开启一轮选举，直到选择出领导人。

#### Raft算法之节点角色

在**Raft**算法之中每一个节点对应于**拜占庭将军问题**之中的一位将军，也对应于分布式系统之中的一台主机或者一个进程。节点一共拥有三种状态，**Follower**、**Candidate**、**Leader**，在这三种状态之中：

1. **Follower**节点，初始状态下所有的节点都是**Follower**状态。每个**Follower**节点有一个随机时间的计时器，倒计时结束时，如果没有收到其他节点的投票请求，那么会将自己转化为**Candidate**节点，并向其他节点发送投票请求。
2. **Candidate**节点会等待其他节点的投票返回，如果该**Candidate**收到了过半数的确认投票后，会将自己升级为**Leader**节点，在一个分布式系统之中允许存在多个**Candidate**节点。
3. **Leader**节点则负责整个集群的决策选择。

每个**Raft**节点都有一个任期`Term`，在**Raft**集群刚刚启动时，每个节点的`Term`均为0。同时每个节点都有一个随机时间的倒计时计时器，每次节点的倒计时器结束倒计时的时候，便会将自己的任期`Term`加一。

## 哨兵模式相关数据结构

在*src/server.h*头文件中定义的服务器全局变量里，记录了*Redis*服务器是否是以哨兵模式在运行：

```c
struct redisServer
{
  ...
  int sentinel_mode;
  ...
};
```

`redisServer.sentinel_mode`这个字段用于标记当前的*Redis*进程是否是哨兵模式，而后续的代码之中，我们将会看到，*Redis*程序正式通过这个字段进行判断，并执行哨兵模式的逻辑的。

首先我们来简要地描述在哨兵模式之中，代码里所定义的相关结构体与用途。

| 结构体对象类型          | 含义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `sentinelState`         | 用于存储**Sentinel**服务器在运行时所需要的数据，类似`redisServer`结构体。 |
| `sentinelRedisInstance` | 表示一个**Sentinel**服务器在运行时所能感知的其他**Redis**服务器，可以是**Master**实例，可以是**Slave**实例，同样可以是其他的**Sentinel**实例。 |
| `instanceLink`          | 用于表示**Sentinel**服务器与其他运行的**Redis**服务器实例之间的网络连接。 |
| `sentinelAddr`          | 用于标记**Sentinel**服务器之中的地址信息数据。               |

### sentinelState数据

另外正如*Redis*会使用`redisServer`结构体来存储服务器在运行时所需要的数据，在哨兵实例之中也存在一个数据结构用于存储哨兵实例在运行时所需的必要数据：

```c
struct sentinelState {
    char myid[CONFIG_RUN_ID_SIZE+1];
    uint64_t current_epoch;
    dict *masters;
    int tilt;
    int running_scripts;
    mstime_t tilt_start_time;
    mstime_t previous_time;
    list *scripts_queue;
    char *announce_ip;
    int announce_port;
    unsigned long simfailure_flags;
    int deny_scripts_reconfig;
} sentinel;
```

在`sentinelState`这个数据结构之中最重要的字段便是`sentinelState.masters`这个哈希表，这里存储了当前这个哨兵实例所监控的**Master**实例。而这些被监控的实例，在哨兵模式之中是使用`sentinelRedisInstance`这个数据结构来表示的：

### sentinelRedisInstance数据

```c
typedef struct sentinelRedisInstance {
    int flags;
    char *name;
    char *runid;
    uint64_t config_epoch;
    sentinelAddr *addr;
    instanceLink *link;
    mstime_t last_pub_time;   /* Last time we sent hello via Pub/Sub. */
    mstime_t last_hello_time; /* Only used if SRI_SENTINEL is set. Last time
                                 we received a hello from this Sentinel
                                 via Pub/Sub. */
    mstime_t last_master_down_reply_time; /* Time of last reply to
                                             SENTINEL is-master-down command. */
    mstime_t s_down_since_time; /* Subjectively down since time. */
    mstime_t o_down_since_time; /* Objectively down since time. */
    mstime_t down_after_period; /* Consider it down after that period. */
    mstime_t info_refresh;  /* Time at which we received INFO output from it. */
    dict *renamed_commands;     /* Commands renamed in this instance:
                                   Sentinel will use the alternative commands
                                   mapped on this table to send things like
                                   SLAVEOF, CONFING, INFO, ... */

    /* Role and the first time we observed it.
     * This is useful in order to delay replacing what the instance reports
     * with our own configuration. We need to always wait some time in order
     * to give a chance to the leader to report the new configuration before
     * we do silly things. */
    int role_reported;
    mstime_t role_reported_time;
    mstime_t slave_conf_change_time; /* Last time slave master addr changed. */

    /* Master specific. */
    dict *sentinels;    /* Other sentinels monitoring the same master. */
    dict *slaves;       /* Slaves for this master instance. */
    unsigned int quorum;/* Number of sentinels that need to agree on failure. */
    int parallel_syncs; /* How many slaves to reconfigure at same time. */
    char *auth_pass;    /* Password to use for AUTH against master & slaves. */

    /* Slave specific. */
    mstime_t master_link_down_time; /* Slave replication link down time. */
    int slave_priority; /* Slave priority according to its INFO output. */
    mstime_t slave_reconf_sent_time; /* Time at which we sent SLAVE OF <new> */
    struct sentinelRedisInstance *master; /* Master instance if it's slave. */
    char *slave_master_host;    /* Master host as reported by INFO */
    int slave_master_port;      /* Master port as reported by INFO */
    int slave_master_link_status; /* Master link status as reported by INFO */
    unsigned long long slave_repl_offset; /* Slave replication offset. */
    /* Failover */
    char *leader;       /* If this is a master instance, this is the runid of
                           the Sentinel that should perform the failover. If
                           this is a Sentinel, this is the runid of the Sentinel
                           that this Sentinel voted as leader. */
    uint64_t leader_epoch; /* Epoch of the 'leader' field. */
    uint64_t failover_epoch; /* Epoch of the currently started failover. */
    int failover_state; /* See SENTINEL_FAILOVER_STATE_* defines. */
    mstime_t failover_state_change_time;
    mstime_t failover_start_time;   /* Last failover attempt start time. */
    mstime_t failover_timeout;      /* Max time to refresh failover state. */
    mstime_t failover_delay_logged; /* For what failover_start_time value we
                                       logged the failover delay. */
    struct sentinelRedisInstance *promoted_slave; /* Promoted slave instance. */
    /* Scripts executed to notify admin or reconfigure clients: when they
     * are set to NULL no script is executed. */
    char *notification_script;
    char *client_reconfig_script;
    sds info; /* cached INFO output */
} sentinelRedisInstance;
```

在`sentinelRedisInstance`这个数据结构之中，主要由三个部分的内容组成：

1. 用于描述实例信息的数据字段，例如地址信息以及连接数据。
2. 用于描述与该实例关联的实例的信息：
   1. 如果这是一个**Master**实例，则会记录与之对应的**Slave**实例的信息，同时由于哨兵模式也可能是由多机构成，因此有会记录监控这个**Master**实例的哨兵实例的信息。
   2. 如果这是一个**Slave**实例，则会记录对应**Master**实例的地址数据，以及复制偏移量等信息。
3. 用于处理故障迁移的数据。

#### 通用数据

| 枚举定义                   | 含义                                                         |
| -------------------------- | ------------------------------------------------------------ |
| `SRI_MASTER`               | 表示这个`sentinelRedisInstance`对象代表一台**Master**服务器。 |
| `SRI_SLAVE`                | 表示这个`sentinelRedisInstance`对象代表一台**Slave**服务器。 |
| `SRI_SENTINEL`             | 表示这个`sentinelRedisInstance`对象代表一台**Sentinel**服务器。 |
| `SRI_S_DOWN`               | 表示这个代表**Master**服务器的`sentinelRedisInstance`对象处于主观掉线的状态。 |
| `SRI_O_DOWN`               | 表示这个代表**Master**服务器的`sentinelRedisInstance`对象处于客观掉线的状态。 |
| `SRI_MASTER_DOWN`          | 表示某个代表**Sentinel**服务器的`sentineRedisInstance`对象，其所监控的**Master**处于掉线状态。 |
| `SRI_FAILOVER_IN_PROGRESS` | 只会设置给`SRI_MASTER`的对象，表示这个对象正在进行故障迁移   |
| `SRI_PROMOTED`             | 只会设置给`SRI_SLAVE`对象，表示在故障迁移流程之中，这个**Slave**服务器被**Sentinel**选择将会提升为**Master**服务器。 |
| `SRI_RECONF_SENT`          | 这个标记表示在故障迁移过程之中，已经向该**Slave**服务器发送了**SLAVEOF**命令让其复制新的**Master**。 |
| `SRI_RECONF_INPROG`        | 这个标记表示在故障迁移过程之中，这个**Slave**服务器正在复制新的**Master**的数据。 |
| `SRI_RECONF_DONE`          | 这个标记表示在故障迁移过程之中，这个**Slave**服务器已经完成对于新的**Master**数据的复制。 |
| `SRI_FORCE_FAILOVER`       |                                                              |
| `SRI_SCRIPT_KILL_SENT`     |                                                              |



#### Master对应数据

#### Slave对应数据

#### 故障迁移对应数据

1. `sentinelRedisInstance.leader`，如果对应一个表示**Master**的对象，这里记录应该执行的故障转移，也就是**Leader**的运行ID；如果对应一个表示**Sentinel**的对象，那么这里记录的是这个对象把选票投给了哪个**Sentinel**作为**Leader**
2. `sentinelRedisInstance.leader_epoch`，对应**Leader**的纪元。
3. `sentinelRedisInstance.failover_epoch`
4. `sentinelRedisInstance.failover_state`
5. `sentinelRedisInstance.leader`
6. `sentinelRedisInstance.promoted_slave`，只会用在代表**Master**的对象，存储了被**Sentinel**选出的，用于替换这个**Master**的**Slave**服务器`sentinelRedisInstance`对象的指针。

现在我们总结一下，在一个哨兵实例之中，存在着三类不同作用的字典`dict`数据结构：

1. `sentinelState.masters`，这里存储这个**Sentienl**服务器监控的所有的**Redis**的**Master**服务器。
2. `sentinelRedisInstance.sentinels`，对于一个在**Sentinel**服务器进程之中表示**Master**服的对象，这个字典`dict`之中存储了监控这个**Master**的所有的**Sentinel**服务器的`sentinelRedisInstance`对象。通过这个字典，一个**Sentinel**服务器可以自动感知到监控同一**Master**的其他**Sentinel**服务器。
3. `sentinelRedisInstance.slaves`，对于一个在**Sentinel**服务器进程之中表示**Master**服的对象，这个字典`dict`之中存储了这个**Master**对应的全部**Slave**服务器。通过这个字典，**Sentinel**服务器可以自动感知到所监控的**Master**服务器所对应的**Slave**服务器信息。

而在哨兵模式之中，是使用`instanceLink`这个结构体来表示哨兵实例与被监控实例之间的连接信息：

### instanceLink数据

```c
typedef struct instanceLink {
    int refcount;          /* Number of sentinelRedisInstance owners. */
    int disconnected;      /* Non-zero if we need to reconnect cc or pc. */
    int pending_commands;  /* Number of commands sent waiting for a reply. */
    redisAsyncContext *cc; /* Hiredis context for commands. */
    redisAsyncContext *pc; /* Hiredis context for Pub / Sub. */
    mstime_t cc_conn_time; /* cc connection time. */
    mstime_t pc_conn_time; /* pc connection time. */
    mstime_t pc_last_activity; /* Last time we received any message. */
    mstime_t last_avail_time; /* Last time the instance replied to ping with
                                 a reply we consider valid. */
    mstime_t act_ping_time;   /* Time at which the last pending ping (no pong
                                 received after it) was sent. This field is
                                 set to 0 when a pong is received, and set again
                                 to the current time if the value is 0 and a new
                                 ping is sent. */
    mstime_t last_ping_time;  /* Time at which we sent the last ping. This is
                                 only used to avoid sending too many pings
                                 during failure. Idle time is computed using
                                 the act_ping_time field. */
    mstime_t last_pong_time;  /* Last time the instance replied to ping,
                                 whatever the reply was. That's used to check
                                 if the link is idle and must be reconnected. */
    mstime_t last_reconn_time;  /* Last reconnection attempt performed when
                                   the link was down. */
} instanceLink;
```

在这个数据结构之中，使用前面我们在介绍Hiredis时介绍的异步连接`redisAsyncContext`来表示哨兵实例与*Redis*实例之间建立的连接，正如前面介绍的需要建立两个网络连接：

1. `instanceLink.cc`用于表示命令连接。
2. `instanceLink.pc`用于表示定于连接。

除此之外，`instanceLink.refcount`字段用于表示这个连接对象的引用计数，这也就意味着多个`sentinelRedisInstance`会引用相同的`instanceLink`连接对象。这主要是因为哨兵实例会在`sentinel.masters`关联其监控的所有实例；而对于**Master**实例对象`sentinelRedisInstance`会用通过`sentinelRedisInstance.sentinels`来反向关联监控它的哨兵对象实例。因此在这里便有了共享`instanceLink`对象的必要性。

下面我们来简单地介绍一下关于`instanceLink`对象上若干操作函数：
```c
instanceLink *createInstanceLink(void);
```
`createInstanceLink`函数用于创建一个`instanceLink`对象，并对相关字段使用默认值进行初始化。

```c
void instanceLinkCloseConnection(instanceLink *link, redisAsyncContext *c);
```
这个函数用于关闭给定`instanceLink`对象上的指定的Hiredis异步API连接`redisAsyncContext`。

```c
instanceLink *releaseInstanceLink(instanceLink *link, sentinelRedisInstance *ri);
```
由于实际上`instanceLink`对象可以被多个`sentinelRedisInstance`对象共享，因此如果要释放一个`instanceLink`对象时，不能简单地将这个对象分配的内存`free`掉。而是需要将对象上的引用计数`instanceLink.refcount`减一，直到减到零位置，在真正地关闭连接上的`redisAsyncContext`，并释放内存。

```c
int sentinelTryConnectionSharing(sentinelRedisInstance *ri);
```
上面这个`sentinelTryConnectionSharing`函数则可以用来实现`sentinelRedisInstance`实例对象上`instanceLink`对象的共享，遍历`sentinel.masters`之中的每一个`sentinelRedisInstance`对象，并从`sentinelRedisInstance.sentinels`查找运行ID与`ri`相同的`sentinelRedisInstance`，然后便可通过共享来复用已有的`instanceLink`对象。

前面也介绍了哨兵模式下，如果监控的*Redis*实例出现了故障，哨兵实例会将事件通知给系统管理员，通知的方式有多种多样，可以通过邮件进行通知，也可以通过其他的系统进行通知。因此在哨兵模式之中，具体的通知方式被下放给了哨兵模式的使用者，使用者可以自行实现通知脚本，并将通知脚本的路径写入哨兵模式的配置文件之中，当系统出现故障时，哨兵模式会自动创建一个子进程用于用于事件的通知。

在哨兵模式下，系统使用`sentinelScriptJob`来表示这些使用脚本来进行事件通知的任务：
```c
typedef struct sentinelScriptJob
{
  int flags;
  int retry_num;
  char **argv;
  mstime_t start_time;
  pid_t pid;
} sentinelScriptJob;
```

而在哨兵模式的服务器全局变量之中，也会使用一个双端链表`list`来存储这些通知的任务：
```c
struct sentinelState
{
  ...
  list *scripts_queue;
  int running_scripts;
  ...
}
```
除了`running_scripts`用于存储脚本任务之外，`running_scripts`字段则会用来记录正在运行的脚本任务的数量，哨兵模式下，同一个时刻最多可以运行不超过`SENTINEL_SCRIPT_MAX_RUNNING`个脚本任务。

通过下面这个函数，我们可以释放一个`sentinelScriptJob`对象分配的内存：
```c
void sentinelReleaseScriptJob(sentinelScriptJob *sj);
```

而如何发起一个脚本任务，在哨兵模式之中则是使用一个队列来实现的，如果需要启动一个脚本任务，则在`sentinel.scripts_queue`之中插入一个表示任务的`sentinelScriptJob`，哨兵模式会在心跳之中自动检查队列并创建子进程执行脚本任务。

将一个脚本任务加入调度队列，是通过`sentinelScheduleScriptExecution`这个函数来执行的：
```c
void sentinelScheduleScriptExecution(char *path, ...);
```
这个函数需要调用者给出脚本在文件系统之中的路径以及运行脚本所需要的脚本参数。同时需要注意的是，哨兵模式的脚本任务调度队列`sentinel.scripts_queue`具有一个最大的长度上限的`SENTINEL_SCRIPT_MAX_QUEUE`，当调度队列长度达到上限的话，会删除队列里还没有被执行的任务中最早被加入队列的任务。

而在心跳之中运行脚本任务则是通过下面这个函数来执行的：
```c
void sentinelRunPendingScripts(void);
```
这个函数会通过`fork`系统调用创建子进程来运行指定的任务脚本，并在哨兵父进程之中更新对应任务的`sentinelScriptJob.pid`，以及系统当前运行脚本任务计数器`sentinel.running_scripts`。

由于脚本任务是在子进程之中异步运行的，因此哨兵父进程需要定期查询子进程的运行状态，在系统心跳之中哨兵模式会使用`sentinelCollectTerminatedScripts`来执行这个检查逻辑：
```c
void sentinelCollectTerminatedScripts(void);
```
该函数会通过`wait3`系统调用检查子进程的状态，对于成功运行结束的脚本任务哨兵模式会将其从`sentinel.scripts_queue`队列之中删除；而对于那些因为信号被异常中断或者脚本内部返回退出码`1`时，这些异常任务将在下一个心跳时被重新执行。

最后如果某个脚本任务运行的事件过长，则可以在系统心跳之中通过`sentinelKillTimedoutScripts`进行检查，并通过`kill`系统调用将对应的子进程杀掉。

最后用户可以通过`SENTINEL PENDING-SCRIPTS`命令来检查当前的哨兵实例上运行的脚本任务的数据：
```c
void sentinelPendingScriptsCommand(client *c);
```
这个函数会遍历`sentinel.scripts_queue`队列，将队列之中的脚本任务的具体信息返回给执行命令的用户。

## 哨兵模式代码实现

### Sentinel实例的初始化

让我们使用`redis-sentinel sentinel.conf`这个命令来启动一个**Sentinel**时，我们可以在*src/server.c*源文件的入口`main`函数中了解到整个**Sentinel**的启动过程：

```c
int main(int argc, char **argv)
{
  ...
	server.sentinel_mode = checkForSentinelMode(argc,argv);
  ...
  if (server.sentinel_mode)
  {
    initSentinelConfig();
    initSentinel();
  }
  ...
  if (argc >= 2)
  {
    ...
    loadServerConfig(configfile,options);
  	...
  }
  ...
  if (!server.sentinel_mode)
  {
    ...
  }
  else
  {
    InitServerLast();
    sentinelIsRunnig();
  }
  ...
  aeSetBeforeSleepProc(server.el,beforeSleep);
  aeSetAfterSleepProc(server.el,afterSleep);
  aeMain(server.el);
  aeDeleteEventLoop(server.el);
  return 0;
}
```

这里我们可以看到在**Sentinel**实例的入口`main`函数之中，执行了如下的逻辑：

1. 通过`checkForSentinelMode`检查当前的**Redis**是否是以**Sentinel**模式启动的，这里当使用`redis-sentinel`或者`redis-server --sentinel`来启动时，都会被认为是**Sentinel**模式。
2. 通过`initSentinelConfig`以及`initSentinel`来执行**Sentinel**的初始化逻辑，包括对`sentinelState`结构体之中数据字段的初始化，以及可执行名字的字典`redisServer.commands`。
3. 在`loadServerConfig`函数之中，通过调用`sentinelHandleConfiguration`函数，读取配置文件，对**Sentinel**实例的配置信息进行初始化。这里最为重要的一个步骤是，读取配置文件之中监控**Master**实例的配置`sentinel monitor <name> <host> <port> <quorum> `，来初始化该**Sentinel**监听列表字典`sentinelState.masters`。这里会调用`createSentinelRedisInstance`函数为每一个被监听的**Master**创建一个对应的`sentinelRedisInstance`实例。
4. 在最后，通过调用的`sentinelIsRunning`函数，为**Sentinel**分配一个随机运行ID，并通过`sentinelGenerateInitialMonitorEvents`函数向`+monitor`频道上发布一条消息。

#### **Sentinel**数据初始化

我们前面在介绍**Redis**命令系统之中介绍过，**Redis**通过定义在*src/server.c*源文件之中的`redisCommand`数组来初始化**Redis**的命令数据：

```c
struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,"as",0,NULL,0,0,0,0,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    ...
};
```

而**Sentinel**实例是一种按照特殊启动方式启动的**Redis**服务器，其只能执行特定的，专属于**Sentinel**模式下的命令，这个命令的数据则被定义在*src/sentinel.c*源文件之中：

```c
struct redisCommand sentinelcmds[] = {
    {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
    {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
    {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
    {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
		...
};
```

在**Sentinel**启动后的初始化函数`initSentinel`之中，会将这个`sentinelcmds`数组插入到服务器的命令字典`server.commands`之中：

````c
void initSentinel(void) {
  ...
	dictEmpty(server.commands,NULL);
  for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {
    int retval;
    struct redisCommand *cmd = sentinelcmds+j;
		retval = dictAdd(server.commands, sdsnew(cmd->name), cmd);
		serverAssert(retval == DICT_OK);
	}
  ...
}
````

#### Sentinel配置加载

```c
void loadServerConfigFromString(char *config)
{
  ...
	else if (!strcasecmp(argv[0],"sentinel")) {
		if (argc != 1) {
			if (!server.sentinel_mode) {
				err = "sentinel directive while not in sentinel mode";
				goto loaderr;
			}
			err = sentinelHandleConfiguration(argv+1,argc-1);
			if (err) goto loaderr;
		}
	}
  ...
}
```

**Sentinel**在加载配置时，会读取配置文件之中所有以`sentinel`开头的行作为配置项，通过`sentinelHandleConfiguration`函数进行加载，在所有的配置项之中，有两项是与建立同**Master**的连接相关：

```
sentinel monitor <master-name> <host> <port> <quorum>
sentinel auth-pass <master-name> <password>
```

其中，如果指定的**Master**没有密码认证的话，`auth-pass`这一项可以省略。

```c
char *sentinelHandleConfiguration(char **argv, int argc);
sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master);
```

在`sentinelHandleConfiguration`函数之中，当读取到一条`monitor`配置项的时候，便会调用`createSentinelRedisInstance`函数，根据配置项之中的内容，创建一个`SRI_MASTER`类型的`sentinelRedisInstance`对象，并将之加入到`sentinelState.masters`字典之中。不过这里只是创建在代表**Master**的实例对象，但是并没有建立**Sentinel**与**Master**之前的实际网络连接。

在读取到`auth-pass`配置项时，会通过`<master-name>`在`sentinelState.masters`之中产查找到对应的`sentinelRedisInstance`实例，并将密码赋值给`sentinelRedisInstance.auth_pass`字段，用于后续建立连接之用途。

#### Sentinel完成初始化

除了上述`initSentinel`初始化函数之外，**Sentinel**还提供了另外一个函数用于执行剩余的初始化逻辑:

```c
void sentinelIsRunning(void);
```

这个函数大致会执行三种逻辑：

1. 调用`getRandomHexChars`为**哨兵**实例初始化一个随机的运行ID，并将之写入`sentinleState.myid`之中。
2. 调用`sentinelGenerateInitialMonitorEvents`函数，为**Sentinel**所监控的每一个**Master**实例生成一个初始的监控事件。

### Sentinel建立连接

前面我们了解到，在**Sentinel**读取配置文件时所创建的`sentinelRedisInstance`对象后，实际上并没有与对应的**Master**建立网络连接。建立网络连接的过程实际上是由**Sentinel**的心跳机制负责的。

```c
void sentinelTimer(void);
```

上面这个函数负责整个**Sentinel**的心跳机制，这里我们主要来看一下其中所负责的建立网络连接的过程。

而负责建立连接的是：

```c
void sentinelReconnectInstance(sentinelRedisInstance *ri)
{
    if (ri->link->disconnected == 0) return;
    ...
}
```

这个函数从函数名可以看出来，它的一个重要的用途便是用于对断开的网络连接进行重连操作。当然在**Sentinel**实例启动之后，这个函数也会负责为**Sentinel**所监控的每一个`sentinelRedisInstance`对象建立网络连接。这里会为每个`sentinelRedisInstance`建立两个网络连接，分别用于传输**Redis**的查询命令，以及用于订阅/发布**Redis**消息。

#### 命令连接的建立

**Sentinel**为一个`sentinelRedisInstance`建立命令连接的步骤是按照如下几步进行的：

1. 通过**Hiredis**之中异步建立连接的`redisAsyncConnectBind`函数，按照`sentinelRedisInstance`对象之中配置的地址信息，异步地建立网络连接。
2. 初始化`sentinelRedisInstance`对象上的连接信息，包含`instanceLink.pending_commands`以及`instanceLink.cc_conn_time`信息。
3. 通过`redisAeAttach`，将这个连接数据加入到**Redis**的主事件循环`redisServer.el`之中。
4. 为这个异步连接设置连接建立的回调函数`sentinelLinkEstablishedCallback`以及断开连接的回调函数`sentinelDisconnectCallback`
5. 调用`sentinelSendAuthIfNeeded`函数尝试进行认证，如果`sentinelRedisInstance`实例对应的**Redis**服务器需要进行密码认证，那么这个函数会通过`redisAsyncCommand`函数，发起一条异步的**AUTH**命令用于验证密码信息。
6. 调用`sentinelSendPing`函数，异步地发起一条**PING**命令，并设置**PING**命令返回的回调函数`sentinelPingReplyCallback`。

不过需要注意的是，`sentinelReconnectInstance`函数在建立连接后所尝试发送**AUTH**命令以及**PING**命令均是异步流程，此时并没有真正地发送出去，要等到连接正式建立之后，才会真正地将命令数据发送到网络上。

#### 订阅连接的建立

`sentinelRedisInstance`对象订阅连接的建立过程与命令连接的建立过程相似，不同之处在于建立订阅连接不会清空`instanceLink.pending_commands`字段上的计数；同时在`sentinelSendAuthIfNeeded`认证完密码之后，**Sentinel**还会向`sentinelRedisInstance`对象对应的**Redis**服务器发送**SUBSCRIBE**命令，订阅该**Redis**服务器上的`__sentinel__:hello`这个频道的消息，并设置订阅消息的回调函数`sentinelReceiveHelloMessages`。

### Sentinel的服务发现机制

通过前面的介绍，以及代码之中提供的**Sentinel**默认配置文件，我们不难发现我们仅可以配置**Sentinel**实例监控的**Master**服务器的信息，并没有对应**Slave**服务器的信息，以及**Sentinel**分布式集群之中其他**Sentinel**实例的信息。那么在这一部分将会介绍一台**Sentinel**实例是如何发现其他的**Slave**服务器以及**Sentinel**服务器的。

与建立与`sentinelRedisInstance`对象对应的**Redis**服务器的网络连接的机制一样，**Sentinel**实例的服务发现机制也是心跳`sentinelTimer`之中完成的。

在**Sentinel**的心跳机制之中，会遍历系统之中的每一个`sentinelRedisInstance`对象，调用`sentinelSendPeriodicCommands`函数，向该对象所对应的**Redis**服务器发送周期性命令，这其中包括：

1. 向所有非`SRI_SENTINEL`对象发送**INFO**命令。
2. 向所有的`sentinelRedisInstance`对象发送**PING**命令。
3. 向所有的`sentinelRedisInstance`对象的`__sentinel__:hello`频道上发布**Hello**信息。

#### Slave服务器的发现机制

**Sentinel**服务器会在`sentinelSendPeriodicCommands`函数之中向所有的非`SRI_SENTINEL`类型的`sentinelRedisInstance`对象发送**INFO**命令，通过这个机制，**Sentinel**实现了对**Slave**服务器的自动发现。我们先来看一下针对一个**Master**服务器发送**INFO**命令后的返回数据：

```
# Server
...
multiplexing_api:epoll
atomicvar_api:atomic-builtin
gcc_version:4.8.5
process_id:31734
run_id:2f2a6b4e4097fa114630d4e7370bd33c914e37af
...
# Replication
role:master
connected_slaves:1
slave0:ip=10.236.192.20,port=6379,state=online,offset=2596808391,lag=0
master_replid:1c1b0f14e1207ca9b2c54627bcf709fdb27967dd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2596808391
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2595759816
repl_backlog_histlen:1048576
...
```

同时我们可以来看一下对于一个**Slave**服务器执行**INFO**命令之后的返回信息：

```
# Server
...
run_id:b9fd714edd72716912d18adcd8e12bd0955dfffc
...
# Replication
role:slave
master_host:10.236.100.244
master_port:6379
master_link_status:up
master_last_io_seconds_ago:2
master_sync_in_progress:0
slave_repl_offset:2597448187
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:1c1b0f14e1207ca9b2c54627bcf709fdb27967dd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2597448187
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2596399612
repl_backlog_histlen:1048576
...
```

这里我们可以看到**INFO**命令返回数据中包含了主从集群的各项配置信息以及数据，包括对应这个**Slave**服务器针对**Master**服务器的复制偏移量`slave_repl_offset`，**Sentinel**则是通过`sentinelRefreshInstanceInfo`这个函数读取**INFO**命令的返回来实现对于**Slave**服务器的发现的。

```c
void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info);
```

如果对于一个`SRI_MASTER`类型的`sentinelRedisInstance`对象，这个函数会从**INFO**命令的返回数据之中读取出对应的**SLAVE**服务器的连接信息。并使用从中解析出来的`ip`以及`port`信息，在`SRI_MASTER`的`sentinelRedisInstance.slaves`这个字典之中查找是否存在对应**SLAVE**服务器的`sentinelRedisInstance`对象，如果不存在就通过`createSentinelRedisInstance`函数创建一个表示对应**SLAVE**服务器的`sentinelRedisInstance`对象，并将其加入到对应**Master**服务器的`sentinelRedisInstance.slaves`字段之中。

通过这样一个机制，**Sentinel**服务器可以自动发现**Master**服务器对应的**Slave**服务器。

另外，针对一个**SLAVE**对象定期执行**INFO**命令后，通过命令返回的信息，可以用于设置以及更新对应的`sentinelRedisInstance`结构体之中关于**SLAVE**服务器的相关数据字段。

#### Sentinel服务器的发现机制

监控同一个**Master**服务器的**Sentinel**服务器是通过**Redis**的发布/订阅机制来实现相互发现的。简要而言，**Sentinel**服务器在建立与**Master**服务器的连接之后，会订阅**Master**服务器的`__sentinel__:hello`频道；**Sentinel**服务器会周期性地向所监控的**Master**服务器的`__sentinel__:hello`频道发布**Hello**消息，这条**Hello**消息会通报这个**Sentinel**服务器的ip地址、端口号、运行ID等参数信息，以及对应这个**Master**的参数信息。这样一来，这个**Sentinel**的这条**Hello**消息就可以被监控同一个**Master**的其他所有**Sentinel**服务器接收，如此一来这个**Sentinel**集群便可以实现彼此之间的相互发现。

```c
void sentinelProcessHelloMessage(char *hello, int hello_len);
```

**Sentinel**便是通过上面这个`sentinelProcessHelloMessage`函数实现对订阅频道之中接收到的**Hello**信息的处理的。我们可以首先看一下，**Hello**消息的具体内容：

```
"message"
"__sentinel__:hello"
"127.0.0.1,26379,0ff83f9b554269ce614f4e792aa5169eeb444f21,0,mymaster,127.0.0.1,6379,0"
```

在这条消息之中，各个字段数据的含义为：

```
<sentinel_ip>,<sentinel_port>,<sentinel_runid>,<current_epoch>, <master_name>,<master_ip>,<master_port>,<master_config_epoch>
```

`sentinelProcessHelloMessage`这个函数发现其他**Sentinel**的逻辑为：

1. 调用`sentinelGetMasterByName`函数，根据`<master_name>`查找该**Master**对应的`sentinelRedisInstance`的对象。
2. 调用`getSentinelRedisInstanceByAddrAndRunID`函数，根据`<sentinel_ip>`、`<sentinel_port>`以及`<seninel_runid>`在**Master**的`sentinelRedisInstance.sentinels`字典之中查找对对应的**Sentinel**的`sentinelRedisInstance`对象。此处的查找必须做到地址、端口、运行ID完全匹配才返回对应的对象。
3. 如果没有找到对应的**Sentinel**对对象，那么根据**Hello**消息之中通告的参数信息，调用`createSentinelRedisInstance`创建一个`SRI_SENTINEL`类型的对象，并将其添加到对应**Master**的`sentinelRedisInstance.sentinels`字典之中。
4. 通过`sentinelTryConnectionSharing`函数，尝试共享底层网络连接的`instanceLink`数据。后面我们会介绍为什么需要共享网络连接。
5. 更新数据
6. 在后续心跳之中，通过`sentinelReconnectInstance`建立网络连接。

除此之外，每当**Sentinel**服务器从订阅连接上接收到消息进而触发`sentinelReceiveHelloMessages`回调函数时，会更新`instanceLink.pc_last_activity`这个时间戳，用于后续检测订阅连接可用性的依据。

##### 共享网络连接

```c
void sentinelReconnectInstance(sentinelRedisInstance *ri)
{
    ...
    instanceLink *link = ri->link;
    ...
    if (link->cc == NULL)
    {
        link->cc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);
        ...
    }
    
    if ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && link->pc == NULL)
    {
        link->pc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);
        ...
    }
}
```

在上述的`sentinelReconnectInstance`函数的代码片段之中，我们可以发现如果**Sentinel**服务器尝试向另外一个代表**Sentinel**的`sentinelRedisInstance`建立连接时，只会创建命令连接，而不会创建订阅连接。也就是说**Sentinel**集群之中的服务器节点彼此之间只有一个命令连接。

而这个特性便是前面所说的共享底层网络连接的由来。设想我们有**SentinelA**以及**SentinelB**两个服务器，监控**Master1**以及**Master2**两个服务器。当**SentinelA**通过**Master1**发现了**SentinelB**之后，为**SentinelB**创建一个`sentinelRedisInstance`对象，并通过这个对象建立与**SentinelB**服务器的底层网络连接；后续**SentinleA**又通过**Master2**发现了**SentinelB**，但是此前两台服务器之间已经建立了一条网络连接，我们没有必要为两个服务器再建立一条网络连接，基于这个理由，我们便可以再新的`sentinelRedisInstance`对象上，复用前面已经创建好的网络连接`instanceLink`。

#### 发现机制的其他用途

前面我们可以看到，**Sentine**通过周期性地点**INFO**命令可以用来发现所监控的**Master**服务器对应的**Slave**服务器。除此之外这个机制可以用于刷新**Sentinel**服务器所监控的其他服务器的当前状态。例如每次收到**INFO**命令的返回数据时，**Sentinel**服务器会将当前的时间更新到对应的`sentinelRedisInstance`对象的`info_refresh`字段之中；同时对于**Slave**服务器的返回数据，**Sentinel**也会从中解析出该**Slave**对于**Master**的复制偏移量，并将之更新到对应的`sentinelRedisInstance.slave_repl_offset`字段之中。

### 故障发现

前面在介绍**Sentinel**的服务器发现机制时，我们提到过，**Sentinel**会在心跳之中周期性调用`sentinelSendPing`函数向其所连接的所有包括**Master**、**Slave**、**Sentinel**服务器发送**PING**探测命令，并通过注册回调函数`sentinelPingReplyCallback`来处理返回信息。

上述这套机制的用于是通过发送**PING**命令以及处理命令的返回数据来刷新对应网络连接`instanceLink`的计时时间戳，用于后续检测连接状态。这其中所涉及到的`instanceLink`中的计时器包括：

1. `instanceLink.last_avail_time`，用于记录对应的**Redis**服务器实例上一次的可用时间，每次收到**PING**命令的返回时，会刷新该时间戳数据。
2. `instanceLink.act_ping_time`，用记录执行**PING**命令的时间戳，每次发送**PING**命令时被设置，收到**PING**命令返回后，这个时间戳记录会被清零。
3. `instanceLink.last_ping_time`，记录上一次执行**PING**命令的时间戳，每次发送**PING**命令后，会更新该时间戳。
4. `instanceLink.last_pong_time`，记录上一次收到**PING**命令返回的时间戳，每次在`sentinelPingReplyCallback`回调函数之中会更新该字段。

#### 主观下线

**Sentinel**服务器检测其所连接的其他的**Redis**服务器的是否处于**主观下线**状态则是在心跳机制之中通过调用`sentinelCheckSubjectivelyDown`这个函数实现的。

```c
void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri);
```

该函数会检查对应的`sentinelRedisInstance`对象网络连接的可用性，并更新该对象的状态。

首先`sentinelCheckSubjectivelyDown`函数会检查命令连接`instanceLink.cc`的可用状态，如果已经向该`sentinelRedisInstance`对应的**Redis**服务器发送过**PING**命令，也就是`instanceLink.act_ping_time`上的时间戳不为0；同时在配置项`down-after-milliseconds`规定的一半的时间内没有收到对方的返回数据时，那么便认为这个对象的命令连接已经掉线，会调用`instanceLinkCloseConnection`函数关闭这个对象上的命令连接。

接下来`sentinelCheckSubjectivelyDown`会检查对应的订阅连接`instanceLink.pc`的可用状态。主要检查截止上一次从订阅连接上接收到消息的时间`instanceLink.pc_last_activity`，如果超过三倍的`SENTINEL_PUBLISH_PERIOD`时间没有接收到新的消息，那么便认为这条订阅连接掉线，调用`instanceLinkCloseConnection`函数关闭这个对象上的订阅连接。

在完成了对于命令连接以及订阅连接的检查后，**Sentinel**便会对该`sentinelRedisInstance`对象的在线状态进行判断。如果这个`sentinelRedisInstance`对象超过`down-after-milliseconds`时间没有接收到**PING**命令的返回，或者已经因为异常而断开网络连接超过`down-after-milliseconds`没有重新建立间接，便将其设置为**主观下线**的状态。这会在`sentinelRedisInstance.flags`字段上设置`SRI_S_DOWN`掩码用于标记。

#### 客观下线

**主观下线**可能只是该**Sentinel**与对应的**Redis**服务器之间的网络连接出现波动，并不一定意味着对应的**Redis**服务器真的出现故障下线，因此**Sentinel**服务器需要从其他的**Sentinel**服务器中获取信息，用于判断究竟是**Redis**服务器真的出现问题，还是仅仅是当前**Sentinel**与对应的**Redis**服务器出现网络波动。

**Sentinel**服务器会在心跳机制之中，针对每一个代表**Master**服务器的`sentinelRedisInstance`对象执行状态查询以及**客观下线**的判断：

```c
void sentinelHandleRedisInstance(sentinelRedisInstance *ri)
{
  ...
	if (ri->flags & SRI_MASTER)
  {
		sentinelCheckObjectivelyDown(ri);
		...
		sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
	} 
}
```

##### 状态询问

所谓查询**Master**服务器的状态，就是向同样监控该**Master**服务器的**Sentinel**服务器发起状态查询请求，用于确定该**Master**服务器的状态。**Sentinel**服务器之中执行状态查询的逻辑主要在下面三个函数接口之中：

```c
void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags);
void sentinelReceiveIsMasterDownReply(redisAsyncContext *c, void *reply, void *privdata);
void sentinelCommand(client *c);
```

首先我们会调用`sentinelAskMasterStateToOtherSentinels`函数，如果`master`对象处于`SRI_S_DOWN`这个**客观下线**的状态，那么会遍历对应**Master**的`sentinelRedisInstance.sentinels`这个字典数据之中的每一个代表**Sentinel**服务器的`sentinelRedisInstance`对象，向其对应的**Sentinel**服务器发起**SENTINEL**查询命令，同时注册命令返回数据的回调处理函数`sentinelReceiveIsMasterDownReply`，查询命令的格式为：

```
SENTINEL is-master-down-by-addr <master-ip> <master-port> <current-epoch> <runid>
```

此时由于**Master**服务器仅仅是**主观下线**的状态，因此**SENTINEL**命令之中，`<runid>`所携带的参数并非是当前**Sentinel**的运行ID，而是特殊符号`*`。

对应的**Sentinel**服务器在收到查询命令时，则会通过`sentinelCommand`函数来处理这条特殊的**SENTINEL**查询命令。`sentinelCommand`函数会根据命令参数之中的`master-ip`以及`master-port`查找到对应的**Master**的`sentinelRedisInstance`对应，检查该对象是也处于`SRI_S_DOWN`这个**主观下线**的状态，并将这个状态返回给发起查询请求的**Sentinel**服务器。

**Sentinel**服务器接收到其他**Sentinel**服务器的**SENTINEL**命令返回数据后，会触发`sentinelReceiveIsMasterDownReply`回调函数。在`sentinelReceiveIsMasterDownReply`函数中如果返回该**Master**服务器**主观下线**，那么会将代表该**Sentinel**的`sentinelRedisInstance`对象的`flags`字段设置上`SRI_MASTER_DOWN`，表示这个**Sentinel**实例也认为这个**Master**下线，算作一次投票。

##### 客观下线判断

**Sentinel**服务器会在心跳机制之中，通过一下调用`sentinelCheckObjectivelyDown`函数来检查一个**Master**服务器是否从**主观下线**转为**客观下线**。

如果一个**Master**服务器的`sentinelRedisInstance`对象的`flags`对象具有`SRI_S_DOWN`掩码，那么**Sentinel**就会遍历这个**Master**的`sentinelRedisInstance.sentinels`字典中的每一个`SRI_SENTINEL`类型的`sentinelRedisInstance`对象。统计其中具有`SRI_MASTER_DOWN`掩码的对象的个数，如果这个数量超过了配置文件之中配置的最低票数，那么便认为这个**Master**服务器处于**客观下线**的状态。

一旦**Sentinel**服务器认为某个**Master**服务器处于**客观下线**的状态，那么这个**Sentinel**便会用于**Raft**算法对该故障的**Master**服务器执行故障迁移的逻辑。

### 故障迁移

**Sentinel**会在心跳之中检查其所监控的**Master**服务器是否处于**客观下线**的状态需要进行故障迁移。

```c
void sentinelHandleRedisInstance(sentinelRedisInstance *ri)
{
  ...
  /* Only masters */
	if (ri->flags & SRI_MASTER)
	{
		sentinelCheckObjectivelyDown(ri);
		if (sentinelStartFailoverIfNeeded(ri))
			sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);
		...
	}
}
```

这里**Sentinel**会通过`sentineStartFailoverIfNeeded`函数来判断对应的**Master**节点实例`ri`是否需要进行故障迁移。其判断的策略为：

1. **Master**服务器对应的实例必须处于**客观下线**状态。
2. 没有正在进行的故障迁移
3. 最近没有正在尝试的故障迁移流程。

如果通过判断，确认该**Master**需要进行故障迁移，那么便会通过`sentinelStartFailover`函数开启故障迁移流程：

```c
void sentinelStartFailover(sentinelRedisInstance *master);
```

这个函数会执行如下的逻辑：

1. 为**Master**对应的`sentinelRedisInstance`对象的`sentinelRedisInstance.flags`字段上添加`SRI_FAILOVER_IN_PROGRESS`掩码，表示这个**Master**实例正在进行故障迁移；将`sentinelRedisInstance.failover_state`字段设置为`SENTINEL_FAILOVER_STATE_WAIT_START`，表示当前正在等待故障迁移正式开始
2. 将**Sentinel**的当前纪元`sentinelState.current_epoch`加一，并将之赋值给**Master**实例的`sentinelRedisInstance.failover_epoch`字段上。
3. 设置**Master**实例上的故障迁移相关时间戳，包括`sentinelRedisInstance.failover_start_time`以及`sentinelRedisInstance.failover_state_change_time`。

#### 选择Leader

##### 发起选举

在**Sentinel**为对应的**Master**对象开启了故障迁移流程之后，首先要做的便是基于**Raft**算法进行选举Leader的过程了，这一步是通过`sentinelAskMasterStateToOtherSentinels`函数进行的。前面我们在介绍检查**Master**是否**客观下线**时，便是通过这个函数向监控相同**Master**的其他**Sentinel**发送**SENTINEL**命令来查询对应**Master**的状态的，

```
SENTINEL is-master-down-by-addr <master-ip> <master-port> <current-epoch> <runid>
```

开启选择Leader的过程也是通过上面这个格式的命令来开启的，只不过`<runid>`这一项在这种情况下会发送**Sentinel**的运行ID。

```c
void sentinelCommand(client *c)
{
  ...
  if (!strcasecmp(c->argv[1]->ptr,"is-master-down-by-addr"))
  {
    ...
    if (ri && ri->flags & SRI_MASTER && strcasecmp(c->argv[5]->ptr,"*"))
    {
      leader = sentinelVoteLeader(ri,(uint64_t)req_epoch, c->argv[5]->ptr, &leader_epoch);
    }
    ...
  }
  ...
}
```

这里我们可以看到，如果对应`<runid>`这一项为正常的运行ID的话，便会调用`SentinelVoteLeader`函数进行选择Leader：

```c
char *sentinelVoteLeader(sentinelRedisInstance *master, uint64_t req_epoch, char *req_runid, uint64_t *leader_epoch);
```

这个函数会接收请求投票的**Sentinel**服务器对应的请求纪元`req_epoch`以及该**Sentinel**的运行ID `req_runid`，接下来`sentinelVoteLeader`函数便会按照**Raft**算法的相关规则，进行投票。其规则为：

1. 如果请求纪元`req_epoch`大于当前**Sentinel**的当前纪元`sentinelState.current_epoch`，那么用请求纪元来替换**Sentinel**的当前纪元。
1. 如果当前**Sentinel**监控的**Master**实例的记录的**Leade**纪元`sentinelRedisInstance.leader_epoch`小于请求纪元`req_epoch`，并且**Sentinel**的当前纪元`sentinelState.current_epoch`不大于请求纪元`req_epoch`的话，那么这个**Sentinel**便会为发起请求的**Sentinel**投票，选举它作为**Leader**，同时将投票记录更新到**Master**服务器对应的`sentinelRedisInstance`对象上。这样一来可以防止该**Sentinel**重复地进行投票。
1. 最后，无论是否响应了投票请求与否，**Sentinel**都会将当前记录的`sentinelRedisInstance.leader`这个**Leader**运行ID，以及`leader_epoch`做为**SENTINEL**命令的返回发送给请求一方的**Sentinel**服务器。

```
<down-state> <leader-runid> <vote-epoch>
```

发起投票请求的**Sentinel**服务器在接收到返回数据后，会通过`sentinelReceiveIsMasterDownReply`函数来处理返回数据：

```c
void sentinelReceiveIsMasterDownReply(redisAsyncContext *c, void *reply, void *privdata)
{
  sentinelRedisInstance *ri = privdata;
  ...
  if (strcmp(r->element[1]->str,"*"))
  {
    ri->leader = sdsnew(r->element[1]->str);
    ri->leader_epoch = r->element[2]->integer;
  }
}
```

这里如果返回数据之中的`<leader-runid>`不是特殊的`*`，那么便认为对方已经在选举之中进行了投票，并将投票结果赋值给代表响应方**Sentinel**的对象。

##### 确定Leader

接下来**Sentinel**服务器便会在心跳之中，通过状态机函数来处理确定**Leader**的逻辑：

```c
void sentinelFailoverStateMachine(sentinelRedisInstance *ri);
```

前面我们介绍了，在**Sentinel**为一个**Master**服务器开启故障迁移流程后，会首先将这个**Master**对象上的故障迁移状态`failover_state`设置成`SENTINEL_FAILOVER_STATE_WAIT_START`，表示其在等待**Sentinel**集群选出**Leader**来正式开始故障迁移流程。在状态机迁移函数`sentinelFailoverStateMachine`之中，当发现实例处于`SENTINEL_FAILOVER_STATE_WAIT_START`状态，那么将会调用`sentinelFailoverWaitStart`函数来检测是否已经选举出负责此次故障迁移的**Leader**服务器。

```c
void sentinelFailoverWaitStart(sentinelRedisInstance *ri);
char *sentinelGetLeader(sentinelRedisInstance *master, uint64_t epoch);
int sentinelLeaderIncr(dict *counters, char *runid);
```

上述三个函数的主要逻辑便是确定在当前的选举过程之中，是否已经选举出了**Leader**服务器节点。这其中`sentinelLeaderIncr`函数用于向`counters`这个统计数据之中增加给定`runid`的**Sentinel**服务器实例在选举**Leader**之中所获得的投票数，这个统计数据将最终用于确定**Leader**服务器。

而函数`sentinelGetLeader`则是会按照指定的纪元`epoch`在给定的**Master**服务器对象所绑定的**Sentinel**服务器对象之中扫描是否存在已经被选出的**Leader**。

首先`sentinelGetLeader`函数会遍历的**Master**对象的`sentinelRedisInstance.sentinels`字典之中监控该**Master**的所有**Sentinel**实例，收集各个**Sentinel**对象的得票情况，同时通过`sentinelRedisInstance.leader_epoch`与`sentinelState.current_epoch`比较来确保收集到的都是针对当前的**客观下线**而进行的投票。

接下来`sentinelGetLeader`会查找得票最多的**Sentinel**对象，如果找到，那么将自己的一票投给得票最高的**Sentinel**对象，否则便投票给自己。

```c
void sentinelHandleRedisInstance(sentinelRedisInstance *ri)
{
  ...
	if (ri->flags & SRI_MASTER) {
		sentinelCheckObjectivelyDown(ri);
		if (sentinelStartFailoverIfNeeded(ri))
			sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);
		sentinelFailoverStateMachine(ri);
		sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
	}
}
```

回看处理`sentinelRedisInstance`对象的心跳逻辑的代码片段，这里可以保证如果一个**Sentinel**服务器率先发现自己所监控的**Master**服务器**客观下线**，那么它会优先投票给自己。这也印证**Raft**算法投票逻辑之中，当一个**Follower**节点的倒计时器结束时，如果没有投票给其他节点，那么便会投票给自己。

最后`sentinelGetLeader`函数会检查得票最高的**Sentinel**对象的票数是否过半，如果过半说明该对象便是最终选出的**Leader**，函数返回该对象的运行ID；否则说明暂时没有完成投票，那么会在下一次心跳之中继续调用该接口扫描是否选出了**Leader**。

而`sentinelFailoverWaitStart`函数则通过调用`sentinelGetLeader`在心跳之中检查自己是否被选举为**Leader**，如果超过`SENTINEL_ELECTION_TIMEOUT`时间后，**Sentinel**发现自己仍然没有被选举为**Leader**，那么便认为这个**Sentinel**选举失败。否则说明自己被选举为**Leader**，那么后续将由这个**Sentinel**服务器负责故障迁移后续的流程，包括将对应**Master**服务器的故障迁移状态`sentinelRedisInstance.failover_state`字段切换为`SENTINEL_FAILOVER_STATE_SELECT_SLAVE`，这表示后续要从**Master**服务器对应的**Slave**服务器之中选择一个服务器将其提升为**Master**服务器。

#### 选择Slave服务器

**Sentinel**会在心跳机制之中，检查对应的**Master**服务器的`sentinelRedisInstance.failover_state`字段的状态，发现这个字段是`SENTINEL_FAILOVER_STATE_SELECT_SLAVE`，便会进入选择**Slave**服务器的流程。

```c
void sentinelFailoverSelectSlave(sentinelRedisInstance *ri);
sentinelRedisInstance *sentinelSelectSlave(sentinelRedisInstance *master);
```

函数`sentinelSelectSlave`会从指定的`master `实例的`sentinelRedisInstance.slaves`字典之中选择出合适的**Slave**服务器来将之提升为**Master**服务器，其收集的逻辑为：

1. 对应的**Slave**服务器不应该处于断开连接、**主观下线**、**客观下线**这些状态。
2. 在5个**PING**命令周期时间内（定义在`SENTINEL_PING_PERIOD`），该**Slave**有响应过**Sentinel**服务器的**PING**命令。
3. 在3个**INFO**命令周期时间内（定义在`SENTINEL_INFO_PERIOD`），该**Slave**有响应过**Sentinel**的**INFO**命令。
4. **Slave**对象上记录的**Master**掉线时间`sentinelRedisInstance.master_link_down_time`不应该超过`(now - master->s_down_since_time) + (master->down_after_period * 10)`。一般来说从**Sentinel**的视角来看**Master**服务器已经处于了**客观下线**的状态的话，那么在**Slave**服务器一侧同样发现**Master**断开连接的时间不会超过10倍的`down_after_period`时间。这里其实是一个潜规则，其中心思想是当**Master**处于不可用的状态时，**Slave**对此的感知可能会之后，但是终究不会超过一定的时间。
5. **Slave**服务器的`slave_priority`不能为0，否则将会忽略这个**Slave**服务器。

**Sentinel**在收集到满足条件的**Slave**服务器对象后，按照如下的规则对这些对象进行排序：

1. 具有更低的优先级`sentinelRedisInstance.slave_priority`。
2. 具有更高的已处理复制偏移量`sentinelRedisInstance.slave_repl_offset`。
3. 按照字典顺序具有更小的运行ID。

按照上述的规则，这个**Sentinel**会在一组候选**Slave**服务器之中选出最佳的**Slave**，准备将其替换为**Master**服务器，记录相关的数据，并将对应的故障迁移的状态`failover_state`切换为`SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE`，表示将会向对应的**Slave**服务器发送`SLAVEOF NO ONE`，使之从一台**Slave**服务器转换为一个独立的**Redis**服务器。

#### Master服务器迁移

与前面的逻辑相似，**Sentinel**会在心跳之中通过`sentinelFailoverStateMachine`函数检验**Master**对象的迁移状态，如果发现是`SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE`，那么便会通过`sentinelFailoverSendSlaveOfNoOne`函数向对应的**Slave**服务器发起迁移命令，命令将以事务的形式被发送：

```
MULTI
SLOVEOF NO ONE
CONFIG REWRITE
CLIENT KILL TYPE normal
EXEC
```

这里除了**MULTI**与**EXEC**命令用于封装事务的命令序列之外，另外三条有意义的命令的作用为：

1. `SLOVEOF NO ONE`，正如在前面介绍**主从复制**机制时所描述的，这个命令会使一个**Slave**服务器转化为一台独立的**Redis**服务器，且不在继续复制原来配置的**Master**服务器。
2. `CONFIG REWRITE`，由于涉及了主从身份的转换，通过这条命令可以强制将**Slave**的当前状态覆盖回写到配置文件之中。
3. `CLIENT KILL TYPE normal`，通过这条命令，**Sentinel**可以要求这个**Slave**服务器关闭连接到这个服务器上的所有用户客户端的网络连接，防止在迁移的过程之中用户的查询命令干扰了整个流程。这里需要注意的一点是，当前**Sentinel**与**Slave**服务器之间建立的网络连接并不会因为这条命令被关闭，具体**CLIENT**命令的代码实现可以在*src/networking.c*文件之中的`clientCommand`这个函数之中看到。

在向被选出的**Slave**服务器发送迁移命令后，**Sentinel**会将当前的迁移状态切换为`SENTINEL_FAILOVER_STATE_WAIT_PROMOTION`，表示**Sentinel**正在等待这个**Slave**服务器完成迁移，同时还刷新迁移状态切换的时间戳`sentinelRedisInstance.failover_state_change_time`。

#### 等待迁移完成

**Sentinel**等待迁移完成的工作主要分为两个部分，其一是迁移过程是否超时；其二是检验迁移过程是否成功。

##### 超时验证

**Sentinel**服务器会在心跳逻辑之中，对处于`SENTINEL_FAILOVER_STATE_WAIT_PROMOTION`状态的**Master**周期性地检查迁移是否超时。

```c
void sentinelFailoverWaitPromotion(sentinelRedisInstance *ri);
```

在函数`sentinelFailoverWaitPromotion`中会通过`failover_state_change_time`这个时间戳记录的时间来判断迁移是否超时。如果超时没有完成迁移，那么会调用`sentinelAbortFailover`中断此次迁移过程。

##### 成功验证

验证迁移成功则是基于**Sentinel**周期性发送**INFO**命令这个机制来完成的。**INFO**命令的返回数据之中会通报对应的服务器的角色信息，如果是**Master**服务器，那么命令的返回数据之中会有`role:master`这样的数据，反之则会有`role:slave`这样的数据。基于这个基础，**Sentinel**如果发现一个具有`SRI_SLAVE`以及`SRI_PROMOTED`的**Slave**对象，其角色切换成了**Master**，那么便说明对应的**Slave**服务器完成了迁移过程。

接下来**Sentinel**便会将当前的故障迁移状态`failover_state`切换为`SENTINEL_FAILOVER_STATE_RECONF_SLAVES`，这个状态表示**Sentinel**将重新配置其他的**Slave**服务器，要求它们复制这个新的**Master**服务器。

#### 重新配置Slave

**Sentinel**服务器在心跳逻辑之中检测如果迁移状态处于`SENTINEL_FAILOVER_STATE_RECONF_SLAVES`时，会通过`sentinelFailoverReconfNextSlave`函数，要求这个`master`的其他**Slave**服务器去复制新的**Master**服务器。

```c
void sentinelFailoverReconfNextSlave(sentinelRedisInstance *master);
```

这个函数会遍历`master`对象对应的`sentinelRedisInstance.slaves`字典，其中会跳过已经选做故障迁移的带有`SRI_PROMOTED`标记的**Slave**服务器对象，并向其他的**Slave**服务器发送**SLAVEOF**命令，要求其复制新的**Master**，并为其在`sentinelRedisInstance.flags`上设置`SRI_RECONF_SENT`标记，表示已经向其发送了**SLAVEOF**命令。

接下来**Sentinel**服务器会继续在周期性的**INFO**命令返回数据之中检验这些**Slave**服务器的状态。如果通过**INFO**命令返回数据上报的`slave_master_host`以及`slave_master_port`与被提升为**Master**的那台服务器相同，说明**SLAVEOF**命令已经生效，这台**Slave**服务器已经开始复制新的**Master**，此时将对应的`sentinelRedisInstance.flags`字段设置为`SRI_RECONF_INPROG`，表明该服务器的复制正在进行中。

因为复制新的**Master**需要一定时间，因此**Sentinel**服务器还会继续在周期性的**INFO**命令的返回数据之中校验那些处于`SRI_RECONF_INPROG`状态的**Slave**服务器，检查其是否已经完成对于新的**Master**服务器的复制。在前面我们介绍主从复制时有提到过，当一个**Slave**服务器完成对于**Master**数据的复制之后，会将`redisServer.repl_state`这个字段设置为`REPL_STATE_CONNECTED`，这样在**INFO**命令的返回数据之中就会出现一条`master_link_status:up`的信息，**Sentinel**在检查到这个返回后，便会将这个对象的`sentinelRedisInstance.flags`字段设置为`SRI_RECONF_DONE`，表示这个服务器的复制已经完成。

#### 迁移完成

最后**Sentinel**服务器会周期性地调用`sentinelFailoverDetectEnd`函数来检测原**Master**的**Slave**服务器是否已经全部完成了对新**Master**的复制，如果有个别的**Slave**超过一定时间`sentinelRedisInstance.failover_timeout`，则会向这个**Slave**重新发送**SLAVEOF**命令强制重新复制。

当检测到所有的**Slave**都完成了复制之后，便会将故障迁移状态切换至`SENTINEL_FAILOVER_STATE_UPDATE_CONFIG`，这样**Sentinel**服务器便会通过`sentinelFailoverSwitchToPromotedSlave`函数调用`sentinelResetMasterAndChangeAddress`将这一组主从集群之中的**Master** 对象切换成新的迁移之后的**Master**服务器，并重新构建这个新的**Master** 对象的`sentinelRedisInstance.slaves`字典，完成整个故障迁移的流程。

### 配置回写







































***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*