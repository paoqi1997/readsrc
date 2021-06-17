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
    IUINT32 cmd;
    IUINT32 frg;
    IUINT32 wnd;
    IUINT32 ts;
    IUINT32 sn;
    IUINT32 una;
    IUINT32 len;
    IUINT32 resendts;
    IUINT32 rto;
    IUINT32 fastack;
    IUINT32 xmit;
    char data[1];
};
```

另一个是 IKCPCB。

```c
// 控制块
struct IKCPCB
{
    IUINT32 conv, mtu, mss, state;
    IUINT32 snd_una, snd_nxt, rcv_nxt;
    IUINT32 ts_recent, ts_lastack, ssthresh;
    IINT32 rx_rttval, rx_srtt, rx_rto, rx_minrto;
    IUINT32 snd_wnd, rcv_wnd, rmt_wnd, cwnd, probe;
    IUINT32 current, interval, ts_flush, xmit;
    IUINT32 nrcv_buf, nsnd_buf;
    IUINT32 nrcv_que, nsnd_que;
    IUINT32 nodelay, updated;
    IUINT32 ts_probe, probe_wait;
    IUINT32 dead_link, incr;
    struct IQUEUEHEAD snd_queue;
    struct IQUEUEHEAD rcv_queue;
    struct IQUEUEHEAD snd_buf;
    struct IQUEUEHEAD rcv_buf;
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
