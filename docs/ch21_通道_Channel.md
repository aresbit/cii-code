# 第 21 章  通道 (Channels)

> 原著: David R. Hanson, *C Interfaces and Implementations*, Chapter 20 (Threads, Section 20.1.3 及实现)
> 翻译与扩展: 基于 CII 源代码 `chan.h` / `chan.c` / `sem.h` 重述

---

## 21.1 引言: CSP 风格的同步通信

### 21.1.1 历史渊源

通道 (Channel) 的设计直接源自 **CSP** (Communicating Sequential Processes, Hoare 1978)。Tony Hoare 在 1978 年发表的经典论文中提出: 并发计算的基本单元不是共享内存, 而是通过**无缓冲的同步通道**进行消息传递的进程。核心思想是:

> "不要通过共享内存来通信, 而要通过通信来共享内存。" (Do not communicate by sharing memory; instead, share memory by communicating.)

这条原则后来成为 Go 语言并发模型的第一信条。在 CII 中, Hanson 将 CSP 通道缩小到线程间通信的级别: 一个 `Chan_T` 是两个线程(或协程)之间的**双向同步会合点 (rendezvous point)**。

本书在 Further Reading 中指出:

> Channels are based on CSP — communicating sequential processes (Hoare 1978). Threads and channels also appear in Newsqueak, an applicative concurrent language. (p.464)

Newsqueak 是 Rob Pike 在贝尔实验室设计的并发语言, 其通道概念后来直接催生了 Go 语言的 `chan` 类型。Hanson 的 CII 通道精简到了 Newsqueak 模型的最小子集: **每次只允许一个发送方和一个接收方同步交换一条消息**。

### 21.1.2 核心语义

Channel 的语义只有两条规则:

1. **Rendezvous (会合)**: `Chan_send` 和 `Chan_receive` 必须成对出现。发送方阻塞直到接收方到达, 接收方阻塞直到发送方到达。二者在同一个 channel 上相遇时, 数据被拷贝, 然后双方同时返回。

2. **零容量**: 同步通道没有缓冲区。数据直接从发送方的内存拷贝到接收方的内存, 中间不在通道内暂存。这使得 `Chan_T` 结构体可以做到极简——只需要信号量, 不需要环形缓冲区。

---

## 21.2 接口: `chan.h` 逐行解析

```c
/* $Id$ */
#ifndef CHAN_INCLUDED
#define CHAN_INCLUDED
#define T Chan_T
typedef struct T *T;
```

`Chan_T` 是**不透明指针类型 (opaque pointer)**。客户端代码永远看不到 `struct T` 的内部结构, 只能持有 `Chan_T` 句柄。这是 CII 全书的设计惯例——接口暴露句柄, 实现隐藏细节。

```c
extern T   Chan_new    (void);
```

`Chan_new` 分配并初始化一个新的同步通道, 返回其句柄。可引发 `Mem_Failed` (当 `NEW(c)` 内存分配失败时)。

```c
extern int Chan_send   (T c, const void *ptr, int size);
```

`Chan_send` 将 `ptr` 指向的 `size` 字节数据通过通道 `c` 发送。调用线程阻塞直至另一线程在同一通道上调用 `Chan_receive`。

- **返回值**: 接收方实际接受的字节数 (接收方可以设定 `size` 的上限)。
- **0 字节发送**: 当 `size == 0` 时, `Chan_send` 执行纯同步——发送方和接收方在通道上握手, 但不传输任何数据。这是**信号机制**的关键用法。
- **Alerts**: 如果调用线程已设置了 alert-pending 标志 (`t->alerted`), `Chan_send` 立即引发 `Thread_Alerted`。如果线程在阻塞期间被 alert, 它将停止等待并引发异常。
- **检查运行时错误**: `c == NULL || ptr == NULL || size < 0`。

```c
extern int Chan_receive(T c,       void *ptr, int size);
```

`Chan_receive` 从通道 `c` 接收最多 `size` 字节的数据到 `ptr` 指向的缓冲区。调用线程阻塞直至另一线程在同一通道上调用 `Chan_send`。

- **返回值**: 实际接收的字节数 `n`。如果 `n < size` 说明发送方提供了少于 `size` 字节的数据; 如果 `n == 0` 说明是纯信号握手。
- **截断行为**: 如果发送方提供的字节数超过 `size`, 超出的部分被**静默丢弃**, 但返回值反映的是截断后的 `size`。
- **Alerts**: 与 `Chan_send` 相同。
- **检查运行时错误**: 同 `Chan_send`。

```c
extern void Chan_free  (T *c);
```

`Chan_free` 释放通道 `c` 占用的所有资源。如果 `c` 指向的通道仍有线程阻塞在其上, 行为是未定义的——调用者必须在释放前确保无活跃通信。这是一个**非标准扩展**: CII 原始接口未提供释放函数 (通道通常在进程退出时由操作系统回收), 但在长期运行的服务中, 显式释放通道是必要的。

```c
extern int Chan_trysend  (T c, const void *ptr, int size);
```

`Chan_trysend` 是非阻塞版本的 `Chan_send`。如果接收方已在通道上等待, 它立即发送数据并返回 `size`; 如果接收方尚未到达, 它**立即返回 0**, 不阻塞调用线程。内部实现依赖信号量的非阻塞 P 操作: `Sem_trywait` 若计数为 0 则立即返回而非阻塞。

```c
extern int Chan_tryreceive(T c,       void *ptr, int size);
```

`Chan_tryreceive` 是非阻塞版本的 `Chan_receive`。如果发送方已准备就绪, 它立即接收数据并返回实际字节数; 如果发送方尚未到达, 它**立即返回 0**。这允许接收方在不阻塞的情况下轮询通道状态, 可配合定时器实现超时逻辑。

```c
#undef T
#endif
```

---

## 21.3 实现: `chan.c` 逐行解析

### 21.3.1 Chan_T 结构体

```c
static char rcsid[] = "$Id$";
#include <string.h>
#include "assert.h"
#include "mem.h"
#include "chan.h"
#include "sem.h"
#define T Chan_T
struct T {
    const void *ptr;   // 指向发送方消息缓冲区的指针
    int *size;         // 指向发送方字节数的指针 (接收方可修改)
    Sem_T send, recv, sync; // 三个信号量协调握手协议
};
```

一个通道只需要 **五个字段**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `ptr` | `const void *` | 指向发送方的数据缓冲区 (发送方在发送前设置) |
| `size` | `int *` | 指向发送方的字节数变量; 注意是**指针**, 接收方可修改其值, 从而将实际接收字节数通知发送方 |
| `send` | `Sem_T` | 控制发送方对 `ptr`/`size` 的写入权限; 初值=1, 表示"可发送" |
| `recv` | `Sem_T` | 通知接收方 `ptr`/`size` 已就绪; 初值=0, 表示"尚无数据" |
| `sync` | `Sem_T` | 通知发送方接收方已完成拷贝; 初值=0, 表示"拷贝未完成" |

### 21.3.2 Chan_new — 创建通道

```c
T Chan_new(void) {
    T c;
    NEW(c);                    // Mem接口: ALLOC(sizeof *c) 并赋值
    Sem_init(&c->send, 1);     // 发送方初始获得许可
    Sem_init(&c->recv, 0);     // 接收方初始阻塞
    Sem_init(&c->sync, 0);     // 发送方初始阻塞 (等待对方拷贝完成)
    return c;
}
```

初始信号量状态:

```
send = 1   (发送方可以进入临界区, 填充 ptr/size)
recv = 0   (接收方被阻塞, 直到发送方填好数据)
sync = 0   (发送方被阻塞, 直到接收方拷贝完成)
```

这个信号量三元组的初始值定义了通道的**握手协议**。理解这三个信号量的状态转换, 就理解了整个同步通道的实现。

### 21.3.3 Chan_send — 发送操作

```c
int Chan_send(Chan_T c, const void *ptr, int size) {
    assert(c);
    assert(ptr);
    assert(size >= 0);

    Sem_wait(&c->send);        // (1) 等待发送许可 (send从1变0)
    c->ptr = ptr;              // (2) 设置消息指针
    c->size = &size;           // (3) 设置字节数指针 (指向局部变量size!)
    Sem_signal(&c->recv);      // (4) 唤醒接收方 (recv从0变1)
    Sem_wait(&c->sync);        // (5) 等待接收方完成拷贝 (sync从0变1后通过)
    return size;               // (6) 返回接收方接受的字节数
}
```

**关键细节: `c->size = &size`**

注意 `c->size` 存储的是**指向发送方局部变量 `size` 的指针**。这意味着:
- `Chan_receive` 可以**修改**发送方栈上的 `size` 变量, 从而将实际接收的字节数反馈给发送方。
- 这是 CII 中一个精妙的指针技巧: 利用 C 的指针语义, 用一个字段实现了双向数据流。

**状态转换图 (发送方视角)**:

```
初始: send=1, recv=0, sync=0

步骤1: Sem_wait(&send)   → send=0, recv=0, sync=0  (占用发送权)
步骤2-3: 设置 ptr 和 size                           (临界区)
步骤4: Sem_signal(&recv)  → send=0, recv=1, sync=0  (唤醒接收方)
步骤5: Sem_wait(&sync)    → 阻塞, 等待接收方完成     (交出CPU)
      (接收方运行中...)
      Sem_signal(&sync)    → send=0, recv=0, sync=1  (接收方完成拷贝)
      发送方继续            → send=0, recv=0, sync=0
                             (sync的P操作将其归零, 发送方返回)
```

注意: `send` 在此刻仍然是 0。`send` 的恢复 1 是由**接收方**在 `Chan_receive` 末尾执行的 `Sem_signal(&c->send)`, 这一点见下文。

### 21.3.4 Chan_receive — 接收操作

```c
int Chan_receive(Chan_T c, void *ptr, int size) {
    int n;
    assert(c);
    assert(ptr);
    assert(size >= 0);

    Sem_wait(&c->recv);        // (1) 等待数据就绪 (recv从1变0)
    n = *c->size;              // (2) 读取发送方提供的字节数
    if (size < n)              // (3) 如果接收缓冲区不够大,
        n = size;              //     截断到接收缓冲区大小
    *c->size = n;              // (4) 将实际接收字节数写回发送方!
    if (n > 0)                 // (5) 如果有数据,
        memcpy(ptr, c->ptr, n);//     拷贝到接收缓冲区
    Sem_signal(&c->sync);      // (6) 通知发送方"拷贝完成"
    Sem_signal(&c->send);      // (7) 释放发送权 (send恢复到1)
    return n;                  // (8) 返回实际接收的字节数
}
```

**关键细节: `*c->size = n`**

这是整个通道协议的精髓。`c->size` 是发送方局部变量 `size` 的地址。接收方通过 `*c->size = n` **直接修改发送方栈上的变量**。因此 `Chan_send` 返回时, 它的 `size` 参数可能已被接收方修改——这正是 `Chan_send` 能返回"接收方实际接受字节数"的机制。

**状态转换图 (接收方视角)**:

```
初始 (发送方已填好数据): send=0, recv=1, sync=0

步骤1: Sem_wait(&recv)   → send=0, recv=0, sync=0  (消耗数据就绪信号)
步骤2-5: 读取、截断、写回、拷贝                      (临界区)
步骤6: Sem_signal(&sync)  → send=0, recv=0, sync=1  (告诉发送方"拷贝完了")
步骤7: Sem_signal(&send)  → send=1, recv=0, sync=1  (释放发送权, 下个发送方可进入)
                            此时通道恢复到初始状态
```

注意第 7 步: `Sem_signal(&c->send)` 是恢复发送权的**唯一位置**。这确保了:
- 在下一次发送之前, 通道数据结构 (`ptr`, `size`) 不会再被修改。
- `send` 回到 1 意味着通道已准备好接受下一条消息。

### 21.3.5 Chan_free — 释放通道 (扩展)

CII 原始接口不包含释放函数——在大多数示例程序中, 通道与进程同生命周期。但长期运行的服务需要在通信完成后回收通道资源。`Chan_free` 是一个直接了当的扩展:

```c
void Chan_free(Chan_T *cp) {
    assert(cp && *cp);
    /*
     * [UNSPECIFIED] 警告: 如果仍有线程阻塞在 (*cp) 上,
     * 释放后的行为是未定义的。CII 的 Sem_T 未提供 "唤醒所有等待者"
     * 的广播原语, 因此调用者必须确保通道已无活跃通信。
     * 一种安全做法是: 在调用 Chan_free 之前, 通过 0 字节消息
     * 或 Thread_join 确保所有相关线程已终止。
     */
    FREE(*cp);   // Mem 接口: 释放 + 置空指针
}
```

### 21.3.6 Chan_trysend / Chan_tryreceive — 非阻塞操作 (扩展)

非阻塞版本的关键在于检查信号量而不阻塞。这需要假设 `Sem_T` 暴露了内部 `count` 字段 (CII 的 `sem.h` 确实将 `count` 作为结构体的公开字段):

```c
/*
 * Chan_trysend: 非阻塞发送
 *
 * 原理: 如果 send 信号量的计数为 1 (表示通道空闲, 可以发送),
 * 则执行完整的同步发送流程; 否则立即返回 0。
 *
 * 注意: 这是 [UNSPECIFIED] 实现，因为 CII 的 Sem_T 没有提供
 * 原子的 trywait 操作。以下代码直接读取 sem.count 是竞态敏感的——
 * 在抢占式调度下, 检查和使用之间可能被中断。
 * 生产级实现需要用 C11 原子操作或 pthread_mutex_trylock。
 */
int Chan_trysend(Chan_T c, const void *ptr, int size) {
    assert(c);
    assert(ptr);
    assert(size >= 0);

    if (c->send.count <= 0)
        return 0;   // 通道忙碌, 不阻塞, 立即返回

    /* 注意: 从此刻到 Sem_wait 之间, 存在竞态窗口 [UNSPECIFIED] */
    Sem_wait(&c->send);
    c->ptr = ptr;
    c->size = &size;
    Sem_signal(&c->recv);
    Sem_wait(&c->sync);
    return size;
}

/*
 * Chan_tryreceive: 非阻塞接收
 *
 * 原理: 如果 recv 信号量的计数为 1 (表示有数据待接收),
 * 则执行完整的同步接收流程; 否则立即返回 0。
 */
int Chan_tryreceive(Chan_T c, void *ptr, int size) {
    int n;
    assert(c);
    assert(ptr);
    assert(size >= 0);

    if (c->recv.count <= 0)
        return 0;   // 无待收数据, 不阻塞, 立即返回

    /* 注意: 竞态窗口 [UNSPECIFIED] */
    Sem_wait(&c->recv);
    n = *c->size;
    if (size < n)
        n = size;
    *c->size = n;
    if (n > 0)
        memcpy(ptr, c->ptr, n);
    Sem_signal(&c->sync);
    Sem_signal(&c->send);
    return n;
}
```

**生产级替代方案**:

上文的计数检查 (`c->send.count <= 0`) 存在竞态, 仅在非抢占式调度 (协程) 下安全。对于抢占式线程, 替代方案包括:
- 使用 `pthread_mutex_trylock` 包装 `send` 的互斥访问
- 使用 C11 的 `atomic_flag_test_and_set` (参见第 21.7 节)
- 在 Linux 上使用 `futex(FUTEX_WAIT | FUTEX_NONBLOCK)` 系统调用

---

## 21.4 信号量协调机制深度分析

### 21.4.1 三阶段握手协议

CII 通道使用**三个信号量**而非两个信号量, 原因在于需要区分两个不同的同步点:

| 步骤 | 发送方 | 信号量变化 | 接收方 |
|------|--------|-----------|--------|
| **Phase 1**: 发送就绪 | `Sem_wait(send)` → 进入临界区 | `send: 1→0` | (可能是在做其他事情) |
| | 填充 `ptr`, `size` | | |
| | `Sem_signal(recv)` | `recv: 0→1` | |
| **Phase 2**: 数据拷贝 | `Sem_wait(sync)` → **阻塞** | | `Sem_wait(recv)` → 进入临界区 |
| | (等待...) | | 读取, 截断, 写回, `memcpy` |
| | | `sync: 0→1` | `Sem_signal(sync)` |
| **Phase 3**: 释放 | | | `Sem_signal(send)` |

如果只用两个信号量 (send, recv), 发送方在 `Sem_signal(recv)` 后立即返回, 无法保证接收方已完成数据拷贝——发送方的 `ptr` 可能指向一个即将失效的栈上缓冲区。

`sync` 信号量的引入解决了这个问题: 发送方在 `Sem_signal(recv)` 后不立即返回, 而是 `Sem_wait(sync)`, 等待接收方确认数据已安全拷贝到其私有缓冲区。这个设计使得通道的同步语义变得**完全确定**: 当 `Chan_send` 返回时, 数据一定已被接收方 (或其上层) 安全接收。

### 21.4.2 Sem_T 结构 (来自 `sem.h`)

```c
#define T Sem_T
typedef struct T {
    int count;    // 信号量计数值 (≥0)
    void *queue;  // 等待该信号量的线程队列
} T;
```

CII 的信号量是**计数信号量 (counting semaphore)**, 基于 Dijkstra (1968) 的经典定义:
- `Sem_init(&s, n)`: 初始化为 `n`
- `Sem_wait(&s)` (P 操作): 若 `count > 0`, `count--`; 否则阻塞调用线程
- `Sem_signal(&s)` (V 操作): 若有线程等待, 唤醒之; 否则 `count++`

在 `Chan_T` 中, `send` 初始值为 1 意味着它是一个**二元信号量 (binary semaphore)**, 等价于互斥锁。`recv` 和 `sync` 初始值为 0, 充当**事件信号量 (event semaphore)**——P 操作必然阻塞, 等待对方的 V 操作将其唤醒。

### 21.4.3 同步通道 vs 缓冲通道

上述实现是**同步 (无缓冲) 通道**。从代码中可以直接看出: 没有任何循环缓冲区、没有 `head`/`tail` 指针、没有 `count` 字段——通道结构体中根本没有存储用户数据的地方。`ptr` 仅仅是发送方缓冲区的指针, 数据在 `memcpy` 完成后就直接离开通道了。

**同步通道 vs 缓冲通道对比**:

| 特性 | 同步通道 (Chan_T) | 缓冲通道 (Go `make(chan T, N)`) |
|------|-------------------|--------------------------------|
| 缓冲区大小 | 0 (零容量) | N ≥ 1 |
| 存储方式 | 指针传递 (无中间存储) | 环形缓冲区 |
| 发送方阻塞条件 | 总是阻塞直到接收方到达 | 仅当缓冲区满时阻塞 |
| 接收方阻塞条件 | 总是阻塞直到发送方到达 | 仅当缓冲区空时阻塞 |
| 结构体复杂度 | 5 字段, ~40 行 | 需要环形缓冲区 + 信号量 |
| 数据结构 | `ptr`, `size`, 3 个信号量 | `buf[]`, `sendx`, `recvx`, `lock` |

**从同步通道构建缓冲通道**:

只需在同步通道上添加一层适配。核心思路: 用一个专门的 "buffer 线程" 夹在发送方和接收方之间, 该线程维护一个队列缓冲区。这是 Go 语言的 `hchan` 实现 (`runtime/chan.go`) 的基本原理:

```c
// 概念代码: 缓冲通道适配器 (基于 Chan_T 实现)
struct BufferedChan {
    Chan_T sync_chan;    // 底层的同步通道
    void **buffer;       // 缓冲区数组
    int cap, head, tail, count;
    Sem_T mutex, not_full, not_empty;
};
```

---

## 21.5 并发模式: 完整代码示例

### 21.5.1 示例一: 生产者-消费者

这是最基础的并发模式: 一个线程生产数据, 另一个线程消费数据, 通过通道同步。

```c
/*
 * prodcons.c — 生产者-消费者模式
 *
 * 一个生产者线程连续发送整数 0..9,
 * 一个消费者线程接收并打印。
 * 同步通道确保: 消费者不会在生产者发送前读取,
 * 生产者不会在消费者读取前覆盖数据。
 *
 * 编译: cc -I../code prodcons.c ../code/chan.c ../code/sem.c \
 *            ../code/thread.c ../code/mem.c ../code/except.c \
 *            ../code/assert.c ../code/arena.c
 */

#include <stdio.h>
#include <stdlib.h>
#include "thread.h"
#include "chan.h"
#include "mem.h"

/* ---- 生产者 ---- */
int producer(void *cl) {
    Chan_T c = ((void **)cl)[0];  // 通道句柄
    int n = *(int *)((void **)cl)[1];  // 生产数量
    int i;

    for (i = 0; i < n; i++) {
        Chan_send(c, &i, sizeof i);
    }

    /* 发送结束信号: 0 字节表示"不再生产" */
    Chan_send(c, &i, 0);
    return EXIT_SUCCESS;
}

/* ---- 消费者 ---- */
int consumer(void *cl) {
    Chan_T c = ((void **)cl)[0];
    int val, total = 0;

    for (;;) {
        int n = Chan_receive(c, &val, sizeof val);
        if (n == 0) {
            /* 0 字节 = 结束信号 */
            printf("Consumer: received shutdown signal. ");
            printf("Total consumed: %d\n", total);
            break;
        }
        printf("Consumer: got %d\n", val);
        total += val;
    }
    return EXIT_SUCCESS;
}

/* ---- main ---- */
int main(void) {
    Chan_T c;
    int n_items = 10;
    void *args[2];

    Thread_init(1, NULL);     // 1 = 抢占式调度

    c = Chan_new();

    args[0] = c;
    args[1] = &n_items;
    Thread_new(consumer, args, sizeof args, NULL);
    Thread_new(producer, args, sizeof args, NULL);

    Thread_exit(EXIT_SUCCESS);
    return EXIT_SUCCESS;
}
```

**模式要点**:
- 发送方用零字节消息作为**带内终止信号 (in-band shutdown signal)**。
- 通道天然提供了**背压 (back-pressure)**: 如果消费者慢, 生产者会在 `Chan_send` 中阻塞, 不会继续产生数据。这是 CSP 模型的内建流控。

### 21.5.2 示例二: Pipeline (管道) — 素数筛

这是 CII 原书 20.2.3 节的标志性示例, 也是 CSP 论文中最经典的管道模式。

```c
/*
 * pipeline.c — 管道模式: Eratosthenes 素数筛
 *
 * 管道结构: source → filter(2) → filter(3) → filter(5) → ... → sink
 *
 * 每个 stage 是一个线程, stage 之间通过 Chan_T 连接。
 * source 发送整数流; 每个 filter 过滤掉其对应素数的倍数;
 * 后端的 sink 收集素数, 并在积累到 n 个素数后
 * "克隆"自身——将现有的 sink 变为 filter, 创建新的 sink。
 *
 * 编译: cc -I../code pipeline.c ../code/chan.c ../code/sem.c \
 *            ../code/thread.c ../code/mem.c ../code/except.c \
 *            ../code/assert.c ../code/arena.c
 */

#include <stdio.h>
#include <stdlib.h>
#include "thread.h"
#include "chan.h"
#include "fmt.h"

/* ---- 管道阶段参数 ---- */
struct args {
    Chan_T c;          // 输入通道 (对于 source 是输出)
    int n;             // 每个 filter 持有的素数个数 (默认5)
    int last;          // 素数上界
};

/* ====== source: 产生整数流 ====== */
int source(void *cl) {
    struct args *p = cl;
    int i = 2;

    /* 先发 2, 然后发所有奇数 */
    if (Chan_send(p->c, &i, sizeof i))
        for (i = 3; Chan_send(p->c, &i, sizeof i); )
            i += 2;

    return EXIT_SUCCESS;
}

/* ====== filter: 过滤素数的倍数 ====== */
void filter(int primes[], Chan_T input, Chan_T output) {
    int j, x;
    for (;;) {
        Chan_receive(input, &x, sizeof x);

        /* 检查 x 是否是 primes[] 中任一素数的倍数 */
        for (j = 0; primes[j] != 0 && x % primes[j] != 0; j++)
            ;

        /* 如果没有匹配的素数, x 可能是素数, 往下游发送 */
        if (primes[j] == 0) {
            if (Chan_send(output, &x, sizeof x) == 0)
                break;  // 下游已关闭
        }
    }

    /* 向上游传播关闭信号 */
    Chan_receive(input, &x, 0);
}

/* ====== sink: 收集并打印素数 ====== */
int sink(void *cl) {
    struct args *p = cl;
    Chan_T input = p->c;
    int i = 0, j, x, primes[256];

    primes[0] = 0;

    for (;;) {
        Chan_receive(input, &x, sizeof x);

        /* 检查 x 是否是素数的倍数 */
        for (j = 0; primes[j] != 0 && x % primes[j] != 0; j++)
            ;

        if (primes[j] == 0) {
            /* x 是素数 */
            if (x > p->last)
                break;

            Fmt_print(" %d", x);
            primes[i++] = x;
            primes[i] = 0;

            /* 积累到 n 个素数: 克隆自己, 变为 filter */
            if (i == p->n) {
                p->c = Chan_new();
                Thread_new(sink, p, sizeof *p, NULL);
                filter(primes, input, p->c);
                return EXIT_SUCCESS;
            }
        }
    }

    Fmt_print("\n");
    /* 向上游传播关闭信号 */
    Chan_receive(input, &x, 0);
    return EXIT_SUCCESS;
}

/* ====== main ====== */
int main(int argc, char *argv[]) {
    struct args args;

    Thread_init(1, NULL);

    args.c   = Chan_new();
    args.n   = argc > 2 ? atoi(argv[2]) : 5;
    args.last = argc > 1 ? atoi(argv[1]) : 1000;

    Thread_new(source, &args, sizeof args, NULL);
    Thread_new(sink,   &args, sizeof args, NULL);

    Thread_exit(EXIT_SUCCESS);
    return EXIT_SUCCESS;
}
```

**运行结果** (`pipeline 100`):

```
2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97
```

**管道动态增长的机制**:

当 sink 积累到 `n` 个素数后:
1. `sink` 的 `Chan_new()` 创建一个新通道 `c_new`
2. `Thread_new(sink, ...)` 创建一个**新的 sink 线程**, 其输入通道是 `c_new`
3. 当前的 `sink` 线程调用 `filter(primes, input, c_new)` — 用旧通道 `input` 接收, 用新通道 `c_new` 发送
4. 原来的 sink 变成了一个 filter, 夹在 source 和新的 sink 之间

管道结构动态扩展为:

```
source → [旧sink→filter(2,3,5,7,11)] → [新sink(13,17,...)]
```

这正是图 20.2 在原文中描绘的动态管道扩展过程。

### 21.5.3 示例三: Fan-Out / Fan-In — 并行归并排序

第三个模式展示如何用多个通道实现工作分发和结果收集。

```c
/*
 * fanout.c — Fan-Out / Fan-In 模式: 并行归并排序
 *
 * 结构:                    ┌→ worker0 →┐
 *  dispatcher →[ch0..chN]→ ├→ worker1 →├→ collector
 *                          └→ worker2 →┘
 *
 * dispatcher: 把数组分片, 通过独立通道发送给各 worker
 * worker:     对自己的分片执行归并排序, 通过回传通道发送结果
 * collector:  收集各 worker 的排序结果, 归并成一个有序数组
 *
 * 编译: cc -I../code fanout.c ../code/chan.c ../code/sem.c \
 *            ../code/thread.c ../code/mem.c ../code/except.c \
 *            ../code/assert.c ../code/arena.c
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "thread.h"
#include "chan.h"
#include "mem.h"

#define NWORKERS 4
#define ARRAY_SIZE 100

/* ---- 传给 worker 的参数 ---- */
struct work {
    Chan_T in;     // 接收任务 (分片)
    Chan_T out;    // 回传结果
};

/* ---- 比较函数 ---- */
static int cmp_int(const void *a, const void *b) {
    return (*(int *)a - *(int *)b);
}

/* ---- worker: 排序分片并回传 ---- */
int worker(void *cl) {
    struct work *w = cl;
    int slice[ARRAY_SIZE];
    int n;

    /* 接收分片大小和数据 */
    n = Chan_receive(w->in, &slice[0], 0);  // 先收大小 (0字节握手)
    Chan_receive(w->in, slice, sizeof slice);

    /* 快速排序 */
    qsort(slice, ARRAY_SIZE / NWORKERS, sizeof(int), cmp_int);

    /* 回传排序结果 */
    Chan_send(w->out, slice, sizeof(int) * (ARRAY_SIZE / NWORKERS));

    return EXIT_SUCCESS;
}

/* ---- dispatcher: 分发任务 ---- */
int dispatcher(void *cl) {
    struct work *works = cl;
    Chan_T collector_in = works[NWORKERS].in;  // 收集器通道
    int array[ARRAY_SIZE];
    int i, j;

    /* 生成随机数据 */
    srand((unsigned)time(NULL));
    for (i = 0; i < ARRAY_SIZE; i++)
        array[i] = rand() % 1000;

    /* 向每个 worker 发送分片 */
    int chunk_size = ARRAY_SIZE / NWORKERS;
    for (i = 0; i < NWORKERS; i++) {
        Chan_send(works[i].in, &chunk_size, sizeof chunk_size);
        Chan_send(works[i].in, array + i * chunk_size,
                  sizeof(int) * chunk_size);
    }

    /* 分发完成, 等待 collector 收集 */
    Thread_join(NULL);
    return EXIT_SUCCESS;
}

/* ---- collector: 收集并归并 ---- */
int collector(void *cl) {
    Chan_T in = cl;
    int *sorted[NWORKERS];
    int merged[ARRAY_SIZE];
    int i;

    /* 从每个 worker 收集排序结果 */
    for (i = 0; i < NWORKERS; i++) {
        sorted[i] = ALLOC(sizeof(int) * ARRAY_SIZE / NWORKERS);
        Chan_receive(in, sorted[i],
                     sizeof(int) * ARRAY_SIZE / NWORKERS);
    }

    /* 验证: 每个分片内部有序 */
    for (i = 0; i < NWORKERS; i++) {
        int j;
        for (j = 1; j < ARRAY_SIZE / NWORKERS; j++) {
            if (sorted[i][j] < sorted[i][j - 1]) {
                printf("ERROR: slice %d not sorted!\n", i);
                return EXIT_FAILURE;
            }
        }
    }

    printf("Collector: all %d slices sorted correctly.\n", NWORKERS);
    printf("First values: ");
    for (i = 0; i < NWORKERS; i++)
        printf("%d ", sorted[i][0]);
    printf("\n");

    /* 在实际系统中, 这里会执行 k-way merge */
    for (i = 0; i < NWORKERS; i++)
        FREE(sorted[i]);

    return EXIT_SUCCESS;
}

/* ---- main ---- */
int main(void) {
    struct work works[NWORKERS + 1];  // 最后一个给 collector
    int i;
    Thread_T threads[NWORKERS + 2];

    Thread_init(1, NULL);

    /* 创建通道: 每个 worker 一个输入通道, 外加一个收集通道 */
    Chan_T collector_chan = Chan_new();
    for (i = 0; i < NWORKERS; i++) {
        works[i].in  = Chan_new();
        works[i].out = collector_chan;
    }

    /* 启动 workers */
    for (i = 0; i < NWORKERS; i++)
        threads[i] = Thread_new(worker, &works[i],
                                sizeof works[i], NULL);

    /* 启动 collector (复用最后一个 work 槽传入通道) */
    works[NWORKERS].in = collector_chan;
    threads[NWORKERS] = Thread_new(collector, &works[NWORKERS].in,
                                    sizeof(Chan_T), NULL);

    /* 启动 dispatcher */
    threads[NWORKERS + 1] = Thread_new(dispatcher, works,
                                        sizeof works, NULL);

    Thread_exit(EXIT_SUCCESS);
    return EXIT_SUCCESS;
}
```

**模式要点**:
- **Fan-Out**: dispatcher 通过 N 个独立通道把任务发给 N 个 worker, 每个 worker 独立排序。
- **Fan-In**: 所有 worker 共享同一个 `collector_chan` 回传结果。注意这是不安全的共享——多发送方同时发送到同一通道可能导致竞态。在 CII 的原始设计中, 通道是为**单发送方-单接收方**设计的。多发送方并发发送需要额外的互斥保护, 或者使用下文第 21.7.3 节讨论的 **mpsc** 模式。
- 更好的做法是每个 worker 使用独立的回传通道, collector 使用 `Thread_join` 或轮询方式逐个收集。

---

## 21.6 0 字节消息: 纯同步/信号机制

在 `Chan_send` 和 `Chan_receive` 中, 当 `size == 0` 时:

```c
// 发送方
Chan_send(c, &dummy, 0);   // 不传数据, 仅握手

// 接收方
int n = Chan_receive(c, buf, 0);  // 返回 0, 表示握手完成
```

此时通道退化为一个**同步屏障 (synchronization barrier)**:
1. 发送方和接收方在通道上 "会合"
2. `memcpy(ptr, c->ptr, 0)` 什么也不做
3. 双方同时知道对方已达同步点

**应用场景**:

| 场景 | 发送方 | 接收方 |
|------|--------|--------|
| **终止信号** | 发送 0 字节表示 "不再有数据" | 收到 0 字节后关闭 |
| **流水线排空** | 发送 0 字节表示 "前序数据已处理完毕" | 收到后触发下一阶段 |
| **心跳检测** | 周期性发送 0 字节 | 超时未收到则判定故障 |
| **单次通知** | 发送 0 字节触发一次性事件 | 收到后执行动作 |

在 pipe-prime 示例中, 关闭信号就是通过 0 字节消息从 sink 层层传播到 source 的: 每个 filter 在收到下游的 0 字节关闭信号后, 再向上游发送 0 字节, 实现**优雅停机 (graceful shutdown)**。

---

## 21.7 Modern-C 改进: C11 原子操作的无锁通道

### 21.7.1 CII 原始实现的局限

CII 的 `Chan_T` 基于内核/用户线程级别的信号量, 每次 `Chan_send`/`Chan_receive` 涉及:
- 至少 2 次 `Sem_wait` (可能触发上下文切换)
- 至少 2 次 `Sem_signal` (可能触发上下文切换)
- 1 次 `memcpy`

对于高频消息传递, 上下文切换开销可能成为瓶颈。

### 21.7.2 C11 原子操作的 SPSC (单生产者单消费者) 无锁通道

利用 C11 `<stdatomic.h>`, 可以实现真正的**无锁 (lock-free)** SPSC 通道, 消除所有内核态上下文切换 (在无竞争情况下):

```c
/*
 * lockfree_chan.h — C11 原子无锁 SPSC 通道 (Modern-C 改进版)
 *
 * 原理: 使用原子变量的生产者-消费者环形缓冲区
 * 核心参考文献:
 *   - Lamport (1983) "Specifying Concurrent Program Modules"
 *   - Herlihy & Shavit (2012) "The Art of Multiprocessor Programming"
 *   - Vyukov (2010) "Single-Producer/Single-Consumer Queue"
 *
 * 与 CII Chan_T 的关键区别:
 *   1. 无锁: 使用 fetch_add / atomic_load / atomic_store
 *   2. 缓冲: 支持预分配容量的有界缓冲区
 *   3. 无阻塞发送: 缓冲区满时返回 0 (而非阻塞)
 */

#include <stdatomic.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>

#define LFCACHELINE 64  // 避免 false sharing

/* ---- SPSC 无锁通道 ---- */
typedef struct {
    /* 生产者只写, 消费者只读 (cacheline 对齐) */
    _Alignas(LFCACHELINE) atomic_size_t head;  // 消费者读取位置
    _Alignas(LFCACHELINE) atomic_size_t tail;  // 生产者写入位置

    size_t capacity;       // 容量 (必须是 2 的幂)
    size_t elem_size;      // 每个元素的大小 (字节)
    char  buffer[];        // 柔性数组成员: buffer[capacity * elem_size]
} LockFreeChan;

/* ---- 初始化 ---- */
LockFreeChan *lfchan_new(size_t capacity, size_t elem_size) {
    LockFreeChan *c;

    /* 将 capacity 向上取整到 2 的幂 (mask 求余更快) */
    size_t cap = 1;
    while (cap < capacity) cap <<= 1;

    c = malloc(sizeof(LockFreeChan) + cap * elem_size);
    if (!c) return NULL;

    atomic_init(&c->head, 0);
    atomic_init(&c->tail, 0);
    c->capacity  = cap;
    c->elem_size = elem_size;
    return c;
}

/* ---- 发送 (非阻塞) ---- */
bool lfchan_send(LockFreeChan *c, const void *elem) {
    size_t head = atomic_load_explicit(&c->head, memory_order_acquire);
    size_t tail = atomic_load_explicit(&c->tail, memory_order_relaxed);

    /* 满检查: tail + 1 == head (mod capacity) */
    if ((tail - head) == c->capacity)
        return false;  // 缓冲区满

    size_t idx = (tail & (c->capacity - 1)) * c->elem_size;
    memcpy(c->buffer + idx, elem, c->elem_size);

    /* 发布写入: 确保 memcpy 在 tail 更新前完成 */
    atomic_store_explicit(&c->tail, tail + 1, memory_order_release);
    return true;
}

/* ---- 接收 (非阻塞) ---- */
bool lfchan_recv(LockFreeChan *c, void *elem) {
    size_t head = atomic_load_explicit(&c->head, memory_order_relaxed);
    size_t tail = atomic_load_explicit(&c->tail, memory_order_acquire);

    /* 空检查 */
    if (head == tail)
        return false;  // 缓冲区空

    size_t idx = (head & (c->capacity - 1)) * c->elem_size;
    memcpy(elem, c->buffer + idx, c->elem_size);

    /* 发布读取: 确保 memcpy 在 head 更新后完成 */
    atomic_store_explicit(&c->head, head + 1, memory_order_release);
    return true;
}

/* ---- 销毁 ---- */
void lfchan_free(LockFreeChan *c) {
    free(c);
}
```

**关键设计与内存序分析**:

| 操作 | 内存序 | 原因 |
|------|--------|------|
| `head` load (send) | `acquire` | 读到消费者最新的 head, 确保看到之前消费者读取过的数据 |
| `tail` store (send) | `release` | 确保 `memcpy` 完成后再发布 tail 更新 |
| `tail` load (recv) | `acquire` | 读到生产者最新的 tail, 确保看到刚写入的数据 |
| `head` store (recv) | `release` | 确保 `memcpy` 完成后 head 才对生产者可见 |

**与 CII Chan_T 的对比**:

| 特性 | CII Chan_T (1996) | LockFreeChan (C11) |
|------|-------------------|-------------------|
| 同步/异步 | 同步 (无缓冲) | 异步 (有界缓冲) |
| 阻塞语义 | 总是阻塞 | 非阻塞 (满/空时返回 false) |
| 并发原语 | 信号量 (`Sem_T`) | 原子操作 (`_Atomic`) |
| 上下文切换 | 每次通信 2-4 次切换 | 无竞争时 0 次切换 |
| 缓存友好性 | 不适用 | Cacheline 对齐防 false sharing |
| 容量 | 0 (rendezvous) | 2^N (有界环形队列) |
| 标准依赖 | C89 + POSIX 线程 | C11 `<stdatomic.h>` |

### 21.7.3 MPSC (多生产者单消费者) / SPMC 通道

SPSC 通道的天然扩展:

- **MPSC (Multi-Producer Single-Consumer)**: 多个生产者可以并发发送。关键做法是让 `tail` 的递增变为 `atomic_fetch_add(&tail, 1)` (原子 CAS 循环), 但需要额外处理满检测的 ABA 问题。

```c
// MPSC 的核心差异: tail 必须用原子递增
size_t tail = atomic_fetch_add_explicit(
    &c->tail, 1, memory_order_relaxed);
// tail 现在是"已预留"的槽位
// 需要等待 head 移动到足够远才能写入
while ((tail - atomic_load(&c->head)) >= c->capacity)
    ; // 自旋等待 (spin-wait)
memcpy(c->buffer + idx, elem, c->elem_size);
```

- **SPMC**: 类似, 但 `head` 用原子递增, 多个消费者竞争读取。

- **MPMC**: 需要更复杂的协议, 因为在获取了索引后到实际写入/读取之间, 其他线程可能已经更改了缓冲区状态。

这些在现代高性能系统中广泛使用, 例如:
- Linux `io_uring` 的提交队列/完成队列就是一对 SPSC 通道
- DPDK 的 `rte_ring` 使用了类似的 MPSC/MCMP 设计
- Rust `crossbeam::channel` 使用了分段队列 (segmented queue) 优化 MPMC 性能

---

## 21.8 跨语言对比: CII Channel vs Go Channel vs Rust mpsc

### 21.8.1 语义对比

| 维度 | CII `Chan_T` | Go `chan` | Rust `std::sync::mpsc` |
|------|-------------|-----------|----------------------|
| **设计来源** | CSP (Hoare 1978) | Newsqueak → CSP | CSP + 所有权系统 |
| **同步/异步** | 仅同步 (无缓冲) | 同步/缓冲可选 | 仅异步 (无限或有限缓冲) |
| **数据结构** | 3 个信号量 + 2 个指针 (无缓冲) | `hchan`: 环形缓冲 + 信号量 + 等待队列 | `Flavor::Array` / `Flavor::List` |
| **多发送方** | API 允许, 但不安全 | 支持 (竞态由 runtime 管理) | 支持 (`Sender` 可 clone) |
| **多接收方** | API 允许, 但不安全 | 支持 | 不支持 (Single Consumer) |
| **Select** | 不支持 (无 `select` 原语) | `select { case <-ch: }` | 不支持 (需用 `crossbeam::select!`) |
| **关闭语义** | 0 字节消息约定 | `close(ch)` 广播关闭 | `drop(Sender)` 自然关闭 |
| **内存管理** | 手动 (指针 + 信号量) | GC (运行时) | 所有权 + Drop |

### 21.8.2 为什么 Go Channel 更强大

Go 的 `chan` 在 CII Chan_T 基础上做了三个关键扩展:

1. **缓冲**: `make(chan int, 10)` 创建容量为 10 的缓冲通道, 发送方只在 buffer 满时才阻塞。
2. **Select 多路复用**: `select` 语句允许一个 goroutine 同时等待多个通道操作, 这是 CSP 原始论文中的核心特性 (CII 未实现)。
3. **关闭广播**: `close(ch)` 会唤醒所有阻塞在该通道上的 goroutine, 接收方得到零值 + `ok=false`。

这些扩展使得 Go channel 能处理比 CII 通道复杂得多的并发模式。但代价是每个 `hchan` 结构体远比 `Chan_T` 庞大——`runtime/chan.go` 中的 `hchan` 结构体约 40 行。

### 21.8.3 为什么 Rust mpsc 走不同路线

Rust 的 `std::sync::mpsc` 选择了**多生产者-单消费者 (MPSC)** 路线, 而非通用双向通道:

```rust
use std::sync::mpsc;
let (tx, rx) = mpsc::channel::<i32>();  // tx 可 clone, rx 不可
```

核心理由:
- **所有权系统**: 单向数据流天然契合 Rust 的 ownership 模型; `tx` 发送所有权, `rx` 接收所有权, 无竞态。
- **Drop 作为关闭信号**: `Sender` 被 drop 时自动关闭通道, 无需显式 `close()`。
- **零成本抽象**: 编译期即可确定通道的 "生命周期", 不需要 GC。

`crossbeam` crate 提供了更接近 Go channel 的 MPMC 实现, 包括 `select!` 宏。

---

## 21.9 Thread + Chan: CII 并发编程的最小可用组合

### 21.9.1 设计哲学

CII 中最极简的并发编程组合是:

```
Thread_init + Thread_new + Chan_new + Chan_send + Chan_receive + Thread_join
```

只有 6 个函数, 但已经涵盖了 CSP 的精髓——线程是执行单元, 通道是同步媒介。这种极简主义有几个好处:

1. **可理解性**: 整个通道实现在 40 行内, 每个信号量状态都可以手工推导。
2. **可移植性**: Chan.c 完全不包含任何机器相关代码; 它只依赖 `Thread_init` 保证的线程系统和 `Sem_T`。
3. **组合性**: Thread + Chan 可以组合成任何并发模式——管道、扇出扇入、工作池、屏障——而不需要额外的同步原语。

### 21.9.2 最小并发程序骨架

```c
#include "thread.h"
#include "chan.h"

int worker(void *cl) {
    Chan_T c = *(Chan_T *)cl;
    int msg;
    Chan_receive(c, &msg, sizeof msg);
    /* ... work ... */
    return EXIT_SUCCESS;
}

int main(void) {
    Chan_T c = Chan_new();
    Thread_init(1, NULL);
    Thread_new(worker, &c, sizeof c, NULL);
    int msg = 42;
    Chan_send(c, &msg, sizeof msg);
    Thread_exit(EXIT_SUCCESS);
}
```

这就是全部。一个线程、一个通道、一条消息、一次会合。

### 21.9.3 局限性

CII 的 Thread+Chan 组合有以下已知局限 (原书练习中明确指出):

1. **无 Select**: 线程无法同时等待多个通道。Exercise 20.3 建议重新实现 Chan 以解决此问题。
2. **无缓冲通道**: Exercise 20.4 要求设计异步带缓冲通道。
3. **无死锁检测**: Exercise 20.2 建议为信号量实现死锁检测。
4. **无优先级调度**: Exercise 20.9 建议扩展 Thread 支持优先级。
5. **多发送方不安全**: 多个线程并发调用同一通道的 `Chan_send` 可能产生竞态。

这些限制在 CII 的设计哲学下是合理的——CII 是一个教学性质的库, 它的目标是展示核心概念的最小可用实现, 而非生产级的并发运行时。

---

## 21.10 总结

### 核心贡献

CII 的 Channel 接口用 **40 行 C 代码 + 3 个信号量** 精确实现了 CSP 同步通信通道的 rendezvous 语义。它的贡献不在于性能或功能完备性, 而在于:

1. **可验证的正确性**: 用三个信号量表达的握手协议可以直接通过状态枚举验证无死锁。
2. **指针技巧**: `c->size = &size` 使得接收方能修改发送方的返回值——这是 C 语言用指针实现双向通信的典范。
3. **最小接口**: 只有 3 个 API 函数 (`Chan_new`, `Chan_send`, `Chan_receive`), 但足以构建所有标准并发模式。
4. **教学价值**: 在 ~400 行的 Thread 实现基础上, 只需 ~40 行即可完整实现通道, 展示了分层设计的威力。

### 从 CII 到现代

| CII (1996) | Modern-C (C11+) | Go (2009+) | Rust (2015+) |
|-----------|-----------------|-----------|-------------|
| `Sem_T` 三元组 | `_Atomic` 无锁队列 | `hchan` runtime | `crossbeam` / `mpsc` |
| ~40 行 | ~100 行 (含内存序) | ~500 行 (含 select) | ~1000 行 (含所有权) |
| 同步 (rendezvous) | 异步 (有界缓冲) | 同步 + 异步 | 异步 (无限缓冲) |
| 教科书典范 | 生产级无锁 | 语言级一等公民 | 零成本抽象 |

---

## 21.11 练习

1. **实现 `Chan_trysend` 和 `Chan_tryreceive`**: 给 CII 通道添加非阻塞发送/接收操作。这些函数不应阻塞调用线程, 如果无法立即完成操作, 返回 0。(提示: 参考信号量的非阻塞 P 操作。)

2. **从 CII Chan_T 构建缓冲通道**: 在不修改 `chan.c` 的前提下, 写一个适配器结构体 `BufferedChan`, 使用 `Chan_T` 作为底层传输, 在适配器层维护一个环形缓冲区。(提示: 参考 21.5.1 的模式。)

3. **实现 Channel Close**: 给 CII 通道添加显式的关闭操作。关闭后的通道不应再接受发送, 但应允许接收方排空剩余数据。

4. **Select 多路复用**: 设计一个 `Chan_select` 函数, 允许线程同时等待多个通道。这是 CSP 原始论文中最核心但 CII 未实现的原语。(提示: 参考 Go 的 `reflect.Select`。)

5. **性能测量**: 分别用 CII `Chan_T` 和 C11 原子 SPSC 通道实现相同的管道素数筛, 在 1000 个素数规模下测量端到端延迟。记录 `Sem_wait`/`Sem_signal` 触发的上下文切换次数。

---

**引用**:

- Hoare, C.A.R. (1978). "Communicating Sequential Processes." *Communications of the ACM*, 21(8), 666-677.
- Hanson, D.R. (1996). *C Interfaces and Implementations*. Addison-Wesley. Chapter 20: Threads.
- Pike, R. (1990). "The Implementation of Newsqueak." *Software—Practice and Experience*, 20(7).
- Vyukov, D. (2010). "Single-Producer/Single-Consumer Queue." [1024cores.net](http://www.1024cores.net).
- Herlihy, M. & Shavit, N. (2012). *The Art of Multiprocessor Programming*. 2nd ed. Morgan Kaufmann.
- Go Team. `runtime/chan.go` — Go 语言标准库通道实现源码。
- Rust Team. `std::sync::mpsc` — Rust 标准库多生产者单消费者通道。
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch20_线程_Thread.html">第20章 线程 Thread</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div></div>
</div>
