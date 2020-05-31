# Redis中字符串对象类型实现

对于*Redis*中字符串对象的类型的代码主要分布在两个文件之中，其中在*src/object.c*文件中主要是实现了字符串数据类型的构造相关的操作，另外在*src/t_string.c*文件中则实现了字符串的相关命令。

***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*