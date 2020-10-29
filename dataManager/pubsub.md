# Redis发布与订阅
在*2.0.0*版本之后，*Redis*提供了一种消息订阅与发布的机制，客户端可以用过**SUBSCRIBE**命令，订阅一个或者多个给定的*频道*；或者使用**PSUBSCRIBE**命令，通过模式匹配的方式订阅一组特定模式的*频道*。同时客户端可以通过**PUBLISH**命令，向特定的*频道*上发布消息，订阅了这个*频道*或者通过模式匹配的方式订阅这个*频道*的客户端，将会收到发布的消息。

## 发布与订阅功能的使用
### 订阅
*Redis*对于订阅*频道*，给出了两个命令，其对应的格式为：

    SUBSCRIBE channel [channel ...]
    PSUBSCRIBE pattern [pattern ...]

这两个命令分别用于定于订阅特定的频道以及按照模式匹配的方式订阅频道，例如：

    SUBSCRIBE channel:1 channel:2 chat:public
    PSUBSCRIBE channel:* chat:*

上述两条命令中：
1. 第一条命令表示客户端订阅`channel:1`以及`channel:2`和`chat:public`这三个频道。
1. 第二条命令则表示客户端订阅了匹配`channel:*`模式的频道以及匹配`chat:*`模式的频道。例如`channel:1`以及`channel:2`都是匹配`channel:*`模式的频道；而`chat:public`则是匹配`chat:*`模式的频道。

这里需要注意的是，*频道*不需要提前定义，如果客户端订阅了一个新的*频道*，那么服务器会自动为其创建一个频道出来。

### 退订
同样的，当我们不需要监听某些频道上的消息时，*Redis*为我们提供了退订频道的命令：

    UNSUBSCRIBE [channel [channel ...]]
    UNPSUBSCRIBE [pattern [pattern ...]]

使用上述两个命令，客户端可以对其订阅的频道进行退订，这两个退订命令既可以退订给定的频道，也可以退订特定匹配特定模式的频道；当没有给出频道名称或者模式名字，则会退订所有的频道以及模式。这里需要注意一个问题，频道名字与模式名字是相互独立的，也就是说如果我们使用**SUBSCRIBE**订阅了`channel:1`频道，同时使用**PSUBSCRIBE**命令订阅了匹配`channel:*`模式的频道；那么当使用**UNPSUBSCRIBE**命令对`channel:*`模式退订时，不会影响对频道`channel:1`的订阅；同样退订`channel:1`频道，也不会影响对`channel:*`模式的订阅。

### 发布
最后，我们来看一下*Redis*用于向频道发布消息的命令：

    PUBLISH channel message

这个命令会向`channel`频道发布一条内容为`message`的消息。所有通过**SUBSCRIBE**订阅了`channel`频道的客户端，以及所有通过**PSUBSCRIBE**命令订阅了通过模式可以匹配到`channel`频道的客户端，都会收到`message`这条消息。

如同上面在退订中所讲的，频道订阅与模式订阅是相互独立的。在向频道发布消息的时候，这一点也是成立的，如果一个客户端既通过**SUBSCRIBE**订阅了`channel`频道，又通过**PSUBSCRIBE**命令订阅类似`chan*`这样的模式，那么这个客户端会收到两条`message`的消息。

## 发布与订阅的代码实现
首先我们先了解一下，*Redis*为发布与订阅功能设计的数据结构。对于频道，*Redis*是使用一个字符串类型的对象数据`robj`进行表示的；而对于模式，*Redis*定义了一个专门的结构体用于表示：
```c
typedef struct pubsubPattern {
    client *client;        //这个模式对应的客户端
    robj   *pattern;       //使用一个字符串对象来表示模式
} pubsubPattern;
```
这里每一个`pubsubPattern`结构体便代表这一个*client-pattern*数据。

而整个发布与订阅机制，是依靠*Redis*服务器数据结构体以及客户端数据结构体中的下面几个字段维护的：
```c
struct client {
    ...
    dict *pubsub_channels;
    list *pubsub_patterns;
    ...
};

struct redisServer {
    ...
    dict *pubsub_channels;
    list *pubsub_patterns;
    ...
};
```
在上面这两个结构体中的四个字段，其含义为：
1. `client.pubsub_channels`，这个字段是一个哈希表，不过这个哈希边之中只使用*key*存储了*频道*数据，客户端通过**SUBSCRIBE**命令订阅的频道都存储在这个哈希表之中。
1. `client.pubsub_patterns`，这个字段是一个由模式字符串对象数据构成双端链表，客户端使用**PSUBSCRIBE**订阅的模式数据都存储在这个双端链表之中。
1. `redisServer.pubsub_channels`，这个字段是一个哈希表，其中存储了频道与由客户端组成的链表构成的映射关系，其中每一个*key*对应于一个频道数据，其对应的*value*则是订阅了这个频道的客户端构成的双端链表。
1. `redisServer.pubsub_patterns`，这个字段是一个由`pubsubPattern`构成的双端链表，表示服务器记录的所有的客户端订阅的模式数据。

*Redis*使用下面这个接口实现订阅一个给定频道的操作：
```c
int pubsubSubscribeChannel(client *c, robj *channel);
```
这个函数会将`channel`频道字符串插入`client.pubsub_channels`哈希表中，同时将客户端结构体指针`c`插入到`redisServer.pubsub_channels`中`channels`对应的双端链表之中，如果服务器中不存在对应的频道，那么会创建一对频道信息，将其插入到`redisServer.pubsub_channels`哈希表中。

而定于一个模式，*Redis*则使用下面这个函数来实现：
```c
int pubsubSubscribePattern(client *c, robj *pattern);
```
这个函数会将`pattern`模式插入`client.pubsub_patterns`这个双端链表之中，同时使用客户端结构体指针以及`pattern`构造一个`pubsubPattern`数据，并将其插入到服务器`redisServer.pubsub_patterns`这个双端链表之中。

我们可以看到，无论是在客户端数据中，还是在服务器数据之中，订阅的频道信息以及模式信息都是分别存储在两个独立的数据结构之中的，这也就解释前面所说的订阅频道与订阅模式是相互独立的这一点。

另外我们以订阅频道命令**SUBSCRIBE**为例，看一下其对应的命令实现函数：
```c
void subscribeCommand(client *c)
{
    int j;
    for (j = 1; j < c->argc, j++)
        pubsubSubScribeChannel(c, c->argv[j]);
    c->flags |= CLIENT_PUBSUB;
}
```
这里我们可以看到，当执行一个订阅命令时，会为客户端设置一个`CLIENT_PUBSUB`标记，表明该客户端处于订阅的状态。处于状态的客户端只能够执行**PING**，**SUBSCRIBE**，**UNSUBSCRIBE**，**PSUBSCRIBE**，**PUNSUBSCRIBE**这几个特定的命令，其他命令则被拒绝执行：
```c
int processCommand(client *c)
{
    ...
    if (c->flags & CLIENT_PUBSUB &&
        c->cmd->proc != pingCommand &&
        c->cmd->proc != subscribeCommand &&
        c->cmd->proc != unsubscribeCommand &&
        c->cmd->proc != psubscribeCommand &&
        c->cmd->proc != punsubscribeCommand) {
        addReplyError(c, "only (P)SUBSCRIBE / (P)UNSUBSCRIBE / PING / QUIT allowed in this context");
        return C_OK;
    }
}
```

而取消订阅，*Redis*是使用下面的四个函数来实现单独取消以及全部取消的功能：
```c
int pubsubUnsubscribeChannel(client *c, robj *channel, int notify);
int pubsubUnsubscribePattern(client *c, robj *pattern, int notify);
int pubsubUnsubscribeAllChannels(client *c, int notify);
int pubsubUnsubscribeAllPatterns(client *c, int notify);
```
其中`pubsubUnsubscribeAllChannels`以及`pubsubUnsubscribeAllPatterns`是取消全部订阅频道以及订阅模式的接口，除了会在**UNSUBSCRIBE**命令以及**UNPSUBSCRIBE**命令之中被调用之外，还会在客户端结构被释放的时候被调用，用以清理数据：
```c
void freeClient(client *c) {
    ...
    pubsubUnsubscribeAllChannels(c,0);
    pubsubUnsubscribeAllPatterns(c,0);
    dictRelease(c->pubsub_channels);
    listRelease(c->pubsub_patterns);
    ...
}
```

在执行取消订阅命令时，会将客户端上的`CLIENT_PUBSUB`标记移除，以便客户端能够执行后续的其他命令。

最后，我们来看一下*Redis*用于在频道上发布消息的接口：
```c
int pubsubPublishMessage(robj *channel, robj *message);
```
这个函数的主要逻辑分为两个部分：
1. 从`redisServer.pubsub_channels`这个哈希表中查找频道`channel`对应的客户端列表，将消息`message`发送给这个客户端列表中的所有客户端。
1. 遍历`redisServer.pubsub_patterns`这个双端链表，对于其中的每一个`pubsubPattern.pattern`模式调用`stringmatchlen`与`channel`进行匹配，如果匹配成功，则将`message`消息发送给`pubsubPattern.client`对应的客户端。

通过上面对于*Redis*订阅与发布功能的介绍，我们发现应用这个功能可以实现一个简单的消息队列机制，但是这里也暴露出订阅与发布功能的一些问题：
1. 没有消息的缓存或者持久化功能，消息的发送的是一个瞬间的事件，如果此时客户端因为网络问题掉线，那么它在重新连接到服务器之后，它将丢失一部分消息。
1. 在客户端保持连接，却不从订阅的频道之中将受到的消息取出时，这将造成数据的累积到*TCP*的缓冲区内，同时在服务器端会导致*TCP*写缓存被填满，这可能会引起系统的一些问题。

当然，如果我们能够及时的从订阅频道中提取数据，同时对于消息丢失有一定的容忍能力，那么*Redis*的订阅发布机制，无疑是一种实现消息发布的简单快捷的工具。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*