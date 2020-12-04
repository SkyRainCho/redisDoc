# Redis服务器入口

在前面的文章之中，我们以及介绍了*Redis*之中的基础数据结构、对象数据类型、数据库键空间、网络与客户端、主从复制的相关内容。在这篇文章之中，我们将对*Redis*服务器本身做一个讲解，也就是当我们启动了一个*Redis*服务器的话，其在服务器一端，都处理了哪些内容，以及是以什么样的流程对这些逻辑进行处理的。

## Redis服务器启动的基本流程
*src/server.c*这个源文件，这里是*Redis*服务器的入口点，其中定义了`main`函数。通过这个文件中的`main`函数，我们可以了解到*Redis*服务器在启动时的一个基本流程。

### 服务器配置数据的初始化与加载
在服务器的全量变量`redisServer`结构体之中。定义了*Redis*运行时的配置信息，例如服务器的心跳频率`redisServer.hz`，服务器键空间数据库的数量`redisServer.dbnum`等等，这些配置信息需要在服务器启动的最开进进行初始化。*Redis*将这个工作分成了两个部分：
1. 配置信息的初始化，`initServerConfig`。
1. 配置信息的加载，`loadServerConfig`。

### 服务器配置数据的初始化
```c
void initServerConfig(void);
void populateCommandTable(void);
```
*Redis*在*src/server.h*头文件之中，通过`define`定义了一系列的类似于`CONFIG_DEFAULT_XXX`这样的定义项，用于设置一些服务器配置参数的默认值。通过上面的`initServerConfig`函数，*Redis*可以为`redisServer`中的配置参数设置初始化的默认值。除此之外这个接口还会通过调用`populateCommandTable`函数，遍历*src/server.c*源文件中硬编码的命令表，逐条将命令插入到服务器的命令字典`redisServer.commands`之中。对一部分，在前面介绍*Redis*命令的部分之中进行了详细的介绍。

### 服务器配置数据的加载
```c
void loadServerConfig(char *filename, char *options);
```
当我们希望定制*Redis*服务器的一些配置信息时，我们可以通过配置`redis.conf`这个配置文件来实现。*Redis*服务器在通过`initServerConfig`接口设置了服务器的默认配置之后，会调用`loadServerConfig`这个函数，尝试加载并解析配置文件，更新服务器的配置信息。这部分关于配置数据的解析内容，被*Redis*定义在了*src/config.c*这份源文件之中。


## Redis守护进程
如果设置了*Redis*以后台的方式运行，那么*Redis*会在`main`函数之中调用`daemonize`函数，将*Redis*服务器进程转化为守护进行，这里贴出`daemonize`的代码细节，通过这个代码我们也可以了解到如果讲一个普通进程转化为守护进程。
```c
void daemonize(void)
{
    int fd;
    if (fork() != 0) exit(0);
    setsid();

    if ((fd = open("/dev/null", O_RDWR, 0)) != -1)
    {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > STDERR_FILENO)
            close(fd);
    }
}
```

## Redis服务器的初始化
```c
void initServer(void);
void InitServerLast();
```
在`redisServer`这个服务器全局数据结构之中，除了服务器的配置数据之外，还有很多服务器运行时需要的数据，*Redis*会通过`initServer`函数以及`InitServerLast`函数，完成这些数据的初始化过程。

在上述两个初始化函数中`initServer`接口承担了初始化*Redis*数据的主体工作：
1. 首先`initServer`函数会负责服务器全局变量之中的部分数据内存分配工作，包括但不限于服务器中的客户端对象列表`redisServer.clients`、服务器端*Slave*客户端对象列表`redisServer.slaves`、以及数据库的键空间`redisServer.db`等等。
1. 通过`createSharedObjects`接口对服务器的共享对象数据进行分配内存与初始化，这些共享数据是*Redis*认为会频繁使用的对象数据，通过在启动时就分配好这些数据，可以避免在运行时的对象分配，提高服务器的运行效率。
1. 通过`aeCreateEventLoop`函数接口创建服务器的事件循环，并获取事件循环所使用的文件描述符。
1. 根据服务器配置的地址信息，通过`listenToPort`接口创建服务器连接监听套接字。
1. 通过`aeCreateTimeEvent`函数，将服务器心跳处理函数`serverCron`注册到事件循环的时间事件之中。
1. 为前面所创建的服务器监听套接字创建可读事件回调处理函数`acceptTcpHandle`，用于接收客户端的连接请求。
1. 如果配置了**AOF**持久化，那么调用`open`接口打开磁盘上的**AOF**文件。

`InitServerLast`接口中初始化的，是需要最后处理的内容，现在只有后台异步线程的初始化接口`bioInit`是通过`InitServerLast`调用的。

## Redis数据的加载
```c
void loadDataFromDisk(void);
```
*Redis*在完成服务器的初始化之后，会通过`loadDataFromDisk`来从磁盘上加载**AOF**或者**RDB**文件，以恢复键空间之中的数据。如果*Redis*同时开启了**AOF**以及**RDB**的数据持久化策略，则在恢复数据时，优先选择从**AOF**文件之中进行恢复。具体的**AOF**以及**RDB**文件数据恢复逻辑，可以参考前面关于数据持久化内容的介绍。

## 进出事件循环的回调
```c
void beforeSleep(struct aeEventLoop *eventLoop);
void afterSleep(struct aeEventLoop *eventLoop);
```
接下来通过`aeMain`启动事件循环之前，会向事件循环注册`beforeSleep`以及`afterSleep`两个回调接口，用于在进入事件循环阻塞前处理逻辑以及从事件循环中跳出时调用。

在`beforeSleep`函数中涉及了比较多的逻辑处理，包括：
1. 在被阻塞的客户端被解锁时，完成后续的处理工作`processUnblockClients`。
1. 刷新**AOF**文件，将内核缓冲区中的数据写入磁盘`flushAppendOnlyFile`。
1. 处理客户端对象应用层输出缓冲区中的数据，尽可能地将数据输出到网络上`handleClientsWithPendingWrites`。

完成两个回调函数的注册之后，*Redis*会调用`aeMain`函数，完成启动过程，进入事件循环开始处理客户端的请求。

## Redis服务器的心跳处理逻辑
```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData);
```
由于*Redis*中处理心跳的函数接口`serverCron`设计了大量核心逻辑的处理，因此这里单独拿出来对其细节进行一个介绍：
1. `serverCron`接口会负责服务器运行时数据的刷新工作，包括调用`updateCachedTime`接口来缓存当前的系统时间戳，这样一来当其他模块需要读取系统时间时，便不需要通过`time`系统调用来获取；另外系统的**LRU**时间戳的刷新也是在`ServerCron`函数之中进行的。
1. 调用`clientsCron`来处理客户端心跳，其中的逻辑包含：
    1. 通过`ClientsCronHandleTimeout`来处理长期没有数据交互的客户端对象；以及处于阻塞状态时间过长没有就绪的客户端对象。
    1. 通过`ClientsCronResizeQueryBuffer`来清理客户端的应用层读缓存用于释放空闲的内存空间。
    1. 通过`ClientsCronTrackExpansiveClients`来处理追踪那些占用内存过高的客户端对象。
1. 调用`databasesCron`来处理键空间心跳：
    1. 通过`activeExpireCycle`来处理键空间之中过期的键。
    1. 通过`tryResizeHashTables`尝试对键空间的大哈希表进行扩充。
    1. 通过`incrementallyRehash`对扩容的键空间的大哈希表进行增量的重哈希数据迁移。
1. 检测生成**RDB**文件以及重写**AOF**文件的后台子进程的运行情况，在子进程结束时，通过`backgroundSaveDoneHandler`以及`backgroundRewriteDoneHandle`回调函数来对后续的工作进行处理。
1. 在没有后台子进程生成**RDB**文件或者重写**AOF**文件的情况下，检测是否满足生成**RDB**文件以及重写**AOF**文件的条件；如果条件满足，则通过对应的函数接口来启动后台子进程处理相关逻辑。
1. 调用`replicationCron`来处理主从复制功能的心跳。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*