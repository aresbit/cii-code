# 第20章: 线程 (Thread) — 用户态协程

> 原文: C Interfaces and Implementations, David R. Hanson, Chapter 20
> 源文件: `code/thread.h`, `code/thread.c`, `code/thread-nt.c`, `code/sem.h`

## 一、设计哲学

CII的Thread模块实现了一套**协作式用户态线程 (coroutines)**, 而非操作系统级的抢占式线程。

### 关键设计决策

```
用户态线程 (CII的Thread):
├── 在用户空间切换, 不涉及内核
├── 协作式调度 (线程主动调用Thread_pause或阻塞时切换)
├── 可选的抢占式调度 (SIGVTALRM信号, 每10ms切换)
├── 自定义栈分配 (16KB/线程)
└── 汇编实现的上下文切换 (_swtch)

OS线程 (pthread):
├── 内核调度
├── 抢占式调度 (时间片)
├── 系统分配的栈 (通常8MB)
└── 系统调用实现的上下文切换
```

### 接口

```c
#define T Thread_T
typedef struct T *T;

extern int  Thread_init (int preempt, ...);      // 初始化, 0=协作 1=抢占
extern T    Thread_new  (int apply(void *),       // 创建线程(执行apply)
                         void *args, int nbytes, ...);
extern void Thread_exit (int code);               // 退出当前线程
extern void Thread_alert(T t);                    // 唤醒/中断线程
extern T    Thread_self (void);                   // 获取当前线程
extern int  Thread_join (T t);                    // 等待线程结束
extern void Thread_pause(void);                   // 主动让出CPU

// 信号量 (sem.h)
#define T Sem_T
typedef struct { int count; void *queue; } T;
extern void Sem_init  (T *s, int count);
extern void Sem_wait  (T *s);                     // P操作
extern void Sem_signal(T *s);                     // V操作

#define LOCK(mutex)    do { Sem_T *m = &(mutex); Sem_wait(m);
#define END_LOCK       Sem_signal(m); } while (0)
```

---

## 二、数据结构

### 2.1 Thread_T

```c
struct T {
    unsigned long *sp;    // 栈指针 (必须是第一个字段! _swtch依赖)
    T link;               // 就绪队列链表
    T *inqueue;           // 指向所在队列的指针 (NULL表示不在队列中)
    T handle;             // 线程句柄 (通常==t自身, 退出后==NULL)
    Except_Frame *estack; // 线程独立的异常栈
    int code;             // 退出码
    T join;               // 等待本线程的线程队列
    T next;               // freelist链表
    int alerted;          // 是否被alert中断
};

// 全局状态
static T ready = NULL;    // 就绪队列 (循环链表)
static T current;         // 当前运行线程
static int nthreads;      // 活跃线程数
static T root;            // 根线程 (main线程)
```

### 2.2 就绪队列 — 循环链表

```
ready → [Thread_A] → [Thread_B] → [Thread_C] ─┐
          ↑___________________________________←┘
          循环: last->link = first
```

队列操作的实现:

```c
// put: 将t加入队列尾部
static void put(T t, T *q) {
    if (*q) {
        t->link = (*q)->link;   // t->link = first
        (*q)->link = t;          // last->link = t
    } else
        t->link = t;             // 单元素: 自环
    *q = t;                      // 更新尾指针
    t->inqueue = q;
}

// get: 从队列头部取出
static T get(T *q) {
    T t = (*q)->link;            // first
    if (t == *q)
        *q = NULL;               // 最后一个元素
    else
        (*q)->link = t->link;    // last->link = first->link
    t->link = NULL;
    t->inqueue = NULL;
    return t;
}
```

---

## 三、协程切换 — _swtch

`_swtch(from, to)` 是汇编实现的上下文切换:

```
_swtch(from, to):
    // 保存当前线程的寄存器
    push %ebx, %esi, %edi, %ebp, %esp → from->sp
    // 恢复目标线程的寄存器
    mov to->sp → %esp
    pop %esp, %ebp, %edi, %esi, %ebx
    ret
```

C语言包装:

```c
static void run(void) {
    T t = current;
    current = get(&ready);              // 从就绪队列取出下一个线程
    t->estack = Except_stack;           // 保存当前线程的异常栈
    Except_stack = current->estack;     // 恢复目标线程的异常栈
    _swtch(t, current);                 // 汇编: 切换上下文
}
```

---

## 四、线程生命周期

### 4.1 Thread_new

```c
T Thread_new(int apply(void *), void *args, int nbytes, ...) {
    T t;

    // 分配栈: 16KB + sizeof(T) + args + 16字节对齐
    int stacksize = (16*1024 + sizeof(*t) + nbytes + 15) & ~15;

    NEW0(t);  // 分配并清零T结构体 (在栈底)
    t->sp = (void *)(((unsigned long)t + stacksize) & ~15U); // 栈顶

    t->handle = t;  // 标记为活跃线程

    // 将args复制到栈上
    if (nbytes > 0) {
        t->sp -= ((nbytes + 15U) & ~15) / sizeof(*t->sp);
        memcpy(t->sp, args, nbytes);
        args = t->sp;
    }

    // === 设置初始栈帧 (平台相关!) ===
    // 这是最精妙的部分: 在栈上伪造一个调用帧,
    // 使_swtch第一次切换到该线程时, 自动开始执行apply(args)
    #if (linux || __APPLE__) && i386
        t->sp -= 16/4;
        *t->sp = (unsigned long)_thrstart;   // 返回地址→线程启动函数
        t->sp -= 16/4;
        t->sp[1] = (unsigned long)apply;      // 参数: 函数指针
        t->sp[2] = (unsigned long)args;       // 参数: args
    #endif

    nthreads++;
    put(t, &ready);  // 加入就绪队列
    return t;
}
```

**栈布局** (x86 Linux):

```
栈底 (低地址)
┌──────────────┐
│ Thread_T     │  ← t (ALLOC返回的地址)
│ sp ──────┐   │
│ link      │   │
│ ...       │   │
├──────────────┤
│ args data   │  ← t->sp (对齐后)
├──────────────┤
│ (padding)   │
├──────────────┤
│ _thrstart   │  ← 初始 "返回地址"
├──────────────┤
│ apply       │  ← 函数指针
│ args        │  ← 参数
│ sp          │
├──────────────┤
│ 栈空间      │
│   ...       │
│   ...       │
│   (16KB)    │
└──────────────┘
栈顶 (高地址)
```

### 4.2 Thread_exit

```c
void Thread_exit(int code) {
    release();  // 释放freelist中的线程内存

    if (current != &root) {
        current->next = freelist;  // 加入freelist (延迟释放!)
        freelist = current;
    }
    current->handle = NULL;  // 标记为已退出

    // 唤醒所有join本线程的线程
    while (!isempty(current->join)) {
        T t = get(&current->join);
        t->code = code;            // 传递退出码
        put(t, &ready);
    }

    // 如果只有root线程和一个等待join0的线程了:
    if (!isempty(join0) && nthreads == 2) {
        put(get(&join0), &ready);
    }

    if (--nthreads == 0)
        exit(code);   // 所有线程结束 → 终止进程
    else
        run();        // 切换到下一个线程
}
```

### 4.3 抢占式调度 (可选)

```c
int Thread_init(int preempt, ...) {
    if (preempt) {
        // 设置SIGVTALRM信号: 每10ms触发一次
        struct sigaction sa;
        sa.sa_handler = (void (*)())interrupt;
        sigaction(SIGVTALRM, &sa, NULL);

        struct itimerval it;
        it.it_value.tv_usec    = 10;   // 初始10微秒
        it.it_interval.tv_usec = 10;   // 每10微秒
        setitimer(ITIMER_VIRTUAL, &it, NULL);
    }
}

static int interrupt(int sig, struct sigcontext sc) {
    // 临界区保护: 如果在critical代码段或_MONITOR区域, 延迟切换
    if (critical ||
        (sc.eip >= (unsigned long)_MONITOR &&
         sc.eip <= (unsigned long)_ENDMONITOR))
        return 0;

    put(current, &ready);  // 当前线程放回就绪队列
    run();                  // 切换到下一个线程
    return 0;
}
```

**抢占保护的两种机制:**
1. `critical` 计数: `do { critical++; ... critical--; } while(0)` 包裹关键代码
2. `_MONITOR`/`_ENDMONITOR` 区间: 信号处理函数检查EIP是否在此区间

---

## 五、信号量 (Sem_T)

### 5.1 实现

```c
#define T Sem_T
typedef struct T {
    int count;       // 信号量计数 (>=0)
    void *queue;     // 等待队列 (实际上是Thread_T的循环链表)
} T;

void Sem_wait(T *s) {   // P操作
    testalert();         // 检查是否被alert中断
    if (s->count <= 0) {
        put(current, (Thread_T *)&s->queue);  // 阻塞当前线程
        run();                                // 切换
        testalert();                          // 被唤醒后再次检查
    } else
        --s->count;      // count>0: 直接通过
}

void Sem_signal(T *s) {  // V操作
    if (s->count == 0 && !isempty(s->queue)) {
        Thread_T t = get((Thread_T *)&s->queue);
        put(t, &ready);  // 唤醒一个等待线程
    } else
        ++s->count;       // 没有等待者: 增加计数
}
```

### 5.2 LOCK宏

```c
#define LOCK(mutex) do { Sem_T *_m = &(mutex); Sem_wait(_m);
#define END_LOCK Sem_signal(_m); } while (0)

// 使用:
Sem_T mutex;
Sem_init(&mutex, 1);  // 初始化为1 → 互斥锁
LOCK(mutex)
    // 临界区代码
END_LOCK;
```

---

## 六、Modern-C 改进

### 6.1 C11 threads.h 对比

```c
// CII Thread
Thread_new(worker, args, sizeof(args));
Sem_wait(&sem);

// C11 threads.h
#include <threads.h>
thrd_t t;
thrd_create(&t, worker, args);
mtx_lock(&mtx);
```

### 6.2 现代项目推荐

| 场景 | 推荐 |
|------|------|
| 生产环境并发 | pthread (Unix) / Win32 threads |
| 轻量级并发 | libdill / libmill (协程) |
| C11移植性 | `<threads.h>` (功能有限) |
| 学习并发原理 | CII Thread (代码量小, 概念完整) |

---

## 七、完整示例

```c
#include "thread.h"
#include "sem.h"
#include <stdio.h>

Sem_T mutex;
int counter = 0;

int worker(void *arg) {
    int id = *(int *)arg;
    for (int i = 0; i < 10; i++) {
        LOCK(mutex)
            counter++;
            printf("Thread %d: counter=%d\n", id, counter);
        END_LOCK;
        Thread_pause();  // 让出CPU
    }
    return id;
}

int main() {
    Thread_init(0);  // 协作式调度
    Sem_init(&mutex, 1);

    int id1 = 1, id2 = 2;
    Thread_T t1 = Thread_new(worker, &id1, sizeof(id1));
    Thread_T t2 = Thread_new(worker, &id2, sizeof(id2));

    Thread_join(t1);
    Thread_join(t2);

    printf("Final counter: %d\n", counter);  // 20
    return 0;
}
```
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch19_多精度计算器_Calc.html">第19章 多精度计算器 Calc</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch21_通道_Channel.html">第21章 通道 Channel</a></div>
</div>
