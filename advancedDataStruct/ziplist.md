# Redis中压缩链表的实现
前面我们已经介绍了*Redis*中所实现的经典双端链表了，而这种链表存在一个问题，便是在存储小数据的时候，
内存使用效率过低，例如当一个链表节点中只保存一个字节的`unsigned char`数据时，我们需要为这个节点保存24个字节的额外数据，
其中包含`listNode.prev`指针，`listNode.next`指针，以及指向具体数据的`listNode.value`指针。
为了解决经典双端链表在保存小数据而导致内存效率过低的问题，*Redis*设计了一套                                                                        


***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*