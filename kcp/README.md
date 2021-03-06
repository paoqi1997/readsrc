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
    IUINT32 nrcv_que, nsnd_que; // 接收队列/发送队列
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
