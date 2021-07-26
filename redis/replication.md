# Redis主从复制

虽然*Redis*单机可以承载十万量级的并发请求，但是依然具有处理能力的上限，考虑正常情况下，对于数据的读取请求要远大对数据的写入请求。基于这个前提，*Redis*设计了一个复制的机制，应用这个复制机制，其他的*Redis*服务器可以拥有一个不断更新的数据副本，这样这些拥有数据副本的*Redis*服务器可以分担用户的读取请求，如此一来可以进一步提升*Redis*的并发能力，本篇文章将对*Redis*主从复制机制的相关功能与实现细节进项介绍。


## 复制功能概述
首先，彼此连接的**Master**实例与**Slave**实例是互为客户端的关系，在前面我们介绍客户端相关内容时，我们介绍了`client.flags`这个字段以掩码的形式记录了当前客户端的状态。而**Master**实例与**Slave**实例从各自的角度看对方，都是一个`client`对象，而在*src/server.h*之中也为**Master**与**Slave**实例定义了对应的状态掩码：
```c
#define CLIENT_SLAVE (1 << 0)
#define CLIENT_MASTER (1 << 1)
```

*Redis*本身也支持一个*Matser*实例连接多个**Slave**实例的模式，这样可以进一步提升系统的并发能力。*Redis*甚至可以在配置文件*redis.conf*之中通过配置项`min-reolicas-to-write`来指定**Master**实例所需要的*Slave*实例的最小数量。一旦*Matser*实例对应的*Slave*数量不足，便可以认为这个**Master**处于一种异常的状态，**Master**实例可以拒绝在这个状态下执行写命令。

然而如果一个**Master**实例连接了过多的**Slave**实例的话，**Master**与**Slave**之间的数据传输对于**Master**实例本身来说也会造成一定的网络负载，因此*Redis*还支持一种层次化的主从模式，也就是说一个**Slave**实例也可以拥有它自己的**Slave**，我们可以称之为*Sub-Slave*实例。基于这种层次化的结构，顶层的**Master**实例只需要与若干少数个**Slave**实例进行数据传输，再由这些与顶层**Master**直连的**Slave**将数据发送给更为下层的**Slave**实例，以降低数据传输对**Master**实例性能所带来的影响。

简单来说*Redis*的复制功能主要可以划分两个阶段：
1. **数据同步**：**Slave**节点通过某种方式获取**Master**节点的数据副本，并将其加载到内存键空间之中，使该**Slave**节点成为**Master**节点的镜像节点。
1. **命令转发**：**Master**节点在执行命令时，将该命令转发给自己的*Slave*节点，以便在运行之中保证**Master**实例与**Slave**实例之间数据的一致性。

### 数据同步
*Redis*可以通过两种方式开启**数据同步**：

1. 通过配置文件，在*redis.conf*配置文件之中，配置`replicaof <masterip> <masterport>`这条配置项，可以设置对应**Master**节点的地址；如果**Master**实例需要密码认证，可以通过`masterauth <master-password>`配置项来设置连接**Master**所需要认证密码。
1. 执行**REPLICAOF**命令，命令的格式为`REPLICAOF host port`，通过这条命令，可以使执行这条命令的*Redis*成为给定地址`host`以及`port`的另一个*Redis*实例的*Slave*节点，不过这条命令有一个缺陷，如果你对应的**Master**节点需要密码验证，而在*redis.conf*配置文件之中没有提前设定**Master**节点的认证密码，那么需要在执行**REPLICAOF**命令之前，先通过**CONFIGSET**命令来设置`masterauth`的认证密码，否则将无法建立主从实例之间的数据同步。

无论上面哪种建立主从复制的方式，**数据同步**都不是立即开始的，这两种方式仅是为将要充当**Slave**的服务器设置了**Master**的地址，真正开启数据同步则是在设置了**Master**地址信息之后，**Slave**通过一系列内部命令来实现的：
1. 通过设置的**Master**地址信息，建立*Slave*与**Master**之间的Socket连接。
1. *Slave*向**Master**发送**PING**命令以检验二者连接是否正常。
1. 如果**Master**需要密码认证，则发送**AUTH**命令验证密码。
1. *Slave*会向**Master**发送一系列的**REPLCONF**命令，将*Slave*的信息通知给**Master**。
1. *Slave*向**Master**发送**PSYNC**命令，以正式的开启**数据同步**。

数据同步相当于将*Master*上键空间之中的全部数据发送给**Slave**实例，如果看过前面对于**RDB**持久化的介绍，我们可以发现这其中有很多相似之处，而实际上*Master*实例与**Slave**实例之间的数据同步也是通过**RDB**的机制来实现的。在*Redis*之中，复制机制中待同步数据的准备与传输方式有两种：

1. *Redis*使用正常的方式，启动一个后台子进程来生成**RDB**文件到磁盘上，等待磁盘上的*RDB*文件生成结束，在通过网络连接将这个**RDB**文件发送给**Slave**节点。
2. 如果*Master*实例所在的服务器上的磁盘性能很低的话，那么先向磁盘中写入数据再从磁盘进行读取与网络传输的话，会导致数据同步的效率很低。对于这种情况，*Redis*设计了一种无盘的数据同步方式，也就是不通过磁盘文件，而是在生成**RDB**文件的过程之中，直接将数据写入与**Slave**实例所建立的连接的套接字文件描述符上，**Slave**实例直接通过网络来接收*Master*发来的同步数据，这样可以大大提升针对单一**Slave**实例同步的速度。而这得益于**Linux**操作系统中*一切皆文件*的设计理念以及*Redis*自身实现的`rio`通用IO对象数据结构对于IO操作的封装。

虽然采用无盘方式进行数据同步的策略，会提升单一**Slave**实例的同步效率，免去了磁盘IO所带来的延时，但是这种方式也有一个潜在的问题，便是在多个**Slave**需要数据同步的时候，无法复用**RDB**文件。当*Master*实例正在为一个**Slave**实例进行无盘的**数据同步**，如果另外一个**Slave**实例也来请求**数据同步**的话，*Master*不得不进行重复两次**RDB**的生成过程。那么采用有磁盘的方式来进行数据同步的话，如果在生成**RDB**文件的过程中，又有一个*Slave*实例期望与*Master*实例之间进行数据同步的话，那么我们可以在**RDB**文件生成结束之后，将这个文件发送给多个等待数据同步的*Slave*实例，免去了重复生成**RDB**文件所带来的系统负载。

上面所介绍的**数据同步**方式，无论是有盘的还是无盘的，其本质都是对*Master*实例上的数据进行一次全量的备份，*Slave*实例将网络连接上接收到的同步数据写入一个本地文件之中，当传输接收之后通过`rdbLoad`接口使用这个全量的数据备份还原*Master*上的数据。通过后面我们将要介绍的**命令转发**，*Master*与*Slave*之间可以保持数据上的一致性。但是如果在日常的运行之中，*Master*与*Slave*之间的网络连接出现问题，被转发的命令无法传递到*Slave*实例上，这将导致二者数据的不一致，为了解决这个问题，当*Slave*与*Master*直接重新建立连接时，会重新执行一次**数据同步**以继续维持两者间数据的一致性。早期的*Redis*也确实使用这种方式在重连时进行全量的**数据同步**的，但是这其中也存在一个问题，

*如果两者之间断线的时间很短呢？是否有必要为了极小的数据差异进行一次全量的数据同步呢？*。

在新版本的*Redis*之中，作者针对这个问题做出了一个优化，也就是引入了**增量数据同步**的机制，应用这个机制，*Redis*可以在*Master*实例与*Slave*实例数据差异较小的情况下，仅对增量的数据差异进行同步，而无须再进行全量的**数据同步**，可以大大提升同步的效率。

所谓**增量数据同步**，其设计思路是：
1. 在*Master*一侧，
    1. 维护一个唯一复制ID`replid`，用于标识当前的*Redis*实例；
    1. 维护一个复制偏移`master_repl_offset`，用于记录当前*Master*实例累积转发命令的计数，每转发一条命令，便会将该命令的长度信息累加到`master_repl_offset`这个偏移量上；
    1. 维护一个有限长度的，先进先出的积压缓冲区`repl_backlog`，当一条命令被转发给*Slave*实例的时候，也会被写入积压缓冲区之中，同时如果积压缓冲区已满，会将最早被写入的数据清空，以便能够继续写入新的命令。
1. 而在*Slave*一层，代表*Master*的客户端之中也存储了*Master*实例同步过来的复制偏移量。

正常请求下，*Master*实例与*Slave*实例之中，各自的复制偏移量应该是相同的。一旦某个*Slave*实例掉线，那么将导致*Slave*上存储的复制偏移量落后于*Master*实例中的复制偏移量。如果这部分差异的数据恰好落在*Master*实例的积压缓冲区里的话，那么在重新连接建立之后，*Master*只需要将积压缓冲区里对应的数据发送给*Slave*实例就可以，而无需重新执行**RDB**的生成过程。只要在第一次建立连接或者重连时两者数据差异过大的时候，才会执行**RDB**的生成过程。

### 命令转发
参考前面介绍的**AOF**持久化策略，可以很好地理解命令转发这一概念。在*Master*实例与*Slave*实例完成数据同步之后，*Master*会在运行过程中将执行的命令发送给*Slave*对应的`client`客户端对象，同时也会将这个命令写入积压缓冲区，以便后续执行**增量数据同步**的流程。而在*Slave*端，*Master*发送来的命令则会被*Master*对应的`client`对象接收，并完成命令的解析与命令的执行，将*Master*端执行的命令在*Slave*一层重新执行一次，已完成数据的同步。而对应层次化结构的主从模式，*Slave*实例在接收到*Master*发送的命令之后，还会将这个命令转发给自己的*Slave*实例，也就是*Sub-Slave*实例，帮助*Sub-Slave*完成数据的同步。

### 同步复制

当用户在**Master**上执行一条写命令时，他可能需要确保这条命令已经被转发到**Slave**实例之后才可以进行后续的操作。原则上在主从复制机制之中，命令的转发是一个异步的过程，**Master**默认自己转发给**Slave**的命令均已经被**Slave**接收并处理。但是基于前面描述的这个场景，需要*Redis*能够实现一种对于转发命令的确认机制，因此*Redis*实现了这个**同步复制**的功能，来确保**Master**实例上执行的命令已经被**Slave**实例所执行。

## 复制功能相关数据结构

### 服务器全局变量中复制相关的数据结构
*Redis*在`redisServer`这个数据结构之中，存储了一系列用于维护复制功能相关的字段。而服务器全局数据之中的数据字段可以被划分为*Master*实例与*Slave*实例所应用的数据。
#### Master实例需要的数据
在*Master*实例上所有连接在这个服务器上的*Slave*实例节点，被存储在一个双端链表之中：
```c
struct redisServer
{
    list *slaves;
};
```

除此之外，*Master*实例上关于复制机制的相关数据可以被分为三个部分：

```c
struct redisServer
{
    char replid[CONFIG_RUN_ID_SIZE+1];
    char replid2[CONFIG_RUN_ID_SIZE+1];
    long long master_repl_offset;
    long long second_replid_offset;
    int slaveseldb;
    
    char* repl_backlog;
    long long repl_backlog_size;
    long long repl_backlog_histlen;
    long long repl_backlog_idx;
    long long repl_backlog_off;
    
    int repl_ping_slave_period;
    time_t repl_backlog_time_limit;
    time_t repl_no_slaves_since;
    int repl_min_slaves_to_write;
    int repl_min_slaves_max_lag;
    int repl_good_slaves_count;
    int repl_diskless_sync;
    int repl_diskless_sync_delay;
};
```

1. *Master*实例自身数据的维护：
    1. `redisServer.replid`，用于存储*Master*实例自身的复制ID，这是一个40位的随机字符串，用于对不同的*Redis*实例进行区分。
    1. `redisServer.master_repl_offset`，用于记录前面所提到的*Master*实例的复制偏移量。每次*Master*实例将命令写入到积压缓冲区之中的时候，会增加这个`master_repl_offset`复制偏移量。
    1. `redisServer.slaveseldb`，用于记录上一次执行复制命令对应的键空间编号；可以对应理解`redisServer.aof_selected_db`这个字段的作用。
1. *Master*实例上积压缓冲区：
    1. `redisServer.repl_backlog`，*Master*上的积压缓冲区，这个积压缓冲区是使用`malloc`分配的一段固定的连续内存。这个积压缓冲区是循环复用的，当写入的数据已经达到积压缓冲区的尾部时，新的数据会从积压缓冲区头部开始写入，并覆盖积压缓冲区之中的旧数据。
    1. `redisServer.repl_backlog_size`，积压缓冲区的长度。
    1. `redisServer.repl_backlog_histlen`，这个用于记录积压缓冲区之中实际数据的长度，这个字段数值的长度不会超过`repl_backlog_size`规定的上限。 
    1. `redisServer.repl_backlog_idx`，用于标记循环积压缓冲区之中当前的数据偏移，如果有新的数据要被写入积压缓冲区的话，将会从`repl_backlog_idx`的下一个字节开始数据写入。
    1. `redisServer.repl_backlog_off`，积压缓冲区数据偏移量，这个字段用于记录积压缓冲区中有效数据的第一个字节对应于`master_repl_offset`的偏移量。
1. *Master*实例上关于*Slave*的统计与配置数据:
    1. `redisServer.repl_backlog_time_limit`，用于记录积压缓冲区的最大生存时间，如果*Master*实例长时间没有*Slave*实例连接，则会将积压缓冲区释放。
    1. `redisServer.repl_ping_slave_period`，用于配置*Master*实例与*Slave*实例发送**PING**命令的时间间隔。
    1. `redisServer.repl_no_slaves_since`，如果*Master*实例上的所有*Slave*实例都已经断开连接，那么这个字段记录*Master*上开始没有*Slave*连接的开始时间。
    1. `redisServer.repl_min_slaves_to_write`，配置*Master*实例上所需要的最小*Slave*实例数量
    1. `redisServer.repl_min_slaves_max_lag`，
    1. `redisServer.repl_good_slaves_count`，记录*Master*实例上，当前处于*good*状态的*Slave*实例数量。
    1. `redisServer.repl_diskless_sync`，配置*Master*实例上，是否使用启用通过网络的无盘复制方式。
    1. `redisServer.repl_diskless_sync_delay`，配置*Matser*实例上启动无盘复制的延时时间，如一个*Slave*节点请求了一次无盘复制，这是*Master*将会延时一段时间在开启**RDB**数据的传输，这样如果在这段延时时间内，另外一个*Slave*节点也来请求一次无盘的**数据同步**，*Master*可以仅开启一次生成**RDB**文件的处理逻辑。
#### Slave实例需要的数据
在*Slave*实例上，*Redis*则是会一个单独的`client`对象来记录其对应的*Master*实例：
```c
struct redisServer
{
    client* master;
    client* cached_master;
};
```
前面已经介绍了，*Master*实例与*Slave*实例互为彼此的一个客户端，因此这里`redisServer.master`字段作为*Master*实例在*Slave*实例的一个客户端记录了*Matser*与*Slave*之间的连接信息。同时为了实现前面提到的主从之间的增量数据同步，需要在连接断开时，对`redisServer.master`的信息进行缓存，而这个缓存的*Master*客户端对象将会被存储在`redisServer.cached_master`字段之中。

```c
struct redisServer
{
    char *masterauth;
    char *masterhost;
    int masterport;
    int repl_timeout;
    int repl_syncio_timeout;
    int repl_state;
    off_t repl_tarnsfer_size;
    off_t repl_transfer_read;
    off_t repl_transfer_last_fsync_off;
    int repl_transfer_s;
    int repl_transfer_fd;
    char *repl_transfer_tmpfile;
    time_t repl_transfer_lastio;
    int repl_serve_stale_data;
    int repl_slave_ro;
    int repl_slave_ignore_maxmemory;
    time_t repl_down_since;
    int repl_disavle_tcp_nodelay;
    int slave_priority;
    int slave_announce_port;
    char *slave_announce_ip;
};
```

另外还有一些数据字段用于维护*Slave*实例端的复制机制：

1. `redisServer.masterhost`，在*Slave*实例一侧，记录*Master*主机所在的地址，通常在代码里通过这个字段是否为空来判断当前的*Redis*是否是主从模式之中的一个*Slave*实例。
1. `redisServer.masterport`，在*Slave*一侧，记录*Master*对应的端口。
1. `redisServer.repl_syncio_timeout`，在*Slave*实例一侧，在建立连接时，向*Master*实例发送内部命令是采用阻塞整个进程的同步IO方式进行传输的，`repl_syncio_timeout`这个字段用于设定每次同步IO的超时时间。
1. `redisServer.repl_state`，用于记录在主从模式之中，当前*Slave*实例所处于的状态：
    1. `REPL_STATE_NONE`，表示当前服务器没有处于或者发起准备建立*Slave*的状态。
    1. `REPL_STATE_CONNECT`，通过配置或者**REPLICAOF**命令设置了*Master*地址信息之后，*Slave*实例会被置为这个状态，这个状态表示*Slave*将要与*Master*实例建立连接。
    1. `REPL_STATE_CONNECTING`，当*Slave*已经设置了地址信息之后，会通过`anetTcpNonBlockBestEffortBindConnect`函数接口建立与*Master*实例的连接，当连接建立并注册到事件循环之后，*Slave*节点将会被设置成`PEPL_STATE_CONNECTING`这个状态。
    1. `REPL_STATE_RECEIVE_PONG`，由于连接是以非阻塞的方式建立的，因此第一次从事件循环之中返回时，表明连接已经被建立，此时*Slave*会向*Master*同步发送一条**PING**命令，并将状态设置为`REPL_STATE_RECEIVE_PONG`，以表明自己处于等待接受*Master*返回的状态。
    1. `REPL_STATE_SEND_AUTH`，这个状态表示*Slave*实例将要将*Master*发送认证命令**AUTH**。
    1. `REPL_STATE_RECEIVE_AUTH`，这个状态表示*Slave*实例已近发送送完**AUTH**命令，正在等待*Master*的返回。
    1. `REPL_STATE_SEND_PORT`，表示*Slave*实例将要向*Master*实例通过**REPLCONF**发送自己的端口数据。
    1. `REPL_STATE_RECEIVE_PORT`，这个状态表示*Slave*已经通过**REPLCONF**命令向*Master*告知了自己的端口数据，正在等待*Master*的返回。
    1. `REPL_STATE_SEND_IP`，表示*Slave*实例将要向*Master*实例通过**REPLCONF**发送自己的端口数据。
    1. `REPL_STATE_RECEIVE_IP`，这个状态表示*Slave*已经通过**REPLCONF**命令向*Master*告知了自己的IP数据，正在等待*Master*的返回。
    1. `REPL_STATE_SEND_CAPA`，表示*Slave*实例将要向*Master*实例通过**REPLCONF**发送自己的CAPA信息。
    1. `REPL_STATE_RECEIVE_CAPA`，这个状态表示*Slave*已经通过**REPLCONF**命令向*Master*告知了自己的CAPA信息。正在等待*Master*的返回。
    1. `REPL_STATE_SEND_PSYNC`，表示*Slave*实例将要向*Master*实例发送**PSYNC**命令以开启**数据同步**过程。
    1. `REPL_STATE_RECEIVE_PSYNC`，*Slave*发送完**PSYNC**命令后，等待*Master*实例的返回。
    1. `REPL_STATE_TRANSFER`，**数据同步**逻辑开始执行，*Slave*实例处于该状态，并开始从连接上获取*Master*发送到的同步数据。
    1. `REPL_STATE_CONNECTED`，**数据同步**逻辑结束，*Slave*将切换至该状态，后续将接受*Master*实例转发来的命令数据，用以维护数据的一致性。
1. `redisServer.repl_transfer_size`，对于使用无盘的**数据同步**，该字段将会被设置为0；而对于有盘的**数据同步**，该字段为整个待传输的**RDB**文件的大小。
1. `redisServer.repl_transfer_read`，用于记录*Slave*实例从网络连接上已经接收到的数据的数量。
1. `redisServer.repl_transfer_last_fsync_off`，由于*Slave*实例收到的同步数据需要先写入磁盘，因此这个字段用于记录上次被写入到磁盘中的数据的偏移。
1. `redisServer.repl_transfer_s`，用于记录*Slave*与*Master*之间连接的套接字文件描述符。
1. `redisServer.repl_transfer_fd`，用于记录*Slave*实例接收同步数据的临时**RDB**文件的文件描述符。
1. `redisServer.repl_transfer_tmpfile`，记录临时**RDB**文件的文件名。
1. `redisServer.repl_transfer_lastio`，记录*Slave*实例上一次接收同步数据的时间戳。
### 客户端对象中复制相关的数据结构
因为在*Redis*的复制机制之中，*Master*实例与*Slave*实例彼此将对方看做一个特殊的客户端，因此在`client`这个表示客户端的数据结构之中也存储了一系列用于维护复制功能相关的数据字段。

```c
struct client
{
    int replstate;
    int repl_put_online_on_ack;
    int repldbfd;
    off_t repldboff;
    off_t repldbsize;
    sds replpreamble;
    long long read_reploff;
    long long reploff;
    long long repl_ack_off;
    long long repl_ack_time;
    long long psync_initial_offset;
    char replid[CONFIG_RUN_ID_SIZE + 1];
    int slave_listening_port;
    char slave_ip[NET_IP_STR_LEN];
    int slave_capa;
    long long woff;
};
```

1. `client.replstate`，这个字段用于从*Master*的视角来看*Slave*对应客户端的状态：
    1. `SLAVE_STATE_WAIT_BGSAVE_START`，对应的*Slave*已经请求**数据同步**，正在等待*Master*开启**RDB**的生成逻辑。
    1. `SLAVE_STATE_WAIT_BGSAVE_END`，对应*Slave*同步请求的**RDB**文件生成流程已经开始执行，正在等待**RDB**文件生成结束。
    1. `SLAVE_STATE_SEND_BULK`，*Master*正在想这个`client`对应*Slave*实例发送**RDB**同步数据。
    1. `SLAVE_STATE_ONLINE`，*Master*已经向这个`client`对应的*Slave*实例完成发送**RDB**文件，开始进入命令转发的流程。
1. `client.repldbfd`，*Master*用于同步的**RDB**文件的文件描述符。
1. `client.repldboff`，*Master*已经传输的**RDB**文件的大小。
1. `client.repldbsize`，*Master*传输的**RDB**文件的总大小。
1. `client.replpreamble`，用于记录*Master*传输的**RDB**文件的一些前缀信息。对于有盘**RDB**传输，前缀信息中存储了**RDB**文件的长度；对于无盘**RDB**传输，前缀信息中存储了用于标记传输结束的`<EOF>`分隔字符串。
1. `client.read_reploff`，在*Slave*一侧记录从*Master*中读取到的命令数据的偏移量
1. `client.reploff`，在*Slave*一侧记录对应*Master*所发送的数据中已经被*Slave*处理的命令数据的偏移量。
1. `client.psync_initial_offset`，在*Master*实例一侧用于记录对应*Slave*的初始偏移。
1. `client.replid`，在*Slave*实例上记录*Master*实例的复制ID。
1. `client.slave_listening_port`，在*Master*实例上记录的*Slave*通过**REPLCONF**命令传来的端口数据。
1. `client.slave_ip`，在*Master*实例上记录的*Slave*通过**REPLCONF**命令传来的IP数据。
1. `client.slave_capa`，在*Master*实例上记录的*Slave*通过**REPLCONF**命令传来的这个**Slave**实例的capa数据。
1. `client.repl_ack_off`，对于一个表示**Slave**实例的`client`对象，这个字段用于存储这个**Slave**当前确认处理的复制偏移量。
1. `client.repl_ack_time`，对于一个**Slave**实例的客户端对象，这个字段记录这个**Slave**上次确认的时间。
1. `client.woff`，这个字段用于存储上一个写命令的全局复制偏移量。

## 积压缓冲区的维护

#### 积压缓冲区的创建

在*Master*实例的一侧，*Redis*会维护一段连续分配的内存作为积压缓冲区，用于存储近期被同步给*Slave*的命令，以便当连接断开重连时，*Redis*执行增量的数据同步。初始情况下，*Redis*不会为积压缓冲区分配内存，当有需要的时候，则会通过下面这个函数来创建一个积压缓冲区：

```c
void createReplicationBacklog(void);
```

*Master*实例上会在下面两种情况下通过调用`createReplicationBacklog`接口来创建积压缓冲区：

1. 当第一个*Slave*实例通过**PSYNC**命令与*Master*开启同步时，会创建积压缓冲区。
2. 当有*Slave*实例断线请求重连时，如果积压缓冲区已经被释放，则会重新创建一个积压缓冲区。

由于主从复制支持层次结构，这也就意味着，一个*Slave*也可能是它的下级*Slave*的*Master*，因此在*Slave*一端也需要维护一段积压缓冲区，当一个*Slave*通过`readSyncBulkPayload`接口开始读取*Master*的同步数据时，也会通过`createReplicationBacklog`来创建一段积压缓冲区。

积压缓冲区的大小通过`redisServer.repl_backlog_size`来存储，不过我们可以在运行期通过**CONF**命令来重新调整*Redis*的积压缓冲区的大小，下面这个函数用来调整积压缓冲区的大小：

```c
void resizeReplicationBacklog(long long newsize);
```

不过这函数会清空原有的积压缓冲区，并将关于积压缓冲区的偏移索引等数据清零重置。

#### 积压缓冲区的释放

*Redis*可以通过调用`freeReplicationBacklog`函数来释放积压缓冲区：

```c
void freeReplicationBacklog(void);
```

在*Master*一端，如果长时间`redisServer.repl_backlog_time_limit`没有*Slave*与该*Master*连接，那么便会通过`freeReplicationBacklog`来释放自己的积压缓冲区。

而对于*Slave*一侧，我们知道*Redis*也会维护一段积压缓冲区，用于处理层次结构的主从复制模式。对于*Slave*实例，一旦其开始请求**PSYNC**的数据同步，也就意味着该实例中的所有数据都已经失效，因此对于其积压缓冲区也应该做释放处理；后续会在处理*Master*发来的同步数据的时候，重新创建积压缓冲区。

#### 积压缓冲区的数据写入

在*Master*实例中，*Master*将命令转发给*Slave*的同时，也会将这份命令数据写入积压缓冲区之中，*Redis*为积压缓冲区的数据写入提供了两个函数接口：

```c
void feedReplicationBacklog(void *ptr, size_t len);
void feedReplicationBacklogWithObject(robj *o);
```

这两个函数分别用于将一段内存数据写入积压缓冲区以及将一个对象数据结构`robj`写入解压缓冲区。由于*Redis*之中积压缓冲区是一个循环缓冲区，当缓冲区已经被写满，新的数据将会从积压缓冲区的起始位置重新进行写入。而这一步操作会导致*Master*实例之中的复制偏移量`redisServer.master_repl_offset`的更新：

```c
void feedReplicationBacklog(void *ptr, size_t len)
{
    server.master_repl_offset += len;
    //向循环积压缓冲区之中写入数据
    ...
        
    if (server.repl_backlog_histlen > server.repl_backlog_size)
        server.repl_backlog_histlen = server.repl_backlog_size;
    
    server.repl_backlog_off = server.master_repl_offset - server.repl_backlog_histlen + 1;
}
```

从上面的代码片段我们可以看出，如果我们把服务器启动时的命令执行的初始状态设置为0的话，那么`redisServer.master_repl_offset`这个复制偏移量可以理解为*Master*实例上最后一条被执行的查询命令相对于服务器启动时初始状态的一个偏移量；而`redisServer.repl_backlog_off`这个积压缓冲区偏移量则可以认为是积压缓冲区之中的第一个数据相对于初始状态的偏移量，二者的差值便是积压缓冲区之中数据的大小。

*Master*实例在向*Slave*实例转发查询命令的接口`replicationFeedSlaves`之中，会执行前面向积压缓冲区之中追加数据的逻辑：

```c
void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc) {
    if (server.slaveseldb != dictid)
    {
        //需要在积压缓冲区之中追加一条SELECT命令，用于切换数据库ID
        robj *selectcmd;
        ...
        if (server.repl_backlog)
            feedReplicationBacklogWithcObject(selectcmd);
    }
    ...
    if (server.repl_backlog)
    {
        char aux[LONG_STR_SIZE+3];
        aux[0] = '*';
        len = ll2string(aux+1,sizeof(aux)-1,argc);
        aux[len+1] = '\r';
        aux[len+2] = '\n';
        feedReplicationBacklog(aux,len+3);

        for (j = 0; j < argc; j++) {
            long objlen = stringObjectLen(argv[j]);
            aux[0] = '$';
            len = ll2string(aux+1,sizeof(aux)-1,objlen);
            aux[len+1] = '\r';
            aux[len+2] = '\n';
            feedReplicationBacklog(aux,len+3);
            feedReplicationBacklogWithObject(argv[j]);
            feedReplicationBacklog(aux+len+1,2);
        }
    }
    ...
}
```

#### 积压缓冲区的数据请求

```c
long long addReplyReplicationBacklog(client *c, long long offset);
```

上面这个函数用于将积压缓冲区之中从指定偏移量`offset`开始的有效数据发送给指定的*Slave*实例，并返回发送数据的长度。这个接口用于函数`masterTryPartialResynchronization`之中，用于处理*Slave*实例的发来的**增量数据同步**请求。

## 主从连接的建立

在积压缓冲区这个主从复制功能之中最为基础的模块了解之后，我们来看一下*Master*实例与*Slave*实例之间是如何建立连接的。

由于主从复制是*Slave*实例主动发起的连接，因此我们先来看*Slave*实例一端，*Slave*可以通过**REPLICAOF**命令来建立与*Master*实例的连接，这个命令有两种形式：

```
REPLICAOF no one
REPLICAOF <host> <port>
```

1. 第一种形式，用于解除当前实例在主从复制之中的*Slave*角色，将其转换为一个独立的可以进行写操作的*Redis*服务器实例。
2. 第二种形式，用于将一个实例设置为某一个给定*Master*的*Slave*实例。

```c
void replicaofCommand(client *c);
```

上面这个命令处理函数会解析客户端命令的参数，分别处理上面的两种解除主从复制以及建立主从复制的过程。

解除主从复制，是通过下面这个`replicationUnsetMaster`函数进行的：

```c
void replicationUnsetMaster(void);
```

这个函数会清空服务器上记录的*Master*数据信息`redisServer.masterhost`，同时释放在*Slave*一侧代表*Master*实例的客户端对象`redisServer.master`。在这个函数接口里会调用一个`shiftReplicationId`：

```c
void shiftReplicationId(void) {
    memcpy(server.replid2,server.replid,sizeof(server.replid));
    server.second_replid_offset = server.master_repl_offset+1;
    //重新为实例生成server.replid复制ID
    changeReplicationId();
}
```

我们知道`redisServer.replid`之中存储了*Master*实例的复制ID，在前面对于数据结构的介绍之中，我们发现还有两个字段分别为`redisServer.replid2`以及`redisServer.scond_replid_offset`，那么这两个看似是复制ID以及复制偏移量的数据是做什么的呢？

在后面我们介绍哨兵模式的时候会讲到，当一组主从复制的*Redis*实例中*Master*实例掉线，那么哨兵服务器会从所有的*Slave*实例之中选择一个实例将其提升为这组实例之中的*Matser*实例，这里`redisServer.replid2`以及`redisServer.scond_replid_offset`这两个字段便是为处理这种情况的。当一个*Slave*实例被提升至*Master*之后，会为这个实例重新生成一个复制ID，并把原有的上一个*Master*实例的复制ID转移到`redisServer.replid2`上，这样一来，其他的*Slave*节点可以使用原来的复制ID来向这个新的*Master*实例来请求数据。

### 与Master建立连接

而**REPLICAOF**命令建立与*Master*实例之间的连接是通过`replicationSetMaster`这个函数开始的：

```c
void replicationSetMaster(char *ip, int port)
{
    int was_master = server.masterhost == NULL;
    
   	sdsfree(server.masterhost);
    server.masterhost = sdsnew(ip);
    server.masterport = port;
    if (server.master)
    {
        freeClient(server.master);
    }
	...
    server.repl_state = REPL_STATE_CONNECT;
}
```

这个函数会设置服务器上记录的*Master*实例的地址信息`redisServer.masterhost`以及`redisServer.masterport`两个字段；如果这个服务器曾经是另外一个*Master*实例的*Slave*，那么释放掉这个旧的*Master*客户端，并将当前服务器的状态`redisServer.repl_state`设置为`REPL_STATE_CONNECT`。

不过这里我们发现`replicationSetMaster`函数并没有建立于*Master*服务器的连接，只是为*Slave*设置了建立连接所需要的数据。而真正进行连接，建立主从复制的逻辑，是在*Redis*的专门处理复制机制的心跳函数中进行的：

```c
void replicationCron(void)
{
    ...
    if (Server.repl_state == REPL_STATE_CONNECT)
    {
        connectWithMaster();
    }
    ...
}
```

这里*Redis*会检测当前服务器是否处于`REPL_STATE_CONNECT`的状态，如果满足条件的话便会调用`connectWithMaster`来与*Master*建立连接的。

```c
int connectWithMaster(void)
{
    fd = anetTcpNonBlockBestEffortBindConnect(NULL, server.masterhost, server.masterport, NET_FIRST_BIND_ADDR);
    aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster,NULL);
    server.repl_transfer_lastio = server.unixtime;
    server.repl_transfer_s = fd;
    server.repl_state = REPL_STATE_CONNECTING;
}
```

这里建立连接的过程是使用非阻塞`connect`的方式发起的，由于非阻塞`connect`会立即返回，而此时连接并未真正建立，因此需要监听这个连接套接字上的可读与可写事件，并注册事件处理函数`syncWithMaster`；同时将当前服务的状态`redisServer.repl_state`设置为`REPL_STATE_CONNECTING`状态。

当异步的连接建立成功时，会触发事件处理回调函数`syncWithMaster`，这个函数会完成在前面概述之中介绍的，内部发送**PING**命令检测连接的建立情况；执行内部**REPLCONF**命令将*Slave*的配置信息通知*Master*；最后根据自身的状态，向*Master*实例发送**PSYNC**命令，开启数据同步。

对于**Master**实例来说，这个*Slave*与一般的客户端没有任何的差异。在*Slave*实例与**Master**实例通过**PING**命令以及**PONG**返回确认了连接之后，*Slave*实例将会发起一系列的**REPLCONF**命令，*Master*实例收到**REPLCONF**命令后，会将相应的配置数据设置到代表这个*Slave*对应的客户端对象`client`的相应字段上。

**REPLCONF**命令的格式为：

```
REPLCONF <option> <value> <option> <value> ...
```

这个命令对应的实现函数为：

```c
void replconfCommand(client *c);
```

这个函数所接收的参数必须是双数，需要满足每一个`<option>`就需要对应的`<value>`。**REPLCONF**这个命令的实现函数比较简单，解析参数之中的`<option>`，并根据`<option>`的不同，将对应的`<value>`设置到`client`结构体的对应字段上。那么**REPLCONF**命令可以接受的`<option>`有：

1. `listening-port`，用于设置*Slave*实例上对应`client.slave_listening_port`的字段信息，存储**Slave**实例监听的端口数据。
1. `ip-address`，用于设置*Slave*实例上对应`client.slave_ip`的字段信息，存储**Slave**实例监听的IP数据。
1. `capa`，用于设置*Slave*实例上对应`client.slave_capa`的字段信息，存储这个**Slave**实例支持的`capa`数据：
   1. `SLAVE_CAPA_EOF`
   1. `SLAVE_CAPA_PSYNC2`
1. `ack`，**Slave**用于向*Master*实例发送**ACK**数据，用于通知当前已经处理的同步数据的数量，通知的数据会被记录到对应客户端`client.repl_ack_off`字段上，另外也会更新对应客户端`client.repl_ack_time`的时间戳。
1. `getack`，上面`ack`是服务器内部使用的命令，由**Slave**实例自动发送；而这个`getack`则是在**Slave**实例一侧被执行的，显式地要求*Slave*实例向*Mater*实例发送一次**ACK**数据。

之所以建立连接的过程需要在`replicationCron`心跳函数之中进行，而不是执行**REPLICAOF**命令时立刻执行。这种方式有一个优点，一旦出现了*Slave*与*Maste*断开连接的情况，那么在`replicationCron`心跳函数之中，会检测连接状态并在断开连接的情况下，重启通过调用`connectWithMaster`来建立连接。

下面这个序列图所展示的，便是在主从机制之中，**Slave**实例是一步一步与**Master**实例建立连接的：

![主从建立连接的序列图](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/%E4%B8%BB%E4%BB%8E%E5%BB%BA%E7%AB%8B%E8%BF%9E%E6%8E%A5%E7%9A%84%E8%BF%87%E7%A8%8B.jpg)

## 数据同步

### Slave实例向Master实例申请数据同步

**Slave**实例在`syncWithMaster`函数中完成建立与**Master**实例的连接之后，便会将`redisServer.repl_state`这个状态字段切换至`REPL_STATE_SEND_PSYNC`这个状态，表示**Slave**实例将要向**Master**发起数据同步请求。

```c
void syncWithMaster(aeEventLoop *el, int fd, void *privdata, int mask) {
	...
	if (server.repl_state == REPL_STATE_SEND_PSYNC)
    {
        if (slaveTryPartialResynchronization(fd, 0) == PSYNC_WRITE_ERROR)
        {
            goto wirte_error;
        }
        server.repl_state = REPL_STATE_RECEIVE_PSYNC;
        return;
    }
    
    if (server.repl_state != REPL_STATE_RECEIVE_PSYNC)
    {
        goto error;
    }
    psync_result = slaveTryPartialResynchronization(fd,1);
    ...
}
```

在上面这段代码片段之中，`syncWithMaster`会分两次调用`slaveTryPartialResynchronization`接口向**Master**尝试请求数据同步：

1. 第一次调用会向**Master**发起**PSYNC**命令，用以通报自身的复制信息。如果这个**Slave**是在连接断开之后重新连接的情况，则会将`redisServer.cached_master`之中缓存的同步信息包括`replid`以及`offset`通报给**Master**，由**Master**判断是进行增量数据同步还是全量的数据同步。

   ```c++
   int slaveTryPartialResynchronization(int fd, int read_reply)
   {
       //发送命令
       if (!read_reply)
       {
           
           return PSYNC_WAIT_REPLY;
       }
       ...
   }
   ```

2. 第二次调用则是接收从**Master**返回的**PSYNC**命令的结果：

   1. `+FULLRESYNC <replid> <offset>`，形如这个格式的返回，表示只能进行全量同步，同时**Master**实例会通告自己的`replid`以及`repl_offset`信息。
   2. `+CONTINUE <replid>`，形如这个格式的返回，表示可以进行增量数据同步，**Master**实例会通过自己的`replid`给**Slave**。

   ```c
   int slaveTryPartialResynchronization(int fd, int read_reply)
   {
       ...
       // 读取命令返回
       reply = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
       aeDeleteFileEvent(server.el,fd,AE_READABLE);
       if (!strncmp(reply,"+FULLRESYNC",11))
       {
           //解析replid以及offset
           ...
           replicationDiscardCachedMaster();
           return PSYNC_FULLRESYNC;
       }
       
       if (!strncmp(reply,"+CONTINUE",9))
       {
           ...
       	replicationResurrectCachedMaster(fd);
           if (server.repl_backlog == NULL) createReplicationBacklog();
           return PSYNC_CONTINUE;
       }
   	...
   }
   ```

而在**Master**一段，则是相当于一次处理**PSYNC**命令的过程。**PSYNC**命令用于*Slave*实例在通过**REPLCONF**命令向*Master*实例设置了复制相关数据之后向*Master*请求数据同步，这个命令也是一条内部命令，由*Slave*实例一侧自动执行，该命令的格式为：

    PSYNC <replid> <reploffset>

这个命令之中：

1. `<replid>`指定了*Matser*实例对应的复制ID，如果*Slave*发送的`<replid>`为`?`，则表示是第一次建立连接，需要进行一次全量的**数据同步**。
1. `<reploffset>`指定了*Slave*自身的复制偏移，同于通知*Master*是否需要进行一次全量的**数据同步**

**PSYNC**命令是通过下面这个`syncCommand`函数来实现的：

```c
void syncCommand(client *c);
```

我们可以来看一下这个函数的一些代码片段，借此窥探**PSYNC**在处理*Master*与*Slave*建立连接的一些细节：

```c
void syncCommand(client *c)
{
    if (c->flags & CLIENT_SLAVE) return;
    if (server.masterhost && server.repl_state != REPL_STATE_CONNECTED)
    {
        ...
        return;
    }
    
    if (clientHasPendingReplies(c))
    {
        ...
        return;
    }
    
    //检查是否可以进行增量的数据同步
    
    c->replstate = SLAVE_STATE_WAIT_BGSAVE_START;
    if (server.repl_disable_tcp_nodelay)
        anetDisableTcpNoDelay(NULL, c->fd);
    c->repldbfd = -1;
    c->flags |= CLIENT_SLAVE;
    listAddNodeTail(server.slaves, c);
    if (listLength(server.slaves) == 1 && server.repl_backlog == NULL)
    {
        chanegReplicationId();
        createReplicationBacklog();
    }
    
    //处理全量同步的细节数据
}
```

通过上面的代码片段，我们可以发现：

1. 在*Master*开始处理**PSYNC**命令之前，*Slave*实例对应的客户端`client`与其他的客户端没有任何区别，并不具有`CLIENT_SLAVE`这个特殊的标记。
2. 一个*Slave*实例也可以接收**PSYNC**命令，用于处理*Sub-Slave*的连接，但是只能在处于`REPL_STATE_CONNECTED`状态下才可以处理**PSYNC**命令。
3. 如果*Slave*实例对应的客户端`client`之中，还有数据在输出缓存里没有被发送到网络上的话，暂时不能执行**PSYNC**命令。
4. 只有在**PSYNC**命令执行之后，才会给*Slave*对应的`client`对象设置上`CLIENT_SLAVE`这个特殊的标记掩码，并且将这个客户端对象加入到`redisServer.slaves`这个服务器双端链表之中。
5. 当第一个*Slave*实例加入到`redisServer.slaves`之中，并且没有分配积压缓冲区的话，会调用`createReplicationBacklog`接口来创建积压缓冲区。

### 执行数据同步

在前面介绍的**PSYNC**命令，仅介绍了一部分关于建立*Master*与*Slave*建立连接的过程，而**数据同步**的相关流程的一部分逻辑也是在**PSYNC**命令的对应函数接口之中实现的。接下来我们就来看一下**全量数据同步**与**增量数据同步**的相关内容。

#### 全量数据同步

我们继续来展示一段`syncCommand`接口的代码片段：

```c
void syncCommand(client *c)
{
    ...
    //如果无法进行增量数据同步，那么后续会开始处理全量同步的逻辑
    c->replstate = SLAVE_STATE_WAIT_BGSAVE_START;
    ...
    if (server.rdb_child_pid != -1 && server.rdb_child_type == RDB_CHILD_TYPE_DISK)
    {
        ...
        listRewind(server.slaves, &li);
        while((ln = listNext(&li)))
        {
            slave = ln->value;
            if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_END) break;
        }

        if (ln && ((c->slave_capa & slave->slave_capa) == slave->slave_capa))
        {
            copyClientOutputBuffer(c, slave);
            replicationSetupForFullResync(c, slave->psync_inirial_offset);
        }
    }
    else if (server.rdb_child_pid != -1 && server.rdb_child_type == RDB_CHILD_TYPE_SOCKET)
    {

    }
    else
    {
        if (server.repl_diskless_sync && (c->slave_capa & SLAVE_CAPA_EOF))
        {
        
        }
        else
        {
            if (server.aof_child_pid == -1)
            {
                startBgsaveForReplication(c->slave_capa);
            }
        }
    }
}
```

这里在我们可以总结出开启**数据同步**的逻辑与流程：

1. 初始将**Slave**实例设置为`SLAVE_STATE_WAIT_BGSAVE_START`状态，表示当前**Slave**实例正在等待*Master*实例开始生成**RDB**文件。
1. 如果**Master**当前有后台子进程在向磁盘上生成**RDB**文件，那么当前*Master*上所有的**Slave**，如果有**Slave**处于`SLAVE_STATE_WAIT_BGSAVE_END`的状态，这意味着这次**RDB**文件是另外一个**Slave**由于全量数据同步而请求的。对于这种情况，如果两个**Slave**的`client.slave_capa`相同，想么后面请求**数据同步**的**Slave**将会复用前面**Slave**所请求的这次**RDB**文件。
1. 如果**Master**当前有后台子进程在通过Socket来生成**RDB**文件，那么说明有其他的**Slave**在进行无盘的**数据同步**，这种情况下不会在开启新的**数据同步**。
1. 对于没有后台进程生成**RDB**文件的**Master**实例，检查服务器是否配置了`server.repl_diskless_sync`，如果服务器配置了无盘数据同步，同时**Slave**支持`SLAVE_CAPA_EOF`，那么**Master**会通过`replicationCron`接口在下一次心跳之中处理无盘的**数据同步**。
1. 如果服务器没有开启无盘**数据同步**，或者**Slave**不支持`SLAVE_CAPA_EOF`，其在没有后台子进程重写**AOF**的情况下，通过调用`startBgSaveForReplication`接口来开启同步。

**Master**开启**数据同步**生成**RDB**文件的逻辑是通过`startBgsaveForReplication`这个函数接口来实现的：

```c
int startBgsaveForReplication(int mincapa);
```

这个函数会根据**Master**实例上`redisServer.repl_diskless_sync`的配置以及**Slave**实例通过**REPLCONF**通报的对应客户端`client.slave_capa`的配置，通过创建子进程的方式进行有盘的**RDB**文件生成，或是通过网络采用无盘的*RDB*文件生成逻辑。

在子进程的处理数据的流程开始之后，**Master**会通过下面这个函数通知**Slave**实例请求同步的结果：

```c
int replicationSetupSlaveForFullResync(client *slave, long long offset);
```

调用这个函数，会将*Slave*对应的客户端`client.replstate`字段设置为`SLAVE_STATE_WAIT_BGSAVE_END`状态，表示其正在等待**RDB**文件生成结束，同时设置这个*Slave*对应客户端对象`client.psync_initial_offset`初始复制偏移字段为系统当前的复制偏移量。最后向*Slave*发送一条返回信息，消息的格式也就是前面介绍的：

    +FULLRESYNC <replid> <offset>

这条返回信息告知**Slave**实例一侧，需要进行一次全量的**数据同步**，并且将当前**Master**的复制ID以及复制偏移量一并发送给*Slave*实例。

##### 有盘的RDB文件数据同步

对于有盘**RDB**文件的**数据同步**，*Redis*会启动一个后台子进程生成**RDB**文件并将这个文件存储到磁盘上。当文件生成结束后，在通过网络连接将这个文件之中的数据发送到对应的一个或者多个**Slave**实例上。`startBgsaveReplication`函数会通过前面我们在介绍**RDB**数据持久化时所介绍的`rdbSaveBackground`函数接口来异步生成**RDB**文件，在子进程结束的回调函数`backgroundSaveDoneHandlerDisk`中会调用`updateSlavesWaitingBgsave`来处理等待**RDB**生成的那些**Slave**对象。

```c
void updateSlavesWaitingBgsave(int bgsaveerr, int type);
```

对于等待**RDB**文件的**Slave**，会将其对应的客户端状态`client.replstate`从`SLAVE_STATE_WIAT_BGSAVE_END`的状态切换至`SLAVE_STATE_SEND_BULK`的状态，这个新状态表示这个的**Slave**客户端进入了传输同步数据的状态。同时在**Master**实例上为这个**Slave**对应的`client`对象在事件循环之中注册可写事件的处理函数`sendBulkToSlave`，这也就意味着基于有盘**RDB**文件的**数据同步**，数据的传输是在*Redis*的主线程之中进行的。

```c
void sendBulkToSlave(aeEventLoop *el, int fd, void *privdata, int mask);
```

`sendBulkToSlave`这个接口会在发送正式的同步数据之前，向*Slave*实例以下面的格式发送**RDB**文件的长度数据：

```
$<length>\r\n
```

通过这个消息，**Slave**实例可以在接收同步数据之前，获得整个待接收数据的大小。接下来`sendBulkToSlave`接口会调用调用`read`系统调用从**RDB**文件之中读取`PROTO_IOBUF_LEN`大小的数据，并调用`write`系统调用接口，将数据发送给**Slave**实例。当所有的数据都发送结束之后，*Redis*会将这个客户端的可写事件处理函数从事件循环之中移除，以结束数据的传输，最后调用`putSlaveOnline`接口完成**数据同步**的最后操作。

```c
void putSlaveOnline(client *slave);
```

`putSlaveOnline`这个接口主要会执行三个逻辑的处理：

1. **Slave状态的刷新**，将*Slave*对应的客户端状态`client.replstate`从`SLAVE_STATE_SEND_BULK`切换至在线状态`SLAVE_STATE_ONLINE`。
1. **重写注册可写事件处理函数**，重新为*Slave*对应的客户端注册可写事件回调处理函数`sendReplyToClient`，用以将执行数据全量同步期间*Master*产生的新命令发送给客户端，以完成数据的同步。
1. **刷新Good Slave数量**，调用`refreshGoodSlavesCount`函数接口，来执行数量刷新的逻辑。

##### 无盘的RDB文件数据同步

前面我们在讲有盘的**数据同步**时介绍了，*Master*在传输正式的同步数据之前，会先将**RDB**文件的长度发送给**Slave**；但是对于无盘的**数据同步**，其本质上是相当于在**RDB**文件的生成过程之中直接就将数据发送给**Slave**，这意味着**Slave**无法提前获知文件的大小，这样在**Slave**一侧也就无法得知**数据同步**的传输是否已经结束。因此需要一种机制，通知**Slave**传输何时结束。为了解决这个问题，*Redis*设计了这样一种方式，在发送正式的**RDB**文件之前，**Master**会向**Slave**发送一段如下面格式的数据：

    $EOF:<40位的随机十六进制字符串>\r\n

这条数据用于通知**Slave**实例，数据传输结束的标记，一旦**Slave**再次收到这段随机字符串，那么就以为意味着数据传输的结束。不过这种情况，需要**Slave**一侧的*Redis*版本能够支持以`SLAVE_CAPA_EOF`的方式来接收数据，对于不支持`SLAVE_CAPA_EOF`的**Slave**，只能使用有盘的**数据同步**。

接下来，我们就详细介绍一下在**Master**实例这边无盘**数据同步**的实现细节。

在*Master*一端，与有盘**数据同步**类似，也是通过`startBgsaveReplication`接口来启动的，不过这里*Redis*为无盘**数据同步**定义了一个新的函数接口`rdbSaveToSlaveSockets`，从名字可以看出，这是一个通过Socket网络连接来生成存储**RDB**文件的过程。

```c
int rdbSaveToSlavesSockets(rdbSaveInfo *rsi)
{
    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1)
        return C_ERR;

    listRewind(server.slaves, &li);
    while ((ln = listNext(&li)))
    {
        client *slave = ln->value;
        if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START)
        {
            fds[numfds++] = slave->fd;
            replicationSetupSlaveForFullResync(slave, getPsyncInitialOffset);
            anetBlock(NULL, slave->fd);
            anetSendTimeout(NULL, slave->fd, server.repl_timeout*1000);
        }    
    }

    if ((childpid = fork()) == 0)
    {
        rio slave_sockets;
        rioInitWithFdset(&slave_sockets, fds, numfds);
        
        rdbSaveRioWithEOFMark(&slave_sockets, NULL, rsi);
        
        rioFlush(&slave_sockets);

        rioFreeFdset(&slave_sockets);
        exitFromChild(0);
    }
    else
    {

    }
}
```

这里从上面`rdbSaveToSlavesSockets`的代码片段，我们可以总结出这个无盘**数据同步**的一些细节：

1. 首先无盘**数据同步**也是通过启动后台子进程来异步进行的。
1. 在开启数据传输之前，遍历`redisServer.slaves`中所有的**Slave**，收集处于`SLAVE_STATE_WAIT_BGSAVE_START`状态的**Slave**，收集其对应网络连接的套接字文件描述符，并调用`replicationSetupSlaveForFullResync`，将其状态切换至`SLAVE_STATE_WAIT_BGSAVE_END`。
1. 创建子进程，在子进程之中通过上面收集到的套接字文件描述符调用`rioInitWithFdset`来初始化一个通用IO对象`rio`，
1. 最后子进程调用`rdbSaveRioWithEOFMark`这个接口，完成**RDB**文件数据的生成与传输。

最后在子进程结束后，**Master**实例依然会通过调用`updateSlavesWaitingBgsave`，不过这里和有盘**数据同步**的一点差异在于，有盘**数据同步**会将**Slave**对应的`client`对象从`SLAVE_STATE_WAIT_BGSAVE_END`状态切换至`SLAVE_STATE_SEND_BULK`的状态，进入数据发送的阶段；然而对于无盘**数据同步**在生成**RDB**的过程之中，就已经将数据同步给了**Slave**，这样**Slave**实例在`SLAVE_STATE_WAIT_BGSAVE_END`状态结束之后，会跳过`SLAVE_STATE_SEND_BULK`状态，直接进入`SLAVE_STATE_ONLINE`的状态。

##### Slave对于同步数据的接收

如果**Slave**在`slaveTryPartialResynchronization`函数中获取到了**Master**发来的形如：

```
+FULLRESYNC <replid> <reploffset>\r\n
```

这样的返回数据，则说明此时需要进行全量的数据同步，这时缓存的**Master**对应的`client`对象`redisServer.cached_master`将会被`replicationDiscardCachedMaster`函数释放。

依然是在`syncWithMaster`这个处理函数之中：

```c
void syncWithMaster(aeEventLoop *el, int fd, void *privdata, int mask)
{
    ...
    psync_result = slaveTryPartialResynchronization(fd,1);
    if (psync_result == PSYNC_CONTINUE)
    {
        return;
    }
    while(maxtries--)
    {
        dfd = open(tmpfile,O_CREAT|O_WRONLY|O_EXCL,0644);
        if (dfd != -1)
        	break;
        sleep(1);
    }
    aeCreateFileEvent(server.el,fd, AE_READABLE,readSyncBulkPayload,NULL);
    ...
    return;
}
```



这里我们看到，如果**Master**实例向**Slave**返回需要**全量数据同步**，那么**Slave**会创建一个临时的**RDB**文件，并注册可读事件的处理函数`readSyncBulkPayload`来接收来自**Master**的同步数据。

```c
void readSyncBulkPayload(aeEventLoop *el, int fd, void *privdata, int mask);
```

`readSyncBulkPayload`这个函数首先会从*Master*传输的数据之中解析出前序的附加信息：

1. 对于使用无盘数据同步的形式，则需要从`$EOF:<40位随机十六进制结束分隔符>\r\n`中解析出标记传输结束的标记字符串。
1. 对于使用有盘数据同步的形式，则需要从`$<length>\r\n`中解析出标记**RDB**总长度的数据信息。

接下来便是从连接之中持续读取数据，并将数据写入磁盘中的临时**RDB**文件之中，并判断传输是否结束。

当**RDB**文件传输结束时，*Slave*一侧会先清空键空间之中的全部数据，同时调用`rdbLoad`函数从**RDB**文件之中完成数据的加载，最后在调用`replicationCreateMasterClient`函数，从连接的套接字文件描述符之中，创建一个表示*Master*的客户端对象，最终完成整个数据的同步。

#### 增量数据同步

##### 增量数据的发送

**Master**实例会在**PSYNC**的命令处理函数`syncCommand`之中来判断**Slave**发来的**PSYNC**命令，是否可以进行一次增量的**数据同步**。

```c
void syncCommand(client *c)
{
    if (c->flags & CLIENT_SLAVE) return;
    ...
    if (!strcasecmp(c->argv[0]->ptr, "psync"))
    {
        if (masterTryPartialResynchronization(c) == C_OK) {
            server.stat_sync_partial_ok++;
            return;
        }
        else
        {
            ...
        }
    }
}
```

如果可以，便进行**增量数据同步**，否则便进行**全量数据同步**。

```c
int masterTryPartialResynchronization(client *c);
```

上面这个函数，是**Master**判断是否可以进行增量**数据同步**的接口，如果可以进行增量同步，那么函数会返回`C_OK`；否则的话，函数返回`C_ERR`，后续则会进行全量数据的同步。`masterTryPartialResyncChronization`这函数的实现逻辑较为简单：

1. 通过从客户端命令之中解析出**PSYNC**的参数`<replid>`以及`<reploffset>`，来判断这个**Slave**是否可以进行增量数据的同步：
   1. **Slave**发送的`<replid>`要与**Master**的`replid`相同。
   1. **Slave**通告的自己的复制偏移量`<reploffset>`要恰好落在**Master**的积压缓冲区内。
1. 由于省去了全量数据同步的过程，因此直接将**Slave**的状态设置为在线的状态`SLAVE_STATE_ONLINE`。
1. 向**Slave**发送一条返回消息，格式为`+CONTINUS\r\n`，用于通知客户端将要进行增量**数据同步**。
1. 通过`addReplyReplicationBacklog`将积压缓冲区之中的数据发送给**Slave**。
1. 最后通过`refreshGoodsSlaveCount`函数，来刷新**Slave**的计次。

##### Slave增量数据的接收

**Slave**实例会在`slaveTryPartialResynchronization`函数中等待从**Master**的返回数据。从前面介绍**Master**一侧的**数据同步**我们可以知道，如果**Slave**的**PSYNC**请求恰好可以完成一次增量的同步，**Master**会向**Slave**发送一个`+CONTINUE\r\n`的返回数据，通知可以进行增量**数据同步**；然后会加积压缓冲区之中的数据发送给**Slave**。而在**Slave**这一端，当检测返回数据为`+CONTINUE`时，也就意味着与**Master**重连成功，此时可以复用之前缓存的**Master**客户端对象`redisServer.cached_master`。

```c
void replicationResurrectCachedMaster(int newfd);
```

在**Master**积压缓冲区之中的增量同步数据，可以被认为是一系列的命令数据的集合，因此**Slave**在通过`replicationResurrectCachedMaster`重用`redisServer.cached_master`时，会为这个客户端注册可读事件的回调函数`readQueryFromClient`，后续**Slave**将以正常处理客户端命令的形式来处理**Master**积压缓冲区之中的增量同步数据。

## 命令同步

在前面介绍**数据同步**这一部分时可能会有一个疑问，如果在执行数据同步的时候**Master**又接收到了新的查询命令，导致数据发生了变化，而真正生成同步数据的又是在子进程之中进行的，这样一来要如何将新的数据的变化同步给**Slave**节点呢？这一部分将会就这个问题进行解释。

### 命令的转发

*Master*实例向*Slave*实例转发命令的过程，与向**AOF**文件之中追加命令的逻辑类似在执行客户端查询命令的`call`接口之中通过调用`propagate`函数来实现的：

```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc, int flags)
{
    if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
        feedAppendOnlyFile(cmd, dbid, argv, argc);
    if (flags & PROPAGATE_PREL)
        replicationFeedSlaves(server.slaves, dbid, argv, argc);
}
```
通过上面的代码，可以看出*Master*实例向*Slave*实例转发命令的逻辑，与追加**AOF**文件是先后进行的。

```c
void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc);
```
这个函数会将命令数据转发给`slaves`列表之中对应的所有的*Slave*实例，并将命令数据写入到服务器的积压缓冲区之中，这个函数会执行如下的逻辑：

1. 校验服务器的状态，检查是否可以执行命令转发的逻辑。这里需要注意的是，虽然主从复制支持层次话的结构，但是中间层的节点无法通过`replicationFeedSlaves`函数将命令转发给自己的*Sub-Slave*子节点，而是通过另外一个函数`replicationFeedSlavesFromMasterStream`函数来实现命令的转发的。
2. 如果执行命令针对的`dictid`和服务器上缓存的数据库ID`redisServer.slaveseldb`不同，那么会追加一条**SELECT**命令。
3. 将命令数据通过`feedReplicationBacklog`以及`feedReplicationBacklogWithObject`接口追加到积压缓冲区之中。
4. 最后遍历存储*Slave*实例的双端链表`slaves`，将命令转发给所有没处于`SLAVE_STATE_WAIT_BGSAVE_STATR`状态的*Slave*。对于处于`SLAVE_STATE_ONLINE`状态的*Slave*，命令数据直接会被发送到对端的*Slave*实例上；对于那些处于`SLAVE_STATE_WAIT_BGSAVE_END`或者`SLAVE_STATE_SEND_BULK`状态的实例，命令数据将会被写入客户端输出缓冲区里，等待全量数据同步结束后，通过前面所讲的重新注册可写事件处理函数`sendReplyToClient`，将缓冲区内的命令数据输出给*Slave*实例。这也就解释本节开头的那个疑问，同步途中的数据变化，要如何通知给**Slave**实例。

对于*Slave*实例在收到*Master*实例发送来的命令数据后也是会运行正常的命令执行逻辑，但是对于多层结构的主从复制来说，*Slave*实例将数据转发给*Sub-Slave*实例的逻辑并不是在`replicationFeedSlaves`接口之中实现的，而是在命令执行结束时，通过调用另外一个`replicationFeedSlavesFromMasterStream`接口实现的：

```c
void processInputBufferAndReplicate(client *c)
{
    if (!(c->flags & CLIENT_MASTER))
    {
        processInputBuffer(c);
    }
    else
    {
        size_t prev_offset == c->reploff;
        //在processInputBuffer之中会调用call函数接口执行查询命令
        processInputBuffer(c);
        size_t applied = c->reploff - prev_offset;
        if (applied)
        {
            replicationFeedSlavesFromMasterStream(server.slaves, c->pending_querybuf, applied);
            sdsrange(c->pending_querybuf,applied,-1);
        }
    }
}
```

`replicationFeedSlavesFromMasterStream`接口的作用与`replicationFeedSlaves`类似，也会将命令数据写入积压缓冲区并转发给下一级的*Sub-Slave*实例。之所以使用的这种方式，按照*Redis*给出的解释是，这么做是方便传递“相同”数据流，并允许*Slave*实例共享*Master*实例的复制历史记录，相同的积压缓冲区以及偏移量。

### 接收命令转发
```c
void replicationFeedSlavesFromMasterStream(list *slaves, char *buf, size_t buflen);
```
对于*Slave*一端来说，*Master*实例实际上是一个具有特殊标记的客户端对象。因此，*Master*在数据同步之后转发来的命令，在*Slave*一次可以按照正常处理客户端命令的逻辑来处理，这里对于处理细节不在详细介绍，可以回看前面关于*Redis*客户端以及*Redis*命令的相关内容。不过这里还有一点特殊的情况，就是对于层次结构的主从复制结构，*Slave*对象在接收到*Master*转发来的命令之后，还需要将其发送给自己的*Slave*，也就是*Sub-Slave*，而这个工作则是通过上面`replicationFeedSlavesFromMasterStream`这个接口完成的。

## 同步复制

### 确认机制

### 心跳机制



***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*