# 第4章 异常与断言 (Exceptions and Assertions)

> 源码文件: `code/except.h`, `code/except.c`, `code/assert.h`, `code/assert.c`
> 原文: C Interfaces and Implementations, David R. Hanson, Chapter 4

---

## 4.1 C语言中模拟异常处理的挑战

C语言没有内建的异常处理机制。C++和Java将 `throw`/`catch` 作为语言的一等公民——当异常抛出时，运行时系统自动执行栈展开（stack unwinding），依次调用每个栈帧中对象的析构函数，直到找到匹配的 `catch` 块。这一过程依赖于编译器生成的异常表（exception tables）、landing pads 和 LSDA（Language-Specific Data Area）。

C语言不提供以上任何设施。通常的错误处理方式只有三种：

| 方式 | 缺点 |
|------|------|
| `return -1` / `return NULL` | 调用者必须主动检查返回值, 遗漏即bug；错误沿调用链逐层手动传播, 侵入性强 |
| `errno` + `perror` | 全局状态, 多线程环境下需 `errno` 线程局部化 (POSIX)，且错误码语义模糊 |
| `exit` / `abort` | 过于粗暴, 无法在调用栈上层恢复 |

CII 库的核心洞察是：**异常处理的本质是一个非局部跳转问题**——从深层嵌套的调用栈跳回某个上层调用点。C89/C99标准库恰好提供了 `setjmp`/`longjmp` 这对非局部跳转原语。在此之上，只需构建一个**栈式异常帧管理器**，配合宏预处理器即可形成一套简洁、可移植的异常处理框架。

---

## 4.2 异常数据结构深度分析

### 4.2.1 Except_T —— 异常类型（except.h: 第6-8行）

```c
#define T Except_T                            // 宏T临时别名Except_T, reduce typing verbosity
typedef struct T {
    const char *reason;                      // 异常的描述字符串, 可为NULL
} T;
```

`Except_T` 是所有异常的基类型。与C++中 `std::exception` 的继承层次不同，这里没有子类型体系——所有异常共享同一结构体，仅通过 `reason` 字段和结构体**地址**来区分类型。

**关键设计决策 —— 地址比较作为类型标签:**

```c
const Except_T Assert_Failed  = { "Assertion failed" };   // assert.c 第3行
const Except_T Mem_Failed     = { "Allocation failed" };   // mem.h 中引用
```

每个异常类型是一个**全局 `const` 对象**。在 `EXCEPT(e)` 宏中，通过 `&(e)` 取地址来识别异常类型（而非通过字符串比较或枚举）。这是一种极简主义方法——不需要RTTI、不需要类型注册表、不引入字符串比较开销。每一个异常对象就是一个全局常量，其地址在编译链接时就已确定。

### 4.2.2 Except_Frame —— 异常帧栈节点（except.h: 第9-16行）

```c
typedef struct Except_Frame Except_Frame;    // 前向声明, 使prev指针可引用自身类型
struct Except_Frame {
    Except_Frame *prev;                      // 指向上一个异常帧, 形成单向链式栈
    jmp_buf       env;                       // setjmp/longjmp的跳转缓冲区 (通常128-256字节)
    const char   *file;                      // 异常抛出点的源文件名 (由RAISE宏的__FILE__填充)
    int           line;                      // 异常抛出点的行号 (由RAISE宏的__LINE__填充)
    const T      *exception;                 // 当前捕获或抛出的异常对象指针
};
```

**设计分析 —— 三核心原则:**

1. **零堆分配 (Zero Heap Allocation):** `Except_Frame` 始终分配在调用栈上（作为 `TRY` 块中的局部变量），不需要 `malloc`。它天然地与外围代码块的生命周期绑定，无内存泄漏风险。

2. **线程安全基础:** 栈上分配天然与线程绑定。在Unix版本中，`Except_stack` 指向当前线程调用栈上的帧；在Windows版本中，TLS机制使每个线程维护自己独立的帧链。

3. **信号安全基础 (Signal Safety):** 没有动态内存分配使得该机制在信号处理器中有条件地可用（虽然仍需谨慎处理 `longjmp` 在信号上下文中的平台差异）。

**Except_Frame 栈的链式结构 (Unix版本):**

```
调用栈方向 (地址递减)
 ←──────────────────────────────────────────────────────────
  栈底 (高地址)                          栈顶 (低地址)
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ TRY frame 1 │←───│ TRY frame 2 │←───│ TRY frame 3 │←── Except_stack (全局指针)
  │ prev ───────┘    │ prev ───────┘    │ prev = NULL │
  │ env          │   │ env          │   │ env          │
  │ exception    │   │ exception    │   │ exception    │
  └─────────────┘    └─────────────┘    └─────────────┘
     (最外层)          (中间层)            (最内层/当前)
```

`Except_stack` 始终指向当前**最内层** `TRY` 块的帧。每个帧的 `prev` 指针指回外层帧。这是一种**栈上帧的链式叠加**——物理上在调用栈上，逻辑上形成异处理栈。

### 4.2.3 异常状态枚举 —— 四种状态的状态机（except.h: 第17-18行）

```c
enum { Except_entered=0, Except_raised,
       Except_handled,   Except_finalized };
```

这四种状态构成了异常处理生存期的完整状态机:

```
                     setjmp 首次返回
  Except_entered (0) ────────────────────→ [用户TRY块代码执行中]
       │                                          │
       │                                          │ RAISE(e) 被调用
       │                                          │ 执行 longjmp(p->env, Except_raised)
       │                                          ▼
       │                                  Except_raised (1)
       │                                          │
       │                             ┌────────────┼────────────┐
       │                             │ 匹配EXCEPT  │ 不匹配     │
       │                             ▼             │            │
       │                     Except_handled (2)     │            │
       │                          (异常被处理)       │            │
       │                                           ▼            │
       │                                    到达END_TRY        │
       │                                    RERAISE ──────────┘
       │                                          │
       │  正常执行到FINALLY                        │ 异常执行到FINALLY
       ▼                                          ▼
  Except_finalized (3)                   Except_raised (1)
  (FINALLY块已执行)                      (FINALLY块已执行, 异常保留)
       │                                          │
       ▼                                          ▼
  到达END_TRY                             到达END_TRY
  不RERAISE (退出)                        RERAISE (继续传播)
```

**状态语义表:**

| 状态 | 值 | 语义 | 触发时机 |
|------|-----|------|----------|
| `Except_entered` | 0 | 正常执行中，无异常 | `setjmp` 首次返回 (返回值 = 0) |
| `Except_raised` | 1 | 异常已抛出，待处理 | `longjmp(p->env, 1)` 跳回时 |
| `Except_handled` | 2 | 异常已被某个 `EXCEPT(e)` 捕获 | `EXCEPT(e)` 匹配成功后设置 |
| `Except_finalized` | 3 | `FINALLY` 块在执行中 | 正常路径进入 `FINALLY` 时设置 |

---

## 4.3 setjmp/longjmp 的工作原理与深层限制

### 4.3.1 工作机制

```c
#include <setjmp.h>
int  setjmp(jmp_buf env);       // 保存当前执行上下文到 env
void longjmp(jmp_buf env, int val);  // 恢复到 env 保存的上下文
```

`setjmp(env)` 保存当前执行上下文到 `env` 中。具体保存内容（取决于平台和ABI）包括:

- **程序计数器 (PC / EIP / RIP):** 当前执行位置
- **栈指针 (SP / ESP / RSP):** 当前栈顶位置
- **帧指针 (BP / EBP / RBP):** 当前栈帧基址
- **被调用者保存寄存器 (Callee-saved registers):** x86-64 下包括 RBX, RBP, R12-R15；ARM64 下包括 X19-X28
- **信号掩码 (Signal Mask):** 使用 `sigsetjmp` 时保存 (CII 使用标准 `setjmp`，不保存信号掩码)

当 `setjmp` 被**首次调用**时，返回 `0`（即代码中的 `Except_entered`）。这是判断"我们是刚进入 TRY 块"还是"被 longjmp 弹回来"的唯一依据。

`longjmp(env, val)` 将所有上述寄存器**恢复到 `setjmp` 保存时的值**，使程序执行流跳回 `setjmp` 调用点，并使 `setjmp` **看起来像是刚返回了 `val`**。注意：`longjmp` **不会**调用栈上任何中间函数正常的返回清理代码（即标准C函数的 `leave` / `ret` 序列）。

### 4.3.2 关键限制及其影响

**限制1 —— 不展开栈帧 (No Stack Unwinding)**

`longjmp` 直接跳转到目标位置，中间所有函数调用帧被丢弃，但不执行任何清理:

```c
void inner(void) {
    void *p = malloc(1024);    // (A) 成功分配
    RAISE(SomeError);          // (B) longjmp, 跳过 free(p)
    free(p);                   // (C) 永不执行 — 内存泄漏!
}

void outer(void) {
    TRY
        inner();
    EXCEPT(SomeError)
        // p 指向的内存已泄漏 — 这就是为什么需要 FINALLY 块
    END_TRY;
}
```

解决方案: 始终在 `FINALLY` 块中释放资源，或将分配操作提升到 `TRY` 块外部。

**限制2 —— `volatile` 变量的必要性 (C标准 7.13.2.1)**

C标准规定: 如果 `setjmp` 调用函数中的一个局部变量在 `setjmp` 和 `longjmp` 之间被修改，且该变量满足以下条件，则它的值在 `longjmp` 后是**不确定的**:
- 不是 `volatile` 限定的
- 存储在寄存器中（而非内存中）

这就是为什么 `Except_flag` **必须**声明为 `volatile int`:

```c
// 正确: volatile 确保 longjmp 后值被可靠恢复
volatile int Except_flag;
Except_flag = setjmp(Except_frame.env);

// 错误: 没有 volatile, longjmp 后 Except_flag 的值未定义
int Except_flag;  // BUG: 编译器可能将其缓存在寄存器中
```

**限制3 —— `setjmp` 只能出现在有限的上下文中 (C89 7.6.2.1 / C99 7.13.1.1)**

```
允许的上下文:
  (1) if (setjmp(env) == 0) { ... }           // if控制表达式
  (2) switch (setjmp(env)) { case 0: ... }     // switch控制表达式
  (3) while (setjmp(env) == 0) { ... }         // while控制表达式
  (4) int x = setjmp(env);                     // 表达式语句 (仅C99+)
  (5) if (!setjmp(env)) { ... }                // 一元!操作

不允许的上下文:
  int x = setjmp(env) + 1;                     // 赋值后参与算术运算
  foo(setjmp(env));                            // 作为函数参数
  if ((x = setjmp(env)) == 0) { ... }          // 赋值后比较 (C89)
```

CII 的所有宏展开都遵循这些限制。`Except_flag = setjmp(...)` 是合法的表达式语句。

**限制4 —— `longjmp` 到已失效的 `jmp_buf`**

如果 `setjmp` 所在的函数已经返回，那么 `jmp_buf env` 中保存的栈帧已经失效（被后续函数调用覆盖），此时 `longjmp` 是**未定义行为**。CII 通过以下机制确保安全性:

1. `Except_raise` 在执行 `longjmp` **之前**先弹出当前帧 (`Except_stack = Except_stack->prev`)
2. `RETURN` 宏在函数返回前也弹出当前帧
3. 用户代码中手动 `return`（而非 `RETURN`）是产生 dangling frame 的主要风险来源

---

## 4.4 跨平台条件编译分析 (`#ifdef WIN32`)

`except.h` 使用 `#ifdef WIN32` 将代码分为两个功能等价但实现迥异的版本。这种条件编译策略是经典的系统编程惯用法——通过宏开关统一接口，隐藏平台差异。

### 4.4.1 两版本核心差异对照

| 操作 | Unix/Linux (`#else` 分支) | Windows (`#ifdef WIN32` 分支) |
|------|--------------------------|------------------------------|
| 栈顶指针 | 全局变量 `Except_stack` | TLS 槽位 `Except_index` (通过 `TlsGetValue`/`TlsSetValue` 访问) |
| 初始化 | 静态初始化 `Except_stack = NULL` | 惰性初始化 `Except_init()` 调用 `TlsAlloc()` |
| 推入帧 | `Except_stack = &Except_frame` (直接赋值) | `Except_push(&Except_frame)` (TLS封装) |
| 弹出帧 | `Except_stack = Except_stack->prev` (直接赋值) | `Except_pop()` (TLS封装) |
| 栈为空判断 | `Except_stack == NULL` | `TlsGetValue(Except_index) == NULL` |
| 线程安全 | **不**安全 (全局变量在并发访问下竞争) | **安**全 (每条线程独立的TLS存储) |

### 4.4.2 全局声明差异

```c
// except.h 第22-29行: Windows版本暴露 TLS 操作函数
#ifdef WIN32
#include <windows.h>
extern int Except_index;                      // TLS槽索引, -1表示未初始化
extern void Except_init(void);                // 分配TLS槽 (首次使用时惰性调用)
extern void Except_push(Except_Frame *fp);    // 将帧推入当前线程的异常栈
extern void Except_pop(void);                 // 弹出当前线程异常栈的栈顶帧
#endif
```

Windows版本需要暴露 `Except_push`/`Except_pop` 函数，因为 `TRY`/`EXCEPT`/`FINALLY`/`END_TRY` 宏都调用它们。Unix版本由于直接操作全局变量，不需要这些封装函数。

### 4.4.3 用户代码透明性

无论哪个平台，用户代码完全一致:

```c
TRY
    x = y / z;
EXCEPT(Arithmetic_Error)
    handle_division_by_zero();
END_TRY;
```

这正是条件编译的价值——将平台差异封装在宏和少量辅助函数中，上层接口保持干净。

---

## 4.5 TRY/EXCEPT/ELSE/FINALLY/END_TRY 宏展开的逐行分析

以下以**Windows版本**为主线进行逐行分析（Unix版本仅在不同处标注）。每一行代码附带中文注释解释其作用和设计意图。

### 4.5.1 TRY 宏 —— Windows 版本（except.h: 第36-43行）

```c
#define TRY do {                                    \
    volatile int Except_flag;                       \
    Except_Frame Except_frame;                      \
    if (Except_index == -1)                         \
        Except_init();                              \
    Except_push(&Except_frame);                     \
    Except_flag = setjmp(Except_frame.env);         \
    if (Except_flag == Except_entered) {
```

**逐行分析:**

| 行 | 代码 | 中文注释 |
|----|------|----------|
| 1 | `#define TRY do { \` | 宏定义开始。`do { ... } while(0)` 是经典宏包装惯用法: (a) 确保宏作为一个完整语句使用, (b) 需要尾随分号以保持语法一致性, (c) 避免 `if/else` 悬挂问题 |
| 2 | `volatile int Except_flag; \` | 声明局部状态变量。**`volatile` 修饰符至关重要** — 没有它，`longjmp` 跳回后该变量的值是不确定的（C标准 7.13.2.1）。编译器可能将普通局部变量缓存在寄存器中，而 `longjmp` 恢复的是 `setjmp` 时的寄存器快照 |
| 3 | `Except_Frame Except_frame; \` | 在**栈**上创建异常帧（不是堆）。帧的生命周期自动绑定到外围代码块，无需手动 `free`。这是设计中最精妙之处 |
| 4 | `if (Except_index == -1) \` | 检查 TLS 槽位是否已初始化。`-1` 是全局初始值，作为"未初始化"哨兵 |
| 5 | `Except_init(); \` | 惰性初始化: 仅在首次使用时调用 `TlsAlloc()` 分配 TLS 索引。避免在程序启动时分配资源（可能有依赖顺序问题） |
| 6 | `Except_push(&Except_frame); \` | 调用 `TlsSetValue()` 将当前帧链入线程的异常帧栈。`Except_push` 内部保存旧栈顶到 `fp->prev` |
| 7 | `Except_flag = setjmp(Except_frame.env); \` | **核心操作**: 保存当前CPU上下文。首次调用返回 `Except_entered` (0)；`longjmp` 跳转回来时，`setjmp` 看起来返回 `Except_raised` (1) |
| 8 | `if (Except_flag == Except_entered) {` | `setjmp` 首次返回时条件为真，控制流进入 TRY 块体。`longjmp` 跳回时条件为假，直接跳过 TRY 块体 |

**Unix版本对比 (except.h: 第66-72行):**

```c
#define TRY do { \
    volatile int Except_flag; \
    Except_Frame Except_frame; \
    Except_frame.prev = Except_stack;    /* 手动链入全局栈 */           \
    Except_stack = &Except_frame;         /* 成为新栈顶 */               \
    Except_flag = setjmp(Except_frame.env); \
    if (Except_flag == Except_entered) {
```

Unix版本直接将 `Except_frame` 推入全局栈（两行赋值），不调用任何函数。这更高效（无函数调用开销），但失去了线程安全性。

### 4.5.2 EXCEPT(e) 宏 —— Windows 版本（except.h: 第44-47行）

```c
#define EXCEPT(e) \
        if (Except_flag == Except_entered) Except_pop(); \
    } else if (Except_frame.exception == &(e)) { \
        Except_flag = Except_handled;
```

**逐行分析:**

| 行 | 代码 | 中文注释 |
|----|------|----------|
| 1 | `if (Except_flag == Except_entered) Except_pop(); \` | **正常路径**: 如果我们在 TRY 块正常执行（无异常抛出），在进入 EXCEPT 子句时弹出异常帧。这意味着 TRY 块已成功执行完毕 |
| 2 | `} else if (Except_frame.exception == &(e)) { \` | 关闭 TRY 块的 `{`，用 `else if` 检查异常匹配。`&(e)` 取异常对象地址进行比较 —— 这是一种**结构性类型比较**（通过地址而非内容）。注意: 只有当 `Except_flag != Except_entered`（即 `Except_raised`）时才会执行到此行 |
| 3 | `Except_flag = Except_handled;` | 异常匹配成功，标记 `Except_flag` 为 `Except_handled` (2)。此标记在 `END_TRY` 中被检查: 如果仍为 `Except_raised`，将执行 `RERAISE` |

**设计精妙之处: `else if` 的两条互斥路径**

`except.h` 中宏的结构创建了两条互斥路径，完全由 `Except_flag` 的值驱动:

```
Except_flag == Except_entered (0)          Except_flag == Except_raised (1)
         │                                           │
         │ 进入此分支                                  │ 进入此分支 (setjmp被longjmp跳回)
         ▼                                           ▼
  if (Except_flag == 0)                     else if (exception == &(e))
      Except_pop();  // 正常退出                   Except_flag = Except_handled;
      // 然后跳过EXCEPT块                          // 然后执行EXCEPT块体
```

这种结构利用 `if/else if` 将 `setjmp` 的两种返回路径合并在同一个代码块中，而无需任何显式的控制流跳转。

**Unix版本对比 (except.h: 第73-76行):**

```c
#define EXCEPT(e) \
        if (Except_flag == Except_entered) Except_stack = Except_stack->prev; \
    } else if (Except_frame.exception == &(e)) { \
        Except_flag = Except_handled;
```

Unix版本用 `Except_stack = Except_stack->prev` 替代 `Except_pop()`，直接操作全局指针。

### 4.5.3 ELSE 宏 —— Windows 版本（except.h: 第48-51行）

```c
#define ELSE \
        if (Except_flag == Except_entered) Except_pop(); \
    } else { \
        Except_flag = Except_handled;
```

**语义与 EXCEPT(e) 的差异:**

`ELSE` 是 `EXCEPT(e)` 的无条件版本 —— 不进行 `&(e)` 地址比较，捕获**所有**未被前面 `EXCEPT(e)` 匹配的异常。它关闭了前一个 `EXCEPT(e)` 的 `else if` 块，开启一个新的 `else` 块。

```
展开后的 if/else if/else 链:

if (Except_flag == 0) {         ← TRY 块体
    ... 用户代码 ...
    Except_pop();               ← 正常路径: EXCEPT 处弹出
} else if (exception == &(e1)) {  ← 第一个 EXCEPT
    Except_flag = Except_handled;
    ... handler 1 ...
    Except_pop();               ← 正常路径: ELSE 处弹出
} else {                        ← ELSE 块
    Except_flag = Except_handled;
    ... catch-all handler ...
}
```

### 4.5.4 FINALLY 宏 —— Windows 版本（except.h: 第52-56行）

```c
#define FINALLY \
        if (Except_flag == Except_entered) Except_pop(); \
    } { \
        if (Except_flag == Except_entered) \
            Except_flag = Except_finalized;
```

**FINALLY 是整套宏设计中最为精妙的部分。**它解决了 core 问题: `finally` 块不论是否有异常都必须执行。

**核心机制 —— 两个连续花括号 `} {`:**

```
前面的宏结束于:  } else { ... }      ← EXCEPT/ELSE 链
FINALLY 开始于:  } {                ← 注意: 没有 else 关键字!
```

这两个连续的花括号创建了一个**独立代码块**，它不属于 `if/else if/else` 链，因此**总会执行**。

**三种执行场景分析:**

**场景A —— 无异常，无EXCEPT匹配:**
```
Except_flag = Except_entered (0)
  → TRY 块正常执行
  → FINALLY: if (Except_flag == 0) Except_pop();  ← 弹出帧
  → FINALLY: { if (Except_flag == 0) Except_flag = Except_finalized;  ← 标记已finalized
  → FINALLY 块内容执行 (Except_flag = 3)
  → END_TRY: Except_flag != Except_raised → 不RERAISE
```

**场景B —— 有异常，被EXCEPT捕获:**
```
Except_flag = Except_raised → 被EXCEPT匹配 → Except_flag = Except_handled
  → FINALLY: if (Except_flag == 0) … 不执行 ← Except_flag = 2, 不弹出帧
  → FINALLY: { if (Except_flag == 0) … 不执行 ← 不改变flag
  → FINALLY 块内容执行 (Except_flag = 2)
  → END_TRY: Except_flag != Except_raised → 不RERAISE
```

**场景C —— 有异常，未被捕获:**
```
Except_flag = Except_raised (1), 无匹配的EXCEPT
  → FINALLY: if (Except_flag == 0) … 不执行
  → FINALLY: { if (Except_flag == 0) … 不执行
  → FINALLY 块内容执行 (Except_flag 保持=1)
  → END_TRY: if (Except_flag == 1) RERAISE;  ← 重新抛出!
```

关键洞察: FINALLY块通过将 `Except_flag` 设置为 `Except_finalized` (3)（仅在正常路径下），防止了正常完成时误触发 `RERAISE`。

### 4.5.5 END_TRY 宏 —— Windows 版本（except.h: 第57-60行）

```c
#define END_TRY \
        if (Except_flag == Except_entered) Except_pop(); \
        } if (Except_flag == Except_raised) RERAISE; \
} while (0)
```

**逐行分析:**

| 行 | 代码 | 中文注释 |
|----|------|----------|
| 1 | `if (Except_flag == Except_entered) Except_pop(); \` | 处理仅含 TRY + FINALLY (无 EXCEPT) 的情况。如果代码正常执行到此处（`Except_flag == 0`），弹出异常帧（因为 FINALLY 尚未弹栈） |
| 2 | `} if (Except_flag == Except_raised) RERAISE; \` | 关闭 FINALLY 块 (或 ELSE/EXCEPT 块) 的 `}`。然后 **最关键的一行**: 检查 `Except_flag` 是否仍为 `Except_raised` (1)，如果是则调用 `RERAISE` 将异常传播给外层 TRY 块 |
| 3 | `} while (0)` | 关闭 `do { ... } while(0)` 包装。用户调用 `END_TRY;` 时分号由用户提供 |

**`RERAISE` 在 END_TRY 中的重新抛出逻辑:**

```
END_TRY 到达时 Except_flag 的可能值:
  = Except_finalized (3): 正常执行完毕 + FINALLY → 不RERAISE ✓
  = Except_handled (2): 异常已被捕获 → 不RERAISE ✓
  = Except_raised (1): 异常未被处理 → RERAISE! (传播给外层)
  = Except_entered (0): (在FINALLY/ELSE中已被处理，不会到达此处)
```

### 4.5.6 Unix版本完整对照（except.h: 第61-90行）

Unix版本所有宏的结构与Windows版本完全一致，唯一的系统差异在于栈操作:

```c
/* ========================== Unix 版本 ========================== */
#define RAISE(e) Except_raise(&(e), __FILE__, __LINE__)
#define RERAISE Except_raise(Except_frame.exception, \
    Except_frame.file, Except_frame.line)
#define RETURN switch (Except_stack = Except_stack->prev,0) default: return
#define TRY do { \
    volatile int Except_flag; \
    Except_Frame Except_frame; \
    Except_frame.prev = Except_stack;    /* 手动链入 */           \
    Except_stack = &Except_frame;         /* 设为栈顶 */           \
    Except_flag = setjmp(Except_frame.env); \
    if (Except_flag == Except_entered) {
#define EXCEPT(e) \
        if (Except_flag == Except_entered) Except_stack = Except_stack->prev; \
    } else if (Except_frame.exception == &(e)) { \
        Except_flag = Except_handled;
#define ELSE \
        if (Except_flag == Except_entered) Except_stack = Except_stack->prev; \
    } else { \
        Except_flag = Except_handled;
#define FINALLY \
        if (Except_flag == Except_entered) Except_stack = Except_stack->prev; \
    } { \
        if (Except_flag == Except_entered) \
            Except_flag = Except_finalized;
#define END_TRY \
        if (Except_flag == Except_entered) Except_stack = Except_stack->prev; \
        } if (Except_flag == Except_raised) RERAISE; \
} while (0)
```

---

## 4.6 RAISE / RERAISE / RETURN 宏详解

### 4.6.1 RAISE(e) — 抛出异常（except.h: 第62行）

```c
#define RAISE(e) Except_raise(&(e), __FILE__, __LINE__)
```

- `&(e)`: 传递异常对象的**地址**，使 `EXCEPT(e)` 可以通过地址比较 (`== &(e)`) 识别异常类型
- `__FILE__`: ANSI C 预定义宏，展开为当前源文件名（字符串字面量）
- `__LINE__`: ANSI C 预定义宏，展开为当前行号（整型常量）

每次调用 `RAISE(e)` 自动记录抛出点的文件和行号，这对于调试和错误报告至关重要。这种"编译时自省"是 C 预处理器与运行时错误处理相结合的优雅范例。

### 4.6.2 RERAISE — 重新抛出（except.h: 第63-64行）

```c
#define RERAISE Except_raise(Except_frame.exception, \
    Except_frame.file, Except_frame.line)
```

`RERAISE` 与 `RAISE` 的关键区别：
- `RAISE(e)` 创建**新的**异常信息 (指定新的异常对象，记录当前 `__FILE__` / `__LINE__`)
- `RERAISE` **复用**当前异常帧中已记录的异常信息 (异常类型、文件、行号均保持不变)

这意味着:
1. 外层的 `EXCEPT` 仍能正确匹配 `exception == &(SomeException)` — 因为 `exception` 指针未变
2. 错误报告中的位置信息指向**原始抛出点**，而非重新抛出处 — 这对调试至关重要

### 4.6.3 RETURN — 安全返回（except.h: Windows第65行, Unix第65行）

```c
/* Windows */
#define RETURN switch (Except_pop(),0) default: return

/* Unix */
#define RETURN switch (Except_stack = Except_stack->prev,0) default: return
```

**问题背景:**

`Except_Frame` 是 `TRY` 块内的**栈上局部变量**。如果在 `TRY` 块中执行普通的 `return`:

```c
TRY
    if (early_exit)
        return 42;   // BUG: Except_stack 仍指向即将失效的 Except_frame!
EXCEPT(e)
    ...
END_TRY;
```

当 `return` 执行后，`Except_frame` 所在栈帧被销毁，但 `Except_stack` 仍指向它。之后的任何 `RAISE` 操作会访问悬垂指针（dangling pointer），导致未定义行为（通常是崩溃）。

**`RETURN` 宏的解决方案:**

`switch (expr, 0) default: return` 是一个巧妙的 C 惯用法:

```c
RETURN 42;
// 展开为:
switch (Except_pop(), 0) default: return 42;
// 等价于:
Except_pop();         // ← 先执行: 弹出当前异常帧
return 42;            // ← 再返回: 此时栈已恢复安全状态
```

关键点:
- `(Except_pop(), 0)` 是**逗号操作符**: 左侧表达式被求值（副作用是弹出异常帧），整个表达式的值是右侧的 `0`
- `switch (0)` 确保 `default:` 标签总是被执行
- 这种模式将"返回前清理"封装在 `return` 语句中，用户只需写 `RETURN val` 而非 `return val`

---

## 4.7 Except_Frame 栈结构详解

### 4.7.1 栈的生命周期

CII 的异常帧栈是一个**逻辑栈**，由 `Except_Frame.prev` 指针链构成。关键特性:

1. **物理上是调用栈的一部分**: 每个 `Except_Frame` 实例存在于其创建函数的栈帧中
2. **逻辑上是一个独立链表**: 通过 `prev` 指针将各调用层级的帧链接起来
3. **生命周期自动管理**: 当函数正常返回 (`RETURN`) 时，帧从逻辑栈中弹出；当 `longjmp` 发生时，`Except_raise` 在跳转前弹出帧

### 4.7.2 嵌套 TRY 块的栈动态

考虑以下嵌套结构:

```c
void h(void) {
    TRY                              // 帧A: 栈顶 = &帧A
        RAISE(E1);                   // 帧A被弹出, longjmp(帧A.env)
    EXCEPT(E1)
        TRY                          // 帧B: 栈顶 = &帧B (prev = 帧A.prev = NULL?)
            RAISE(E2);               // 帧B被弹出, longjmp(帧B.env)
        EXCEPT(E2)
            /* 处理 E2 */
        END_TRY;
    END_TRY;
}
```

**栈状态追踪:**

```
初始:    Except_stack = NULL

进入 TRY(h) 层:     Except_stack → 帧A (prev = NULL)
RAISE(E1):          帧A被填充异常信息 → Except_stack = NULL → longjmp(帧A.env, 1)
setjmp返回(帧A):    Except_flag = 1, 进入EXCEPT(E1) → Except_flag = 2

进入 TRY(内层):      Except_stack → 帧B (prev = NULL)
RAISE(E2):          帧B被填充异常信息 → Except_stack = NULL → longjmp(帧B.env, 1)
setjmp返回(帧B):    Except_flag = 1, 进入EXCEPT(E2) → Except_flag = 2

END_TRY(内层):      Except_flag=2 ≠ raised → 不RERAISE → while(0)结束
END_TRY(外层):      Except_flag=2 ≠ raised → 不RERAISE → while(0)结束
```

注意: 第一次 `RAISE(E1)` 后，`Except_stack` 变为 `NULL`。内层 TRY 块再进入时，`frameB.prev = NULL`。这是因为异常帧在 `Except_raise` 中已经被弹出。

---

## 4.8 Windows TLS 线程安全实现对比

### 4.8.1 Windows TLS 完整实现分析（except.c: 第42-72行）

```c
/* ========== TLS 索引 (全局, 所有线程共享) ========== */
int Except_index = -1;                    // 初始值 -1 = "未分配TLS索引"

/* ========== 惰性初始化 ========== */
void Except_init(void) {
    BOOL cond;
    Except_index = TlsAlloc();                          // 向OS申请TLS槽位索引
    assert(Except_index != TLS_OUT_OF_INDEXES);         // 系统TLS槽位有限 (通常64-1088个)
    cond = TlsSetValue(Except_index, NULL);             // 当前线程的TLS值初始化为NULL
    assert(cond == TRUE);
}

/* ========== 推入异常帧 ========== */
void Except_push(Except_Frame *fp) {
    BOOL cond;
    fp->prev = TlsGetValue(Except_index);               // 保存当前线程的旧栈顶
    cond = TlsSetValue(Except_index, fp);               // 当前线程的新栈顶 = fp
    assert(cond == TRUE);
}

/* ========== 弹出异常帧 ========== */
void Except_pop(void) {
    BOOL cond;
    Except_Frame *tos = TlsGetValue(Except_index);      // 当前线程的栈顶
    cond = TlsSetValue(Except_index, tos->prev);        // 回退到旧栈顶
    assert(cond == TRUE);
}
```

**TLS 安全模型:**

```
        全局 (进程级)
    ┌──────────────────────────┐
    │ Except_index = 2         │  ← 一个进程只有一个索引
    └──────────────────────────┘

     线程A                     线程B                     线程C
    ┌────────────────┐       ┌────────────────┐       ┌────────────────┐
    │ TLS[2] → 帧A3  │       │ TLS[2] → 帧B1  │       │ TLS[2] → NULL  │
    │       → 帧A2   │       │                │       │                │
    │       → 帧A1   │       │                │       │                │
    │       → NULL   │       │                │       │                │
    └────────────────┘       └────────────────┘       └────────────────┘
```

每个线程通过相同的 `Except_index` 访问自己的 TLS 槽位，但每个槽位中的数据是线程独立的。

**`Except_init` 的竞态条件分析:**

多个线程可能同时首次调用 `TRY`，导致并发执行 `Except_init`:

```
线程A: if (Except_index == -1)  ← 真
线程B: if (Except_index == -1)  ← 真 (A尚未设置)
线程A: Except_index = TlsAlloc();  // 分配索引=2
线程B: Except_index = TlsAlloc();  // 分配索引=3
```

结果: `Except_index` 最终为 `3`，索引 `2` 泄漏（永不使用）。这在实践中是可接受的，因为:
1. 泄漏只发生在程序启动时（一次性的）
2. 代价（多分配一个TLS槽位）远小于每次检查都加锁的代价
3. 所有线程最终使用同一个 `Except_index`（最后写入的值获胜）

### 4.8.2 Unix vs Windows 安全性对比

| 维度 | Unix 全局栈 | Windows TLS |
|------|-----------|-------------|
| 线程安全 | **不安全** — 多线程共享 `Except_stack` | **安全** — 每线程独立TLS存储 |
| 信号处理安全 | 谨慎可用 (无锁/无堆分配) | 谨慎可用 (TLS访问不分配内存) |
| fork 安全性 | **不安全** — 子进程继承父进程的 `Except_stack` | **不安全** — Windows 无 fork |
| 性能 | 极快 (单次指针赋值) | 稍慢 (API调用 + 内核模式TLS查找) |
| 代码复杂度 | 低 | 中 (需要初始化和封装函数) |

### 4.8.3 Windows CRT `_assert` 的替换（except.c: 第43-45行）

```c
_CRTIMP void __cdecl _assert(void *, void *, unsigned);
#undef assert
#define assert(e) ((e) || (_assert(#e, __FILE__, __LINE__), 0))
```

Windows版的`except.c`需要**额外覆盖**`assert`宏。原因: Windows CRT的默认`assert`行为是弹出消息对话框（非控制台程序）或打印到stderr。在CII的`except.c`自身内部（特别是`Except_init`等初始化函数中），如果断言失败，需要确保使用CRT原生的`_assert`而非CII的`RAISE`版本——因为此时异常处理框架本身可能尚未就绪。

`#e` 是 C 预处理器的"字符串化"操作符，将表达式 `e` 转换为字符串字面量。

---

## 4.9 Assert_Failed 异常的设计（assert.h, assert.c）

### 4.9.1 assert.h 接口（完整源码）

```c
/* $Id$ */
#undef assert                             // 取消标准库assert的定义 (可能来自<assert.h>)
#ifdef NDEBUG
#define assert(e) ((void)0)               // NDEBUG定义时: assert编译为空操作 (零开销)
#else
#include "except.h"
extern void assert(int e);                // 函数声明: 提供链接符号 (供外部模块引用)
#define assert(e) ((void)((e)||(RAISE(Assert_Failed),0)))
#endif
```

### 4.9.2 assert 宏展开分析

```c
#define assert(e) ((void)((e)||(RAISE(Assert_Failed),0)))
```

逐层分解（以 `assert(x != NULL)` 为例）:

```
assert(x != NULL)

  → ((void)((x != NULL) || (RAISE(Assert_Failed), 0)))

情况1: x != NULL 为真 (非零)
  → ((void)(1 || (RAISE(...), 0)))
  → || 短路求值: 右侧不执行
  → ((void)1)
  → 编译为空操作

情况2: x != NULL 为假 (0)
  → ((void)(0 || (RAISE(Assert_Failed), 0)))
  → || 不短路: 右侧求值
  → RAISE(Assert_Failed) → Except_raise(&(Assert_Failed), __FILE__, __LINE__)
  → longjmp 到最近的 TRY 块 (不返回)
  → 如果 longjmp 理论上返回: (RAISE(...), 0) = 0, ((void)0)
```

关键设计细节:
- **`(RAISE(...), 0)` 的逗号操作符**: 确保 `||` 右侧表达式整体求值为 `0`，满足 `||` 操作符的类型要求
- **`(void)` 转换**: 抑制编译器警告 "expression result unused"
- **短路求值**: 正常路径（断言通过）零额外开销

### 4.9.3 CII assert vs 标准C assert 对比

| 特性 | C 标准 `assert` (`<assert.h>`) | CII `assert` (`"assert.h"`) |
|------|-------------------------------|----------------------------|
| 失败行为 | `fprintf(stderr, ...)` + `abort()` | 抛出 `Assert_Failed` 异常 |
| 可捕获性 | **不可**捕获（程序终止） | **可**捕获（`EXCEPT(Assert_Failed)`） |
| NDEBUG 行为 | 编译为空 `((void)0)` | 编译为空 `((void)0)` |
| 函数符号 | 无外部函数符号 | `extern void assert(int e)` 供链接 |
| 嵌套使用 | N/A | 可通过 `TRY/EXCEPT` 包装以控制断言失败行为 |

### 4.9.4 assert.c 实现 —— 括号技巧

```c
static char rcsid[] = "$Id$";
#include "assert.h"

const Except_T Assert_Failed = { "Assertion failed" };

void (assert)(int e) {        // ★ 关键: 函数名加括号防止宏展开
    assert(e);                // 此处调用的是 assert 宏 (宏在函数体内展开)
}
```

**为什么用 `(assert)` 而不是 `assert`？**

C预处理器只在看到 `assert` 后紧跟 `(` 时才展开宏。`(assert)(int e)` 中的括号破坏了 `assert(` 的模式，使预处理器将 `(assert)` 视为普通标识符而非宏调用。这是一种经典的"宏抑制"技术。

这种设计同时提供了两种使用方式:
1. **宏形式**: `assert(condition)` — 直接内联展开，编译时常量断言可被完全优化掉
2. **函数形式**: `(assert)(condition)` — 通过函数指针调用，用于需要函数符号的场景

---

## 4.10 Except_raise 函数详解（except.c: 第8-41行）—— 异常抛出核心

```c
void Except_raise(const T *e, const char *file, int line) {
#ifdef WIN32
    Except_Frame *p;
    if (Except_index == -1)                           // 惰性初始化检查
        Except_init();
    p = TlsGetValue(Except_index);                    // 从TLS获取当前线程的异常栈顶
#else
    Except_Frame *p = Except_stack;                   // Unix: 直接读全局指针
#endif
    assert(e);                                        // 异常对象指针不能为NULL

    if (p == NULL) {                                  // 栈为空 → 没有TRY块捕获
        fprintf(stderr, "Uncaught exception");         //   打印 "Uncaught exception"
        if (e->reason)                                //   如果有reason字符串
            fprintf(stderr, " %s", e->reason);         //     打印reason
        else                                          //   否则
            fprintf(stderr, " at 0x%p", e);            //     打印异常对象的地址
        if (file && line > 0)                         //   如果有源文件信息
            fprintf(stderr, " raised at %s:%d\n",      //     打印抛出位置
                    file, line);
        fprintf(stderr, "aborting...\n");              //
        fflush(stderr);                               //   刷新stderr缓冲区
        abort();                                      //   终止程序 ← 不可恢复
    }

    p->exception = e;                                 // 将异常指针写入当前帧
    p->file = file;                                   // 将源文件名写入当前帧
    p->line = line;                                   // 将行号写入当前帧
#ifdef WIN32
    Except_pop();                                     // Win: TLS栈弹出 (删掉当前帧)
#else
    Except_stack = Except_stack->prev;                 // Unix: 全局栈回退 (删掉当前帧)
#endif
    longjmp(p->env, Except_raised);                   // 跳转回setjmp, 返回Except_raised(1)
}                                                     // ★ longjmp 不返回, 此行永不可达
```

**执行流程时序图:**

```
调用 RAISE(SomeError)
    ↓
Except_raise(&(SomeError), "file.c", 42)
    ↓
    [1] 获取当前栈顶帧 p = Except_stack (或 TlsGetValue)
    ↓
    [2] p == NULL? → 是 → 打印 "Uncaught exception" → abort()
    ↓  否
    [3] p->exception = &(SomeError)  ← 填写异常信息
    [4] p->file = "file.c"
    [5] p->line = 42
    ↓
    [6] Except_stack = p->prev  ← ★ 先弹出帧 (longjmp不返回, 此操作不可在之后执行)
    ↓
    [7] longjmp(p->env, 1)  ← 跳回 TRY 块中的 setjmp
    ↓
        (setjmp 此时返回 1, 即 Except_raised)
    ↓
        Except_flag = 1  (volatile 变量, 值被正确恢复)
    ↓
        if (Except_flag == 0) { ... }  ← 条件为假, 跳过TRY体
    ↓
        else if (p->exception == &(SomeError)) { ... }  ← 匹配, 进入EXCEPT块
```

**第6步（先弹出帧）的设计原因:**

`longjmp` 不返回到调用者，因此任何在 `longjmp` 之后的操作永远不会执行。弹出帧必须在 `longjmp` 之前完成。这样做有两个好处:
1. 异常帧栈保持正确状态（当前帧已被移除）
2. 外层的异常处理器可以正常工作（栈顶是外层的 TRY 块）

---

## 4.11 使用示例

### 4.11.1 基本异常抛出与捕获

```c
#include "except.h"
#include <stdio.h>

// 定义自定义异常: 必须是全局 const 对象
const Except_T Allocate_Failed = { "Allocation failed" };

void *allocate(long n) {
    void *p = malloc(n);
    if (p == NULL)
        RAISE(Allocate_Failed);    // 抛出异常: 自动记录 __FILE__, __LINE__
    return p;
}

void worker(void) {
    TRY
        void *buf = allocate(1024 * 1024);
        /* 使用 buf ... */
        free(buf);
    EXCEPT(Allocate_Failed)        // 地址比较: &(Allocate_Failed)
        fprintf(stderr, "Out of memory\n");
    END_TRY;                       // 必须分号
}
```

### 4.11.2 带 FINALLY 的资源清理

```c
void process_file(const char *path) {
    FILE *fp = NULL;

    TRY
        fp = fopen(path, "r");
        if (fp == NULL)
            RAISE(File_Open_Failed);    // 注意: 抛出后不执行 free/close

        char line[256];
        if (fgets(line, sizeof line, fp) == NULL)
            RAISE(Parse_Error);

        printf("First line: %s", line);

    EXCEPT(File_Open_Failed)
        fprintf(stderr, "Cannot open file: %s\n", path);

    EXCEPT(Parse_Error)
        fprintf(stderr, "Cannot read from file: %s\n", path);

    FINALLY
        if (fp) fclose(fp);            // ★ 无论是否异常, 都会执行

    END_TRY;
}
```

### 4.11.3 嵌套异常处理与异常转换

```c
const Except_T Inner_Error  = { "Inner error" };
const Except_T Outer_Error  = { "Outer error" };

void outer(void) {
    TRY
        TRY
            RAISE(Inner_Error);        // 抛出内部异常
        EXCEPT(Inner_Error)
            /* 捕获并转换为外层异常 */
            RAISE(Outer_Error);        // 重新抛出为外层异常
        END_TRY;
    EXCEPT(Outer_Error)
        printf("Caught outer error\n");
    END_TRY;
}
```

### 4.11.4 assert 使用

```c
#include "assert.h"   // CII 的 assert (不是标准库的)

int divide(int a, int b) {
    assert(b != 0);                    // 除数为零时抛出 Assert_Failed 异常
    return a / b;
}

void safe_caller(void) {
    TRY
        int result = divide(10, 0);   // 触发断言
        printf("Result: %d\n", result);
    EXCEPT(Assert_Failed)
        printf("Assertion failed: divisor was zero\n");
        // 程序不会 abort(), 可以优雅恢复
    END_TRY;
}
```

### 4.11.5 TRY 块中使用 RETURN 宏

```c
int search(int *arr, int n, int target) {
    TRY
        assert(arr != NULL);
        assert(n > 0);

        for (int i = 0; i < n; i++) {
            if (arr[i] == target)
                RETURN i;              // ★ 安全返回: 先弹出异常帧再return
        }
        RETURN -1;                     // 未找到: 安全返回

    EXCEPT(Assert_Failed)
        fprintf(stderr, "Invalid arguments\n");
        return -2;                     // EXCEPT块中可以正常return (帧已被except_raise弹出)
    END_TRY;

    return -3;                         // 不可达
}
```

---

## 4.12 Modern-C 改进讨论

### 4.12.1 C11 `_Static_assert` — 编译期断言

C11引入了`_Static_assert`关键字（通过`<assert.h>`提供宏`static_assert`），在**编译期**检查常量表达式:

```c
#include <assert.h>
static_assert(sizeof(void*) >= 8, "64-bit platform required");
static_assert(sizeof(int) == 4, "int must be 32 bits");
```

与CII运行时`assert`的关系是互补的:

| 特性 | `static_assert` (C11) | `assert` (CII) |
|------|----------------------|----------------|
| 检查时机 | **编译期** | **运行时** |
| 错误形式 | 编译错误 (不生成目标代码) | 抛出异常 (可被 TRY/EXCEPT 捕获) |
| 可用表达式 | 整数常量表达式 | 任意运行时表达式 |
| 性能开销 | **零** | NDEBUG下为零, 否则有分支开销 |
| 适用场景 | 平台检测/类型大小/编译时常量 | 函数参数校验/运行时条件/不变式 |

**Modern-C 推荐策略:**

```c
// 编译期检查用 static_assert
static_assert(sizeof(Except_Frame) <= 512, "Except_Frame too large for stack");

// 运行时检查用 CII assert
void *allocate(long n) {
    assert(n > 0);                    // 运行时参数校验, 可被捕获
    return malloc(n);
}
```

### 4.12.2 `errno` 作为替代方案

C标准库的`errno`是一种更轻量的错误报告机制:

```c
errno = 0;
long val = strtol(str, &endptr, 10);
if (errno == ERANGE) {
    /* 溢出处理 */
}
```

与CII异常的对比:

| 维度 | `errno` | CII 异常 |
|------|---------|----------|
| 错误传播 | 调用者手动检查 | 自动非局部传播 |
| 类型安全 | 全局 `int`，类型不安全 | 通过 `const Except_T*` 指针区分 |
| 侵入性 | 低 (不改变控制流) | 高 (需要 TRY/EXCEPT 包装) |
| 资源清理 | 手动在每个层级清理 | FINALLY 块集中处理 |
| 调试信息 | 仅 `errno` 值 + `strerror()` | 保留 file/line + 异常类型 |
| 性能 | 极低 (单次 errno 检查) | 中等 (`setjmp` 需保存大量寄存器) |

### 4.12.3 `setjmp`/`longjmp` 的深层局限性

**1. 信号处理器中的安全性**

`longjmp` 在信号处理器中的行为是平台定义的。POSIX.1 要求: 如果信号处理器是从某个被中断函数中调用的（且该函数在调用 `sigsetjmp` 时保存了信号掩码），那么从该信号处理器中调用 `siglongjmp` 是安全的。但 CII 使用的是普通 `setjmp`（不保存/恢复信号掩码），因此在信号处理器中使用 `RAISE` 需要额外验证。

**2. C++ 互操作性的致命问题**

在 C++ 中，`longjmp` 跳过具有非平凡析构函数的局部对象会导致**未定义行为** (C++17 21.10.4):

```cpp
// 危险: CII 异常 + C++ RAII 混用
TRY
    std::string s = "hello";   // C++ 对象, 有析构函数
    RAISE(SomeError);           // longjmp 跳过 s 的析构 → 未定义行为!
EXCEPT(SomeError)
    // ...
END_TRY;
```

在 C/C++ 混合项目中，不应同时使用 CII 异常和 C++ 异常/C++ RAII 对象。两者应严格分离（如在纯 C 模块中使用 CII 异常，在纯 C++ 模块中使用 C++ 异常）。

**3. 性能分析**

`setjmp` 在 x86-64 下的典型开销:
- 保存 10-15 个通用寄存器 (每个 8 字节)
- 保存 8-16 个 XMM 寄存器 (每个 16 字节，取决于ABI)
- 保存信号掩码 (`sigsetjmp`，约 128 字节)
- 总计约 200-600 字节的保存/恢复操作

这意味着 `TRY` 块的进入有非零开销。对于性能敏感的内循环，应避免使用 `TRY` 包装。

### 4.12.4 现代替代方案总览

| 方案 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| `setjmp`/`longjmp` (CII方案) | 零外部依赖, C89兼容 | 无栈展开, 易泄漏资源 | 嵌入式/内核/纯C项目 |
| C++ 异常 | 完整RAII, 自动栈展开 | 需要C++编译器, ABI复杂 | C++ 项目 |
| `libunwind` | 栈展开能力, 语言无关 | 平台依赖性高, 复杂API | 需要精确栈回溯 |
| 返回值 + `goto` cleanup | 零魔法, 可调试 | 代码冗长, 易遗漏检查 | 内核/关键安全代码 |
| `setcontext`/`getcontext` | 完整协程能力 | POSIX (2008) 已标记废弃 | 不建议新项目使用 |
| C11 `thread_local` + `errno` | 标准化, 简单 | 没有非局部传播 | 简单错误处理 |

---

## 4.13 设计模式与工程洞察

### 4.13.1 宏实现的 DSL (Domain-Specific Language)

`TRY/EXCEPT/ELSE/FINALLY/END_TRY/RAISE/RERAISE/RETURN` 构成了一个微小的领域特定语言。它通过宏展开为标准的 `if/else` + `setjmp/longjmp`，完全不需要编译器扩展。这种技术对 C 程序员的启示:

- 宏不仅仅是常量和内联函数的替代品
- 精心设计的宏可以创建全新的控制流结构
- 关键在于宏必须展开为**语法完整**的 C 结构（本设计中是 `do { if/else if/else { } } while(0)`）

### 4.13.2 栈上分配 + volatile 状态机

将异常帧分配在调用栈上（而非堆上）是本设计的核心。这与 C 语言的调用栈语义自然契合。`volatile` 变量穿越了 `setjmp`/`longjmp` 的语义断层，而状态机的四种状态（entered/raised/handled/finalized）用单个整型变量驱动了所有控制流分支。

### 4.13.3 地址比较作为类型标注

使用 `&(e)` 的地址作为异常类型的身份标识。不需要 RTTI、不需要字符串比较、不需要类型注册表。每个异常就是一个全局常量对象，其地址在编译时确定，在链接时解析，在运行时仅需一次指针比较。

### 4.13.4 平台抽象通过条件编译

`#ifdef WIN32` 将整个 `except.h` 分为两个功能等价但实现迥异的版本。这展示了条件编译做平台抽象的最佳实践——差异仅限于少数几行，99%的宏代码在平台间是共享的。

---

## 4.14 关键代码索引

| 组件 | 文件:行号 | 功能 |
|------|-----------|------|
| `Except_T` 定义 | `except.h:6-8` | 异常类型结构体 |
| `Except_Frame` 定义 | `except.h:10-16` | 异常帧栈节点 |
| 状态枚举 | `except.h:17-18` | 异常处理四状态 |
| `Except_stack` 全局 | `except.c:7` | Unix 全局栈顶 |
| `Except_index` | `except.c:47` | Windows TLS 槽位索引 |
| `RAISE` 宏 | `except.h:62` | 抛出异常 |
| `RERAISE` 宏 | `except.h:63-64` | 重新抛出 |
| `RETURN` 宏 (Unix) | `except.h:65` | 安全返回 |
| `RETURN` 宏 (Win) | `except.h:35` | 安全返回 |
| `TRY` 宏 (Unix) | `except.h:66-72` | Unix TRY 块 |
| `TRY` 宏 (Win) | `except.h:36-43` | Windows TRY 块 |
| `EXCEPT` 宏 (Unix) | `except.h:73-76` | 异常捕获 |
| `EXCEPT` 宏 (Win) | `except.h:44-47` | 异常捕获 |
| `ELSE` 宏 | `except.h:77-80` / `48-51` | 兜底捕获 |
| `FINALLY` 宏 | `except.h:81-86` / `52-56` | 最终清理 |
| `END_TRY` 宏 | `except.h:86-89` / `57-60` | TRY 结束 |
| `Except_raise` 函数 | `except.c:8-41` | 抛出核心实现 |
| `Except_init` | `except.c:48-54` | Windows TLS 初始化 |
| `Except_push` | `except.c:57-63` | Windows TLS 推入帧 |
| `Except_pop` | `except.c:65-71` | Windows TLS 弹出帧 |
| `assert` 宏 | `assert.h:8` | 断言宏 |
| `(assert)` 函数 | `assert.c:4-6` | 断言函数 (括号技巧) |
| `Assert_Failed` 对象 | `assert.c:3` | 断言失败异常 |

---

*本章基于《C Interfaces and Implementations》第4章原文及 `except.h` / `except.c` / `assert.h` / `assert.c` 完整源码逐行分析而成。所有宏展开均标注了每一行的设计意图与技术约束。Modern-C改进讨论基于作者对C11/C17标准的分析。*
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch03_原子_Atoms.html">第3章 原子 Atoms</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch05_内存管理I_Memory.html">第5章 内存管理 I</a></div>
</div>
