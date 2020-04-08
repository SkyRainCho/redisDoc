# Redis中Append Only File重写缓冲区的实现
在*Redis*中实现了一个简单的缓冲区，用于当后台进程重写*AOF*文件时，
记录每次的变化，通常基于这个机制，我们可以在出现机器掉电时，基于这个累积的变化，
来恢复数据。虽然我们只需要追加数据，但是我们不能只对一个大的数据块使用`realloc`，
因为*巨大*的`reallocs`并非总是像人们期望的那样处理，但可能涉及复制数据。
因此，我们使用列表的形式来维护数据块，列表中每个数据块都是`AOF_RW_BUF_BLOCK_SIZE`个字节。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*