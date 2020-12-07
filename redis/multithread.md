# 新Redis中多线程模式

本系列文章主要是根据*Redis*的*5.0.8*版本的源代码进行讲解的，但是在本系列文章的撰写过程之中，*Redis*的作者发布了一个最新的*6.0*的版本。在这个版本之中*Redis*发布了一个重大的新特性，这个新特性便是*Redis*的多线程模式，我觉得有必要在刚刚介绍完*Redis*服务器基础流程、客户端对象以及事件循环的相关内容之后，趁热打铁介绍一下新版*Redis*中的这个多线程模式。

首先我们简单回顾一下老版本之中*Redis*处理客户端请求的流程：
1. *Redis*在启动时，在`initServer`初始化服务器的接口中，为监听套接字的文件描述符在事件循环之中注册可读事件的回调处理函数`acceptTcpHandler`，用以接收客户端建立连接的请求。
1. 当有新的客户端连接到到来时，会触发`acceptTcpHandler`回调函数，这个函数会调用`accept`系统调用获得新连接对应的文件描述符，使用这个文件描述符创建一个客户端对象`client`，并为这个客户端的文件描述符在事件循环之中注册可读事件的回调函数`readQueryFromClient`。
1. 当一个客户端向*Redis*服务器发来命令请求时，会触发`readQueryFromClient`，这个函数从内核中将网络数据读取到客户端对象`client`的应用层缓存，并解析命令参数数据，执行对应的命令。
1. 命令的输出结果会先写入客户端对象`client`的应用层输出缓冲之中，但是先不把数据真正的写入网络。
1. 重复上面步骤2，步骤3，步骤4的过程直到这次事件循环之中所有的就绪事件都被处理完毕。
1. 在重新进入事件循环之前，在`beforeSleep`接口之中，遍历有数据待输出的客户端，将应用层缓冲区中的数据尽量通过`write`接口写入网络；如果内核缓冲区已满，那么为这个客户端注册可写事件的回调处理函数`sendReplyToClient`，等待内核缓冲区中有空闲空间时，触发可写事件通过`sendReplyToClient`接口将应用层缓冲区继续输出到网络上。

这样我们可以总结出，老版本*Redis*的一个处理流程：主线程负责了从网络数据读取、命令数据解析、命令执行，命令返回数据的输出的全过程。同时我们了解，数据的IO操作的速度要明显慢于内存操作，这也就意味着*Redis*主线程实际上将相当多的时间用于IO操作，而不是将时间用于核心的内存数据的操作上。为了解决这一问题，进一步提升*Redis*的并发性能，新版*Redis*提出了**IO线程**的解决方案，在这种方案之中，客户端命令数据的读取与解析交给**IO线程**并发地进行，主线程只负责核心内存数据库的操作，而最终命令的返回结果也交由**IO线程**进行。这种特殊的多线程并发方式，既可以在慢速的IO操作上应用多线程的方式提高数据IO的效率，而在核心数据的操作上依然使用单线程模式可以避免因为加锁导致的系统性能下降，是一种结合了多线程模式与单线程模式有点的解决方案。接下来我们会简要介绍一下，新版中*Redis*的多线程模式的实现细节。

首先我们来简单描述一下新版*Redis*之中多线程模式的工作流程：
1. 主线程阻塞在事件循环之中，等待就绪事件的发生；当事件就绪时，主线程会收集就绪的客户端对象，将其放入一个等待队列之中，而不是去读取客户端网络连接上的数据。
1. 当所有就绪客户端已经被收集到之后，会将这些客户端分配包括主线程在内的后台**IO线程**来并发地读取并解析客户端网络连接上的命令数据；在所有后台线程完成数据读取之前，*Redis*会阻塞地进行等待。
1. 当所有后台线程完成数据读取解析之后，主线程依次处理各个客户端执行对应的客户端命令，并将命令的输入写入客户端的应用层缓冲区；同时将客户端放入另外一个等待队列之中，等待数据的输出。
1. 当主线程处理完所有客户端的命令执行之后，将等待输出的队列之中的数据，分配给包括主线程在内的后台**IO线程**并发地将数据输出到网络上；在所有后台**IO线程**完成输出之前，*Redis*会阻塞地等待。
1. **IO线程**完成输出之后，主线程会遍历前面的等待数据客户端，为还有数据没有传输的客户端注册可写事件的处理函数；等待后续网络可写时继续数据。
1. 完成上述步骤之后，主线程重新进入事件循环阻塞，继续等待新的就绪事件的发生。

这里我们可以发现，**IO线程**要么在等待就绪的客户端，要么都在读取数据，要么都在输出数据，线程的状态永远都是一致的，不存在一部分线程在处理输入，另外一部线程在处理输出的情况；同时每个后台线程能处理的客户端对象在其开始处理逻辑之前，就已经被分配好了，也不存在多个线程访问一个客户端的情况。这样可以极大地降低系统的复杂性，同时可以大大避免使用锁的场景，在简化实现的情况下，提升系统性能。

为了实现多线程，新版*Redis*新增了若干数据字段与数据定义，用于支持多线程系统的实现。

对于客户端状态`client.flags`，老版本*Redis*之中有一个`CLIENT_PENDING_WRITE`的状态，表示客户端应用层缓冲区中存在待输出的数据，正在等待主线程处理完所有就绪的客户端命令时，再进行数据的输出。新版的*Redis*也复用了这个客户端状态，不过稍稍更改了含义，由*等待主线程在处理完客户端请求后由主线程进行数据输出*更改为*等待主线程处理完客户端请求后，由IO线程进行并发的数据输出*。在这个状态的基础上，新版*Redis*还定义了两个新的客户端状态：
1. `CLIENT_PENDING_READ`，这个状态表示对应的客户端上触发了可读事件，已被主线程加入到等待队列之中挂起，正在等待**IO线程**并发地处理读取网络连接上传来的数据。
1. `CLIENT_PENDING_COMMAND`，这个状态表示对应的客户端已经被**IO线程**完了数据的接收与命令的解析，正在等待所有**IO线程**处理结束后，由主线程处理客户端命令的执行。

在服务器全局变量结构之中，老版本*Redis*有一个`redisServer.clients_pending_write`的双端链表字段，用于存储被挂起等待输出的客户端对象；现在在新版本的*Redis*之中，还定义了一个`redisServer.clients_pending_read`的双端链表，用于存储被挂起等待**IO线程**执行数据读取的客户端对象。

在新版*Redis*的*src/networking.c*源文件之中，定义了若干内部变量，用于处理多线程的逻辑：
1. `pthread_t io_threads[IO_THREADS_MAX_NUM]`，用于记录**IO线程**的线程ID，其中0号索引为主线。
1. `list *io_threads_list[IO_THREADS]`，存储每个**IO线程**对应处理的客户端队列。
1. `_Atomic unsigned long io_threads_pending[IO_THREADS_MAX_NUM]`，每个**IO线程**的处理队列中客户端对象的数量。
1. `int io_threads_op`，由于所有的**IO线程**同一时间，只能执行同一类工作，这个变量用于存储当前后台线程执行工作的类型：
    1. `IO_THREADS_OP_WRITE`，多线程输出。
    1. `IO_THREADS_OP_READ`，多线程读取。

新版*Redis*多线程数据读写的基本逻辑是类似的，不过多线程读取这一部分的内容相对更加复杂一些，因此本文将主要讲解**IO线程**多线程数据读取这一部分的内容。

新版*Redis*定义了一个新函数，用于推迟客户端对象数据的读取：
```c
int postponeClientRead(client *c)
{
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !ProcessingEventsWhileBlocked **
        !(c->flags & (CLIENT_MASTER | CLIENT_SLAVE | CLIENT_PENDING_READ)))
    {
        c->flags |= CLIENT_PENDING_READ;
        listAddNodeHead(server.clients_pending_read, c);
        return 1;
    }
    else
    {
        return 0;
    }
}

void readQueryFromClient(connection *conn)
{
    client *c = connGetPrivateData(conn);
    
    if (postponeClientRead(c)) return;
    ......
}
```
上面这个函数会在客户端处理可读事件的回调函数`readQueryFromClient`逻辑的一开始被调用，也就是说主线程遍历处理从事件循环之中返回的就绪客户端时，如果*Redis*开启了多线程读取，并且这个对应的客户端没有`CLIENT_PENDING_READ`的状态，会将这个客户端加入到等待队列`redisServer.clients_pending_read`之中，并添加`CLIENT_PENDING_READ`状态。这种情况下主线程并不会调用`read`系统调用为这个客户端从网络连接上读取数据。

当主线程处理完事件循环之中所有的就绪客户端之后，应该重新进入事件循环的阻塞状态之中，重新等待就绪客户端的返回。在重新进入阻塞状态前，会默认调用`beforeSleep`这个回调函数。新版*Redis*在`beforeSleep`这个回调函数之中，新增了一段`handleClientsWithPendingReadsUsingThreads`的逻辑，用于多线程处理客户端数据的读取。
```c
int handleClientsWithPEndingReadsUsingThreads(void)
{
    //将等待队列之中的客户端分配给各个IO线程
    listRewind(server.clients_pending_read.&li);
    while((ln = listNext(&li)))
    {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id], c);
        item_id++;
    }

    //设置线程操作类型，并设置各个线程队列中挂起的客户端的数量
    //完成这段代码之后，IO线程将会开始处理客户端的数据读取逻辑
    io_threads_op = IO_TREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++)
    {
        int count = listLength(io_threads_list[j]);
        io_threads_pending[j] = count;
    }

    //主线程开始处理分配给自己的那部分客户端
    listRewind(io_threads_list[0], &li);
    while((ln = listNext(&li)))
    {    
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }
    listEmpty(io_threads_list[0]);

    //主线程自旋地阻塞并等待其他线程完成数据处理
    while(1)
    {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }

    //主线程处理客户端命令的执行
    while(listLength(server.clients_pending_read))
    {
        ln = listFirst(server.clients_pending_read);
        client *c - listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_READ;
        listDelNode(server.clients_pending_read, ln);
        
        if (c->flags & CLIENT_PENDING_COMMAND)
        {
            c->flags &= ~CLIENT_PENDING_COMMAND;
            processCommadnAndResetClient(c);
        }
        procrssInputBuffer(c);
    }
}
```
通过上述的代码，我们可以了解到主线程是如何处理多线程数据读取操作的，接下来我们来看一下**IO线程**的入口函数：
```c
void *IoThreadMain(void *myid)
{
    while(1)
    {
        //自旋地等待等待队列之中有数据可用
        for (int j = 0; j < 1000000; j++)
        {
            if (io_theads_pending[id] != 0) break;
        }

        if (io_threads_pending[id] == 0)
        {
            continue;
        }
        
        listRewind(io_threads_list[id], &li);
        while ((ln = listNext(&li)))
        {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_TREADS_OP_WRITE) {
                writeToClient(c, 0);
            }
            else if (io_threads_op == IO_THREADS_OP_READ)
            {
                readQueryFromClient(c->conn);
            }
        }
        listEmpty(io_threads_list[id]);
        io_treads_pending[id] = 0;
    }
}
```
思考一下这里一个比较有意思的细节，主线程向各个**IO线程**分配客端户对象时，看起来应该使用条件变量来辅助实现。然后在*Redis*的代码之中并没有这么做，其关键点就在于`io_threads_pending`这个原子数组。**IO线程**中是通过`io_threads_pending`中的数据来检测客户端对象是否已经分配完毕的，并不是通过`io_threads_list`的长度来判断的；而主线程是先向`io_threads_list`之中分配客户端对象，当分配结束之后，才会设定`io_threads_pending`数据中数据的长度，同时由于这个数组是原子的，这就保证了在主线程完成分配工作之前，**IO线程**不会启动后续逻辑的处理。

最后，多线程的输出操作，则是通过在`beforeSleep`函数之中调用`handleClientsWithPendingWritesUsingThreads`来实现的，因为与读取操作类似，这里就不在详细介绍了：
```c
int handleClientsWithPendingWritesUsingThreads(void);
```

以上便是新版*Redis*之中，关于多线程处理机制的介绍。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*