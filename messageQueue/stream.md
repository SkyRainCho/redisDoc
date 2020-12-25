# Redis可持久化的消息队列

## 消息队列的概述

简单来说消息队列可以被看做是一个存放小的容器，作为消息的生产者可以将消息存放到消息队列之中；而作为消息的消费者，当需要消息时可以从消息队列之中取出消息供自己使用。

### 为什么要使用消息队列

消息队列是分布式系统之中的一个重要的组件。应用消息队列，我们可以解决在分布式系统之中的三个比较重要的问题：

1. **异步**，分布式系统之中的处理模块只需要处理完自己负责的逻辑之后，就可以将请求扔进消息队列，由后续模块处理后续的逻辑，自己则可以继续处理其他的请求；而不用像同步模式之中那样，必须要一个请求彻底处理结束才能处理新的请求。
2. **削峰**，当分布式系统之中突然涌入大量的请求时，消息队列可以水库一样将这些请求缓存起来，这样后端的逻辑处理模块可以按照合理的速度对这些请求逐步进行处理。而不是让这些突发的请求直接冲击后端的逻辑处理模块，导致系统瘫痪。
3. **解耦**，将生成者逻辑模块与消费者逻辑模块分离，生产者只负责将数据传入消息队列，而不用关心这些数据将会被如何处理；消费者只负责将消息从消息队列之中取出并处理，而不用关系这些消息是如何产生的。

有利便必有一弊，引用消息队列也会为整个分布式系统带来一些问题：

1. 首先便是增大了系统的复杂性。这是由于引入消息队列需要将原来基于同步方式的业务逻辑改为异步的方式来实现，稍微做过一些后端系统开发的读者都会深有体会，异步模型代码的复杂度要远大于同步模型的代码复杂度。
2. 其次便是降低了系统的可用性。这一点比较容易理解，在分布式系统之中又引入了一个新的组件，势必会降低系统的可用性，一旦消息队列组件出现宕机，那么整个系统将无法对外提供服务。

因此，我们可以说消息队列并不是分布式系统之中一个必要的组件，只有在合适的场景下使用，才会提升系统的性能，盲目地随意乱用的话，有可能适得其反。

### Redis中的可以作为消息队列使用的模块

*Redis*之中的很多模块被使用者拿来当作简易的消息队列来使用，这其中包括：

#### 发布/订阅模式。

在这个模式中，生产者作为发布方向指定的频道里发布消息，消费者作为订阅方从频道中接收消息。

但是在这种模式之中存在着一些问题与隐患：

1. 通过前面我们对于发布/订阅模式内容的介绍文章，我们知道这种模式中，生产者所发布的消息会立即广播给频道上的订阅者，消息不会保存在*Redis*服务器上，这就意味着一旦频道上的某个订阅者网络出现问题，那么它将丢失这条消息。
2. 生产者发布的消息是以广播的形式发送给频道上的所有订阅者，机制上不支持消费者以互斥的方式从消息队列中获取消息这一需求。

#### 列表对象数据类型。

在这种模式中，生产者通过**LPUSH**命令向以一个列表对象的左侧插入一条消息；消费者则通过**BRPOP**命令，以阻塞的方式从该列表对象的右侧获取一条消息。

虽然使用列表对象类型，可以避免发布/订阅模式之中存在的隐患。即列表对象本身便会存储在*Redis*的键空间之中，在一定程度上避免了消息丢失的情况，同时由于阻塞操作的机制，也可以支持多个消费者以互斥的方式从消息队列中获取消息这一需要。

但是这种消息队列的实现方式也存在一些问题。一旦通过**BRPOP**命令从列表之中获取出一条消息，那么这条消息将会被从列表队列之中彻底移除，这也就意味着消费者只能获取列表中当前的消息，而无法获取历史消息。同时由于缺乏确认机制，一旦消息在从列表之中移除后没有被消费者接收，或者消费者接收到但是没有成功处理，这将造成消息的丢失。

### Redis中Stream可持久化的消息队列

为了解决前面简易*Redis*消息队列存在的隐患，在*5.0*版本之中作者引入了**Stream**对象类型，作为经典的消息队列的实现方式。通过**Stream**这个对象类型，*Redis*可以实现对消息队列的持久化；可以实现消费者都可以访问任意时刻的消息，同时还可以记录消费者的获取消息的偏移；并为提供了对消息的**ACK**确认机制，可以保证即使消费者掉线也不会出现消息丢失的情况。

同时*Redis*会为消息队列之中的每一条消息维护一个唯一的、自增的消息ID。通过消息ID，我们可以快速索引一条指定的消息，或者从消息队列之中删除指定ID的消息。

最后在*Redis*服务器的角度来看，**Stream**对象上的操作与操作其他类似字符串、列表对象无异。因此对于用作消息队列的*Redis*，同样可以应用主从复制等多机机制，以提高消息队列组件的可用性与并发性能。

#### 消息的生产

在*Redis*之中，每一个**Stream**都可以关联一个*Key*存储在内存数据的键空间之中，每一个**Stream**便是一个消息队列，消息队列可以被认为是一种先进先出的数据结构，属于这个消息队列的消息都会存储这个**Stream**对象之中，同时每个消息由一个唯一的消息ID以及消息内容构成。通过**XADD**命令，我们可以向一个消息队列之中压入一条消息，**XADD**命令的格式为：

```
XADD key [MAXLEN [~|=] <count>] ID field string [field string ...]
```

这条命令向`key`所对应的**Stream**对象之中加入一条消息；`ID`字段用于指定消息的ID，如果传入`*`则表示会由*Redis*自动生成一个消息ID，这个系统自动生成的消息ID由两部分组成并且都是64位整数，第一部分是一个毫秒级的时间戳，第二部分则是一个自增的序列号，每一毫秒会清零一次；而消息的内容则是由`field`以及`string`参数构成，这相当于是一个键值对，一条消息至少要由一组键值对，由可以拥有多个键值对。命令执行成功后，会将消息的ID返回给命令的调用者。这里*Redis*强烈简易，使用系统自动生成的消息ID，而不要用自己定义的消息ID。

同时我们可以通过`MAXLEN`这个可选参数来设置这个消息队列的最大长度。如果我们使用`MAXLEN = <count>`的形式来限制消息列队的最大长度，则这个长度限制是准确的，也就是以`count`参数为上限；然后当我们采用`MAXLEN ~ <count>`的形式来限制消息对了的长度时，则是才有近似的方式进行限制，也就是该消息队列的真实长度可能会比`count`稍微大一些，但是绝对不会低于`count`这个数量限制。近似长度限制的运行效率要大于准确长度限制的运行效率。

接下来我们可以通过**XLEN**命令来获取指定消息队列**Stream**对象内消息的数量，这个命令的格式为：

```c
XLEN key
```

#### 消息的获取

在通过**XADD**命令完成向消息队列中插入消息之后，*Redis*还给出了**XREAD**命令，用于从消息队列之中读取消息，命令的格式为：

```
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]
```

这个命令可以以两种方式从消息队列之中读取消息，一种是按照给定消息的数量进行读取，另外一种则是以阻塞的方式进行读写。

在按照数量进行读取的方式之中，**XREAD**命令将会按照如下的格式对消息进行读取：

```
XREAD COUNT count STREAMS key ID
```

这个命令表示期望从`key`对应的消息队列**Stream**对象中按照先进先出的顺序读取消息ID大于参数`ID`的`count`条消息。如果满足条件的消息数量不足`count`条，那么就会读取全部实际满足条件的消息。

例如在消息生产者一侧：

```
127.0.0.1:6379> xadd mystream * field1 string1
"1608172773676-0"
127.0.0.1:6379> xadd mystream * field2 string2
"1608172779688-0"
127.0.0.1:6379> xadd mystream * field3 string3
"1608172785919-0"
127.0.0.1:6379> xadd mystream * field4 string4
"1608173001629-0"
127.0.0.1:6379> xlen mystream
(integer) 24
```

而在消息的消费者一侧：

```
127.0.0.1:6379> xread COUNT 2 STREAMS mystream 1608172773676
1) 1) "mystream"
   2) 1) 1) "1608172779688-0"
         2) 1) "field2"
            2) "string2"
      2) 1) "1608172785919-0"
         2) 1) "field3"
            2) "string3"
```

而如果将`ID`参数设置为0，便可以实现从消息队列的头部开始读取消息的需求。

在以阻塞方式读取消息队列之中的数据时，则可以使用如下的命令格式：

```
xread BLOCK milliseconds STREAMS key ID
```

这条命令表示调用者希望从`key`指定的消息队列之中读取一条ID大于`ID`的消息。如果这样的消息暂时不存在，那么调用者将会阻塞`milliseconds`毫秒，直到有满足条件的消息，或者阻塞超时。如果`milliseconds`为0，则表示调用者将一直阻塞下去直到有满足条件的消息到来。如果我们希望获取未来将会传入消息队列中的新消息，但又不知道最大的消息ID时，可以将`ID`参数设置为`$`代表当前消息队列中最大的ID。一个应用场景如下所示：

在消息的消费者一侧，以阻塞的方式等待`mystream`消息队列上最新的消息：

```
127.0.0.1:6379> xread BLOCK 0 STREAMS mystream $
```

后续生产者一侧，向这个消息队列之中加入一条新的消息：

```
127.0.0.1:6379> xadd mystream * field5 string5
"1608175440458-0"
```

此时在消费者一侧，命令将会返回，将这条新增的消息发送给消费者：

```
127.0.0.1:6379> xread BLOCK 0 STREAMS mystream $
1) 1) "mystream"
   2) 1) 1) "1608175440458-0"
         2) 1) "field5"
            2) "string5"
(17.46s)
```

除了**XREAD**命令之外，*Redis*还提供了**XRANGE**命令用于批量地获取消息队列之中的消息：

```
XRANGE key start end [COUNT count]
```

**XRANGE**命令用于从`key`这个消息队列之中获取以从小到大的顺序获取消息ID落在`start`以及`end`范围内的消息，如果通过`COUNT`参数设置了数量`count`，那么将获取满足条件的前`count`条消息。而特殊的参数符号`-`以及`+`则分别用来表示消息队列之中最小的ID以及最大的ID。

与之对应的*Redis*还提供了一个**XREVRANGE**的命令，这个命令与**XRANGE**命令类似，不过是按照消息ID从大到小的顺序来获取消息。

#### 消息的删除

*Redis*提供了**XDEL**用于将从给定的消息队列之中删除指定消息ID的消息，命令的格式为：

```
XDEL key ID [ID ...]
```

此外*Redis*还提供了一个对消息队列进行裁剪的命令，其格式为：

```
XTRIM key MAXLEN [~] count
```

**XTRIM**命令会将给定的消息队列裁剪到指定的`count`大小，最旧的一部分消息（也就是消息ID较小的那部分）将被从消息队列之中移除。当我们对裁剪掉的数量没有特别精确的需求是，可以添加`~`参数，这时被裁剪掉的消息数量可能会比`count`参数稍微大一些，但是绝对不会少于`count`的值，应用这个机制可以在牺牲一些准确性的情况下，更快地实现对消息队列的裁剪。

#### 消费者组

前面介绍的**XREAD**以及**XRANGE**这两个获取消息队列中消息的本质上还是对消息队列的一个读取与查找操作，可以认为是一种功能更加完备的阻塞列表对象，但者距离真正的消息队列还有一定的差距。上述两个命令缺少消息的确认机制，而且也不具备重连后恢复数据的功能。

为了实现更加完备的消息队列功能，*Redis*给了一个**消费者组**的概念。每一个**消费者组**中有若干个**消费者**，**消费者组**会从消息队列中获取消息，并将消息提供给组内的**消费者**取处理。即每个**消费者组**对应一个消息队列，每个**消费者组**中拥有多个**消费者**；而每个消息队列则可以关联多个**消费者组**，组与组之间是相互独立的。

应用**消费者组**可以保证：

1. 每条消息会被派发给组内不同的**消费者**，这样可以确保在一个**消费者组**内消息不会被重复处理。
2. 在同一个**消费者组**内部，**消费者**是通过大小写敏感的名字来是识别的，这就需要在创建**消费者**的时候需要为**消费者**指定名字。这意味这即使连接断开，**消费者组**依然可以维持所有的状态，这样客户端在重新连接后可以重新申请成为相同的**消费者**。
3. 每个**消费者组**都有一个对应消息ID的游标，这样以来，当**消费者**请求消息的时候，**消费者组**会根据这个游标为**消费者**派发消息。
4. 每消费一条消息，都需要通过指定的命令进行确认。
5. **消费者组**中会缓存所有已经派发但是还没有被确认的消息。这样可以确保消息的处理不会丢失。

##### 创建消费者组

*Redis*通过**XGROUP**命令可以创建一个**消费者组**，命令的格式为：

```
XGROUP CREATE key groupname id-or-$
```

这个命令用于在`key`这个消息列表上创建一个名字为`groupname`的**消费者组**，并通过`id`参数设置**消费者组**起始的消息ID游标，如果`id`设置`$`，则意味着这个**消费者组**将仅处理被创建之后才存入消息队列之中的消息。

如果我们希望修改一个**消费者组**上的消息派送游标的话，同样可以通过**XGROUP**命令来实现：

```
XGROUP SETID ID groupname id-or-$
```

##### 释放消费者组

当某一个**消费者组**不在被需要的时候，可以通过**XGROUP**命令来释放这个组，对应命令的格式为：

```
XGROUP DESTROY key groupname
```

这条命令会删除`key`这个消息队列上关联的名字为`groupname`的**消费者组**。

##### 创建消费者

完成了**消费者组**的创建之后，我们便可以通过**XREADGROUP**命令来创建**消费者**并从**消费者组**中获取命令。**消费者**获取消息有两种方式，一种是非阻塞的方式获取消息；另外一种是阻塞的获取方式。

在非阻塞的方式中，我们可以按照如下的命令创建一个**消费者**来获取消息：

```
XREADGGROUP GROUP group consumer [COUNT count] STREAMS key [ID]
```

这个命令会在`key`这个消息队列的`group`**消费者组**中以`consumer`消费者的名义请求一条消息，或者根据`COUNT`参数请求`count`条消息。同时我们可以通过`ID`参数来请求指定的消息，不过通过这种方式请求的消息必须时没有被确认的消息，如果请求一条已经被确认的消息，那么命令会返回一个空值。

此外，我们还可以通过阻塞的方式，来从消息队列之中获取一条消息：

```
XREADGROUP GROUP group consumer BLOCK milliseconds STREAMS key
```



##### 释放消费者

当我们需要释放某一个**消费者**的时候，我们可以通过**XGROUP**命令来进行操作：

```
XGROUP DELCONSUMER key groupname consumername
```

这条命令会释放`key`这个消息队列上，名为`groupname`的**消费者组**中名为`consumer`的**消费者**。

##### 未确认消息队列的查询

当**消费者**调用了**XREADGROUP**命令获取消息后，这条消息会被**消费者组**缓存在未确认消息队列之中。我们可以通过**XPENDING**这条命令来获取到一个**消费者组**中未确认消息队列中的具体数据。

```
XPENDING key group [start end count] [consumer]
```

这条命令会从返回`key`这个消息队列上的`group`**消费者组**中的未确认消息队列的内容。按照默认的参数，命令会返回未确认消息队列中消息的数量，消息ID的最大最小值，以及拥有未确认消息的**消费者**列表。

通过`start end count`我们可以进一步查看给定ID范围以及数量下的未确认消息的详细内容，例如消息ID，消息的所有者，已经从消息被派送后等待确认的毫秒时间等。

如果给出了`consumer`参数，那么我们可以查看指定**消费者**已经接收但是还未处理的消息的详细信息。

应用这一个功能，我们可以处理在断线重连的情况下，**消费者**能够获取到曾经派送给它但是没有来记得处理确认消息，这样在重连之后**消费者**可以重新处理这些消息，防止消息丢失。

##### 确认消息

在**消费者**一侧，当从**消费者组**中获取到的消息已经被正确处理时，**消费者**可以向**消费者组**发送一条确认信息，告知该消息已经被正确的处理没有丢失。这样**消费者组**便可以将对应的消息从缓存的未确认消息队列中移除。**消费者**一侧，可以通过**XACK**命令对消息进行确认。

```
XACK key group ID [ID ...]
```

这条命令的含义为，向`key`对应的消息队列上的`group`**消费者组**确认`ID`消息已被正确处理。

##### 消息转移

当一个**消费者**断线重连之后，可以通过**XPENGING**命令请求**消费者组**上的未确认消息列表，进而找回曾经被派发给自己的消息进行处理。但是如果这个**消费者**因为故障长时间下线的话，可能会导致一部分消息会在相当长的一段时间内无法被处理。对于这种情况，*Redis*给出了一个**XCLAIM**命令，应用这个命令，可以转移待处理消息的所有权，将消息转移给**消费者组**内其他可用的**消费者**进行处理。

```
XCLAIM key group consumer min-idle-time ID [ID ...] [IDLE ms] [TIME ms-unix-time] [RETRYCOUNT count] [force] [justid]
```

这条命令，会判断`key`消息队列上的`ID`消息，如果这条消息在`group`**消费者组**中已被派发，但是等待超过了`min-idle-time`的时间依然没有收到确认，那么就会将这个消息转移给另外一个`consumer`**消费者**进行处理。

****

## Redis中Stream的基础数据结构

### 消息ID

```c
typedef struct streamID {
    uint64_t ms;
    uint64_t seq;
} streamID;
```

*Redis*使用`streamID`这个数据结构来表示消息的ID：

1. `streamID.ms`，这是一个毫秒级别的时间戳，用于记录消息生成时的系统时间。
2. `streamID.seq`，相同时间戳内的消息序列号，如果同一毫秒内产生了多条消息，那么便会用这个序列号进行区分。每当这个毫秒时间戳被更新时，序列好会被清零重新计次。

### Stream

```c
typedef struct stream {
    rax *rax;
    uint64_t length;
    streamID last_id;
    rax *cgroups;
} stream;
```

`stream`这个数据结构便是*Redis*中消息队列的主体：

1. `stream.rax`，这是一个基数树，以消息ID为*Key*，用来存储消息队列之中的消息。
2. `stream.length`，存储消息队列中消息的数量。
3. `stream.last_id`，记录上一次生成的消息ID。
4. `stream.cgroups`，同样是一个基数树，用于存储关联这个消息队列的**消费者组**，使用**消费者组**的名字作为基数树的*Key*。

在`stream.rax`这个基数树中，以消息ID为*Key*，而对应的*Value*则是以前面我们介绍过的紧凑列表**listpack**。简单描述一下基数树使用紧凑列表来存储消息的布局：

1. 每条消息的消息内容都存储在一个紧凑列表之中。
2. 每个紧凑列表中可以保存多条消息，每个消息都是这个紧凑列表的一个`entry`。
3. 紧凑列表会使用表中第一条消息的ID，记作`master_id`，作为这个紧凑列表在整个`stream.rax`基数树中的*Key*。
4. 服务器在全局变量之中限制了单一紧凑列表的内存上限`redisServer.stream_node_max_bytes`以及单一紧凑列表最多可以容纳消息条数的上限`redisServer.stream_node_max_entries`。当超过整个上限，*Redis*会新分配一个紧凑列表存储在基数树之中。
5. 当我们需要删除一条消息时，通常不是将这条消息从紧凑列表之中删除，而是将其设置为一个删除状态。因为删除会涉及到内存的移动已经重新分配，这样会大大地降低系统的性能。

下面我们可以来看一下在紧凑列表中，消息是按照何种格式存储的。

```
+-----------+-------+-------+-----+-------+
|<MSGHeader>|<MSG-1>|<MSG-2>|.....|<MSG-N>|
+-----------+-------+-------+-----+-------+
```

在`MSGHeader`之中记录了当前这个紧凑列表之中的**消息表头**，后面的`MSG-X`为存储这个紧凑列表之中的消息内容。

首先我们来看一下紧凑列表之中这个消息表头的内存分布：

```
+-------+---------+------------+---------+--/--+---------+---------+-+
| count | deleted | num-fields | field_1 | field_2 | ... | field_N |0|
+-------+---------+------------+---------+--/--+---------+---------+-+
```

1. `count`之中记录这个紧凑列表之中存储的消息的数量。
2. `deleted`中记录了这个紧凑列表之中被标记为删除的消息的数量。
3. `fileds`这一组相关的数据是消息的`field`数据，使用的是第一个加入这个紧凑列表之中的消息的`field`数据。我们知道*Redis*消息队列之中的消息内容都是都按照`field-string`这样格式的键值对，而能够加入同一个消息队列中的消息，其键值对中的`fields`域大致应该是相同的。那么我们使用第一个加入这个紧凑列表中的消息的`fields`域来作为这个列表的公共`fields`域，如果后续消息的`fields`域与第一个消息相同，那么可以公用这个紧凑列表中`MSGHeader`里的`fields`域，以达到节省内存的目的。
   1. `num-fields`用于记录这个公用消息`fields`域中`field`的个数。
   2. `field_XXX`则是用于记录每个`field`的具体内容。
4. 最后一个`0`字符，相当于一个特殊的标记，用于标记`MSGHeader`的结束。



在这个`MSGHeader`之后，便是每条消息的具体内容，由于紧凑列表之中的消息有可能会被标记为已经删除的状态，因此*Redis*出了一个状态定义`STREAM_ITEM_FLAG_DELETED`用于表示这个节点是否已经被删除。同时*Redis*还给出了一个状态定义`STREAM_ITEM_FLAG_SAMEFIELDS`用于标记这个消息的`fields`域是否与整个紧凑列表的`fields`域相同，根据是否相同*Redis*对消息的内存分布格式给出两种定义。

首先我们看一下`fields`与紧凑列表公共`fields`域不同的消息的存储格式：

```
+-----+--------+----------+-------+-------+-/-+-------+-------+--------+
|flags|entry-id|num-fields|field-1|value-1|...|field-N|value-N|lp-count|
+-----+--------+----------+-------+-------+-/-+-------+-------+--------+
```

在上面这个内存分布之中：

1. `flags`，这个字段以掩码的形式记录了当前消息的状态，前面提到的`STREAM_ITEM_FLAG_DELETED`以及`STREAM_ITEM_FLAG_SAMEFIELDS`都会被记录在这里。
2. `entry_id`，前面我们介绍了，在`stream.rax`这个基数树中，*Value*是存储消息的紧凑列表，而*Key*是紧凑列表之中第一条消息的ID，我们称之为`master_id`；同时我们也知道每条消息都用一个唯一的属于自己的ID。这里`entry_id`便是用于记录消息的ID，只不过不是消息的完整ID，而是记录的自身ID与`master_id`的差值。
3. `num-fields`，由于这条消息的`fields`域与公共`fields`不相同，故此这里记录了该消息的键值对的个数。
4. `field-value`，这一系列字段记录了消息的键值对之中的内容。
5. `lp-count`，这个字段记录前面介绍的`flags`，`entry-id`等字段的个数，用于方便从后向前的反向查找。

最后我们来看一下`fields`域与公共`fields`字段的

```
+-----+--------+-------+-/-+-------+--------+
|flags|entry-id|value-1|...|value-N|lp-count|
+-----+--------+-------+-/-+-------+--------+
```

这里类似`flags`、`entry-id`以及`lp-count`字段与前面的含义相同，只不过由于省区了`fields`域数据的存储，仅以`value-XXX`来存储每个`field`对应的`value`的数据。

### 消费者组

```c
typedef struct streamCG {
    streamID last_id;
    rax *pel;
    rax *consumers;
} streamCG;
```

*Redis*使用`streamCG`来表示一个**消费者组**：

1. `streamCG.last_id`，消息派发游标，用于记录下一条需要发送给**消费者**的消息ID，每次**消费者**请求消息，**消费者组**便会将`streamCG.last_id`对应的消息发送给**消费者**，并将`streamCG.last_id`向后移动一位。
2. `streamCG.pel`，这是一个基数树结构，用于存储未确认消息的列表。
3. `streamCG.consumers`，同样时一个基数树结构，用于存储这个**消费者组**内的**消费者**，基数树的*Key*为**消费者**的名字，对应的`Value`为后面介绍的`streamConsumer`数据结构。

### 消费者

```c
typedef struct streamConsumer {
	mstime_t seen_time;
    sds name;
    rax *pel;
} streamConsumer;
```

`streamConsumer`这个数据结构就是*Redis*用于表示**消费者**：

1. `streamConsumer.seen_time`，记录**消费者**上次活动的时间戳。
2. `streamConsumer.name`，存储**消费者**的名字。
3. `streamConsumer.pel`，这个**消费者**对应的还没有确认的消息的列表。

### 未被确认的消息

```c
typedef struct streamNACK {
    mstime_t delivery_time;
    uint64_t delivery_count;
    streamConsumer *consumer;
} streamNACK;
```

`streamNACK`这个数据结构就是表示已被派发但是没有被确认的消息：

1. `streamNACK.delivery_time`，记录该条消息上次被派发的时间戳。
2. `streamNACK.delivery_count`，存储该条消息被派发的次数。
3. `streamNACK.consumer`，记录该条消息上次被派发的**消费者**。

### Stream迭代器

```c
typedef struct streamIterator {
    stream *stream;
    streamID master_id;
    uint64_t master_fields_count;
    unsigned char *master_fields_start;
    unsigned char *master_fields_ptr;
    int entry_flags;
    int rev;
    uint64_t start_key[2];
   	uint64_t end_key[2];
    raxIterator ri;
    unsigned char *lp;
    unsigned char *lp_ele;
    unsigned char *lp_flags;
    unsigned char field_buf[LP_INTBUF_SIZE];
    unsigned char value_buf[LP_INTBUF_SIZE];
} streamIterator;
```

在这个迭代器数据结构之中：

1. `streamIterator.stream`，记录了当前迭代器关联的消息队列`stream`的指针。
2. `streamIterator.master_id`，记录迭代器当前所在的紧凑列表对应的`master_id`。
3. `streamIterator.master_fields_count`，记录迭代器当前所在的紧凑列表之中公共`fields`域中`field`的个数。
4. `streamIterator.master_fields_start`，记录迭代器当前所在的紧凑列表之中公共`fields`域中第一个`field`的指针。
5. `streamIterator.master_fields_ptr`，迭代器用于遍历当前在的紧凑列表中公共`fields`域的指针。
6. `streamIterator.entry_flags`，迭代器当前遍历消息的标记字段。
7. `streamIterator.rev`，用于记录当前的迭代器是正向迭代器还是逆向迭代器。
8. `streamIterator.start_key`，记录这个迭代器初始化时的起始消息ID，默认是`0-0`。
9. `streamIterator.end_key`，记录这个迭代器初始化时的终止消息ID，默认是`UINT64_MAX-UINT64_MAX`。
10. `streamIterator.ri`，表示这个迭代器用于迭代遍历`stream.rax`这个基数树的底层迭代器`raxIterator`。
11. `streamIterator.lp`，记录迭代器当前指向的紧凑列表指针。
12. `streamIterator.lp_ele`，这是迭代器在遍历当前紧凑列表时使用的，指向列表元素的一个内部指针。
13. `streamIterator.lp_flags`，迭代器在当前的紧凑列表中指向列表中记录消息标记字段`flags`内存的指针，`streamIterator.entry_flags`可以通过这个字段解析获得。
14. `streamIterator.field_buf`，迭代器用于遍历某一条消息的键值对时，存储当前所看到的`field`的内容。
15. `streamIterator.value_buf`，迭代器用于遍历某一条消息的键值对时，存储当前所看到的`value`的内容。

## Redis中Stream的实现逻辑

### Stream的基础实现

#### StreamID的基础操作

对于消息ID`streamID`，*Redis*给出了若干基础的操作函数接口：

1. ```c
   void streamIncrID(streamID *id);
   ```

   这个函数用于递增一个ID的序号部分`streamID.seq`。虽然不太可能，但是一旦`streamID.seq`这个字段超过了`UINT64_MAX`的限制，就需要在ID的时间戳部分`streamID.ms`上进一位。

2. ```c
   void streamNextID(streamID *last_id, streamID *new_id);
   ```

   这个函数会根据给定的消息ID`last_id`来生成下一个新的消息ID`new_id`。

3. ```c
   void streamEncodeID(void *buf, streamID *id);
   ```

   这个函数用于将一个`streamID`按照大端的字节序列编码成一个128比特的数据，这样可以保证编码后的数据依然是满足字典顺序的。

4. ```c
   void streamDecodeID(void *buf, streamID *id);
   ```

   这个函数用于将一个128比特的数据解码成一个`streamID`对象。
   
5. ```c
   int streamCompareID(streamID *a, streamID *b);
   ```
   
   用于比较两个`streamID`对象的大小。

#### Stream中消费者组的基础操作

首先我们来看一组关于消息队列之中，**消费者组**相关的底层函数接口：

1. ```c
   streamNACK *streamCreateNACK(streamConsumer *consumer);
   ```

   这个函数用于从一个**消费者**`consumer`来创建一个未确认消息对象`streamNACK`。

2. ```c
   void streamFreeConsumer(streamConsumer* sc);
   ```

   这个函数用于释放一个给定的**消费者**`sc`。

3. ```c
   streamCG *streamCreateCG(stream *s, char *name, size_t namelen, streamID *id);
   ```

   这个函数用于在一个给定的消息队列`s`上，创建一个名为`name`的**消费者组**。同时会指定**消费者组**的派发游标`streamCG.last_id`，并会将这个新的**消费者组**加到消息队列的**消费者组**队列之中`stream.cgroups`。

4. ```c
   void streamFreeCG(streamCG *cg);
   ```

   这个函数用于释放一个给定的**消费者组**。

5. ```c
   streamConsumer *streamLookupConsumer(streamCG *cg, sds name, int create);
   ```

   这个函数用于从`cg`这个**消费者组**中查找名为`name`的**消费者**。在没有找到的情况下，如果`create`不为0，那么会创建一个新的**消费者**加入到**消费者组**中，并返回这个新的**消费者**；否则返回`NULL`。

6. ```c
   uint64_t streamDelConsumer(streamCG *cg, sds name);
   ```

   这个函数则用于从`cg`这个**消费者组**中删除一个名为`name`的**消费者**。同时也会释放未确认消息队列**PEL**之中的与这个**消费者**相关的未确认消息对象`streamNACK`。

#### Stream中消息的插入与删除

接下来我们看一下如何向消息队列之中插入一条新的消息：

```c
int streamAppendItem(stream *s, robj **argv, int64_t numfields, streamID *added_id, streamID *use_id);
```

`streamAppendItem`用于向一个指定的消息队列之中插入一条新的消息。如果参数`added_id`不为`NULL`，那么新消息对应的消息ID将会通过`added_id`进行返回；如果`use_id`参数为`NULL`，这表示该条新消息将使用系统自动生成的消息ID，也就是根据`stream.last_id`来生成新的ID，否则使用调用者给出的`use_id`作为新消息的ID。

`streamAppendItem`是消息队列插入逻辑底层实现函数，而前面所介绍的消息在消息队列对应基数树中存储方式的细节，都是在这个函数之中实现的。

而从消息队列之中的删除一条指定的消息，则是通过下面的函数来实现的：

```c
int streamDeleteItem(stream *s, streamID *id);
```

这个函数会使用后续介绍的迭代器操作来对给定ID的消息进行删除。

除了删除单条消息的接口，*Redis*还提供了一个用于对消息队列按照指定的大小进行裁剪的函数接口：

```c
int64_t streamTrimByLength(stream *s, size_t maxlen, int approx);
```

当消息队列之中的消息数量过多时，我们可以通过`streamTrimByLength`这个函数接口来裁剪消息队列的大小，删除消息队列中过于老旧的消息。`maxlen`用于限定消息数量的上限，`approx`则是通知*Redis*是使用精确裁剪，还是使用近似裁剪。对于`stream.length > maxlen`的情况下，如果使用精确裁剪，那么裁剪的结果为`stream.length == maxlen`；而使用近似裁剪的方式，裁剪后的最终结果将是`stream.length >= maxlen`。

我们知道，在消息队列的基数树之中，实际存储的是一个一个的紧凑列表，而每个紧凑列表又是多条相邻消息的集合。因此在裁剪时，会将较为老旧的紧凑列表通过`raxRemove`这个接口从基数树之中删除。而精确裁剪与近似裁剪的区别就在于对边界上紧凑列表的处理方式上，假设我们需要将消息队列裁剪到只剩下1000条消息，而从995条消息到1005条消息都是存储在同一个紧凑列表之中的，也就是第1000条消息恰好落在一个紧凑列表的中间，这是两种裁剪的不同处理方式就体现出来了：

1. 近似裁剪方式，在处理到这个边界紧凑列表时，直接返回停止裁剪。此时消息队列之中还剩余1005条消息，虽然没有达到1000条的目的，但是由于*Redis*为每个紧凑列表所能容纳的消息数量做出了限制，因此剩余的消息数量不会大于`maxlen`很多，同时也保证了，绝对不会多删消息。
2. 精确裁剪方式，则会遍历这个边界紧凑列表，将需要删除的消息设置上`STREAM_ITEM_FLAG_DELETED`标记，同时更新紧凑列表中已删除消息的计数。由于这种方式可能涉及到内存移动以及内存的重新分配，因此效率要低于近似的裁剪方式。

而上面这个函数，也正是**XTRIM**命令的底层实现逻辑。

#### Stream中消息的获取

对于消息队列之中消息的获取，*Redis*给出了两个底层的操作接口。类似**XREAD**命令、**XRANGE**命令、**XREVRANGE**命令都是通过这里的底层操作接口来实现的；对于**消费者组**的操作中，**XREADGROUP**命令、**XPENDING**这两个**消费者**用于从**消费者组**中获取消息的命令，也是通过这里的底层接口来实现。

首先我们来看一下从消息队列之中获取消息的函数接口：

````c
size_t streamReplyWithRange(client *c, stream *s, streamID *start, streamID *end, size_t count, int rev, streamCG *group, streamConsumer *consumer, int flags, streamPropInfo *spi);
````

这个函数会将消息队列`s`上的ID范围在`[start, end]`闭区间范围内的消息，发送给客户端`c`；`count`参数如果不为0，则会按照顺序从区间内返回`count`条消息发送给客户端；而`rev`则指定了消息的方向，究竟是从前向后发送，还是从后先前发送。

如果参数`group`以及`consumer`这两个参数不为空的时候，表示是在处理**XREADGROUP**命令通过**消费者组**来获取消息的情况，因为**XREADGROUP**命令与**XREAD**命令一样，底层都是通过`streamReplyWithRange`这个函数来实现的。

### Stream的迭代器操作

#### 迭代器基础操作

我们先来看一下，*Redis*对于消息队列迭代器的基础操作接口：

1. ```c
   void streamIteratorStart(streamIterator *si, stream *s, streamID *start, streamID *end, int rev);
   ```

   这个函数用于初始化一个消息队列迭代器`si`。该函数会将迭代器`si`与消息队列`s`相关联；同时通过`start`以及`end`这两个参数用于限定迭代遍历的范围，这两个范围上下限可以为空值，分别对应`0`以及`UINT64_MAX`；最后可以通过`rev`来指定遍历方向。

2. ```c
   void streamIteratorStop(streamIterator *si)
   ```

   这个函数用于释放一个消息队列迭代器`si`。

3. ```c
   int streamIteratorGetID(streamIterator *si, streamID *id, int64_t *numfields);
   ```

   这个函数用于实现对消息队列之中的消息进行遍历，每次遍历会通过`id`返回消息对应的消息ID，以及通过`numfields`参数来返回该消息拥有的键值对的数量。如果遍历到达了结束的位置，则函数会返回0；否则函数会返回1，表示当前可以遍历到一条合法的消息。仅仅通过`streamIteratorStart`函数初始化后的迭代器无法直接对数据进行操作，需要通过`streamIteratorGetID`这个函数调用之后，才可以真正地对消息队列之中的消息进行访问。

4. ```c
   void streamIteratorGetField(streamIterator *si, unsigned char **fieldptr, unsigned char **valueptr, int64_t *fieldlen, int64_t *valuelen);
   ```

   这个函数用于遍历一条消息的键值对，每次调用会从迭代器当前消息的键值对之中，通过`fieldptr`以及`fieldlen`来返回键数据；通过`valueptr`以及`valuelen`来返回值数据。

5. ```c
   void streamIteratorRemoveEntry(streamIterator *si, streamID *current);
   ```

   这个函数用于删除迭代器当前对应的消息，不过我们需要显式地提供消息ID。如果该消息所在的紧凑列表只有该条消息这一个未删除的消息，那么就可以将这个紧凑列表从消息队列的基数树之中删除；如果该消息所在的紧凑列表还有其他的未删除消息，那么仅需要将这个消息加上删除标记`STREAM_ITEM_FLAG_DELETED`。删除这条当前的消息之后，迭代器会被重新定位到下一条消息之上。

#### 迭代器使用范式

结合上面关于消息队列迭代器的基础操作，我们可以总结出迭代器使用的两个范式：

1. 对消息队列之中一定范围内的消息进行迭代遍历：

   ```c
   streamIterator myiterator;
   streamIteratorStart(&myiterator, ...);
   int64_t numfields;
   while(streamIteratorGetID(&myiterator,&ID,&numfields))
   {
       while (numfields) {
           unsigned char *key, *value;
           size_t key_len, value_len;
           streamIteratorGetField(&myiterator,&key,&value,&key_len,&value_len);
           //对收集到的键值对进行处理
       }
   }
   ```

   这里我们可以看到遍历消息队列的一个基本流程，即先通过`streamIteratorStart`对迭代器进行初始化；然后通过`streamIteratorGetID`函数对消息进行遍历，获取到消息ID，以及该条消息中的键值对的个数；最后通过`streamIteratorGetField`函数完成对当前消息中键值对数据的遍历。

2. 通过迭代器删除指定ID的消息：

   ```c
   streamIterator si;
   streamIteratorStart(&si, s, id, id, 0);
   streamID myid;
   int64_t numfields;
   if (streamIteratorGetID(&si, &myid, &numfields))
   {
       streamIteratorRemoveEntry(&si, &myid);
   }
   streamItertorStop(&si);
   ```

   前面介绍的，通过`streamDeleteItem`来删除消息队列之中指定ID消息，就是通过上面这种范式来实现的。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*