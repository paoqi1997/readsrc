# test

关于 KCP 提供的一个例子的源码剖析。

## test.cpp

首先是 udp_output 函数，这个是给 ikcpcb 对象准备的，作为 KCP 的下层协议输出函数，KCP 需要发送数据的时候会用到它。

其调用栈如下所示：

ikcp_update -> ikcp_flush -> ikcp_output -> kcp->output

这里 union 的使用比较巧妙，先以 union 的 void* 形式接收指针，再以 union 的 int 形式传递 peer。

```cpp
LatencySimulator *vnet;

int udp_output(const char *buf, int len, ikcpcb *kcp, void *user) {
    union { int id; void *ptr; } parameter;
    parameter.ptr = user;
    vnet->send(parameter.id, buf, len);
    return 0;
}
```

test 这里是有3种模式，主要体现在 ikcp_nodelay 设置的不同。

再把目光转向 while (1)，首先调用 ikcp_update 更新 KCP 状态，然后 kcp1 调用 ikcp_send 发数据，
vnet 的1端收到数据后会调用 ikcp_input 将数据交给 kcp2，0端收到数据后也会调用 ikcp_input 将数据交给 kcp1。

如果 kcp2 通过 ikcp_recv 收到数据就调用 ikcp_send 发回，而如果 kcp1 收到数据就打印数据并计算 RTT，next 自增一次。

当 next 超过1000后跳出循环，随后释放 KCP 对象并进行流量统计。

```
#0# ikcp_send(kcp1, ---> vnet->recv(1, ---> ikcp_input(kcp2, ---> ikcp_recv(kcp2, #1#
#0# ikcp_recv(kcp1, <--- ikcp_input(kcp1, <--- vnet->recv(0, <--- ikcp_send(kcp2, #1#
```

## test.h

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

### LatencySimulator

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
    this->nmax = nmax;
    tx1 = tx2 = 0;
}
```

然后是 send 函数。

```cpp
void send(int peer, const void *data, int size) {
    if (peer == 0) {
        ++tx1;

        if (r12.random() < lostrate) { // 丢包
            return;
        }
        if (p12.size() >= nmax) {      // 对端已满
            return;
        }
    } else {
        ++tx2;

        if (r21.random() < lostrate) {
            return;
        }
        if (p21.size() >= nmax) {
            return;
        }
    }

    DelayPacket *pkt = new DelayPacket(size, data);
    current = iclock();

    IUINT32 delay = rttmin;
    if (rttmax > rttmin) {
        delay += rand() % (rttmax - rttmin);
    }

    // 加上延迟时间
    pkt->setts(current + delay);

    if (peer == 0) {
        p12.push_back(pkt);
    } else {
        p21.push_back(pkt);
    }
}
```

如果 LatencySimulator::send 认为 peer 为0就是 `由0发送，由1接收` 的话，那么应该有以下模型：

```
0 p1 -----> p2 1 (p12)
0 p1 <----- p2 1 (p21)
```

最后是 recv 函数。

```cpp
int recv(int peer, void *data, int maxsize) {
    DelayTunnel::iterator it;
    if (peer == 0) {
        it = p21.begin();
        if (p21.size() == 0) return -1;
    } else {
        it = p12.begin();
        if (p12.size() == 0) return -1;
    }

    DelayPacket *pkt = *it;
    current = iclock();

    if (current < pkt->ts()) {   // 数据包尚未到达
        return -2;
    }
    if (maxsize < pkt->size()) { // 给定空间不足以容纳数据
        return -3;
    }

    if (peer == 0) {
        p21.erase(it);
    } else {
        p12.erase(it);
    }

    maxsize = pkt->size();
    memcpy(data, pkt->ptr(), maxsize);
    delete pkt;

    return maxsize;
}
```
