# Redis服务器入口

*src/server.c*这个源文件，这里是*Redis*服务器的入口点，其中定义了`main()`函数。
以下是启动*Redis*服务器的几个重要步骤：
1. 通过调用`initServerConfig()`函数来设置`server`结构体的默认值。
2. 通过调用`initServer`来分配操作所需要的数据结构，设置监听的*Socket*套接字。
3. 通过调用`eaMain(()`来启动事件循环，用以监听新的连接。

在*Redis*的事件循环中会定期调用两个特殊函数：
1. 根据`server`结构体重定义的频率`server.hz`来定期调用`serverCron()`
来执行那些必须被不时执行的任务，例如定期检查超时的客户端。
2. 而`beforeSleep()`会在每次事件循环被触发的时候调用，*Redis*会处理一些请求，
然后返回到事件循环之中。

在这个*src/server.c*源文件中，可以找到很多处理*Redis*服务器其他重要内容的代码:
1. `call()`用于在一个给定的客户端上下文中调用给定的命令。
2. 当使用`EXPIRE`命令为某个*key*设置生存时间时，会调用`activeExpireCycle()`。
3. 当执行一个写入命令时，而此时*Redis*内存不足的时候，会调用`freeMemoryIfNeeded()`。
4. 全局变量`redisCommandTable`定义了所有的*Redis*命令，指定了命令的名字，
该命令的实现函数，所需的参数的数量以及这个命令的其他一些属性。