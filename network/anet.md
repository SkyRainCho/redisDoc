# Redis中基础TCP套接字接口

## Redis中设置TCP套接字属性的操作接口

### 阻塞相关操作
```c
int anetSetBlock(char *err, int fd, int non_block);
int anetNonBlock(char *err, int fd);
int anetBlock(char *err, int fd);
```
其中`anetSetBlock`是这三个函数的基础，另外两个函数都是通过调用它来实现相关功能的。
对于阻塞类*FD*，当调用`read/write/accept`等阻塞类操作时，会将进程阻塞，直到事件发生；
而非阻塞类*FD*，当调用阻塞操作时，如果没有获得期待的结果，会立即在这些操作中返回，
同时会设定`EAGAIN`错误码。而`anetSetBlock`会通过调用`fcntl`系统调用，
来为给定的*FD*设定是阻塞模式，还是非阻塞模式。

### TCP保活机制的设定
```c
int anetTcpKeepAlive(char *err, int fd);
int anetKeepAlive(char *err, int fd, int interval);
```
在*TCP*中，如果一个空闲的*TCP*连接上没有任何数据传输，那么连接的两端无法得知，该连接是否依然可用。
故此*TCP*提供了*KeepAlive*机制，可以检测当前连接是否存在，其基本的概念是：
1. 如果在一条连接上，2个小时都没有数据传输，那么会启动*KeepAlive*机制。
2. 设定了*KeepAlive*的一端会向对端发送一个探测报文，如果75秒还没有收到对端的*ACK*，那么认为超时；
当连续发送10个探测报文均超时的情况下，那么认为这一个连接已经被关闭。

`anetTcpKeepAlive`函数使用`setsockopt`系统调用，通过设置`SO_KEEPALIVE`选项，来为*TCP*启动保活机制。
对于*Linux*系统，2小时计时器，75秒超时，10次重发，这些都是可以配置的：
|选项名|*TCP*默认配置|含义|
|-----|-----------|----|
|`TCP_KEEPIDLE`|7200秒|空闲计时器的时间，当一条连接的空闲时间达到了该选项设置的时间，会启动探测机制|
|`TCP_KEEPINTVL`|75秒|探测超时的时间，发送一个探测报文后，多少时间内没有收到*ACK*，则认为超时|
|`TCP_KEEPCNT`|10次|重发探测数量，当所有的探测报文均超时的时候，会重发多少次|

函数`anetKeepAlive`可以开启TCP的保活检测机制，将空闲计时器的时间设置为`interval`，将探测超时时间设置为`interval/3`，
将重发次数设置为3次。

### TCP的Nagle算法设定
为了减少网络上过多微小分组导致的拥塞问题，TCP设计了*Nagle*算法，该算法要求在连接上，最多只有有一个未被确认的未完成的微小分组，
在该分组的确认到达之前不能再发送其他的微小分组，这个算法虽然减少了网络上发生拥塞的概率，但是提高了数据的延时。因此*Redis*提供了
相关的操作函数，可以用来开启和关闭*TCP*上的*Nagle*算法。
```c
static int anetSetTcpNoDelay(char *err, int fd, int val);
int anetEnableTcpNoDelay(char *err, int fd);
int anetDisableTcpNoDelay(char *err, int fd);
```
使用`setsockopt`函数，通过设定`TCP_NODELAY`选项的值来开启或者关闭*Nagle*算法。

### TCP发送选项设置
```c
int anetSetSendBuffer(char *err, int fd, int buffsize);
int anetSendTimeout(char *err, int fd, long long ms);
```
这两个接口函数分别用于设定*Redis*的*TCP*发送缓冲区大小，以及发送超时时间。

### TCP用于地址解析的接口
```c
int anetGenericResolve(char *err, char *host, char *ipbuf, size_t ipbuf_len, int flags);
int anetResolve(char *err, char *host, char *ipbuf, size_t ipbuf_len);
int anetResolveIP(char *err, char *host, char *ipbuf, size_t ipbuf_len);
```

### TCP创建套接字接口
```c
static int anetSetReuseAddr(char *err, int fd);
static int anetCreateSocket(char *err, int domain);
```
在*TCP*中，如果断开连接，那个会进入一个*TIME_WAIT*状态，这个状态会持续2个*MSL*的时间，
在这段时间内，定义这个连接所对应的套接字将处于不可用的状态，这种设计的原则思想在于：
> 当*TCP*执行一个主动关闭，并发回最后一个*ACK*，该连接必须在*TIME_WAIT*状态停留的时间
为2倍的*MSL*，这样可以让*TCP*再次发送最后的*ACK*以防这个*ACK*丢失。

对于服务器来说，通常我们不需要其在*TIME_WAIT*状态中不可用，
这时我们可以使用`anetSetReuseAddr`通过`setsockopt`函数设置`SO_REUSEADDR`属性，来是连接在*TIME_WAIT*状态中可用。

而通过调用`anetCreateSocket`，我们可以创建一个连接地址可以重用的套接字，并返回套接字的文件描述符。

### TCP处理套接字Connect接口
```c
static int anetTcpGenericConnect(char *err, char *addr, int port, char *source_addr, int flags);
int anetTcpConnect(char *err, char *addr, int port);
int anetTcpNonBlockConnect(char *err, char *addr, int port);
int anetTcpNonBlockBindConnect(char *err, char *addr, int port, char *source_addr);
int anetTcpNonBlockBestEffortBindConnect(char *err, char *addr, int port, char *source_addr);
int anetUnixGenericConnect(char *err, char *path, int flags);
int anetUnixConnect(char *err, char *path);
int anetUnixNonBlockConnect(char *err, char *path);
```
> *UNIX*域套接字用于在同一台计算机上运行的进程之间的通信，比起因特网域套接字，*UNIX*域套接字的效率更高。
*UNIX*域套接字仅仅复制数据，它们并不执行协议处理，不需要添加或删除网络报头，无需计算校验和，不要产生顺序号，无需发送确认报文。

### TCP处理套接字上的读写操作
```c
int anetRead(int fd, char *buf, int count);
int anetWrite(int fd, char *buf, int count);
```

### TCP以服务器方式启动套接字
```c
static int anetListen(char *err, int s, struct sockaddr *sa, socklen_t len, int backlog);
static int anetV6Only(char *err, int s);
static int _anetTcpServer(char *err, int port, char *bindaddr, int af, int backlog);
int anetTcpServer(char *err, int port, char *bindaddr, int backlog);
int anetTcp6Server(char *err, int port, char *bindaddr, int backlog);
int anetUnixServer(char *err, char *path, mode_t perm, int backlog);
```

### TCP处理套接字接收连接的操作接口
```c
static int anetGenericAccept(char *err, int s, struct sockaddr *sa, socklen_t *len);
int anetTcpAccept(char *err, int s, char *ip, size_t ip_len, int *port);
int anetUnixAccept(char *err, int s);
```

### 用于将连接信息转化为字符串
```c
int anetPeerToString(int fd, char *ip, size_t ip_len, int *port);
int anetFormatAddr(char *buf, size_t buf_len, char *ip, int port);
int anetFormatPeer(int fd, char *buf, size_t buf_len);
int anetSockName(int fd, char *ip, size_t ip_len, int *port);
int anetFormatSock(int fd, char *fmt, size_t fmt_len);
```