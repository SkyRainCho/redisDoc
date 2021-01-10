# Redis脚本

我们知道在关系型数据库之中，存在一个存储过程的一个概念。由于关系型数据自身仅仅能够提供增删改查这种基础功能，无法执行复杂逻辑，而所谓存储过程是一种在数据库中存储复杂程序，以便外部程序调用的一种数据库对象。存储过程本质上时为了完成特定功能的SQL语句集，经过编译创建并保存在数据库之中，数据库的用户可以指定存储过程的名字并给定参数来调用执行，用以实现数据库SQL语言层面的代码封装与复用。

*Redis*虽然不是关系型数据库，但是也为用户提供了类似存储过程的功能。这个功能的核心是*Redis*内部集成了一个Lua脚本的解释器，用户可以通过Lua脚本来原子地调用一条或者多条*Redis*命令，并且可以集成一定的用户自定义逻辑。

## Redis脚本概述

除了执行Lua脚本之外，*Redis*可以通过**PIPELINE**以及事务功能来一次性地执行多条命令，那么这三种方式有什么区别呢？

首先，**PIPELINE**机制完全是客户端的行为，客户端一次性将多条命令发送至*Redis*服务器，这个机制完全得益于*Redis*服务器核心所使用的事件驱动处理逻辑。这种方式虽然可以一次性地执行多条*Redis*命令，但是却无法保证命令执行的原子性。这一点在前面介绍*Redis*客户端对象那一部分的文章之中有过介绍。

事务功能，会在客户端执行**MULTI**命令时，将客户端对象设置成一个事务状态，后续用户通过客户端执行的命令都会被缓存起来，而不是立即执行；最后当用户通过客户端执行**EXEC**命令时，前面被缓存起来的命令会一次性地原子的执行。而事务功能中，命令的执行过程不会被其他客户端所打断，具备原子性。然而事务功能也有它的局限性，它只能简单地执行多条*Redis*的原生命令，却对逻辑进行自定义。例如，我们希望根据*Redis*数据之中的某个特定键中存储的数值的不同，后续执行不同的命令，这种场景事务功能则是爱莫能助，如果将这个逻辑放在客户端去执行，先通过**GET**命令获取某一个键的值，在根据值的不同向服务器发起不同的命令，而这样的话又无法保证逻辑的原子性。

而Lua脚本恰恰是为了解决上面的这个场景需求所设计的，用户可以向*Redis*服务器发送一段Lua脚本代码段，在代码段之中可以调用*Redis*命令并获得命令的返回值，同时我们还可以使用Lua脚本语法不同的逻辑执行不同的*Redis*命令。而在一段Lua代码段之中的所有Lua逻辑以及*Redis*命令的执行是原子性的，不会被其他客户端的命令所打断。

### EVAL与EVALSHA命令

上述这两个命令是*Redis*中执行Lua脚本的入口，首先我们来看一下**EVAL**命令的格式：

```
EVAL script numkeys key [key ...] arg [arg ...]
```

这个命令之中，`script`是一段Lua脚本片段，这段代码片段不应该被定义为一个函数。在这段Lua代码之中，可以通过`redis.call()`以及`redis.pcall（）`来调用*Redis*的原生命令。`numkeys`参数可以指定传入脚本之中的`key`参数的个数；后面的`key`则表示在脚本中能够用的*Redis*中的键参数，在脚本之中可以用`KEYS[1]`、`KEYS[2]`这样的形式引用这些参数。而最后的参数`arg`则是表示不是*Redis*键的参数，在脚本之中可以用`ARGV[1]`，`ARGV[2]`这样的形式来进行引用。

```
EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 script:key script:value
```

上述这条命令，相当于在*Redis*上执行一条**SET**命令，为`script:key`这个键设置`script:value`的值。

*Redis*服务器在服务器全局数据结构之中，维护了一个哈希表用于缓存用户通过**EVAL**命令提交的Lua脚本代码段。在这个哈希表之中，键为`f_<hex sha1 sum>`，这其中`<hex sha1 sum>`是通过Lua脚本代码段的字符串计算得出的；哈希表之中的值为Lua脚本代码段的字符串。

而这段Lua代码段相当于在Lua环境之中执行了如下的Lua函数：

```lua
function f_<hex sha1 sum>()
    return redis.call('set', KEYS[1], ARGV[1])
end
```

这样相当于在*Redis*之中维护了一个`<hex sha1 sum> : lua code`的对应关系。通过这个对应关系其他的用户可以复用其他用户上传的Lua脚本代码段。而命令**EVALSHA**恰好是为完成这个需求的，这个命令的格式为：

```
EVALSHA sha1 numkeys key [key ...] arg [arg ...]
```

在这条命令之中，用户不用提交Lua脚本的代码段，而是通过`sha1`参数提交缓存的Lua代码段对应的`sha1`哈希值，这样*Redis*会通过`sha1`这个哈希值查找到缓存的Lua代码段，然后执行这个Lua代码段之中的逻辑。

在*Redis*服务器运行Lua脚本代码时，需要将*Redis*的数据类型作为参数传入Lua环境，在脚本代码结束时需要将Lua数据类型作为返回值返回给*Redis*服务器，下面这个表格便是*Redis*数据与Lua数据的对应转化关系：
Reds
| Redis 数据类型 | Lua 数据类型 | 含义 |
| -------------- | ------------ | ---- |
|                |              |      |


在*Redis*的原生命令之中一些命令属于带有不确定性的命令，例如**HKEYS**这样的命令，即使两个内容完全相同的散列对象，也会因为键值对插入的顺序不同而导致**HKYES**命令的返回结果不同。如果在Lua脚本之中调用这些带有不确定性的命令，*Reds*会通过一个辅助的函数对结果进行排序。这样可以保证，只要两个对象的数据集是一样的，

### SCRIPT命令

*Redis*通过**SCRIPT**命令对于系统缓存的Lua脚本代码段进行操作。

#### 检查脚本缓存是否存在

用户可以通过如下的命令格式来检查给定的`sha1`哈希值对应的Lua代码段是否存在与系统缓存之中：

```
SCRIPT EXISTS script [script ...]
```

这个命令可以接收一个或者多个Lua脚本代码段对应的`sha1`值，结果返回一个包含0或1的列表，0表示对应的脚本不存在于缓存之中，1表示对应的脚本存在于缓存之中。

#### 清空脚本缓存

用户可以通过如下的命令格式来清空系统之中所有缓存的Lua脚本代码段：

```
SCRIPT FLUSH
```

这个命令一定会成功，不会返回失败。

#### 将脚本缓存到系统中

除了执行**EVAL**命令可以将一段Lua代码缓存到系统之中，我们也可以通过**SCRIPT**命令将一段脚本代码缓存到系统之中：

```
SCRIPT LOAD script
```

这个命令与**EVAL**命令不同的地方在，**EVAL**命令会执行Lua代码段中的逻辑，而**SCRIPT**命令则不会执行Lua代码，仅仅是将代码进行缓存，最后命令会返回*Redis*通过Lua代码段生成的`sha1`哈希值。

#### 杀死当前运行的脚本

如果用户提交的Lua代码之中存在一些Bug，导致系统出现问题，例如陷入死循环，我们可以通过**SCRIPT**命令将当前运行的脚本杀掉：

```
SCRIPT KILL
```

这个命令执行之后，执行这个脚本的客户端会从**EVAL**命令的阻塞状态中退出，并收到一个错误作为返回值。这个命令只会杀掉那些执行只读命令的Lua脚本，对于执行过写命令的脚本，`SCRIPT KILL`命令无法将其杀掉，对于这种情况，说明了*Redis*数据库内存中的数据已经被污染，唯一可行的方案就是将*Redis*服务器关掉，防止污染数据被写入磁盘，我们可以通过下面的这条命令来以不存盘的方式关闭*Redis*服务器：
```
SHUTDOWN NOSAVE
```

## Redis脚本代码实现

### Redis脚本数据结构

在*Redis*服务器的全局变量之中，定义执行Lua脚本的相关数据结构：
```c
struct redisServer {
    ...
    lua_State *lua;
    client *lua_client;
    client *lua_caller;
    dict *lua_scripts;
    unsigned long long lua_scripts_mem;
    mstime_t lua_time_limit;
    mstime_t lua_time_start;
    int lua_write_dirty;
    int lua_random_dirty;
    int lua_replicate_commands;
    int lua_multi_emmitted;
    int lua_repl;
    int lua_timeout;
    int lua_kill;
    int lua_always_replicate_comnands;
};
```
这些数据字段之中：
1. `redisServer.lua`，这是一个Lua解释器对象，整个*Redis*服务器所有的客户端只需要一个Lua解释器。
1. `redisServer.lua_client`，这是一个伪客户端，用于执行Lua脚本之中通过`redis.call`调用的*Redis*命令。
1. `redisServer.lua_caller`，当前正在执行Lua脚本的客户端，如果当前没有执行Lua脚本，这个指针会被设置为`NULL`。
1. `redisServer.lua_scripts`，这是一个哈希表，用于存储脚本SHA1哈希值与缓存的Lua脚本代码段的映射关系。
1. `redisServer.lua_scripts_mem`，记录了缓存Lua脚本代码段占用的内存。
1. `redisServer.lua_time_limit`，配置的Lua脚本执行时间上限，如果脚本执行的时间超过这个上限则认为是执行超时。
1. `redisServer.lua_time_start`，记录当前Lua脚本执行的启动时间戳。
1. `redisServer.lua_write_dirty`，如果当前Lua脚本中执行的*Redis*命令修改了键空间之中的数据，这个字段会被设置为1。
1. `redisServer.lua_random_dirty`，如果当前Lua脚本中执行的*Redis*命令是带有随机标记`CMD_RANDOM`，这个字段会被设置为1。
1. `redisServer.lua_replicate_commands`，
1. `redisServer.lua_multi_emmitted`，
1. `redisServer.lua_repl`，
1. `redisServer.lua_timeout`，如果脚本执行的时间超过了`redisServer.lua_time_limit`设置的上限，那么这个字段会被设置为1。
1. `redisServer.lua_kill`，如果脚本执行时间超时，其他用户可以执行`SCRIPT KILL`命令将这个字段设置为1，后续脚本会判断这个字段是否为1来决定是否终止脚本运行。
1. `redisServer.lua_always_replicate_comnands`，


### Redis脚本实现逻辑

首先*Redis*会通过`sha1hex`这个函数来计算Lua脚本代码段的SHA1哈希值：
```c
void sha1hex(char *digest, char *script, size_t len);
```
脚本的哈希值可以通过`digest`参数进行返回。

下面我们按照不同的功能分组，来看一下代码的实现逻辑。

#### Redis与Lua的数据转换

这里我们先来看一下，前面在概述之中介绍的*Redis*数据类型与Lua数据之间的转换。

##### Redis到Lua数据的转换

首先*Redis*定义了下面5个函数，用于实现底层的*Redis*数据到Lua数据的转换：
```c
char *redisProtocolToLuaType_Int(lua_State *lua, char *reply);
char *redisProtocolToLuaType_Bulk(lua_State *lua, char *reply);
char *redisProtocolToLuaType_Status(lua_State *lua, char *reply);
char *redisProtocolToLuaType_Error(lua_State *lua, char *reply);
char *redisProtocolToLuaType_MultiBulk(lua_State *lua, char *reply)
```
这些函数的名字明显地指明了这些函数的功能，分别用于将`reply`数据之中的整数、块数据、状态信息、错误信息，多块数据压入到Lua的虚拟栈之中，这里需要注意如下内容：
1. `redisProtocolToLuaType_Bulk`这个函数中，如果块数据为空，那么会调用`lua_pushboolean`向虚拟栈中压入一个`false`值；否则调用`lua_pushstring`函数将块数据作为Lua字符串压入Lua虚拟栈。
1. `redisProtocolToLuaType_Status`以及`redisProtocolToLuaType_Error`这两个函数，都会向Lua虚拟栈之中压入一个Lua的表数据，并向这个表中插入域值对`["ok"] = status`或者`["err"] = error_message`。

通过上面这个5个函数，*Redis*定义了一个更高级的函数接口用于将*Redis*的数据压入Lua虚拟栈：
```c
char *redisProtocolToLuaType(lua_State *lua, char* reply);
```
这个函数会根据`reply`之中的特殊前缀也就是第一个字符，来对应数据类型的转换：
1. `:`，需要转换为整数。
1. `$`，需要转换为块数据。
1. `+`，需要转换为状态数据。
1. `-`，需要转换为错误数据。
1. `*`，需要转换为多块数据。

##### Lua到Redis数据的转换
而Lua数据到*Redis*数据的转换则是通过单独的一个函数来实现的：
```c
void luaReplyToRedisReply(client *c, lua_State *lua);
```
这个函数的逻辑为：
1. 首先会通过`lua_type`函数获取Lua虚拟栈的栈顶元素类型。
1. 如果栈顶元素类型为`LUA_TSTRING`、`LUA_TBOOLEAN`、`LUA_TNUMBER`，那么执行将对应的数据转换为*Redis*的Reply数据。
1. 如果栈顶元素类型为`LUA_TTABLE`，则会根据Lua表中的数据内容，将其转化为*Redis*的状态、错误以及多块数据。

#### 辅助函数
前面提到了，*Redis*在执行Lua脚本的过程之中，会对带有不确定性的命令的输出结果进行排序，这个功能*Redis*是通过下面这个辅助函数来实现的：
```c
void luaSortArray(lua_State *lua);
```
`luaSortArray`这个函数会通过Lua语言之中的`table.sort`接口，对Lua虚拟栈之中的命令返回数据进行排序，使之有序。

Lua语言之中的一些全局函数例如`loadfile`，*Redis*不希望能在脚本之中能够被使用，因此通过`luaRemoveUnsupportedFunctions`接口将这些函数从Lua环境之中移除。
```c
void luaRemoveUnsupportedFunctions(lua_State *lua);
```
这个函数会通过将指定的Lua全局函数的函数名设置为`nil`，来从Lua环境之中将这些函数移除的。

同时，为了防止数据的残留，*Redis*禁止在Lua脚本之中声明任何全局变量；但是可以使用`local`关键字声明局部变量，这样Lua环境可以在脚本执行结束之后，局部变量可以自动被回收， *Redis*通过下面这个函数，实现这个功能：

```c
void scriptingEnableGlobalsProtection(lua_State *lua);
```



#### Lua脚本之中的Redis接口
前面我们介绍了，可以在Lua脚本代码段之中通过`redis.call`以及`redis.pcall`接口调用*Redis*的原生命令。除了上述两个接口之外，*Redis*还定义了若干个C语言接口共Lua脚本代码段之中进行调用。

|*Redis*的C语言接口|lua代码|接口含义|
|-----------------|--------|--------|
|`int luaRedisCallCommand(lua_State *lua)`|`redis.call`|Lua代码中用于调用*Redis*原生命令的接口|
|`int luaRedisPCallCommand(lua_State *lua)`|`redis.pcall`|Lua代码中用于调用*Redis*原生命令的接口|
|`int luaRedisSha1hexCommand(lua_State *lua)`|`redis.sha1hex`|Lua代码中用于计算一个字符串的SHA1哈希值|
|`int luaRedisErrorReplyCommand(lua_State *lua)`|`redis.error_reply`|在Lua代码中，向Lua虚拟栈压入错误信息|
|`int luaRedisStatusReplyCommand(lua_State *lua)`|`redis.status_reply`|在Lua代码中，向Lua虚拟栈压入状态信息|
|`int luaRedisReplicateCommandsCommand(lua_State *lua)`|`redis.replicate_commands`|如果截止这个接口前，脚本没有执行过写命令的话，这个接口可以开启单一命令的复制并返回`true`；否则这个接口会返回`false`，并继续坚持整个脚本的复制工作。|
|`int luaRedisSetReplCommand(lua_State *lua)`|`redis.set_repl`||
|`int luaLogCommand(lua_State *lua)`|`redis.log`|通过这个接口可以向*Redis*服务器的日志系统之中添加日志|

在上述这些函数接口之中，比较重要的是在Lua脚本代码段之中，通过`redis.call`以及`redis.pcall`来调用*Redis*原生命令的接口，这两个接口都是通过下面这个函数来实现的：
```c
int luaRedisGenericCommand(lua_State *lua, int raise_error);
```
在这个函数之中，会执行如下的逻辑：
1. 通过`lua_gettop`接口我们可以获取Lua虚拟栈中的参数个数并检查参数的个数，`redis.call`以及`redis.pcall`函数至少需要一个参数作为*Redis*命令的名字。
1. 从Lua的虚拟栈之中解析出`redis.call`或者`redis.pcall`函数的参数。这里需要注意的是，这些参数必须是整数或者字符串形式的参数，我们在脚本之中向`redis.call`函数中传输其他类型的参数。
1. 将解析出来的参数传入*Redis*用于在脚本之中执行原生命令的伪客户端`redisServer.lua_client`之中。
1. 通过`lookupCommand`接口查找待执行的命令，判断参数个数，并检查这个命令是否禁止在Lua脚本之中执行（命令携带了`CMD_NOSCRIPT`标记）。
1. 如果脚本命令是一个写命令`CMD_WRITE`，那么会检查当前的脚本之中是否可以执行写命令。
1. 检查是否已经达到内存使用的上限，
1. 根据命令的类型，如果是写命令`CMD_WRITE`，则将`redisServer.lua_write_dirty`设置为1；如果是随机命令`CMD_RANDOM`，则将`redisServer.lua_random_dirty`字段设置为1。
1. 通过`call`接口来执行*Redis*的原生命令。
1. 从`redisServer.lua_client`这个伪客户端之中将应用层输出缓存之中的内容，通过前面介绍的`redisProtocolToLuaType`接口转换为Lua数据类型，作为返回值压入Lua虚拟栈。
1. 最后，如果执行的这个*Redis*原生命令是带有不确定性的命令，那么会通过`luaSortArray`接口对Lua栈之中的数据进行排序。

#### Lua环境的初始化

现在我们来看看，*Redis*是如何初始化一个Lua环境的：

1. *Redis*会在服务器通用的初始化函数`initServer`的最后，为服务器初始化Lua环境。
2. 在执行`SCRIPT FLUSH`命令清空服务器上的Lua脚本缓存时，为了清空Lua环境之中可能残留的数据，也会对系统的Lua环境进行初始化。

```c
void scriptingInit(int setup);
```

通过`setup`参数，可以确定是对Lua环境的初始化是发生在系统启动时，还是通过`SCRIPT`进行的。`scriptingInit`这个初始话函数主要执行的逻辑为：

1. *Redis*会同通过`lua_open`函数来创建一个新的Lua环境，并通过`luaLoadLibraries`为这个Lua环境加载脚本执行时必要的库；并通过`luaRemoveUnsupportedFunctions`来移除*Redis*不支持的Lua原生函数。
2. 初始化*Redis*服务器全局变量中缓存脚本的哈希表`redisServer.lua_script`。
3. 在禁止创建Lua全局变量之前，先为Lua环境创建一个全局的表`redis`，并将前面定义的C语言接口以及其他系统变量注册进这个全局表`redis`之中。例如将`luaRedisCallCommand`这个C语言进口注册进`redis`全局表，成为`redis.call`这个函数。
4. `scriptingInit`会将Lua库函数之中一些具有副作用的函数替换成*Redis*自己实现的函数接口，需要被替换的函数有两个`math.random`以及`math.randomseed`这两个Lua库函数。
5. 为执行Lua脚本之中的*Redis*原生命令，*Redis*需要初始化一个用于执行命令的伪客户端`redisServer.lua_client`。
6. 最后*Redis*会通过`scriptingEnableGlobalsProtection`函数，来保护Lua环境，禁止后续用户在Lua脚本之中声明全局变量。

#### 脚本命令的实现

对于*Redis*原生的脚本命令之中，较为重要的便是通过**EVAL**来执行脚本的命令以及通过**SCRIPT**命令来杀掉出现Bug脚本这两个命令，本小节之中也着重对这两个部分进行介绍。

```c
void evalGenericCommand(client *c, int evalsha);
```

通过**EVAL**命令以及**EVALSHA**命令来执行Lua脚本代码的，下面我们来简要介绍一下`evalGenericCommand`这个函数的实现逻辑：

1. 首先清空服务器全局变量之中关于Lua脚本的数据字段，包括`redisServer.lua_random_dirty`以及`redisServer.lua_write_dirty`等这样的字段，将系统之中的Lua环境的数据重置。
2. 获取Lua脚本对应的SHA1哈希值，可以如果是**EVAL**命令，则是通过命令之中给定的脚本计算出来；如果是**EVALSHA**命令则从命令参数之中获取到脚本的哈希值。
3. 查找脚本的哈希值对应的函数是否存在于Lua环境之中，如果不存在说明是通过**EVAL**命令执行的新的Lua脚本。对于这种情况则调用`luaCreateFunction`，将这个新的Lua脚本加入系统缓存之中。
4. 将命令之中的`key`以及`arg`参数，写入Lua脚本的全局变量`KEYS`以及`ARGV`之中，供脚本的函数逻辑使用。
5. 设置一些服务器关于Lua脚本执行的字段，包括当前执行脚本的客户端指针`redisServer.lua_caller`，脚本执行的开始时间`redisServer.lua_time_start`。如果系统设置了脚本执行时间的上限`redisServer.lua_time_limit`，那么会设置Lua的钩子`luaMaskCountHook`用于检测脚本执行是否超时。
6. 通过`lua_pcall`来执行这个脚本函数。
7. 通过`luaReplyToRedisReply`这个接口，将Lua环境之中的返回数据输出给客户端对象的应用层输出缓冲区。

在了解了*Redis*如何通过**EVAL**以及**EVALSHA**命令执行Lua脚本之后，我们看看*Redis*服务器如何检测Lua脚本执行超时，并通过**SCRIPT**命令将超时的脚本杀掉的实现细节。

首先*Redis*是通过`luaMaskCountHook`这个钩子来检测的：

```c
void luaMaskCountHook(lua_State *lua, lua_Debug *ar);
```

在`evalGenericCommand`函数之中的调用场景为：

```c
void evalGenericCommand(client *c, int evalsha)
{
    ...
    if (server.lua_time_limit > 0 ldb.active == 0)
    {
        lua_sethook(lua, luaMaskCountHook, LUA_MASKCOUNT, 100000);
        delhook = 1;
    }
    ...
}
```

这里相当于Lua脚本每执行100000次执行单元就会调用`luaMaskCountHook`函数来检查脚本是否超时，

```c
void luaMaskCountHook(lua_State *lua, lua_Debug *ar)
{
    long long elapsed = mstime() - server.lua_time_start;
    if (elapsed >= server.lua_time_limit && server.lua_timedout == 0)
    {
        server.lua_timedout = 1;
        protectClient(server.lua_caller);
    }
    if (server.lua_timedout) processEventsWhileBlocked();
    if (server.lua_kill) {
        lua_error(lua);
    }
}
```

每次调用`luaMaskCountHook`这个函数时，都会检查从`redisServer.lua_time_start`这个时间戳开始，Lua脚本执行到现在是否超时，如果超时则会将`redisServer.lua_timedout`字段设置为1。当Lua脚本出现超时时，本质上是主线程卡死在`redisServer.lua_caller`这个客户端上来执行Lua脚本的流程上，此时如果不进行特殊的处理，*Redis*主线程将无法重新进入事件循环去处理其他的客户端发来的命令。因此*Redis*在检测到Lua脚本执行超时时，会调用`processEventsWhileBlocked`来强制*Redis*进入事件循环接收其他客户端的命令，这样其他客户端可以发送**SCRIPT**命令来终结这个超时脚本的执行。

在*Redis*处理客户端命令的函数接口之中：

```c
int processCommand(client *c) {
    ...
    if (server.lua_timedout &&
        c->cmd->proc != authCommand &&
        c->cmd->proc != replconfCommand &&
        !(c->cmd->proc == shutdownCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'n') &&
        !(c->cmd->proc == scriptCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'k'))
    {
        flagTransaction(c);
        addReply(c, shared.slowscripterr);
        return C_OK;
    }
    ...
}
```

上述代码之中，表明当Lua脚本执行超时时，仅允许客户端执行特定的几个命令，其中就包含了杀掉脚本运行的**SCRIPT**命令。

最后我们来看一下**SCRIPT**命令是如何终止脚本运行的：

```c
void scriptCommand(client *c) {
    ...
    if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"kill")) {
    	 if (server.lua_caller == NULL) {
             
         } else if (server.lua_caller->flags & CLIENT_MASTER) {
             
         } else if (server.lua_write_dirty) {
             
         } else {
          	server.lua_kill = 1;
            addReply(c,shared.ok); 
         }
    }
    ...
}
```

这里会通过`redisServer.lua_caller`来检查当前是否运行者Lua脚本，通过`redisServer.lua_write_dirty`来检查脚本之中是否已经运行过写命令，同时检查这个当前运行Lua脚本的客户端是不是主从模式之中代表主服务器的客户端对象。在排除了上述这些检查之后，会将`redisServer.lua_kill`这个字段设置为1。

回头再看`luaMaskCountHook`这个函数，当我们通过**SCRIPT**命令将`redisServer.lua_kill`字段设置为1之后，当脚本执行再次进入`luaMaskCountHook`这个函数后，会通过`lua_error`这个接口来终止脚本的运行。

#### 脚本复制

最后我们来看一下在主从模式之下，脚本的复制机制。


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*