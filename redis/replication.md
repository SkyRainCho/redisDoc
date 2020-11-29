# Redis主从复制

虽然*Redis*单机可以承载十万量级的并发请求，但是依然具有处理能力的上限，考虑正常情况下，对于数据的读取请求要远大对数据的写入请求。基于这个前提，*Redis*设计了一个复制的机制，应用这个复制机制，其他的*Redis*服务器可以拥有一个不断更新的数据副本，这样其他拥有数据副本的*Redis*服务器可以分担用户的读取请求，这样一来可以进一步提升*Redis*的并发能力，本篇文章将对*Redis*主从复制机制的相关功能与实现细节进项介绍。


## 复制功能概述
首先，彼此连接的*Master*实例与*Slave*实例是互为客户端的关系，在前面我们介绍客户端相关内容时，我们介绍了`client.flags`这个字段以掩码的形式记录了当前客户端的状态。而*Master*实例与*Slave*实例从各自的角度看对方，都是一个`client`对象，而在*src/server.h*之中也为*Master*与*Slave*实例定义了对应的状态掩码：
```c
#define CLIENT_SLAVE (1 << 0)
#define CLIENT_MASTER (1 << 1)
```

*Redis*本身也支持一个*Matser*实例连接多个*Slave*实例的模式，这样可以进一步提升系统的并发能力。*Redis*甚至可以在配置文件*redis.conf*之中通过配置项`min-reolicas-to-write`来指定*Master*实例所需要的*Slave*实例的最小数量，一旦*Matser*实例对应的*Slave*数量不足，则可以认为这个*Master*处于一种异常的状态，*Master*实例可以拒绝在这个状态下执行写命令。

然而如果一个*Master*实例连接了过多的*Slave*实例的话，*Master*与*Slave*之间的数据传输对于*Master*实例本身来说也会造成一定的网络负载，因此*Redis*还支持一种层次化的主从模式，也就是说一个*Slave*实例也可以拥有它自己的*Slave*，我们可以称之为*Sub-Slave*。基于这种层次化的结构，顶层的*Master*实例只需要与若干个*Slave*实例进行数据传输，在有这些与顶层*Master*直连的*Slave*将数据发送给更为下层的*Slave*实例，以降低数据传输对*Master*实例性能所带来的影响。

简单来说*Redis*的复制功能主要可以划分两个阶段：
1. **数据同步**：*Slave*节点通过某种方式获取*Master*节点的数据副本，并将其加载到内存键空间之中，使该*Slave*节点成为*Master*节点的镜像节点。
1. **命令转发**：*Master*节点在执行命令时，将该命令转发给自己的*Slave*节点，以便在运行之中保证*Master*实例与*Slave*实例之间数据的一致性。

### 数据同步
*Redis*可以通过两种方式开启**数据同步**：
1. 通过配置文件，在*redis.conf*配置文件之中，配置`replicaof <masterip> <masterport>`这条配置项，可以设置对应*Master*节点的地址；如果*Master*实例需要密码认证，可以通过`masterauth <master-password>`配置项来设置连接*Master*所需要认证密码。
1. 执行**REPLICAOF**命令，命令的格式为`REPLICAOF host port`，通过这条命令，可以使执行这条命令的*Redis*成为给定地址`host`以及`port`的另一个*Redis*实例的*Slave*节点，不过这条命令有一个缺陷，如果你对应的*Master*节点需要密码验证，而在*redis.conf*配置文件之中没有提前设定*Master*节点的认证密码，那么需要在执行**REPLICAOF**命令之前，先通过**CONFIGSET**命令来设置`masterauth`的认证密码，否则将无法建立主从实例之间的数据同步。

无论上面哪种建立主从复制的方式，**数据同步**都不是立即开始的，这两种方式仅是为将要充当*Slave*的服务器设置了*Master*的地址，真正开启数据同步则是在设置了*Master*地址信息之后，*Slave*通过一系列内部命令来实现的：
1. 通过设置的*Master*地址信息，建立*Slave*与*Master*之间的Socket连接。
1. *Slave*向*Master*发送**PING**命令以检验二者连接是否正常。
1. 如果*Master*需要密码认证，则发送**AUTH**命令验证密码。
1. *Slave*会向*Master*发送一系列的**REPLCONF**命令，将*Slave*的信息通知给*Master*。
1. *Slave*向*Master*发送**PSYNC**命令，以正式的开启**数据同步**。

数据同步相当于将*Master*上键空间之中的全部数据发送给*Slave*实例，如果看过前面对于**RDB**持久化的介绍，我们可以发现这其中有很多相似之处，而实际上*Master*实例与*Slave*实例之间的数据同步也是通过**RDB**的机制来实现的。在*Redis*之中，复制机制中待同步数据的准备与传输方式有两种：

1. *Redis*使用正常的方式，启动一个后台子进程来生成**RDB**文件到磁盘上，等待磁盘上的*RDB*文件生成结束，在通过网络连接将这个**RDB**文件发送给*Slave*节点。
2. 如果*Master*实例所在的服务器上的磁盘性能很低的话，那么先向磁盘中写入数据再从磁盘进行读取与网络传输的话，会导致数据同步的效率很低。对于这种情况，*Redis*设计了一种无盘的数据同步方式，也就是不通过磁盘文件，而是在生成**RDB**文件的过程之中，直接将数据写入与*Slave*实例所建立的连接的套接字文件描述符上，*Slave*实例直接通过网络来接收*Master*发来的同步数据，这样可以大大提升针对单一*Slave*实例同步的速度。而这得益于**Linux**操作系统中*一切皆文件*的设计理念以及*Redis*自身实现的`rio`通用IO对象数据结构。

虽然采用无盘方式进行数据同步的策略，会提升单一*Slave*实例的同步效率，免去了磁盘IO所带来的延时，但是这种方式也有一个问题，便是在多个*Slave*需要数据同步的时候，无法复用**RDB**文件，一旦为一个*Slave*实例开启了无盘的**数据同步**，如果另外一个*Slave*实例也来请求**数据同步**的话，*Master*也不得不进行重复两次**RDB**的生成过程。那么采用有磁盘的方式来进行数据同步的话，如果在生成**RDB**文件的过程中，又有一个*Slave*实例期望与*Master*实例之间进行数据同步的话，那么我们可以在**RDB**文件生成结束之后，将这个文件发送给多个等待数据同步的*Slave*实例，免去了重复生成**RDB**文件所带来的系统负载。

上面所介绍的**数据同步**方式，无论是有盘的还是无盘的，其本质都是对*Master*实例上的数据进行一次全量的备份，*Slave*实例将网络连接上接收到的同步数据写入一个本地文件之中，当传输接收之后通过`rdbLoad`接口使用这个全量的数据备份还原*Master*上的数据。通过后面我们将要介绍的**命令转发**，*Master*与*Slave*之间可以保持数据上的一致性。但是如果在日常的运行之中，*Master*与*Slave*之间的网络连接出现问题，被转发的命令无法传递到*Slave*实例上，这将导致二者数据的不一致，为了解决这个问题，当*Slave*与*Master*直接重新建立连接时，会重新执行一次**数据同步**以继续维持两者间数据的一致性。早期的*Redis*也确实使用这种方式在重连时进行全量的**数据同步**的，但是这其中也存在一个问题，

*如果两者之间断线的时间很短呢？是否有必要为了极小的数据差异进行一次全量的数据同步呢？*。

在新版本的*Redis*之中，作者针对这个问题做出了一个优化，也就是引入了**增量数据同步**的机制，应用这个机制，*Redis*可以在*Master*实例与*Slave*实例数据差异较小的情况下，仅对增量的数据差异进行同步，而无须再进行全量的**数据同步**，可以大大提升同步的效率。

所谓**增量数据同步**，其设计思路是：
1. 在*Master*一侧，
    1. 维护一个唯一复制ID`replid`，用于区分各个*Redis*实例；
    1. 维护一个复制偏移`master_repl_offset`，用于记录当前*Master*实例累积转发命令的计数，每转发一条命令，便会将该命令的长度信息累加到`master_repl_offset`这个偏移量上；
    1. 维护一个有限长度的，先进先出的积压缓冲区`repl_backlog`，当一条命令被转发给*Slave*实例的时候，也会被写入积压缓冲区之中，同时如果积压缓冲区已满，会将最早被写入的数据清空，以便能够继续写入新的命令。
1. 而在*Slave*一层，代表*Master*的客户端之中也存储了*Master*实例同步过来的复制偏移量。

正常请求下，*Master*实例与*Slave*实例之中，各自的复制偏移量应该是相同的。一旦某个*Slave*实例掉线，那么将导致*Slave*上存储的复制偏移量落后于*Master*实例中的复制偏移量。如果这部分差异的数据恰好落在*Master*实例的积压缓冲区里的话，那么在重新连接建立之后，*Master*只需要将积压缓冲区里对应的数据发送给*Slave*实例就可以，而无需重新执行**RDB**的生成过程。只要在第一次建立连接或者重连时两者数据差异过大的时候，才会执行**RDB**的生成过程。

### 命令转发
参考前面介绍的**AOF**持久化策略，可以很好地理解命令转发这一概念。在*Master*实例与*Slave*实例完成数据同步之后，*Master*会在运行过程中将执行的命令发送给*Slave*对应的`client`客户端对象，同时也会将这个命令写入积压缓冲区，以便后续执行**增量数据同步**的流程。而在*Slave*端，*Master*发送来的命令则会被*Master*对应的`client`对象接收，并完成命令的解析与命令的执行，将*Master*端执行的命令在*Slave*一层重新执行一次，已完成数据的同步。而对应层次化结构的主从模式，*Slave*实例在接收到*Master*发送的命令之后，还会将这个命令转发给自己的*Slave*实例，也就是*Sub-Slave*实例，帮助*Sub-Slave*完成数据的同步。

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
1. *Master*实例自身数据的维护：
    1. `redisServer.replid`，用于存储*Master*实例自身的复制ID，这是一个40位的随机字符串，用于对不同的*Redis*实例进行区分。
    1. `redisServer.master_repl_offset`，用于记录前面所提到的*Master*实例的复制偏移量。每次*Master*实例将命令写入到积压缓冲区之中的时候，会增加这个`master_repl_offset`复制偏移量。
    1. `redisServer.slaveseldb`，用于记录上一次执行复制命令对应的键空间编号。
1. *Master*实例上积压缓冲区：
    1. `redisServer.repl_backlog`，*Master*上的积压缓冲区，这个积压缓冲区是一段固定的被连续分配的内存。这个积压缓冲区是循环复用的，当写入的数据已经达到积压缓冲区的尾部时，新的数据会从积压缓冲区头部开始写入，并覆盖积压缓冲区之中的旧数据。
    1. `redisServer.repl_backlog_size`，积压缓冲区的长度。
    1. `redisServer.repl_backlog_histlen`，这个用于记录积压缓冲区之中实际数据的长度，这个字段数值的长度不会超过`repl_backlog_size`。 
    1. `redisServer.repl_backlog_idx`，用于标记循环积压缓冲区之中当前的数据偏移，如果有新的数据要被写入积压缓冲区的话，将会从`repl_backlog_idx`的下一个字节开始数据写入。
    1. `redisServer.repl_backlog_off`，这个用于记录积压缓冲区中有效数据的第一个字节对应于`master_repl_offset`的偏移量。
1. *Master*实例上关于*Slave*的统计与配置数据:
    1. `redisServer.repl_backlog_time_limit`，用于记录积压缓冲区的最大生存时间，如果*Master*实例长时间没有*Slave*实例连接，则会将积压缓冲区释放。
    1. `redisServer.repl_ping_slave_period`，用于配置*Master*实例与*Slave*实例发送**PING**命令的时间间隔。
    1. `redisServer.repl_no_slaves_since`，记录*Master*上开始没有*Slave*连接的开始时间。
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
}
```
另外还有一些数据字段用于维护*Slave*实例端的复制机制：
1. `redisServer.masterhost`，在*Slave*实例一侧，记录*Master*主机所在的地址，通常在代码里通过这个字段是否为空来判断当前的*Redis*是否是主从模式之中的一个*Slave*实例。
1. `redisServer.masterport`，在*Slave*一侧，记录*Master*对应的端口。
1. `redisServer.repl_syncio_timeout`，在*Slave*实例一侧，在向*Master*实例发送内部命令时，是采用阻塞整个进程的同步IO方式进行传输的，`repl_syncio_timeout`这个字段用于设定每次同步IO的超时时间。
1. `redisServer.repl_state`，用于记录在主从模式之中，当前*Slave*实例所处于的状态：
    1. `REPL_STATE_NONE`
    1. `REPL_STATE_CONNECT`，通过配置或者**REPLICAOF**命令设置了*Master*地址信息之后，*Slave*实例会被置为这个状态，这个状态表示*Slave*将要与*Master*实例建立连接。
    1. `REPL_STATE_CONNECTING`，当*Slave*已经设置了地址信息之后，会通过`anetTcpNonBlockBestEffortBindConnect`函数接口建立与*Master*实例的连接，当连接建立并注册到事件循环之后，*Slave*节点将会被设置成`PEPL_STATE_CONNECTING`这个状态。
    1. `REPL_STATE_RECEIVE_PONG`，由于连接是以非阻塞的方式建立的，因此第一次从事件循环之中返回时，表明连接已经被建立，此时*Slave*会向*Master*同步发送一条**PING**命令，并将状态设置为`REPL_STATE_RECEIVE_PONG`，以表明自己处于等待接受*Master*返回的状态。
    1. `REPL_STATE_SEND_AUTH`，这个状态表示*Slave*实例将要将*Master*发送认证命令**AUTH**。
    1. `REPL_STATE_RECEIVE_AUTH`，这个状态表示*Slave*实例已近发送送完**AUTH**命令，正在等待*Master*的返回。
    1. `REPL_STATE_SEND_PORT`，表示*Slave*实例将要向*Master*实例通过**REPLCONF**发送自己的端口数据。
    1. `REPL_STATE_RECEIVE_PORT`
    1. `REPL_STATE_SEND_IP`，表示*Slave*实例将要向*Master*实例通过**REPLCONF**发送自己的端口数据。
    1. `REPL_STATE_RECEIVE_IP`
    1. `REPL_STATE_SEND_CAPA`，表示*Slave*实例将要向*Master*实例通过**REPLCONF**发送自己的端口数据。
    1. `REPL_STATE_RECEIVE_CAPA`
    1. `REPL_STATE_SEND_PSYNC`，表示*Slave*实例将要向*Master*实例发送**PSYNC**命令以开启通过过程。
    1. `REPL_STATE_RECEIVE_PSYNC`，*Slave*发送完**PSYNC**命令后，等待*Master*实例的返回。
    1. `REPL_STATE_TRANSFER`，**数据同步**逻辑开始执行，*Slave*实例处于该状态从连接上获取*Master*发送到的同步数据。
    1. `REPL_STATE_CONNECTED`，**数据同步**逻辑结束，*Slave*将切换至该状态，后续将接受*Master*实例转发来的命令数据。
1. `redisServer.repl_transfer_size`，对于使用无盘的**数据同步**，该字段将会被设置为0；而对于有盘的**数据同步**，该字段为整个待传输的**RDB**文件的大小。
1. `redisServer.repl_transfer_read`，用于记录*Slave*实例从网络连接上已经接收到的数据的数量。
1. `redisServer.repl_transfer_last_fsync_off`，由于*Slave*实例收到的同步数据需要先写入磁盘，因此这个字段用于记录上次被写入到磁盘中的数据的偏移。
1. `redisServer.repl_transfer_s`，用于记录*Slave*与*Master*之间连接的套接字文件描述符。
1. `redisServer.repl_transfer_fd`，用于记录*Slave*实例介绍同步数据的临时**RDB**文件的文件描述符。
1. `redisServer.repl_transfer_tmpfile`，记录临时**RDB**文件的文件名。
1. `redisServer.repl_transfer_lastio`，记录*Slave*实例上一次接收同步数据的时间戳。
1. `redisServer.repl_server_stale_data`
1. `redisServer.repl_slave_ro`
1. `redisServer.repl_slave_ignore_maxmemory`
1. `redisServer.repl_down_since`
1. `redisServer.repl_disable_tcp_nodelay`
### 客户端对象中复制相关的数据结构
因为在*Redis*的复制机制之中，*Master*实例与*Slave*实例彼此将对方看做一个特殊的客户端，因此在`client`这个表示客户端的数据结构之中也存储了一系列用于维护复制功能相关的数据字段。
1. `client.replstate`，这个字段用于从*Master*的视角来看*Slave*对应客户端的状态：
    1. `SLAVE_STATE_WAIT_BGSAVE_START`，对应的*Slave*已经请求**数据同步**，正在等待*Master*开启**RDB**的生成逻辑。
    1. `SLAVE_STATE_WAIT_BGSAVE_END`，对应*Slave*同步请求的**RDB**文件生成流程已经开始执行，正在等待**RDB**文件生成结束。
    1. `SLAVE_STATE_SEND_BULK`，*Master*正在想这个`client`对应*Slave*实例发送**RDB**同步数据。
    1. `SLAVE_STATE_ONLINE`，*Master*已经向这个`client`对应的*Slave*实例完成发送**RDB**文件，开始进入命令转发的流程。
1. `client.repl_put_online_on_ack`
1. `client.repldbfd`，*Master*用于同步的**RDB**文件的文件描述符。
1. `client.reoldboff`，*Master*已经传输的**RDB**文件的大小。
1. `client.repldbsize`，**Master**传输的**RDB**文件的总大小。
1. `client.replpreamble`，用于记录*Master*传输的**RDB**文件的一些前缀信息。
1. `client.read_reploff`
1. `client.reploff`，在*Slave*上记录对应*Master*所发送的数据中已经被处理的数据偏移。
1. `client.repl_ack_off`
1. `client.repl_ack_time`
1. `client.psync_initial_offset`，在*Master*实例一段用于记录对应*Slave*的初始偏移。
1. `client.replid`，在*Slave*实例上记录*Master*实例的复制ID。
1. `client.slave_listening_port`，在*Master*实例上记录的*Slave*通过**REPLCONF**命令传来的端口数据。
1. `client.slave_ip`，在*Master*实例上记录的*Slave*通过**REPLCONF**命令传来的IP数据。
1. `client.slave_capa`，在*Master*实例上记录的*Slave*通过**REPLCONF**命令传来的capa数据。
1. `client.woff`
## 从Master角度看复制功能

### 积压缓冲区的维护

#### 积压缓冲区的创建

在*Master*实例的一侧，*Redis*会维护一段连续分配的内存作为积压缓冲区，用于存储近期被同步给*Slave*的命令，以便当连接断开重连时，*Redis*执行部分的增量同步。初始情况下，*Redis*不会为积压缓冲区分配内存，当有需要的时候，则会通过下面这个函数来创建一个积压缓冲区：

```c
void createReplicationBacklog(void);
```

*Master*实例上会在下面两种情况下通过调用`createReplicationBacklog`接口来创建积压缓冲区：

1. 当第一个*Slave*实例通过**PSYNC**命令与*Master*开启同步时，会创建积压缓冲区。
2. 当有*Slave*实例断线请求重连时，如果积压缓冲区已经被释放，则会重新创建一个积压缓冲区。

在*Slave*一侧，由于主从复制支持层次结构，这也就意味着，一个*Slave*也可能是它的下级*Slave*的*Master*，因此在*Slave*一端也需要维护一段积压缓冲区，当一个*Slave*通过`readSyncBulkPayload`接口开始读取*Master*的同步数据时，也会通过`createReplicationBacklog`来创建一段积压缓冲区。

积压缓冲区的大小通过`redisServer.repl_backlog_size`来存储，不过我们可以在运行期通过**CONF**命令来重新调整*Redis*的积压缓冲区的大小：

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
void feedReplicationWithObject(robj *o);
```

这两个函数分别用于将一段内存数据写入积压缓冲区以及将一个对象数据结构`robj`写入解压缓冲区。

### 与Slave建立连接

### 执行数据同步

### 执行命令转发


## 从Slave角度看复制功能

### 与Master建立连接

### 执行数据同步

### 接收命令转发

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*
