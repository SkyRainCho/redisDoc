# Redis对客户端阻塞操作的支持

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