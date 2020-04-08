# Redis在整个内核中对于套接字与文件的同步IO操作

在大多数的情况下*Redis*都是以非阻塞的方式进行IO操作的，但是也存在例外情况
例如当执行*SYNC*指令时，*slave*节点就是以阻塞的方式进行的。
而*MIGRATE*指令，也必须是以阻塞的方式来进行，只有这样，从两个实例的角度来看，
这个操作才是原子的。因此这也就是我们为什么需要同步IO操作的原因。
在*Redis*中，所有这些同步IO操作都是以毫秒级时间来计算超时返回的。