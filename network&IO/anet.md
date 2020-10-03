# Redis中基础TCP套接字接口

### 阻塞相关操作
在Linux/Unix操作系统中，通过`fcntl`函数，可以获取或者设置一个文件描述符的阻塞状态：
```c
    int flags;
    flags = fcntl(fd, F_GETFL);

    flags |= O_NONBLOCK;

    fcntl(fd, F_SETFL, flags);
```
对于一个文件描述符阻塞状态的操作，*Redis*在`fcntl`系统调用的基础上定义了下面三个较容易理解的函数：
```c
int anetSetBlock(char *err, int fd, int non_block);
int anetNonBlock(char *err, int fd);
int anetBlock(char *err, int fd);
```
其中`anetSetBlock`是这三个函数的基础，另外两个函数都是通过调用它来实现相关功能的。对于阻塞类*FD*，当调用`read/write/accept`等阻塞类操作时，会将进程阻塞，直到事件发生；而非阻塞类*FD*，当调用阻塞操作时，如果没有获得期待的结果，会立即在这些操作中返回，同时会设定`EAGAIN`错误码。

### TCP保活定时器的操作
根据《TCP/IP详解（卷1，协议）》一书中的描述：
> ......可以存在没有任何数据流通过一个空闲的TCP连接。...... 这意味着我们可以启动一个客户与服务器建立一个连接，......，但是只要两端的主机没有被重启，则连接依然保持建立。

这也就意味着，当TCP连接被异常中断时，两端的主机是无法感知到这一异常事件的发生。因此需要提供一种机制来检查一条长期空闲的TCP连接究竟是对端确实没有发送任何数据，还是连接异常被中断。基于此种需求，TCP实现了一个保活定时器的功能用于检测一条TCP连接是否依然存在，通常这个保活定时器的机制开启于服务器端，当然如果在客户端开启也并非不可以，其检测的流程为：
1. 如果服务检测到在一条TCP连接上2个小时都没有数据传输，那么会向这条连接上发送一个探测报文，如果对端主机响应这个探测报文，那么则认为这个连接正常，并将保活计定时器重置到两个小时之后。
2. 如果服务器发送探测报文之后75秒还没有收到对端的响应，那么认为这个探测报文响应超时，服务器会重新向连接上发送一个探测报文；当连续发送10个探测报文均超时的情况下，那么认为这一个连接已经被关闭。

借由套接字编程接口`setsockopt`，我们可以设置套接字的`SO_KEEPALIVE`接口设定这个套接字所对应的TCP连接是否启用保活定时器的机制。这样*Redis*便可以利用`setsockopt`这个接口定义了两个专门用于设置TPC保活定时器的函数：
```c
int anetTcpKeepAlive(char *err, int fd);
int anetKeepAlive(char *err, int fd, int interval);
```
上述这两个函数都是用于为一个套接字文件描述符开启保活定时器功能，那么这两个函数有什么区别呢？首先TCP默认保活定时器机制比较鸡肋，其检测的间隔只能是每2个小时启动一次，无法进行修改，`anetKeepAlive`函数就是通过设置`SO_KEEPALIVE`开启TCP的默认保活机制。对于*Linux*系统，2小时计时器，75秒超时，10次重发，这些都是可以配置的：

|选项名|*TCP*默认配置|含义|
|-----|-----------|----|
|`TCP_KEEPIDLE`|7200秒|空闲计时器的时间，当一条连接的空闲时间达到了该选项设置的时间，会启动探测机制|
|`TCP_KEEPINTVL`|75秒|探测超时的时间，发送一个探测报文后，多少时间内没有收到*ACK*，则认为超时|
|`TCP_KEEPCNT`|10次|重发探测数量，当所有的探测报文均超时的时候，会重发多少次|

函数`anetKeepAlive`可以开启TCP的保活检测机制，将空闲计时器的时间设置为`interval`，将探测超时时间设置为`interval/3`，将重发次数设置为3次。

### TCP的Nagle算法设定
为了减少网络上过多微小分组导致的拥塞问题，TCP设计了*Nagle*算法，该算法要求在连接上，最多只能有一个未被确认的未完成的微小分组，在该分组的确认到达之前不能再发送其他的微小分组，这个算法虽然减少了网络上发生拥塞的概率，但是提高了数据的延时。套接字可以使用`setsockopt`函数接口通过设置`TCP_NODELAY`选项的值来关闭或者开启对应TCP连接上的**Nagle**函数。*Redis*提供了相关的三个操作函数，可以用来开启和关闭连接上的**Nagle**算法。
```c
static int anetSetTcpNoDelay(char *err, int fd, int val);
int anetEnableTcpNoDelay(char *err, int fd);
int anetDisableTcpNoDelay(char *err, int fd);
```
这三个函数中`anetEnableTcpNoDelay`用于在连接上开启**Nagle**算法，在*Redis*，当服务器接收到客户端连接并为其构造一个`client`结构体时，会开启对应连接上的**Nagle**算法：
```c
client *createClient(int fd) {
    client *c = zmalloc(sizeof(client));
    
    if (fd != -1)
    {
        anetNonBlock(NULL, fd);
        anetEnableTcpNoDely(NULL, fd);
        if (server.tcpkeepalive)
            anetKeepAlive(NULL, fd, server.tcpkeepalive);
    }
    ...
}
```
通过上面这个创建客户端`client`结构体函数的片段，我们可以看到，*Redis*服务器接收一个新的客户端连接时，会将这个TCP连接对应的套接字文件描述符设置为非阻塞状态；同时调用`anetEnableTcpNoDely`函数关闭这条连接上的**Nagle**算法；如果服务器配置保活定时器的参数，则调用`anetKeepAlive`函数启动这条连接上的保活定时器。


### TCP发送选项设置
Linux系统之中每一个TCP连接在内核之中拥有一个发送缓冲区，当程序调用`write`系统调用向套接字发送数据时，数据实际上是被写入到内核的TCP发送缓冲区之中，而TCP发送缓冲区之中的数据究竟何时被发送到网络上，则是由内核决定的。当内核之中TCP发送缓冲区已满的情况下，如果套接字文件描述符是阻塞的状态，那么调用`write`系统调用时，进行将会被阻塞，知道内核将CTP发送缓冲区中的数据发送到网络上，使发送缓冲区中有空闲的空间可写；如果套接字文件描述符是非阻塞的状态，那么调用`write`则会立即返回，同时会设定`EAGAIN`错误码。

同时对于对于套接字的阻塞IO操作，我们可以设置对应的TCP套接字的发送超时间，如果因为某些原因发送行为没有正常进行，那么达到设置的超时时间后，将返回错误。

*Redis*给出下面两个函数，分别用于设置发送缓冲区的大小以及发送超时时间。
```c
int anetSetSendBuffer(char *err, int fd, int buffsize);
int anetSendTimeout(char *err, int fd, long long ms);
```

### TCP用于地址解析的接口
在Linux，系统提供了结构体`struct addrinfo`以及函数接口`getaddrinfo`用于地址查询，解析IP地址。*Redis*应用这个函数接口定义了下面的三个函数用于从域名解析出IP地址。
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
在TCP中，如果断开连接，那个会进入一个*TIME_WAIT*状态，这个状态会持续2个*MSL*的时间，在这段时间内，定义这个连接所对应的套接字将处于不可用的状态，这种设计的原则思想在于：
> 当*TCP*执行一个主动关闭，并发回最后一个*ACK*，该连接必须在*TIME_WAIT*状态停留的时间为2倍的*MSL*，这样可以让*TCP*再次发送最后的*ACK*以防这个*ACK*丢失。

对于服务器来说，通常我们不需要其在*TIME_WAIT*状态中不可用，这时我们可以使用`anetSetReuseAddr`通过`setsockopt`函数设置`SO_REUSEADDR`属性，来是连接在*TIME_WAIT*状态中可用。

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
我们可以发现，在上述的函数之中根据函数名字有几个函数采用的是非阻塞的`connect`操作。也许有人会疑问，发起连接是客户端的主动行为，为何要采用非阻塞的方式来调用`connect`操作呢？根据《UNIX网络编程》之中对于TCP套接字的`connect`以及`accept`介绍，我们可以知道，客户端调用`connect`主动发起连接时，相当于TCP建立连接三次握手之中，客户端向服务器发送的**SYN**报文，此时客户端进程会阻塞在`connect`操作之中，直到客户端收到服务器的应答**ACK**报文。这种情况对于客户端服务器同属于同一台主机或者在同一个局域网的情况下，客户端可能会立即收到或者在极短的时间内收到服务器的应该**ACK**报文；但是对于广域网环境，服务器与客户端的网络环境可能相差很大，客户端在发送**SYN**报文之后，很有可能需要等待数秒钟的时间才能够收到服务器的**ACK**报文。那么对于这种情况，使进程阻塞在`connect`操作而无法处理其他的逻辑显然不是一个很好设计。因此非阻塞`connect`操作便是针对这种情况而设计的。

当我们对一个非阻塞套接字文件描述符调用`connect`操作时，会向网络上的服务器对端发送一个**SYN**报文请求请求建立连接，同时函数会立即返回，并将`errno`错误码设置为`EINPROGRESS`；接下来需要我们将这个文件描述符加入事件循环队列并监听其可写事件，当服务器的应答**ACK**报文到达时，将触发该套接字上的可写事件，表明TCP连接建立成功。

另外我们发现，在上面这几个函数之中还有三个用于处理UNIX域套接字的函数，对于什么是UNIX域套接字，我们可以参考《UNIX网络编程》中对相关概念的描述：
> *UNIX*域套接字用于在同一台计算机上运行的进程之间的通信，比起因特网域套接字，*UNIX*域套接字的效率更高。
*UNIX*域套接字仅仅复制数据，它们并不执行协议处理，不需要添加或删除网络报头，无需计算校验和，不要产生顺序号，无需发送确认报文。

*客户端为什么要在执行`connect`之前，执行一次`bind`调用？？？*


### TCP处理套接字上的读写操作
```c
int anetRead(int fd, char *buf, int count);
int anetWrite(int fd, char *buf, int count);
```
函数`anetRead`会从连接对应的文件描述符上调用`read`系统调用，读取最多`count`个字节的数据到`buf`缓冲区上。
函数`anetWrite`会向连接对应的文件描述符上，通过调用`write`系统调用，从`buf`缓冲区上写入最多`count`个字节。

### TCP以服务器方式启动套接字
```c
static int anetListen(char *err, int s, struct sockaddr *sa, socklen_t len, int backlog);
static int anetV6Only(char *err, int s);
static int _anetTcpServer(char *err, int port, char *bindaddr, int af, int backlog);
int anetTcpServer(char *err, int port, char *bindaddr, int backlog);
int anetTcp6Server(char *err, int port, char *bindaddr, int backlog);
int anetTcp6Server(char *err, char *path, mode_t perm, int backlog);
```

静态函数`anetListen`是一个基础操作函数，该函数首先调用`bind`系统调用，将文件描述符`s`和地址`sa`进行绑定，然后调用`listen`系统调用，为该条连接设置积压值，并启动监听。静态函数`_anetTcpServer`也是一个基础操作函数，构建一个套接字连接，同时调用`anetListen`函数，启动监听。

而静态函数`_anetTcpServer`则是*Redis*用于创建服务器监听的通用接口，根据参数中的绑定地址`bindaddr`，绑定端口`port`，域参数`af`以及积压值`backlog`来创建一个可以重用的套接字文件描述符，并通过`anetListen`函数接口来进行地址的绑定与`listen`设定。

而函数`anetTcpServer`、`anetTcp6Server`、`anetTcp6Server`则分别对应于**IPV4**和**IPV6**以及UNIX域的服务器套接字的建立。

在*Redis*服务器启动逻辑之中，会调用上述的函数来创建服务器监听用的套接字：
```c
int listenToPort(int port, int *fds, int *count) {
    ...
    for (j = 0; j < server.bindaddr_count || j == 0; j++)
    {
        if (server.bindaddr[j] == NULL) {
            ...
            fds[*count] = anetTcpServer(server.neterr,port,NULL,server.tcp_backlog);
            ...
        }
        else if (strchr(server.bindaddr[j],':')) 
        {
            ...
            fds[*count] = anetTcp6Server(server.neterr,port,server.bindaddr[j],server.tcp_backlog);
        }
        else 
        {
            ...
            fds[*count] = anetTcpServer(server.neterr,port,server.bindaddr[j],server.tcp_backlog);
        }
        ...
    }
    ...
}
```

### TCP处理套接字接收连接的操作接口
```c
static int anetGenericAccept(char *err, int s, struct sockaddr *sa, socklen_t *len);
int anetTcpAccept(char *err, int s, char *ip, size_t ip_len, int *port);
int anetUnixAccept(char *err, int s);
```
上面这三个函数则是通过调用`accept`系统调用，从监听套接字`s`之中建立一条与客户端之间的连接，并返回这个新连接对应的套接字文件描述符。

在服务器启动的`initServer`函数中，*Redis*会将前面调用`anetTcpServer`创建的监听套接字加入事件循环之中，同时为其可读事件注册处理函数`acceptTcpHandler`：
```c
void initServer(void)
{
    ...
    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,acceptTcpHandler,NULL);
    }
    ...
}
```
在事件处理函数`acceptTcpHandler`之中，*Redis*会调用上述`anetTcpAccept`接口用于接收客户端主动发起的连接。

### 用于将连接信息转化为字符串
```c
int anetPeerToString(int fd, char *ip, size_t ip_len, int *port);
int anetFormatAddr(char *buf, size_t buf_len, char *ip, int port);
int anetFormatPeer(int fd, char *buf, size_t buf_len);
int anetSockName(int fd, char *ip, size_t ip_len, int *port);
int anetFormatSock(int fd, char *fmt, size_t fmt_len);
```
上述这几个函数则主要是用于将地址网络地址信息转化为可读的字符串，与网络的核心逻辑关联不大，因此不做详细解释。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*