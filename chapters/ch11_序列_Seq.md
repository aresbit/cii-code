# 第11章 序列 (Sequences)

> **核心贡献**：序列是本书中最通用的 ADT 之一。它用一个动态数组实现了环形缓冲区，在保持 O(1) 随机访问的同时，同时支持双端 O(1) 插入和删除。一个 Seq 就可以充当数组、链表、栈、队列和双端队列，不需要为每种数据结构单独维护一个 ADT。

---

## 11.1 概述

序列 (Sequence) 持有 N 个值，值与整数索引 0 到 N-1 一一对应（N 为正值）。空序列不持有任何值。序列具有以下关键特性：

- 支持按索引随机访问（像 Array）
- 支持在任一端添加值（双端 push）
- 支持从任一端移除值（双端 pop）
- 容量自动扩展，无需使用者管理内存

尽管规范相对简单，序列却是本书中使用最广泛的 ADT。它可以替代数组、链表、栈、队列和双端队列，且通常能够完全取代为这些数据结构单独设计的 ADT。序列可以视为第 8 章动态数组（Array）的更抽象版本——它将内部的簿记（bookkeeping）和扩容细节隐藏在实现背后。

**值的类型**：序列中存储的值统一为 `void *` 指针。

---

## 11.2 接口 (Interface)

### 头文件：`seq.h`

```c
/* seq.h — 第11章 序列接口 */
#ifndef SEQ_INCLUDED
#define SEQ_INCLUDED
#define T Seq_T
typedef struct T *T;
```

`Seq_T` 是不透明指针类型（opaque pointer type），使用者无法看到其内部结构，从而实现了封装。通过 `#define T Seq_T` 的技巧，后续代码可以直接用 `T` 指代 `Seq_T`，这是一种减少冗长类型名的惯用法。

**运行时检查错误**：向本接口的任何函数传递 `NULL` 的 `T` 指针均属于受检查的运行时错误（checked runtime error）。

### 创建函数

```c
extern T Seq_new(int hint);
extern T Seq_seq(void *x, ...);
```

#### `Seq_new(int hint)`

创建一个空序列。`hint` 是预期最大元素数量的估计值。如果数量未知，可以传入 `hint = 0`，此时内部会创建一个较小的初始容量（默认 16）。不论 `hint` 为何值，序列都会在需要时自动扩容。`hint` 不得为负数（受检查的运行时错误）。可能抛出 `Mem_Failed` 异常。

#### `Seq_seq(void *x, ...)`

创建并返回一个已初始化的序列，元素值由非空指针参数提供。参数列表以第一个空指针作为终止标志。例如：

```c
Seq_T names;
names = Seq_seq("C", "ML", "C++", "Icon", "AWK", NULL);
```

这会创建一个包含 5 个值的序列，分别对应索引 0 到 4。可变参数部分中的指针被假定为 `void *` 类型，因此传递非 `char *` 或 `void *` 类型的指针时需要显式转型（强制类型转换）。可能抛出 `Mem_Failed` 异常。

### 析构函数

```c
extern void Seq_free(T *seq);
```

释放序列 `*seq` 所占用的所有内存，并将 `*seq` 清零（设为 `NULL`）。`seq` 本身或 `*seq` 为空指针均为受检查的运行时错误。

### 查询函数

```c
extern int Seq_length(T seq);
```

返回序列 `seq` 中当前存储的值的数量。

### 随机访问函数

```c
extern void *Seq_get(T seq, int i);
extern void *Seq_put(T seq, int i, void *x);
```

- **`Seq_get(seq, i)`**：返回序列中第 `i` 个值。索引 `i` 必须在 `[0, N-1]` 范围内（受检查的运行时错误）。**时间复杂度 O(1)**。

- **`Seq_put(seq, i, x)`**：将序列中第 `i` 个值替换为 `x`，并返回该位置原来的值。同样 O(1)。

这两项操作利用环形缓冲区的特性，通过 `(head + i) % array.length` 计算实际内存位置，因此保证了常数时间。

### 双端添加函数

```c
extern void *Seq_addlo(T seq, void *x);
extern void *Seq_addhi(T seq, void *x);
```

- **`Seq_addlo(seq, x)`**：将 `x` 添加到序列的**低端**（开头），并返回 `x`。添加后，所有已有元素的索引自动加 1，序列长度加 1。可能抛出 `Mem_Failed` 异常。

- **`Seq_addhi(seq, x)`**：将 `x` 添加到序列的**高端**（末尾），并返回 `x`。添加后序列长度加 1（已有元素的索引不变）。可能抛出 `Mem_Failed` 异常。

### 双端移除函数

```c
extern void *Seq_remlo(T seq);
extern void *Seq_remhi(T seq);
```

- **`Seq_remlo(seq)`**：移除并返回序列低端（开头）的值。移除后，剩余值的索引各减 1，序列长度减 1。

- **`Seq_remhi(seq)`**：移除并返回序列高端（末尾）的值。移除后序列长度减 1（剩余值的索引不变）。

向空序列调用这两个函数属于受检查的运行时错误。

### 完整接口一览

```c
#ifndef SEQ_INCLUDED
#define SEQ_INCLUDED
#define T Seq_T
typedef struct T *T;

extern T    Seq_new  (int hint);
extern T    Seq_seq  (void *x, ...);
extern void Seq_free (T *seq);
extern int  Seq_length(T seq);
extern void *Seq_get  (T seq, int i);
extern void *Seq_put  (T seq, int i, void *x);
extern void *Seq_addlo(T seq, void *x);
extern void *Seq_addhi(T seq, void *x);
extern void *Seq_remlo(T seq);
extern void *Seq_remhi(T seq);

#undef T
#endif
```

接口设计遵循 CII 全书一以贯之的风格：极简、正交（orthogonal）、用不透明指针封装实现细节。虽然功能涵盖了栈（push/pop = addhi/remhi）、队列（enqueue/dequeue = addhi/remlo）和双端队列的全部操作，却没有引入任何额外的类型或函数——一套 API 覆盖了四种数据结构的需求。

---

## 11.3 实现 (Implementation)

### 11.3.1 数据结构

序列的表示结构嵌入了一个 `Array_T` 结构体（而非指向 `Array_T` 的指针），利用了 C 语言中结构体首成员地址等于结构体自身地址的特性。

```c
#include <stdlib.h>
#include <stdarg.h>
#include <string.h>
#include "assert.h"
#include "seq.h"
#include "array.h"
#include "arrayrep.h"
#include "mem.h"

#define T Seq_T

struct T {
    struct Array_T array;   /* 内嵌的 Array_T 结构体 (动态数组) */
    int length;             /* 序列中实际存储的元素数量 */
    int head;               /* 第 0 号元素在底层数组中的起始索引 */
};
```

**三个字段详解**：

| 字段 | 含义 | 取值范围 |
|------|------|----------|
| `array` | 内嵌的 `Array_T` 结构体，其 `.array` 指向存放 `void *` 指针的内存块，`.length` 是该内存块可容纳的最大元素数 | `array.length` >= max(hint, 16)，且 >= `length` |
| `length` | 序列中实际存储的元素数量 | 0 <= `length` <= `array.length` |
| `head` | 环形缓冲区的起始偏移量——序列的第 0 号元素存储在底层数组的 `head` 位置 | 0 <= `head` < `array.length` |

**环形缓冲区（Circular Buffer）方案**：

底层数组被用作环形缓冲区来存储序列的值。第 0 号值存储在底层数组的第 `head` 号元素位置，后续值依次存储在 `(head + 1) % array.length`、`(head + 2) % array.length`，依此类推。也就是说，如果第 `i` 个值存储在数组的 `array.length - 1` 位置，那么第 `i+1` 个值就存储在数组的 0 号位置——数据可以从数组尾部"回绕"到数组头部。

下图展示了在一个 16 元素的底层数组中存储 7 个值的示例：

```
head = 12
length = 7
底层数组容量 = 16

数组索引: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
          ───────────────────────────────── ─────────────────
           第3个 第4个 第5个 第6个         第0个 第1个 第2个
          (i=3) (i=4) (i=5) (i=6)         (i=0)(i=1)(i=2)
```

在这张图中，`head = 12`，所以序列的 0 号元素存储在底层数组的索引 12 处。序列中的第 `i` 个元素存储在底层数组的索引 `(12 + i) % 16` 处。

### 11.3.2 创建：`Seq_new`

```c
T Seq_new(int hint) {
    T seq;
    assert(hint >= 0);               /* 检查: hint 不能为负数 */
    NEW0(seq);                       /* 分配并零初始化 Seq_T 结构体
                                        (NEW0 展开为 CALLOC, 保证
                                        length 和 head 被置为 0) */
    if (hint == 0)
        hint = 16;                   /* 默认初始容量: 16 个指针 */
    ArrayRep_init(&seq->array, hint, sizeof(void *),
                   ALLOC(hint * sizeof(void *)));
    /* 初始化内嵌的 Array_T 结构体:
       - array.length = hint (16)
       - array.size   = sizeof(void *)
       - array.array  = 指向新分配的 hint*sizeof(void*) 字节的内存 */
    return seq;
}
```

**设计要点**：

1. `NEW0(seq)` 展开为 `(seq) = CALLOC(1, (long)sizeof *(seq))`，以 `calloc` 方式分配内存。`calloc` 保证所有位为零，因此 `seq->length = 0`、`seq->head = 0` 是自动获得的——无需手动赋值。

2. 默认初始容量 16 是一个平衡点：够小不浪费内存，够大能容纳典型小序列。这个值是 [UNSPECIFIED]——原始论文未解释为何选择 16，但基于经验推测是兼顾了缓存行大小（一个指针 4 或 8 字节，16 个指针 = 64~128 字节，恰好 1-2 条缓存行）。

3. `ArrayRep_init` 绕过 `Array_new` 而直接初始化 Array_T 的内部字段。这是因为 Seq 内嵌了 Array_T 结构体本身（不是指针），所以不能使用 `Array_new`（后者会另外分配一个 Array_T 结构体）。

### 11.3.3 批量创建：`Seq_seq`

```c
T Seq_seq(void *x, ...) {
    va_list ap;
    T seq = Seq_new(0);             /* 创建空序列 (默认容量 16) */
    va_start(ap, x);
    for ( ; x; x = va_arg(ap, void *))
        Seq_addhi(seq, x);          /* 逐个追加到高端的朴素方式 */
    va_end(ap);
    return seq;
}
```

`Seq_seq` 使用了 C 语言标准库的 `va_list`、`va_start`、`va_arg`、`va_end` 宏来处理可变长参数列表。参数列表以第一个 `NULL` 指针终止。它将每个非空参数通过 `Seq_addhi` 逐一追加到序列末尾——这种方式简单直接，但在参数较多时会引起多次扩容。

**潜在优化**：如果已知参数个数，可以先用 `Seq_new(n)` 预分配足够容量，避免中间的扩容开销。但 `Seq_seq` 的实现选择了简洁性而非极致性能。

### 11.3.4 释放：`Seq_free`

```c
void Seq_free(T *seq) {
    assert(seq && *seq);
    assert((void *)*seq == (void *)&(*seq)->array);
    /* 关键断言: *seq 的地址 == (*seq)->array 的地址
       即 Array_T 必须是 Seq_T 结构体的第一个字段 */
    Array_free((Array_T *)seq);
    /* 利用了 C 语言结构体内存布局的等价性,
       将 Seq_T 指针直接转型为 Array_T 指针来释放 */
}
```

**精巧的 C 语言技巧**：

1. `*seq` 指向的 `Seq_T` 结构体的第一个字段是 `struct Array_T array`。
2. 因此 `(void *)*seq` 必定等于 `(void *)&(*seq)->array`——结构体首字段的地址等于结构体自身的地址。
3. 借助这一等价性，`Seq_free` 可以直接调用 `Array_free((Array_T *)seq)` 来释放底层的动态数组（`array.array` 指向的内存）和描述符（`Seq_T` 结构体本身）。

这个设计避免了为 Seq 编写重复的释放逻辑，复用了 Array 的 `Array_free`，但前提是 `Array_T` 必须是 `Seq_T` 的首个成员——这个不变量通过上述 `assert` 在 debug 构建中运行时检查。

### 11.3.5 索引宏：`seq[i]`

在分析 `Seq_get`、`Seq_put` 等函数之前，先看一个贯穿整个实现的核心索引表达式：

```c
/* 计算序列第 i 个元素在底层数组中的实际位置 */
#define seq_i(seq, i) \
    ((void **)seq->array.array)[(seq->head + i) % seq->array.length]
```

这个公式将逻辑索引（面向使用者的 0 到 N-1）转换为物理索引（底层数组中实际的存储位置）：

```
物理索引 = (head + i) mod array.length
```

`(void **)` 转型是必要的，因为 `seq->array.array` 的实际类型是 `char *`（在 `ArrayRep` 中声明为 `char *array`，以支持任意类型元素的泛型存储）。转型为 `void **` 后，`[]` 运算符将以 `sizeof(void *)` 为步长正确索引指针数组。

### 11.3.6 随机访问：`Seq_get` 和 `Seq_put`

```c
void *Seq_get(T seq, int i) {
    assert(seq);
    assert(i >= 0 && i < seq->length);   /* 索引越界检查 */
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length];
}

void *Seq_put(T seq, int i, void *x) {
    void *prev;
    assert(seq);
    assert(i >= 0 && i < seq->length);   /* 索引越界检查 */
    prev = ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length];
    ((void **)seq->array.array)[
        (seq->head + i) % seq->array.length] = x;
    return prev;                          /* 返回被替换的旧值 */
}
```

- **`Seq_get`**：直接通过 `(head + i) % array.length` 读取并返回第 `i` 个值。O(1)。

- **`Seq_put`**：先保存旧值，再将 `x` 写入相同位置，最后返回旧值。这是一种"交换语义"（swap semantics），使调用者可以在替换值的同时不丢失原有数据。O(1)。

### 11.3.7 高端移除：`Seq_remhi`

```c
void *Seq_remhi(T seq) {
    int i;
    assert(seq);
    assert(seq->length > 0);             /* 空序列不能移除 */
    i = --seq->length;                   /* length 先减 1, 然后用作索引 */
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length];
    /* 注意: 不需要清除旧值, 因为后续 add 操作会覆盖 */
}
```

`Seq_remhi` 是最简单的移除操作：只需将 `length` 减 1，然后返回（新 length 值指示的）最后一个元素。不需要移动任何数据，也不需要修改 `head`。O(1)。

**注意**：被移除元素的指针仍然残留在底层数组中。这不是内存泄漏（指针指向的对象的生命周期由调用者管理），下次在该位置 `Seq_put` 或扩容后在该位置 `Seq_add` 时会被覆盖。

### 11.3.8 低端移除：`Seq_remlo`

```c
void *Seq_remlo(T seq) {
    int i = 0;
    void *x;
    assert(seq);
    assert(seq->length > 0);             /* 空序列不能移除 */
    x = ((void **)seq->array.array)[
           (seq->head + i) % seq->array.length];  /* 获取第 0 号元素 */
    seq->head = (seq->head + 1) % seq->array.length;
    /* head 前进 1 (模 array.length), 即第 1 号元素变为新的第 0 号 */
    --seq->length;                       /* 长度减 1 */
    return x;
}
```

`Seq_remlo` 比 `Seq_remhi` 稍微复杂：它需要返回 `head` 位置的元素（即序列的第 0 号元素），然后将 `head` 循环前进一位，再递减 `length`。

**关键语义**：移除低端值后，所有剩余值的索引自动减 1。这是因为"第 0 号"的定义被重新映射到了原来"第 1 号"的位置——通过 `head` 的循环递增，这一语义被高效地实现了，无需移动任何数据。O(1)。

### 11.3.9 高端添加：`Seq_addhi`

```c
void *Seq_addhi(T seq, void *x) {
    int i;
    assert(seq);
    if (seq->length == seq->array.length)
        expand(seq);                     /* 容量已满 → 先扩容 */
    i = seq->length++;                   /* 新元素的逻辑索引 = 旧 length */
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length] = x;
    /* 将 x 写入 (head + 旧length) mod array.length 位置 */
}
```

`Seq_addhi` 在序列末尾追加元素：

1. 检查容量：如果 `length == array.length`（底层数组已满），调用 `expand` 扩容。
2. 计算新元素的逻辑索引 `i = 旧length`，并将其写入物理位置 `(head + i) % array.length`。
3. 递增 `length` 并返回 `x`。

O(1) 摊销（amortized）——扩容触发时才需要 O(n) 的数据搬迁。

### 11.3.10 低端添加：`Seq_addlo`

```c
void *Seq_addlo(T seq, void *x) {
    int i = 0;                           /* 新元素总是变成第 0 号 */
    assert(seq);
    if (seq->length == seq->array.length)
        expand(seq);                     /* 容量已满 → 先扩容 */
    if (--seq->head < 0)
        seq->head = seq->array.length - 1;
    /* head 循环后退 1 位, 开出一个新的 0 号槽位 */
    seq->length++;                       /* 长度加 1 */
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length] = x;
    /* 将 x 写入新的 head 位置 (即新的第 0 号) */
}
```

`Seq_addlo` 在序列开头插入元素：

1. 检查容量，必要时扩容。
2. 将 `head` 循环后退一位（`head = (head - 1) mod array.length`）。等价代码为：
   ```c
   if (--seq->head < 0)
       seq->head = seq->array.length - 1;
   ```
   或者使用 Arith 模块的数学模运算（两者语义相同）：
   ```c
   seq->head = Arith_mod(seq->head - 1, seq->array.length);
   ```
   本书选择了分支判断，因为它避免了函数调用开销。
3. 将 `x` 写入新 `head` 位置（此时该位置被映射为序列的第 0 号索引）。
4. 递增 `length` 并返回 `x`。

**关键语义**：在低端添加元素后，所有已有元素的索引自动加 1。这是因为 `head` 向前移动了一格，而逻辑索引到物理索引的映射公式 `(head + i) % array.length` 自然处理了这种偏移——同样是无需移动任何数据。O(1) 摊销。

### 11.3.11 核心扩容逻辑：`expand`

```c
static void expand(T seq) {
    int n = seq->array.length;
    Array_resize(&seq->array, 2 * n);    /* 容量翻倍 (2n) */
    if (seq->head > 0) {
        /* 环形缓冲区数据搬迁: 将 [head, n) 区间的元素
           平移到新数组的 [head+n, 2n-1] 区间 */
        void **old = &((void **)seq->array.array)[seq->head];
        memcpy(old + n, old, (n - seq->head) * sizeof(void *));
        seq->head += n;                  /* head 向后偏移 n */
    }
    /* 如果 head == 0, 数据已经在正确位置, 无需搬迁 */
}
```

**扩容过程详解**：

1. `Array_resize(&seq->array, 2*n)` 将底层数组容量从 `n` 扩大为 `2n`。`Array_resize` 内部使用 `RESIZE` 宏 → `Mem_resize` → `realloc`（或类似机制），因此数据在内存中可能保持原地址或搬迁到新地址。

2. **关键问题**——如果 `head > 0`，那么 `[head, n)` 区间的元素（在原始数组中位于"尾部"）在扩容后仍然在 `[head, n)` 位置，但新扩容出来的空间 `[n, 2n-1]` 在它们"后面"。为了让环形缓冲区的连续性不被破坏，需要将 `[head, n)` 区间的元素平移到 `[head+n, 2n-1]` 区间，从而在 `[0, head)` 和 `[head+n, 2n-1]` 之间释放出 `[head, head+n)` 区间作为"中间空隙"。

3. `head += n`——将 `head` 向后移动 `n` 位以适应新的数组布局。

**图解扩容过程**（扩容前容量 n=8，扩容后 2n=16）：

```
扩容前 (n=8, head=3, length=6):
数组: [ -  -  -  V0 V1 V2 V3 V4 ]
索引:   0  1  2   3  4  5  6  7
                       ↑head=3
                    使用的区间: [3,7]+[0,1) = 6 个元素

扩容后 Array_resize 初始状态 (2n=16):
数组: [ -  -  -  V0 V1 V2 V3 V4  ?  ?  ?  ?  ?  ?  ?  ? ]
索引:   0  1  2   3  4  5  6  7   8  9 10 11 12 13 14 15
                       ↑head=3
                    但 V5 在索引 0, V5 的存储被破坏!

memcpy(old+n, old, n-head) 搬迁后:
数组: [ V5 -  -  V0 V1 V2 V3 V4  ?  ?  ?  ?  ?  ?  V5  ? ]
索引:   0  1  2   3  4  5  6  7   8  9 10 11 12 13  14 15
                                         ↑head=3+8=11
                    使用的区间: [11,15]+[0,2] = 7 个元素 ✓
```

**注意**：扩容后，中间会多出一大段未使用空间 `[head-n, head)`（原书称为"opened up the middle"）。这段空间可被后续的 `addlo` 和 `addhi` 利用，使它们在不触发再次扩容的情况下容纳更多元素。

**扩容策略**：容量翻倍（growth factor = 2），这是动态数组的经典策略，保证了均摊 O(1) 的插入时间复杂度。

### 11.3.12 完整实现代码

```c
/* seq.c — 第11章 序列实现 */
static char rcsid[] = "$Id$";
#include <stdlib.h>
#include <stdarg.h>
#include <string.h>
#include "assert.h"
#include "seq.h"
#include "array.h"
#include "arrayrep.h"
#include "mem.h"

#define T Seq_T

struct T {
    struct Array_T array;
    int length;
    int head;
};

/* ---- 静态函数: 扩容 ---- */
static void expand(T seq) {
    int n = seq->array.length;
    Array_resize(&seq->array, 2 * n);      /* 容量翻倍 */
    if (seq->head > 0) {
        void **old = &((void **)seq->array.array)[seq->head];
        memcpy(old + n, old, (n - seq->head) * sizeof(void *));
        seq->head += n;
    }
}

/* ---- 公共接口实现 ---- */

T Seq_new(int hint) {
    T seq;
    assert(hint >= 0);
    NEW0(seq);
    if (hint == 0) hint = 16;
    ArrayRep_init(&seq->array, hint, sizeof(void *),
                  ALLOC(hint * sizeof(void *)));
    return seq;
}

T Seq_seq(void *x, ...) {
    va_list ap;
    T seq = Seq_new(0);
    va_start(ap, x);
    for ( ; x; x = va_arg(ap, void *))
        Seq_addhi(seq, x);
    va_end(ap);
    return seq;
}

void Seq_free(T *seq) {
    assert(seq && *seq);
    assert((void *)*seq == (void *)&(*seq)->array);
    Array_free((Array_T *)seq);
}

int Seq_length(T seq) {
    assert(seq);
    return seq->length;
}

void *Seq_get(T seq, int i) {
    assert(seq);
    assert(i >= 0 && i < seq->length);
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length];
}

void *Seq_put(T seq, int i, void *x) {
    void *prev;
    assert(seq);
    assert(i >= 0 && i < seq->length);
    prev = ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length];
    ((void **)seq->array.array)[
        (seq->head + i) % seq->array.length] = x;
    return prev;
}

void *Seq_remhi(T seq) {
    int i;
    assert(seq);
    assert(seq->length > 0);
    i = --seq->length;
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length];
}

void *Seq_remlo(T seq) {
    int i = 0;
    void *x;
    assert(seq);
    assert(seq->length > 0);
    x = ((void **)seq->array.array)[
           (seq->head + i) % seq->array.length];
    seq->head = (seq->head + 1) % seq->array.length;
    --seq->length;
    return x;
}

void *Seq_addhi(T seq, void *x) {
    int i;
    assert(seq);
    if (seq->length == seq->array.length)
        expand(seq);
    i = seq->length++;
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length] = x;
}

void *Seq_addlo(T seq, void *x) {
    int i = 0;
    assert(seq);
    if (seq->length == seq->array.length)
        expand(seq);
    if (--seq->head < 0)
        seq->head = seq->array.length - 1;
    seq->length++;
    return ((void **)seq->array.array)[
              (seq->head + i) % seq->array.length] = x;
}
```

---

## 11.4 复杂度分析

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| `Seq_new` | O(1) | 分配结构体 + 初始数组 |
| `Seq_seq` | O(k) | k 为参数个数 |
| `Seq_free` | O(1) | 释放数组 + 结构体 |
| `Seq_length` | O(1) | 直接读取 length 字段 |
| `Seq_get` | O(1) | `(head + i) % array.length` 直接索引 |
| `Seq_put` | O(1) | 同上 |
| `Seq_remhi` | O(1) | 只递减 length，不移动数据 |
| `Seq_remlo` | O(1) | 只循环递增 head，不移动数据 |
| `Seq_addhi` | O(1) 摊销 | 触发扩容时为 O(n) |
| `Seq_addlo` | O(1) 摊销 | 触发扩容时为 O(n) |

**摊销分析**：扩容时容量翻倍（growth factor = 2），每次扩容操作花费 O(n)，但会将至少 n/2 次插入的"债务"清偿。使用势能法或会计法可以证明每次插入的均摊代价为 O(1)。

**双端 O(1) 策略的核心**：`head` 字段的循环递增/递减使得低端添加/移除不需要移动任何已有数据——这是环形缓冲区比普通线性数组的压倒性优势。

---

## 11.5 序列与 Array 的对比

序列和 Array 在 CII 中构成了一种自然的层级关系：序列建立在 Array 之上，提供更高层次的抽象。

### 设计层次

```
Seq (序列) ── 高层 ADT, 面向最终用户
 │
 ├── 内嵌 Array_T 结构体 (非指针)
 │
 └── Array_T ── 底层动态数组, 提供内存管理和随机访问能力
      │
      └── 原始 char *array ── 以字节为单位的通用内存块
```

Seq 不持有指向 `Array_T` 的指针，而是直接将 `struct Array_T` 作为自己的第一个字段。这个设计决策带来了两个后果：

1. **内存局部性更好**：Seq 的描述符和 Array 的描述符在连续内存中，减少了一次指针间接寻址。
2. **释放路径简化**：`Seq_free` 可以通过指针转型直接委托给 `Array_free`，利用了结构体首字段规则。

### 功能差异对比

| 特性 | Array | Seq |
|------|-------|-----|
| **数据存储** | 任意类型的字节序列 (`size` 字节/元素) | 仅存储 `void *` 指针 |
| **容量管理** | 手动 `Array_resize` 或创建时固定 | 自动扩容 (容量翻倍) |
| **高效操作** | 仅随机访问 O(1) | 随机访问 O(1) + 双端添加/移除 O(1) |
| **索引映射** | 直接映射 (索引 == 物理位置) | 环形映射 (索引 + head) % capacity |
| **低端插入** | O(n) — 需要移动所有元素 | O(1) 摊销 — 仅调整 head |
| **低端移除** | O(n) — 需要移动所有元素 | O(1) — 仅调整 head |
| **泛型性** | 更泛型 (任意元素大小) | 弱泛型 (仅 `void *`) |
| **透明度** | 基本透明 (暴露内部结构) | 完全不透明 (`Seq_T` 是不透明指针) |

### 使用场景划分

**使用 Seq 当**：
- 数据是指针（或可以封装为指针）
- 需要在双端频繁添加/移除元素
- 需要栈、队列或双端队列的行为
- 不想手动管理容量

**使用 Array 当**：
- 需要存储非指针类型的原始数据（`int`、`double`、`struct` 等）
- 只需要按索引读写
- 需要手动控制内存布局和大小
- 构建更底层的数据结构

**Seq 对 Array 的抽象溢价**：Seq 的 107 行实现代码中，约 60% 的复杂度用于处理环形缓冲区的索引映射和数据搬迁（`addlo`/`remlo`/`expand`）。这部分复杂度是消费者不需要承受的——这正是 ADT 的价值所在：将复杂性封装在接口之下。

---

## 11.6 一些设计考量和局限

### 11.6.1 容量只增不减

序列当前实现只扩容不缩容。`Seq_remlo` 和 `Seq_remhi` 在元素数量远低于底层数组容量时不会缩小数组。这可能导致：

- **优势**：避免了"抖动"（thrashing）——在容量边界反复扩容和缩容的灾难性性能退化。
- **劣势**：长期运行的程序如果序列先膨胀后萎缩，会浪费内存。

**改进方向**（练习 11.4）：当 `seq->length < seq->array.length / 2` 时触发缩容。但需要警惕：
- 如果缩容发生在容量边界附近（如 `length` 在 `capacity/2` 上下抖动），会导致频繁的 `realloc` 和数据搬迁，产生灾难性的性能退化。
- 稳妥的策略是加入迟滞（hysteresis）：缩容阈值应远低于扩容阈值，例如容量减半仅在 `length < capacity/4` 时触发。

### 11.6.2 可变参数的局限性

`Seq_seq` 使用 `NULL` 作为参数列表终止标志，这意味着序列中不能存储 `NULL` 指针值。这是 C 语言可变参数机制的固有局限性——无法在编译期确定参数个数。

### 11.6.3 对 C 结构体内存布局的依赖性

`Seq_free` 依赖 `Array_T` 是 `Seq_T` 结构体的首字段。虽然这在所有主流平台和编译器上都能正常工作（C 标准保证结构体首字段无 padding），但它是一个隐式约定而非显式接口契约。如果将来有人重构结构体布局（例如在 `array` 前添加一个 `magic` 字段），程序将在 debug 构建的断言处失败，但 release 构建下会静默地产生未定义行为。

### 11.6.4 `char *` 数组的 `void **` 转型

底层数组的类型是 `char *`（定义在 `arrayrep.h` 中），但在 Seq 的实现中频繁被转型为 `void **` 以进行指针级索引。这种转型依赖于 `char *` 和 `void **` 具有相同的表示（representation）——这在绝大多数实现上成立，但严格来说不符合 C 标准的别名规则（strict aliasing rule）。现代 C 编译器可能在启用严格别名优化时产生问题。这是 CII 代码中一个已知的"实用主义折中"。

---

## 11.7 Modern C 改进方案

CII 的 Seq 实现写于 1990 年代初期（第一版 1996 年），在当代 C 编程语境下，有若干可行的改进方向。

### 11.7.1 环形缓冲区优化

**使用位掩码代替取模运算**

当前实现使用 `% array.length` 进行环形索引计算。当 `array.length` 总是 2 的幂时，可用位掩码替代：

```c
/* 原始: (head + i) % array.length
   改进: (head + i) & (array.length - 1)   // 当 array.length 是 2 的幂 */

/* 扩容策略从 2*n 改为保持 2 的幂 */
static void expand(T seq) {
    int n = seq->array.length;
    Array_resize(&seq->array, 2 * n);
    if (seq->head > 0) {
        void **old = array_base(seq) + seq->head;
        memcpy(old + n, old, (n - seq->head) * sizeof(void *));
        seq->head += n;
    }
}
```

`%` 运算（尤其是除数不是编译期常量时）在现代 CPU 上的延迟约为 20-80 个周期，而 `&` 运算仅需 1 个周期。这一优化在环形缓冲区操作为热路径的场景下效果显著。

**代价**：容量不再是从 `hint` 出发的连续值，而是被"圆整"到最近的 2 的幂。初始容量 16 本身已经是 2 的幂，所以从默认配置出发不受影响。

### 11.7.2 缓存友好设计

**问题**：当前扩容策略 `expand` 将尾段数据复制到数组后半部，在数组中间制造一段空白。这在逻辑上是正确的，但可能导致数据在物理上变得非连续，降低 CPU 缓存的利用率。

**改进方案一：扩容时重排数据**

在 `expand` 中，如果 `head > 0`，可选择将整个序列搬移到新数组的起始位置：

```c
static void expand_v2(T seq) {
    int n = seq->array.length;
    Array_resize(&seq->array, 2 * n);
    if (seq->head > 0) {
        void **base = (void **)seq->array.array;
        /* 将 [head, head+length) 整体搬移到 [0, length) */
        int tail_len = n - seq->head;
        if (tail_len >= seq->length) {
            /* head 之后的元素数量足够覆盖全部有效数据 —— 无需搬 */
        } else {
            /* 搬移环绕段 */
            memmove(base, base + seq->head, tail_len * sizeof(void *));
            memmove(base + tail_len, base + n,
                    (seq->length - tail_len) * sizeof(void *));
        }
        seq->head = 0;
    }
}
```

这样数据在扩容后总是从索引 0 开始连续存放，最大化缓存命中率。代价是增加了一次 `memmove` 的开销。选择哪种方案取决于工作负载：如果序列经常被遍历，线性化带来的缓存收益大于搬移的开销；如果序列主要用于双端 push/pop，原始环形方案已经足够好。

**改进方案二：使用 `alignas` 对齐**

```c
#include <stdalign.h>
struct T {
    struct Array_T array;
    int length;
    int head;
} alignas(64);   /* 对齐到 64 字节 (一条缓存行) */
```

将 `Seq_T` 结构体对齐到缓存行边界，避免伪共享（false sharing）——当多个线程分别操作不同的序列时，如果两个 `Seq_T` 共享同一条缓存行，一个线程的写入会使另一个线程的缓存行失效。

### 11.7.3 类型安全改进

**使用 `_Generic` 提供类型安全的接口**

```c
#define Seq_get_type(seq, i, type) \
    ((type)(uintptr_t)Seq_get((seq), (i)))

/* 使用示例 */
int *p = Seq_get_type(seq, 0, int *);
```

或更进一步，为常用类型提供包装：

```c
static inline intptr_t Seq_getint(T seq, int i) {
    return (intptr_t)Seq_get(seq, i);
}
```

### 11.7.4 使用 `restrict` 关键字

对于性能敏感的访问路径，添加 `restrict` 提示编译器不会发生别名：

```c
void *Seq_get(T restrict seq, int i);
```

这在循环中连续访问序列元素时可以启用更激进的编译器优化（如向量化）。

### 11.7.5 初始化容量的自适应性

`Seq_new` 的默认初始容量 16 对所有场景一视同仁。可以改为根据第一次实际使用动态调整，或者提供一个从统计数据学习最佳 `hint` 的启发式方案。但对于通用库而言，16 的简单性胜过过度设计的自适应性。

---

## 11.8 与 C++ `std::deque` 的对比

CII 的 Seq 和 C++ 标准库的 `std::deque` 在概念上相似（都是双端可操作序列），在实现上差异显著。

### 关键差异

| 维度 | CII Seq | C++ std::deque |
|------|---------|----------------|
| **底层结构** | 单一连续数组 + 环形缓冲区 | 分段数组 (chunked / segmented array) — 由多个固定大小的块组成的中控器 |
| **扩容策略** | `realloc` 或重新分配整个数组 | 分配新块，更新中控指针数组，不需要搬迁已有数据 |
| **最大容量** | 受限于连续内存可用性 | 每个块独立分配，可接近系统总内存 |
| **元素类型** | 仅 `void *` 指针 | 任意类型 (模板泛型) |
| **迭代器稳定性** | 扩容后所有迭代器/索引失效 | 仅在插入点所在的块内元素部分失效 |
| **内存开销** | 仅结构体 + 底层数组 | 中控指针数组 + 每个块的额外元数据 |
| **缓存局部性** | 更好（数据连续） | 较差（数据跨块分散） |
| **前端插入** | O(1) 摊销 (直接移 head) | O(1) (分配新块或在块内移动) |
| **异常安全** | 无 (C 语言) | 强异常保证 (基本操作) |
| **内存释放** | 手动 `Seq_free` | RAII 自动析构 |
| **`NULL` 支持** | 不能存储 `NULL` (`Seq_seq` 以 NULL 终止) | 可以存储任何合法值 |
| **类型安全** | 无 (void * 需要手动转型) | 编译期类型检查 |
| **代码量** | ~110 行 (seq.c) | GCC libstdc++ 中约 3000+ 行 (含迭代器) |

### std::deque 的分段数组方案

C++ `std::deque` 的典型实现使用一个**中控器**（map）——一个指向多个固定大小块（chunks）的指针数组，每个块通常为 512 字节或单个元素大小：

```
中控器 (指针数组):
[ptr0][ptr1][ptr2][ptr3][ptr4]...
   |     |     |     |     |
   v     v     v     v     v
[块0] [块1] [块2] [块3] [块4] ...
(每个块容纳固定数量元素，如 512/sizeof(T) 个)

前端插入: 在块0之前分配新块，更新中控器起始指针
后端插入: 在最后一块之后分配新块，更新中控器结束指针
```

**核心优势**：
- **扩容不需要搬迁已有数据**——只分配新块和（可能）更新中控器指针数组。
- **迭代器稳定性**——只有插入点所在块的元素受影响。
- **可以充分利用碎片化内存**——每个块分别分配，不要求大段连续内存。

**核心代价**：
- **多次间接寻址**——访问元素需要先找到所在的块（块号 = i / block_size，块内偏移 = i % block_size）。
- **缓存不友好**——相邻的块可能在物理内存中相距甚远。
- **实现复杂度高**——中控器管理、块内偏移计算、迭代器实现都较复杂。

### Seq 的设计哲学

Seq 选择了一条更简单的路径：单一连续数组。这意味着它牺牲了超大序列（需要大段连续内存）和迭代器稳定性，但换来了：

1. **实现的极致简洁**（~110 行 vs ~3000+ 行）。
2. **更好的缓存局部性**——所有元素物理上连续。
3. **更简单的索引计算**——只需一次模运算，不需要两级间接寻址。
4. **和 Array 的代码复用**——通过内嵌 Array_T 直接继承其内存管理能力。

在典型的小型到中型序列（几百到几万个元素）场景下，Seq 的单一连续数组方案具有更好的常数因子。只有在序列极大或对扩容时的数据搬迁敏感时，`std::deque` 的分段方案才显现优势。

### 适用场景建议

| 场景 | 推荐方案 |
|------|---------|
| 小型序列 (< 1K 元素), C 项目 | CII Seq |
| 中型序列 (1K - 100K), C 项目 | CII Seq 或手动环形缓冲区 |
| 大型序列 (> 100K), C 项目 | 考虑分段数组方案或 C++ std::deque |
| 需要迭代器/引用稳定性 | C++ std::deque (分段方案) |
| 需要缓存友好的遍历 | CII Seq (连续数组) |
| 需要类型安全 | C++ std::deque 或现代 C `_Generic` 包装 |
| 教学/学习/理解 ADT 设计 | CII Seq 是不二之选 |

---

## 11.9 序列的应用模式

### 栈 (Stack)

```c
Seq_T stk = Seq_new(0);
Seq_addhi(stk, item);   /* push */
item = Seq_remhi(stk);  /* pop  */
```

### 队列 (Queue / FIFO)

```c
Seq_T q = Seq_new(0);
Seq_addhi(q, item);     /* enqueue */
item = Seq_remlo(q);    /* dequeue */
```

### 双端队列 (Deque)

```c
Seq_T dq = Seq_new(0);
Seq_addlo(dq, item);    /* push_front */
Seq_addhi(dq, item);    /* push_back  */
item = Seq_remlo(dq);   /* pop_front  */
item = Seq_remhi(dq);   /* pop_back   */
```

### 动态列表 (List)

```c
Seq_T lst = Seq_seq("Alice", "Bob", "Carol", NULL);
for (int i = 0; i < Seq_length(lst); i++)
    printf("%s\n", (char *)Seq_get(lst, i));
```

**关键洞察**：所有这些数据结构共享同一套 API 和同一个实现。使用者不需要为每个抽象概念维护不同的类型——从栈切换到队列只需要将 `remhi` 改为 `remlo`。这种统一性是 CII 全书最重要的设计理念之一。

---

## 11.10 补充阅读与练习

### 来源与演变

- 序列的操作名称取自 DEC Modula-3 库中的 Sequence 接口 (Horning et al., 1993)。
- Icon 语言 (Griswold and Griswold, 1990) 中的列表概念几乎完全等同于 CII 的序列，但 Icon 使用了**双向链表分块**的实现方式（见练习 11.1）。

### 练习 11.1：分块链表实现

Icon 使用双向链表连接多个"块"（chunk），每个块存储 M 个值。不依赖 `Array_resize`，仅通过添加/移除块来响应 `Seq_addlo`/`Seq_addhi`。缺点：访问第 i 个值需要遍历链表（O(i/M) 时间）。

**进一步优化**：如果访问值 i 后几乎总是接着访问值 i-1 或 i+1（缓存上次访问的块），可将这种模式优化为 O(1)。

### 练习 11.2：边向量 (Edge-Vector) 表示

不依赖 `Array_resize`，当 N 元素数组满时，将其转换为一个"指针数组的数组"（edge-vector），其中每个子数组容纳 2N 个元素，转换后的序列最多容纳 N * 2N = 2N^2 个元素。若 N=1024，可容纳超 200 万个元素，每个仍可 O(1) 访问。每个子数组可**惰性分配**——仅在首次写入时分配。

### 练习 11.3：禁止低端操作的对数时间方案

若禁止 `Seq_addlo` 和 `Seq_remlo`，可以使用**跳表 (Skip List)** (Pugh 1990) 实现逐步分配空间但支持 O(log n) 访问任意元素的序列。

### 练习 11.4：自动缩容与"抖动"

修改 `Seq_remlo` 和 `Seq_remhi`，当 `seq->length < seq->array.length / 2` 时缩容。需注意：频繁在边界处添加/移除会导致容量振荡（thrashing）——每次操作都触发 `realloc` 和数据搬迁。解决方案是使用迟滞阈值（如仅在 `length < capacity / 4` 时缩容）。

### 练习 11.5：用 Seq 重写 xref

使用序列（而非集合）保存行号。由于文件顺序读取，行号自然保持递增，无需排序。

### 练习 11.6：`Seq_free` 断言消除

重写 `Seq_free` 使其不需要依赖 `Seq_T` 首字段是 `Array_T` 的断言。注意：不能使用 `Array_free`。

---

## 本章小结

1. **Seq 是建立在 Array 之上的高级 ADT**，它通过环形缓冲区 (`head` 字段) 在保持 O(1) 随机访问的同时实现了双端 O(1) 添加/移除。

2. **一套 API 覆盖四种数据结构**：栈 (addhi/remhi)、队列 (addhi/remlo)、双端队列 (两端均可)、动态列表 (get/put)。不需要为每种抽象维护独立的类型。

3. **扩容采用容量翻倍策略**，确保均摊 O(1) 的插入复杂度。扩容时需处理环形缓冲区的数据搬迁。

4. **与 C++ std::deque 相比**，Seq 的单一连续数组更简单、缓存更友好，但扩容时需搬迁数据且要求连续内存。对于小型到中型序列，Seq 的常数因子优于 std::deque 的分段方案。

5. **已知局限**：不能存储 NULL、容量只增不减、依赖结构体首字段布局、`char *` 到 `void **` 的严格别名问题。在实际使用中应根据场景权衡这些 trade-off。
