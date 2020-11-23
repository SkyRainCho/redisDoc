# Redis主从复制


## 复制功能概述
简单来说*Redis*的复制功能主要可以划分两个阶段：
1. **数据同步**：*Slave*节点通过某种方式获取*Master*节点的数据副本，并将其加载到内存键空间之中，使该*Slave*节点成为*Master*节点的镜像节点。
1. **命令转发**：*Master*节点在执行命令时，将该命令转发给自己的*Slave*节点，以便在运行之中保证*Master*实例与*Slave*实例之间数据的一致性。

### 数据同步
*Redis*可以通过两种方式开启**数据同步**：
1. 通过配置文件，在*redis.conf*配置文件之中，配置`replicaof <masterip> <masterport>`这条配置项，可以设置对应*Master*节点的地址；如果*Master*实例需要密码认证，可以通过`masterauth <master-password>`配置项来设置连接*Master*所需要认证密码。
1. 执行**REPLICAOF**命令，命令的格式为`REPLICAOF host port`，通过这条命令，可以使执行这条命令的*Redis*成为给定地址`host`以及`port`的另一个*Redis*实例的*Slave*节点，不过这条命令有一个缺陷，如果你对应的*Master*节点需要密码验证，而

### 命令转发

## 复制功能相关数据结构

## 从Master角度看复制功能

## 从Slave角度看复制功能


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*