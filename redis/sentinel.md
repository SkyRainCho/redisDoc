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



***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*