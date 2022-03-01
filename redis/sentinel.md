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

## Raft算法

之所以在这里介绍**Raft**算法，是因为在故障迁移的过程之中，当**主从**集群之中的**Master**服务器下线时，负责监控这些**Redis**服务器的**Sentinel**集群便会应用**Raft**算法，从集群中的所有**Sentinel**服务器之中选举出一个**Leader**服务器，在由这个**Leader**服务器对**Master**服务器进行故障迁移的操作。在介绍**Raft**算法之前，我们先来看一下提出**Raft**算法的缘由，也就是分布式系统之中著名的**拜占庭将军问题**。

### 拜占庭将军问题

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

### Raft算法之节点

在**Raft**算法之中每一个节点对应于**拜占庭将军问题**之中的一位将军，也对应于分布式系统之中的一台主机或者一个进程。节点一共拥有三种状态，**Follower**、**Candidate**、**Leader**，在这三种状态之中：

1. **Follower**节点，初始状态下所有的节点都是**Follower**状态。每个**Follower**节点有一个随机时间的计时器，倒计时结束时，如果没有收到其他节点的投票请求，那么会将自己转化为**Candidate**节点，并向其他节点发送投票请求。
2. **Candidate**节点会等待其他节点的投票返回，如果该**Candidate**收到了过半数的确认投票后，会将自己升级为**Leader**节点，在一个分布式系统之中允许存在多个**Candidate**节点。
3. **Leader**节点则负责整个集群的决策选择。

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

另外正如*Redis*会使用`redisServer`结构体来存储服务器在运行时所需要的数据，在哨兵实例之中也存在一个数据结构用于存储哨兵实例在运行时所需的必要数据：

```c
struct sentinelState {
    char myid[CONFIG_RUN_ID_SIZE+1]; /* This sentinel ID. */
    uint64_t current_epoch;         /* Current epoch. */
    dict *masters;      /* Dictionary of master sentinelRedisInstances.
                           Key is the instance name, value is the
                           sentinelRedisInstance structure pointer. */
    int tilt;           /* Are we in TILT mode? */
    int running_scripts;    /* Number of scripts in execution right now. */
    mstime_t tilt_start_time;       /* When TITL started. */
    mstime_t previous_time;         /* Last time we ran the time handler. */
    list *scripts_queue;            /* Queue of user scripts to execute. */
    char *announce_ip;  /* IP addr that is gossiped to other sentinels if
                           not NULL. */
    int announce_port;  /* Port that is gossiped to other sentinels if
                           non zero. */
    unsigned long simfailure_flags; /* Failures simulation. */
    int deny_scripts_reconfig; /* Allow SENTINEL SET ... to change script
                                  paths at runtime? */
} sentinel;
```

在`sentinelState`这个数据结构之中最重要的字段便是`sentinelState.masters`这个哈希表，这里存储了当前这个哨兵实例所监控的**Master**实例。而这些被监控的实例，在哨兵模式之中是使用`sentinelRedisInstance`这个数据结构来表示的：

```c
typedef struct sentinelRedisInstance {
    int flags;      /* See SRI_... defines */
    char *name;     /* Master name from the point of view of this sentinel. */
    char *runid;    /* Run ID of this instance, or unique ID if is a Sentinel.*/
    uint64_t config_epoch;  /* Configuration epoch. */
    sentinelAddr *addr; /* Master host. */
    instanceLink *link; /* Link to the instance, may be shared for Sentinels. */
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

在`sentinelRedisInstance`这个数据结构之中，主要由三个模块的内容：

1. 用于描述实例信息的数据字段，例如地址信息以及连接数据。
2. 用于描述与该实例关联的实例的信息：
   1. 如果这是一个**Master**实例，则会记录与之对应的**Slave**实例的信息，同时由于哨兵模式也可能是由多机构成，因此有会记录监控这个**Master**实例的哨兵实例的信息。
   2. 如果这是一个**Slave**实例，则会记录对应**Master**实例的地址数据，以及复制偏移量等信息。
3. 用于处理故障迁移的数据。

而在哨兵模式之中，是使用`instanceLink`这个结构体来表示哨兵实例与被监控实例之间的连接信息：

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



在*src/sentinel.c*源文件之中，定义了哨兵模式所需要的数据结构。

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



在*src/server.h*之中定义了接口函数：

```c
void initSentinelConfig(void);
void initSentinel(void);
void sentinelTimer(void);
char *sentinelHandleConfiguration(char **argv, int argc);
void sentinelIsRunning(void);
```

重置一个被监视的**Master**实例的状态，

实例之间进行交互的接口：
```c
void sentinelEvent(int level, char *type, sentinelRedisInstance *ri, const char *fmt, ...);
```
这个函数主要用于将一些消息通过**发布/订阅**连接发送给指定的`sentinelRedisInstance`对象所代码的实例。这里`level`用于标记通知消息的等级，当我们发送`LL_WARNING`等级的消息时，将会触发前面介绍用户通知的脚本。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*