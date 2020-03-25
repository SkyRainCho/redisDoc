# Redis中的C语言动态Strings库

这个库主要是用来描述Redis之中的五个基本类型之一的String，
Redis使用 `typedef char *sds;` 来描述这个动态String，
其在内存中的分布格式为一个StringHeader以及在StringHeader后面
一段连续的动态内存，而`sds`则是指向StringHeader后面的连续内存的第一个字节。