# KCP

关于 KCP 的源码剖析。

## test

KCP 提供的一个测试用例。

### test.h

首先封装了一些跨平台的函数。

```cpp
// 获取系统时间的秒和微秒部分
static inline void itimeofday(long *sec, long *usec);

// 获取毫秒级时间戳
static inline IINT64 iclock64();

// 获取毫秒级时间戳的32位形式
static inline IUINT32 iclock();

// 将进程挂起一段时间
static inline void isleep(unsigned long millisecond);
```

然后封装了一些类。

DelayPacket 类是带延迟的数据包，主要用来缓存数据，当然它也提供了 `ts` 和 `setts` 方法，用以获取和设置延迟时间，其中 ts 是 timestamp 的缩写。

Random 类用于产生均匀分布的随机数，这里暂时不展开描述。

LatencySimulator 类另外单独描述。

#### LatencySimulator

LatencySimulator 类是网络延迟模拟器，这里主要关注它的构造函数、send 和 recv 共3个函数。

首先是它的构造函数，即类的初始化。

```cpp
// 往返丢包率默认为10%，RTT默认为[60, 125)ms
LatencySimulator(int lostrate = 10, int rttmin = 60, int rttmax = 125, int nmax = 1000)
    : r12(100), r21(100) {
    current = iclock();
    this->lostrate = lostrate / 2; // lostrate(lost rate) 是往返丢包率，单程要除以2
    this->rttmin = rttmin / 2;
    this->rttmax = rttmax / 2;
    tx1 = tx2 = 0;
}
```

### test.cpp

## src
