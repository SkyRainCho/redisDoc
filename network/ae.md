# Redis中一个基础的事件驱动库

## 事件驱动库的基础定义
首先在*Redis*中注册了四种时间类型:
|事件|描述|
|---|----|
|`#define AE_NONE 0`|没有事件被注册|
|`#define AE_READABLE 1`|当描述符可读时触发|
|`#define AE_WRITABLE 2`|当描述符可写时触发|
|`#define AE_BARRIER 4`|如果在同一个事件循环迭代中，如果有读事件触发，那么即使可写也不触发该时间|

对于事件循环的基本数据类型
```c
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* 用于定义最多可以被追踪的描述符 */
    long long timeEventNextId;
    time_t lastTime;     /* 用于探测系统时钟偏移 */
    aeFileEvent *events; /* 已被注册的事件 */
    aeFiredEvent *fired; /* 已经触发的事件 */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* 用于Poll的API调用的指定数据 */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```