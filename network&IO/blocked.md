# Redis对客户端阻塞操作的支持
在前面的关于*Redis*列表对象类型以及有序集合对象类型的文章中，我们了解到*Redis*针对这些数据对象类型提供了一组阻塞命令。以常用的**BLPOP**命令以及**BRPOP**命令为例，执行这些命令可以从指定的列表数据对象之中弹出数据，如果指令的列表对象为空，那么命令的调用者将会进入阻塞状态，直到这个指定的列表对象中有数据为止。

不过这里需要注意的是，此处所说的阻塞，并不是系统编程中所指的阻塞，也就是说阻塞状态并不会阻塞整个服务器，其仅仅是将执行阻塞命令的客户端对象`client`设置成阻塞状态，而服务器依然可以继续为其他客户端的请求提供服务。

本文将详细讲解*Redis*对于阻塞命令的相关支持。

*Redis*服务器全局变量中关于阻塞操作的相关字段：
```c
struct redisServer{
    ...
    unsigned int blocked_clients;
    unsigned int blocked_clients_by_type[BLOCK_NUM];
    list *unblocked_Clients;
    list *ready_keys;
    ...
};
```

数据库中与阻塞操作相关的字段：
```c
struct redisDb {
    ...
    dict *blocking_keys;
    dict *ready_keys;
};
```

而在客户端对象结构体`client`中与阻塞操作相关的字段为：
```c
struct client {
    ...
    int btype;
    blockingState bpop;
};
```

这里`blockingState`这个结构体的定义为：
```c
struct blockingState {
    mstime_t timeout;
    dict *keys;
    robj *target;
    ...
};
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*