# Redis中原子计数器的实现
如果系统中提供了`__atomic`或者`__sync`宏的支持，
那么*Redis*将使用的相关的机制实现其原子计数器的操作；
否则的话，*Redis*会使用互斥量来完成计数器在不同线程之间的同步。

## 原子操作接口
|宏定义|含义|
|-----|---|
|`atomicIncr(var,count)`|将计数器`var`原子地增加`count`|
|`atomicGetIncr(var,oldvalue_var,count)`|将计数器的值通过`oldvalue_var`返回，同时为计数器原子地加`count`|
|`atomicDecr(var,count)`|将计数器原子地减`count`|
|`atomicGet(var,dstvar)`|原子地获取计数器的值，通过`dstvar`返回|
|`atomicSet(var,value)`|原子地将计数器的值设置为`value`|

## 原子操作实现
对于*Redis*中的原子计数器，源代码中给出了三种实现方式：
### Atomic实现
在*C11*标准中（注意不是*C++11*标准），为并发编程提供了原子操作的支持，
其定义在*stdatomic.h*头文件中，以`atomicIncr`为例，其实现为：
```c
#define atomicIncr(var,count) __atomic_add_fetch(&var,(count),__ATOMIC_RELAXED)
```
这其中，`__atomic_add_fetch`是系统为原子加法提供的支持，
而`__ATOMIC_RELAXED`，则是执行该操作时，所约定的内存模型，对应于
```c
enum memory_order
{
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst,
};
```
在*Atomic*实现中，*Redis*对于所有的原子操作，都采用宽松的内存模型，
其含义是，没有同步或者顺序制约，仅对这次操作要求原子性。
而宽松内存属性是处理计数器自增的典型使用场景。

### Sync实现
在*C11*之前，*gcc*对原子操作的支持是通过`builtin`函数实现的，即`__sync`前缀的函数。
仍然以`atomicIncr`为例，其*Sync*的实现方式为：
```c
#define atomicIncr(var,count) __sync_add_and_fetch(&var,(count))
```

### 互斥锁实现
对于既不支持`__atomic`，也不支持`__sync`的系统，那么将使用*pthread*系列结构中的互斥锁，
来实现对计数器操作的原子性保证，以`atomicIncr`为例，其实现方式为：
```c
#define atomicIncr(var,count) do { \
    pthread_mutex_lock(&var ## _mutex); \
    var += (count); \
    pthread_mutex_unlock(&var ## _mutex); \
} while(0)
```

## 原子操作用法

### Atomic实现与Sync实现
对于以这两种方式实现的原子计数器，我们只需要声明一个计数器变量，
便可以执行对应的原子操作，例如：
```c
    long myvar;
    atomicSet(myvar,12345);
```

### 互斥锁实现
对于以这种方式实现的原子计数器，通过代码分析，我们知道，这个计数器需要配合一个`pthread_mutex_t`的互斥锁一起使用，
那么其使用方法为：
```c
    long myvar;
    pthread_mutex_t myvar_mutex;
    atomicSet(myvar,12345);
```
这里需要注意，配合计数器一起声明的互斥锁的变量名，必须使用计数器变量名加上`_mutex`后缀的形式。