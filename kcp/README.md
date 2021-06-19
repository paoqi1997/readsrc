# KCP

关于 [KCP](https://github.com/skywind3000/kcp) 的源码剖析。

## [ikcp.c](https://github.com/skywind3000/kcp/blob/master/ikcp.c)

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
    char data[1];     // 数据内容
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
    // snd_una:
    // snd_nxt:
    // rcv_nxt:
    IUINT32 snd_una, snd_nxt, rcv_nxt;
    //  ts_recent:
    // ts_lastack:
    //   ssthresh:
    IUINT32 ts_recent, ts_lastack, ssthresh;
    IINT32 rx_rttval, rx_srtt, rx_rto, rx_minrto;
    IUINT32 snd_wnd, rcv_wnd, rmt_wnd, cwnd, probe;
    IUINT32 current, interval, ts_flush, xmit;
    IUINT32 nrcv_buf, nsnd_buf;
    IUINT32 nrcv_que, nsnd_que;
    IUINT32 nodelay, updated;
    IUINT32 ts_probe, probe_wait;
    IUINT32 dead_link, incr;
    struct IQUEUEHEAD snd_queue; // 发送队列
    struct IQUEUEHEAD rcv_queue; // 接收队列
    struct IQUEUEHEAD snd_buf;   // 发送缓冲区
    struct IQUEUEHEAD rcv_buf;   // 接收缓冲区
    IUINT32 *acklist;
    IUINT32 ackcount;
    IUINT32 ackblock;
    void *user;
    char *buffer;
    int fastresend;
    int fastlimit;
    int nocwnd, stream;
    int logmask;
    int (*output)(const char *buf, int len, struct IKCPCB *kcp, void *user);
    void (*writelog)(const char *log, struct IKCPCB *kcp, void *user);
};

typedef struct IKCPCB ikcpcb;
```

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
