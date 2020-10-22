# Redis命令支持

在了解*Redis*命令执行之前，我们先来看一下代表*Redis*命令的结构体的定义，在*src/server.h*这个头文件中：
```c
typedef void redisCommandProc(client *c);
typedef int *redisGetKeysProc(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags;
    int flags;
    redisGetKeysProc *getkeys_proc;
    int firstkey;
    int lastkey;
    int keystep;
    long long microseconds, calls;
};
```
首先讲解一下，这个结构体中常用的字段的含义：
1. `redisCommand.name`
在*Redis*服务器全局数据中，使用一个哈希表存储了所有的*Redis*命令，使用命令名作为哈希表的*Key*用来索引`redisCommand`数据：
```c
struct redisServer {
    ...
    dict *commands;
    ...
}
```
在*src/server.c*源文件之中*Redis*使用硬编码的形式维护了一个`redisCommand`的数组：
```c
struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,"as",0,NULL,0,0,0,0,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    ...
};
```
而*Redis*会通过下面这个函数`populateCommandTable`，在初始化服务器配置`initServerConfig`接口之中，将`redisCommandTable`数组中的数据整合进`redisServer.commands`这个哈希表之中：
```c
void populateCommandTable(void);
```
通过下面这两个函数，我们可以通过给定命令名称来从`redisServer.commands`中检索`redisCommand`对象：
```c
struct redisCommand *lookupCommand(sds name);
struct redisCommand *lookupCommandByCString(char *s);
```

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*