# Redis同步IO操作

在大多数的情况下*Redis*都是以非阻塞的方式进行IO操作的，但是也存在例外情况
例如当执行**SYNC**指令时，*slave*节点就是以阻塞的方式进行发送的。而**MIGRATE**指令，也必须是以阻塞的方式来进行，只有这样，从两个实例的角度来看，这个操作才是原子的。因此这也就是我们为什么需要同步IO操作的原因。在*Redis*中，所有这些同步IO操作都是以毫秒级时间来计算超时返回的。

在*Redis*之中，给出了三个同步IO操作的底层函数接口：

```c
ssize_t syncWrite(int fd, char *ptr, ssize_t size, long long timeout);
ssize_t syncRead(int fd, char *ptr, ssize_t size, long long timeout);
ssize_t syncReadLine(int fd, char *ptr, ssize_t size, long long timeout);
```

者三个函数分别用于同步地向文件中数据，读取给定长度的数据以及读取整行的数据，所有者三个函数都接受一个毫秒级别的时间戳，如果超过这个超时时间，依然没有完成执行操作，函数将会返回-1，标记操作超时。

前面对于客户端读写操作的相关内容之中，我们可以了解到，每个客户端对象`client`自身都维护了自己的应用层读写缓存：

1. 对于读操作，客户端对象在每次触发可读事件时，会通过`readQueryFromClient`函数，将内核缓冲区中的数据一次性全部读取到应用层缓冲区。如果只读取了部分数据，那么该客户端会重新进入事件循环，等待连接的对方继续发来数据，触发可读事件，继续完成数据的读取。
2. 对于写操作，*Redis*会在每次进入事件循环之前，通过调用`handleClientsWithPendingWrites`函数，尝试将应用层缓冲区之中的数据尽可能地写入内核缓冲区；如果应用层缓冲区的数据过大，导致无法一次性写入，那么*Redis*会这个这个客户端在事件循环之中注册一个可写事件，等待内核缓冲区之中有足够空间时，触发写事件，继续将应用层缓存之中的数据写入内核，如此循环，直到所有的数据都完成写入。

同时我们直到，*Redis*通常会调用`anetNonBlock`这个函数将连接设置为非阻塞的状态，那么如何在这样的前提下实现阻塞的同步IO操作呢？

可以回顾一下，我们在前面介绍*Redis*事件循环时，曾经简单描述过的一个函数接口`aeWait`，这个接口的作用是，将整个进程阻塞在某一个文件描述符上，等待可读或者可写事件触发，或者等待超时，下面我们看看这个函数的代码实现片段：

```c
int aeWait(int fd, int mask, long long milliseconds)
{
    struct pollfd pfd;
    ...
    pfd.fd = fd;
    
    ...
    if ((retval = poll(&pfd, 1, milliseconds)) == 1)
    {
        ...
    }
    ...
}
```

之前在看这段代码的时候，便有一些疑问，既然*Redis*已经使用`aeMain`这个函数接口作为系统整个事件循环的入口，为什么还要另外使用`poll`系统调用来实现一个单独的`aeWait`函数呢？

那么这个疑问，现在在这里得到了解决，这个`aeWait`接口便是为了实现系统的同步IO操作而单独设计的，以上面的同步写操作的函数接口为例：

```c
ssize_t syncWrite(int fd, char *ptr, ssize_t size, long long timeout) {
	while(1) {
        ...
        nwritten = write(fd,ptr,size);
        if (nwritten == -1)
        {
            if (errno != EAGAIN) return -1;
        }
        else
        {
            ptr += nwritten;
            size -= nwritten;
        }
        if (size == 0) return ret;
        aeWait(fd, AE_WRITEABLE, wait);
        elapsed = mstime() - start;
        if (elapsed >= timeout)
        {
            errno = ETIMEDOUT;
            return -1;
        }
    }   
}
```

也就是说，`syncWrite`这个函数会尝试将数据通过`write`系统调用写入到内核缓冲区之中，直到写完或者内核缓冲区已满为止，接下来*Redis*会通过`aeWait`这个接口将这个进程阻塞，直到内核缓冲区之中的数据被发送到网络上，再次触发可写事件为止或者等待超时，这样通过一次或者多次的循环，将数据同步地输出出去。

而`syncRead`以及`syncReadLine`这两接口与`syncWrite`接口的实现方式类似，也是通过`aeWait`这个接口，结合多次循环，将整个进程阻塞起来同步地进行数据的读取操作。




***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*