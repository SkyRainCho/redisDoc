# Redis中的后台I/O服务

虽然*Redis*宣称是采用单线程以及I/O多路复用的方式，来实现高并发的数据存取，但这不意味着*Redis*服务器的整个进程是使用单线程模式运行的。*Redis*所宣传单线程是指在访问核心数据逻辑上是采用单线程模式进行的，这样可以最大限度避免由于锁争用而导致的性能降低。实际上*Redis*服务器进程仍然是采用多线程模式运行的，除了处理核心逻辑的主线程之外，*Redis*还启动了若干个线程在后台运行来处理非核心逻辑，这样可以避免这些非核心逻辑长期占用主线程。

在头文件*src/bio.h*以及源文件*src/bio.c*中定义以及实现多线程后台异步处理的相关逻辑，而在*src/bio.h*头文件之中定义了需要以后台线程方式处理的任务类型：
```c
#define BIO_CLOSE_FILE    0
#define BIO_AOF_FSYNC     1
#define BIO_LAZY_FREE     2
#define BIO_NUM_OPS       3
```
这些任务类型分别是：
1. `BIO_CLOSE_FILE`，用于异步地调用`close`系统调用来关闭文件描述符。
2. `BIO_AOF_FSYNC`，用于进行**AOF**数据持久化。
3. `BIO_LAZY_FREE`，用于处理前面我们介绍的对于数据的**惰性删除**。

*Redis*为每个任务类型启动了一个线程，用于处理其对应的业务逻辑。同时也为每一个线程创建了一个任务队列，用以实现一个**生产者-消费者**模式，主线程作为**生产者**将需要处理的非核心逻辑也任务的形式插入到任务队列之中，而后台线程则作为**消费者**从任务队列之中取出任务以异步地处理相关的逻辑。

在*src/bio.c*文件中定义了后台任务的数据结构：
```c
struct bio_job
{
    time_t time;
    void *arg1;
    void *arg2;
    void *arg3;
};
```
这其中：
1. `bio_job.time`，这个字段用于表示任务被创建的时间戳。
2. `bio_job.arg`，则是指定了这个任务的参数。

*Redis*使用在*src/adlist.h*中定义的双端链表存储由`bio_job`组成的任务，同时使用*pthread*中的互斥锁以及条件变量来定义了一个任务队列：
```c
static pthread_mutex_t bio_mutex[BIO_NUM_OPS];
static pthread_cond_t bio_newjob_cond[BIO_NUM_OPS];
static list *bio_jobs[BIO_NUM_OPS];
```

同时*Redis*还定义了一个数组用于记录不同任务类型的任务队列中待执行任务的数量：
```c
static long long bio_pending[BIO_NUM_OPS];
```

最后*Redis*还创建了一个线程池，线程池中的每一个线程分别用于处理一个任务类型所对应的任务队列：
```c
static pthread_t bio_threads[BIO_NUM_OPS];
```



*Redis*在完成对线程池以及任务队列的声明和定义之后，还定义后台操作的相关接口，这些接口的实现逻辑普遍比较简单，但是通过阅读代码可以发现*Redis*在*src/bio.c*源代码中给出关于线程池与任务队列以及**生产者-消费者**模式的一个经典实现范式。

## 任务队列与线程池的初始化

通过`bioInit`这个函数接口对任务队列进行初始化，同时启动线程池。

```c
void bioInit(void){
    
}
```



## 后台线程的函数入口

通过上面`bioInit`这个接口，我们可以发现，*Redis*是使用`bioProcessBackgroundJobs`这个函数作为入口，通过`pthread_create`系统调用来启动线程池中的线程的，可以说这个函数便是整个后台I/O服务的核心，也是在**生产者-消费者**模式中扮演**消费者**的角色。

## 添加后台任务接口

## 后台线程的终止




***
![公众号二维码](https://machiavelli-1301806039.cos.ap-beijing.myqcloud.com/qrcode_for_gh_836beef2355a_344.jpg)

喜欢的同学可以扫描二维码，关注我的微信公众号，*马基雅维利incoding*