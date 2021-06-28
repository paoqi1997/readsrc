# coroutine

关于 [coroutine](https://sourcegraph.com/github.com/cloudwu/coroutine) 的源码剖析。

## [main.c](https://github.com/cloudwu/coroutine/blob/master/main.c)

首先通过 coroutine_open 创建一个 schedule 对象，然后通过 coroutine_new 创建两个协程对象，创建时会给它们传入函数指针及对应的参数，
最后返回对应协程的 ID。

接下来调用 coroutine_resume 恢复协程，进入 foo 调用后执行一些逻辑，随即调用 coroutine_yield 返回 main，如此往复，
期间通过 coroutine_status 判断 foo 是否已经执行完毕。

最后调用 coroutine_close 回收 schedule 对象。

## [coroutine.c](https://github.com/cloudwu/coroutine/blob/master/coroutine.c)

## [coroutine.h](https://github.com/cloudwu/coroutine/blob/master/coroutine.h)

主要就是一些宏、结构体还有函数的声明。
