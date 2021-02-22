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

除此之外，`instanceLink.refcount`字段用于表示这个连接对象的引用计数，这也就意味着多个`sentinelRedisInstance`会引用相同的`instanceLink`连接对象。这主要是因为哨兵实例会在`sentinel.masters`关联其监控的所有实例；而对于**Master**实例对象`sentinelRedisInstance`会用通过`sentinelRedisInstance.sentinels`



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

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*