# KCP

关于 [KCP](https://sourcegraph.com/github.com/skywind3000/kcp) 的源码剖析。

## [ikcp.c](https://github.com/skywind3000/kcp/blob/master/ikcp.c)

首先定义了一大堆常量，如 IKCP_RTO_MIN、IKCP_CMD_PUSH 等，接下来会用到它们。

然后是 ikcp_encodexxx 和 ikcp_decodexxx 一系列函数，如 ikcp_encode8u 和 ikcp_decode8u，用于消息编解码。

接着定义了一些工具函数，如 \_imin_、\_imax_、\_ibound_ 等。

接下来提供了 ikcp_malloc_hook 和 ikcp_free_hook 两个函数指针变量，可以通过 ikcp_allocator 设置它们。

调用 ikcp_malloc 时如果有设置 ikcp_malloc_hook 就调用它，没有就调用 malloc；
调用 ikcp_free 时如果有设置 ikcp_free_hook 就调用它，没有就调用 free。

同时还提供 ikcp_segment_new 和 ikcp_segment_delete 函数，它们分别用于创建和释放 IKCPSEG 对象。

```c
// 注意这个size
static IKCPSEG* ikcp_segment_new(ikcpcb *kcp, int size) {
    return (IKCPSEG*) ikcp_malloc(sizeof(IKCPSEG) + size);
}
```

再然后是 ikcp_log 和 ikcp_canlog 两个函数，它们都受 kcp->logmask 的约束，其中 ikcp_log 顺利的话会调用 kcp->writelog。

ikcp_output 函数首先会检查日志掩码是否符合 IKCP_LOG_OUTPUT 预设的条件，符合的话就调用 ikcp_log，然后调用 kcp->output。

ikcp_qprint 函数打印给定 IQUEUEHEAD 对象的各个 IKCPSEG 的信息，包括 seg->sn 和 seg->ts。

ikcp_create 和 ikcp_release 函数分别用于创建和释放 ikcpcb 对象，其中 ikcp_create 中某些成员的初始化值得注意。

ikcp_setoutput 函数用于设置 ikcpcb 对象的 output 成员。

### 调用关系

如下所示：

```
// 初始化
ikcp_create -> ikcpcb
ikcpcb->output = udp_output(sendto)

// 发端
ikcp_send(ikcpcb, ...)
...
ikcp_update(ikcpcb, ...) -> ikcp_flush -> ikcp_output(sendto) -> 对端

// 收端
对端 -> recvfrom -> ikcp_input
...
ikcp_recv(ikcpcb, ...) # 有事没事都可以调用，rcv_queue 有数据才会写到 buffer
```

### ikcp_send

如果开启了流传输模式，就将 buffer 中的数据尽可能地追加到最后一个报文段中，使其 DATA 段的长度等于 MSS。

```c
if (kcp->stream != 0) {
    if (!iqueue_is_empty(&kcp->snd_queue)) {
        IKCPSEG *old = iqueue_entry(kcp->snd_queue.prev, IKCPSEG, node);
        if (old->len < kcp->mss) {
            ...
        }
    }
    if (len <= 0) {
        return sent;
    }
}
```

计算本次发送的数据需要分片的数量。

```c
if (len <= (int)kcp->mss) count = 1;
else count = (len + kcp->mss - 1) / kcp->mss;
```

本次分片数量超过限制。

```c
if (count >= (int)IKCP_WND_RCV) {
    if (kcp->stream != 0 && sent > 0)
        return sent;
    return -2;
}
```

最小分片数量为1。

```c
if (count == 0) count = 1;
```

对数据进行分片并将对应创建的报文段对象放入 snd_queue。

```c
for (i = 0; i < count; i++) {
    ...
}

return sent;
```

### ikcp_update

驱动 ikcp_flush，但两次调用之间的时间间隔不应小于 interval，默认为100ms。

### ikcp_flush

首先发送 ACK 列表中所有的 ACK。

```c
...
IKCPSEG seg;
...
seg.cmd = IKCP_CMD_ACK;
...
count = kcp->ackcount;
for (i = 0; i < count; i++) {
    size = (int)(ptr - buffer);
    // IKCP_OVERHEAD 为24，是报文头的大小，ACK 报文不需要 DATA 段
    if (size + (int)IKCP_OVERHEAD > (int)kcp->mtu) {
        ikcp_output(kcp, buffer, size);
        ptr = buffer;
    }
    ikcp_ack_get(kcp, i, &seg.sn, &seg.ts);
    ptr = ikcp_encode_seg(ptr, &seg);
}

kcp->ackcount = 0;
```

在对端接收窗口大小为0时探测其大小。

```c
if (kcp->rmt_wnd == 0) {
    ...
} else {
    kcp->ts_probe = 0;
    kcp->probe_wait = 0;
}

if (kcp->probe & IKCP_ASK_SEND) {
    ...
}

if (kcp->probe & IKCP_ASK_TELL) {
    ...
}

kcp->probe = 0;
```

计算拥塞窗口的大小。

```c
cwnd = _imin_(kcp->snd_wnd, kcp->rmt_wnd);
if (kcp->nocwnd == 0) cwnd = _imin_(kcp->cwnd, cwnd);
```

将数据从 snd_queue 搬到 snd_buf。

```c
while (_itimediff(kcp->snd_nxt, kcp->snd_una + cwnd) < 0) {
    ...
}
```

快重传相关。

```c
resent = (kcp->fastresend > 0)? (IUINT32)kcp->fastresend : 0xffffffff;
rtomin = (kcp->nodelay == 0)? (kcp->rx_rto >> 3) : 0;
```

将 snd_buf 中符合条件的报文段都发送出去。

```c
for (p = kcp->snd_buf.next; p != &kcp->snd_buf; p = p->next) {
    IKCPSEG *segment = iqueue_entry(p, IKCPSEG, node);
    int needsend = 0;
    // 第一次发送
    if (segment->xmit == 0) {
        ...
    }
    // 触发超时重传
    else if (_itimediff(current, segment->resendts) >= 0) {
        ...
        if (kcp->nodelay == 0) {
            segment->rto += _imax_(segment->rto, (IUINT32)kcp->rx_rto);
        } else {
            IINT32 step = (kcp->nodelay < 2)? 
                ((IINT32)(segment->rto)) : kcp->rx_rto;
            // RTO *= 1.5
            segment->rto += step / 2;
        }
        segment->resendts = current + segment->rto;
        lost = 1;
    }
    // 跳过了 fastack 个报文段，触发快重传
    else if (segment->fastack >= resent) {
        if ((int)segment->xmit <= kcp->fastlimit ||
            kcp->fastlimit <= 0) {
            ...
            change++;
            ...
        }

    }

    if (needsend) {
        ...
    }
}
```

将剩余报文段也发送出去。

```c
size = (int)(ptr - buffer);
if (size > 0) {
    ikcp_output(kcp, buffer, size);
}
```

根据发生快重传和超时重传与否计算慢开始门限 ssthresh 和拥塞窗口 cwnd 的大小。

```c
// 触发快重传，进入快恢复
if (change) {
    IUINT32 inflight = kcp->snd_nxt - kcp->snd_una;
    // 将 ssthresh 置为发送窗口中报文数量的一半
    kcp->ssthresh = inflight / 2;
    if (kcp->ssthresh < IKCP_THRESH_MIN)
        kcp->ssthresh = IKCP_THRESH_MIN;
    // 将 cwnd 置为比 ssthresh 稍高的值
    kcp->cwnd = kcp->ssthresh + resent;
    kcp->incr = kcp->cwnd * kcp->mss;
}

// 有报文段因超时后仍没有得到确认而需要重传，进入慢开始
if (lost) {
    // 将 ssthresh 置为拥塞窗口大小的一半
    kcp->ssthresh = cwnd / 2;
    if (kcp->ssthresh < IKCP_THRESH_MIN)
        kcp->ssthresh = IKCP_THRESH_MIN;
    // 将 cwnd 置1
    kcp->cwnd = 1;
    kcp->incr = kcp->mss;
}

if (kcp->cwnd < 1) {
    kcp->cwnd = 1;
    kcp->incr = kcp->mss;
}
```

### ikcp_input

会不断地解析接收到的数据，直到数据不足以形成一个报头。

```c
IUINT32 prev_una = kcp->snd_una;
...

while (1) {
    IUINT32 ts, sn, len, una, conv;
    IUINT16 wnd;
    IUINT8 cmd, frg;
    IKCPSEG *seg;

    if (size < (int)IKCP_OVERHEAD) break;

    data = ikcp_decode32u(data, &conv);
    if (conv != kcp->conv) return -1;

    data = ikcp_decode8u(data, &cmd);
    ...
    data = ikcp_decode32u(data, &len);

    size -= IKCP_OVERHEAD;

    ...

    // 更新对端接收窗口大小
    kcp->rmt_wnd = wnd;

    // 将已得到确认的报文段从 snd_buf 中删除
    // 收到的报文段的 una 大于 snd_buf 中的报文段的 sn
    ikcp_parse_una(kcp, una);

    // 向右移动 snd_una
    ikcp_shrink_buf(kcp);

    // 处理报文段
    if (cmd == IKCP_CMD_ACK) {
        ...
    }
    else if (cmd == IKCP_CMD_PUSH) {
        ...
    }
    else if (cmd == IKCP_CMD_WASK) {
        ...
    }
    else if (cmd == IKCP_CMD_WINS) {
        ...
    }
    else {
        return -3;
    }

    data += len;
    size -= len;
}
```

将 snd_buf 中 sn 小于 maxack 且未确认送达的报文段的 fastack +1，maxack 是本轮接收的 ACK 报文中最新的序号。

```c
if (flag != 0) {
    ikcp_parse_fastack(kcp, maxack, latest_ts);
}
```

计算拥塞窗口 cwnd 的大小。

```c
// una 向右移动了
if (_itimediff(kcp->snd_una, prev_una) > 0) {
    if (kcp->cwnd < kcp->rmt_wnd) {
        IUINT32 mss = kcp->mss;
        // 慢开始阶段，每收到一个 ACK，cwnd 就相应+1
        // Round 1: cwnd 为1，发送1个报文段并收到1个确认，cwnd + 1 = 2
        // Round 2: cwnd 为2，发送2个报文段并收到2个确认，cwnd + 2 = 4
        // Round 3: cwnd 为4，发送4个报文段并收到4个确认，cwnd + 4 = 8
        // ...
        if (kcp->cwnd < kcp->ssthresh) {
            kcp->cwnd++;
            kcp->incr += mss;
        }
        // 拥塞避免阶段
        else {
            if (kcp->incr < mss) kcp->incr = mss;
            kcp->incr += (mss * mss) / kcp->incr + (mss / 16);
            // incr 的增加值超过一个 mss 的大小
            if ((kcp->cwnd + 1) * mss <= kcp->incr) {
            #if 1
                kcp->cwnd = (kcp->incr + mss - 1) / ((mss > 0)? mss : 1);
            #else
                kcp->cwnd++;
            #endif
            }
        }
        if (kcp->cwnd > kcp->rmt_wnd) {
            kcp->cwnd = kcp->rmt_wnd;
            kcp->incr = kcp->rmt_wnd * mss;
        }
    }
}

return 0;
```

### ikcp_recv

获取 rcv_queue 中第一个包的大小，它可能由一个或多个分片组成。

```c
...

peeksize = ikcp_peeksize(kcp);

// 不足以形成一个包
if (peeksize < 0)
    return -2;

// 接收缓冲区长度不足
if (peeksize > len)
    return -3;
```

若接收队列的大小大于等于接收窗口的大小，有点拥塞，看看收完一波包后可不可以执行快恢复。

```c
if (kcp->nrcv_que >= kcp->rcv_wnd)
    recover = 1;
```

将 rcv_queue 中的数据写到 buffer。

```c
for (len = 0, p = kcp->rcv_queue.next; p != &kcp->rcv_queue; ) {
    ...

    // 0表示只有一个分片或者最后一个分片
    if (fragment == 0) 
        break;
}

assert(len == peeksize);
```

将数据从 rcv_buf 搬到 rcv_queue。

```c
while (! iqueue_is_empty(&kcp->rcv_buf)) {
    ...
}
```

收完一波包后，接收队列的大小已经小于接收窗口的大小，可以执行快恢复（虽然最后的结果是 do nothing）。

```c
// fast recover
if (kcp->nrcv_que < kcp->rcv_wnd && recover) {
    kcp->probe |= IKCP_ASK_TELL;
}

return len;
```

## [ikcp.h](https://github.com/skywind3000/kcp/blob/master/ikcp.h)

首先对一些基本类型进行平台抽象，比如 test.h 用到的 IINT64 和 IUINT32。

然后提供一系列宏以封装队列操作，其前缀均为 iqueue_，比如 iqueue_init、iqueue_add、iqueue_add_tail 等。

再一个是提供一些字节序和对齐相关的宏。

接着是核心结构体之一：IKCPSEG。

```c
// 报文
struct IKCPSEG
{
    struct IQUEUEHEAD node;
    IUINT32 conv;     // 会话编号，双方相互传输时需保证 conv 一致
    IUINT32 cmd;      // 指令，如 IKCP_CMD_PUSH、IKCP_CMD_ACK 等
    IUINT32 frg;      // 分片数，用于标识是 KCP 包的第几个分片
    IUINT32 wnd;      // 窗口大小，发送方的发送窗口不能超过接收方的窗口大小
    IUINT32 ts;       // 发送时的时间戳
    IUINT32 sn;       // 序号
    IUINT32 una;      // 未确认序号，在未丢包的情况下，如果收到 sn=10 的包的话，una 为11
    IUINT32 len;      // 数据长度
    IUINT32 resendts; // 下次超时重传的时间戳
    IUINT32 rto;      // 超时重传的等待时间
    IUINT32 fastack;  // 分片被跳过的次数，超过既定次数无需等待超时，直接重传，比如收到[1,3,4,5]，收到3表示2被跳过1次，收到4表示2被跳过2次
    IUINT32 xmit;     // 重传次数，每重传1次+1
    char data[1];     // 数据内容（如果是data[0]/data[]就是柔性数组）
};
```

另一个是 IKCPCB。

```c
// 控制块
struct IKCPCB
{
    //  conv: 会话编号
    //   mtu: 最大传输单元（Maximum Transmission Unit, MTU）
    //   mss: 最大报文段长度（Maximum Segment Size, MSS）
    // state: 状态（0/-1）
    IUINT32 conv, mtu, mss, state;
    // snd_una: 第一个未确认的发送包序号
    // snd_nxt: 下一个待使用的发生包序号
    // rcv_nxt: 下一个待使用的接收包序号
    IUINT32 snd_una, snd_nxt, rcv_nxt;
    //  ts_recent: 当前时间戳
    // ts_lastack: 上次确认的时间戳
    //   ssthresh: 慢开始门限
    IUINT32 ts_recent, ts_lastack, ssthresh;
    // rx_rttval: RTT 的变化量
    //   rx_srtt: 平滑后的 RTT（Smoothed Round Trip Time）
    //    rx_rto: 由 ACK 接收延迟计算得到的重传超时时间（RTO, Retransmission Timeout）
    // rx_minrto: 最小重传超时时间
    IINT32 rx_rttval, rx_srtt, rx_rto, rx_minrto;
    // snd_wnd: 发送窗口大小
    // rcv_wnd: 接收窗口大小
    // rmt_wnd: 对端接收窗口大小
    //    cwnd: 拥塞窗口大小
    //   probe: 探测变量，如 IKCP_ASK_SEND、IKCP_ASK_TELL 等
    IUINT32 snd_wnd, rcv_wnd, rmt_wnd, cwnd, probe;
    //  current: 当前时间戳
    // interval: 内部 flush 的时间间隔
    // ts_flush: 下次 flush 的时间戳
    //     xmit: 发送 segment 的次数
    IUINT32 current, interval, ts_flush, xmit;
    IUINT32 nrcv_buf, nsnd_buf; // 接收缓冲区大小/发送缓冲区大小
    IUINT32 nrcv_que, nsnd_que; // 接收队列大小/发送队列大小
    // nodelay: 是否启用 nodelay 模式
    // updated: 是否调用过 ikcp_update 函数
    IUINT32 nodelay, updated;
    //   ts_probe: 下次探测的时间戳
    // probe_wait: 探测前需要等待的时间
    IUINT32 ts_probe, probe_wait;
    // dead_link: IKCP_DEADLINK
    //      incr: 可发送的最大数据量
    IUINT32 dead_link, incr;
    struct IQUEUEHEAD snd_queue; // 发送队列
    struct IQUEUEHEAD rcv_queue; // 接收队列
    struct IQUEUEHEAD snd_buf;   // 发送缓冲区
    struct IQUEUEHEAD rcv_buf;   // 接收缓冲区
    IUINT32 *acklist; // 待发送的 ACK 列表
    IUINT32 ackcount; // acklist 中 ACK 的数量
    IUINT32 ackblock; // acklist 最大可容纳的 ACK 数量，为2的倍数，一开始为8
    void *user;       // 用于回调的指针
    char *buffer;     // 消息存储字节流
    int fastresend;   // 触发快重传的重复 ACK 个数
    int fastlimit;    // 快重传最大次数
    // nocwnd: 是否关闭流控
    // stream: 是否采用流传输模式
    int nocwnd, stream;
    int logmask;      // 日志掩码
    int (*output)(const char *buf, int len, struct IKCPCB *kcp, void *user); // 发送回调
    void (*writelog)(const char *log, struct IKCPCB *kcp, void *user);       // 写日志回调
};

typedef struct IKCPCB ikcpcb;
```

接下来是 IKCP_LOG_OUTPUT、IKCP_LOG_INPUT 等一系列日志掩码，如果 ikcpcb 对象的 logmask 成员设置了某个日志掩码，
那么调用 ikcp_canlog 和 ikcp_log 都将受到约束。

也就是说，外部传入的 mask 和 kcp->logmask 与运算的结果不为0并且 kcp->writelog 不为 NULL，函数才不会提前 return。

最后是由 `extern "C"` 包裹的一些函数声明，如 ikcp_create、ikcp_update 等。

### iqueue 相关

iqueue_head 结构体和 iqueue_xxx 系列操作实现了双向循环链表。

```
IQUEUE_INIT(ptr): ptr -> ptr
IQUEUE_ADD(node, head): head -> head->next ---> head -> node -> head->next
IQUEUE_ADD_TAIL(node, head): head->prev -> head ---> head->prev -> node -> head
```

## TCP vs KCP

首先汇总一下相关的一些概念：

```
MTU(Maximum Transmission Unit): 最大传输单元，就是报头+MSS
MSS(Maximum Segment Size): 最大报文段长度，实际上是指数据段的长度

RTT(Round Trip Time): 往返时延，即从发送数据到收到确认所经历的时间
RTO(Retransmission Timeout): 重传超时时间，意思是从发送数据的时刻开始算起，超过这个时间仍未收到确认便需要重传

cwnd(congestion window): 拥塞窗口
ssthresh(slow start threshold): 慢开始门限

ARQ(Automatic Repeat-reQuest): 自动重传请求
UNA(Unacknowledged): 第一个尚未收到确认的序号，这意味着在它之前的报文段都已经被正确接收
ACK(Acknowledgment): 确认号，即期望收到的下一个报文段的序号
```

主要关注 ARQ 协议、滑动窗口和拥塞控制三个方面。

ARQ 协议

1. 发生超时重传后，TCP 的 RTO 翻倍，而 KCP 的是乘以1.5，这一点通过 ikcp_nodelay 的 nodelay 来控制。

2. TCP 重传用的是 GBN 协议，而 KCP 是选择性重传。

3. KCP 支持快重传，这一点通过 ikcp_nodelay 的 resend 来控制。

4. TCP 有 Delayed ACK 机制，而 KCP 可以调节 ACK 延迟，这一点应该是通过 ikcp_update_ack 来实现。

5. 按照韦神的说法，ARQ 响应有两种方式：UNA 和 ACK。TCP 用的是连续 ARQ 协议，是对按序到达的最后一个分组发送确认，应该是 UNA 模式；而 ACK 模式更贴合停止等待 ARQ 协议的描述一些。KCP 是将 UNA 和 ACK 结合在一起使用。

滑动窗口

1. 两者之间没什么区别。

拥塞控制

1. KCP 方面可以通过 ikcp_nodelay 的 nc 来控制流控的开启/关闭，若将其设置为1，cwnd 将只取决于发送窗口大小和对端接收窗口大小的最小值，不再受 kcp->cwnd 的影响。当然 kcp->cwnd 还是照常调节，和 TCP 一样，引入了慢开始、快恢复等机制。

## TPs

+ [为什么需要 RUDP 协议](https://wmf.im/p/%E4%B8%BA%E4%BB%80%E4%B9%88%E9%9C%80%E8%A6%81-rudp-%E5%8D%8F%E8%AE%AE/)

+ [详解 KCP 协议的原理和实现](https://luyuhuang.tech/2020/12/09/kcp.html)

+ [unix网络编程3.2——UDP（二）UDP可靠性传输1——KCP协议（上）](https://www.cnblogs.com/kongweisi/p/17008361.html)

+ [unix网络编程3.3——UDP（三）UDP可靠性传输2——KCP协议（下）](https://www.cnblogs.com/kongweisi/p/17016446.html)

+ [（续）深入理解KCP库的核心函数和工作流程](https://blog.csdn.net/weixin_45634782/article/details/137439361)

+ [KCP协议浅析](https://www.cnblogs.com/hggzhang/p/17235879.html)

+ [ARQ模型响应有两种](https://avmedia.0voice.com/?id=63807)

## 附录

### 1. KCP 包结构

```
KCP has only one kind of segment: both the data and control messages are
encoded into the same structure and share the same header.

The KCP packet (aka. segment) structure is as following:

0               4   5   6       8 (BYTE)
+---------------+---+---+-------+
|     conv      |cmd|frg|  wnd  |
+---------------+---+---+-------+   8
|     ts        |     sn        |
+---------------+---------------+  16
|     una       |     len       |
+---------------+---------------+  24
|                               |
|        DATA (optional)        |
|                               |
+-------------------------------+
```
