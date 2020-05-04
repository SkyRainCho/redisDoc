# Redis中压缩链表的实现
压缩链表主要用做*Redis*中列表数据类型的底层实现之一，前面我们已经介绍了*Redis*中所实现的经典双端链表了，
那么为什么还需要一个压缩链表作为列表数据类型的实现之一呢？


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*