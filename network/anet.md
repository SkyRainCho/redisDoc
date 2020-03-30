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