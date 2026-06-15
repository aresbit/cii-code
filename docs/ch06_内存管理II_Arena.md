# 第6章: 内存管理 II -- Arena (基于区域的分配器)

> 原文: C Interfaces and Implementations, David R. Hanson, Chapter 6
> 源文件: `code/arena.h`, `code/arena.c`
> 关联章节: 第5章 Mem (`mem.h` / `mem.c`)

---

## 一、设计哲学: 批量分配, 统一释放

### 1.1 问题背景

传统的 `malloc/free` 接口要求每一次分配 (allocation) 都必须配对一个显式的释放 (deallocation)。如第5章所述, 这种一对一契约极易出错:

- **忘记释放**: 造成内存泄漏。
- **重复释放**: 导致 double-free 未定义行为。
- **释放时机错误**: 产生悬空指针 (dangling pointer)。
- **代码膨胀**: 释放逻辑遍布程序各处, 使代码复杂化。

然而, 大量实际程序的内存分配存在天然的**批量生命周期**。图形用户界面 (GUI) 是一个典型例子: 窗口创建时通过 `malloc` 分配滚动条、按钮等子控件的内存; 窗口销毁时所有这些内存一同回收。编译器 `lcc` (Fraser and Hanson 1995) 是另一个典型: 编译一个函数期间持续分配内存, 函数编译完成后一次性释放所有中间数据。在这些场景下, 对象生命周期由**逻辑阶段**的起止决定, 而非由单个对象本身决定。

### 1.2 Arena 的核心思想

Arena (又名 pool allocator / region-based allocator) 将 "一次性回收一组对象" 这个语义固化为原语:

> **从 Arena 中分配 (alloc), 无需逐一 free; 回收 Arena 时, 其内所有内存一同归还。**

Hanson 将这种哲学概括为:

- **分配**: `Arena_alloc(arena, nbytes)` 等价于 "给我 nbytes 内存, 归这个 arena 管"。
- **释放**: `Arena_free(arena)` 等价于 "回收该 arena 管理的一切"。
- **销毁**: `Arena_dispose(&arena)` 等价于 "再也不需要这个 arena 了"。

这种设计带来的关键收益在于**简化代码** (Hanson 称之为 "the most important benefit")。适用式算法 (applicative algorithms) 通过分配新数据结构而非就地修改旧数据结构来编程; Arena 消解了他们在每个分配点记住何时释放的心智负担。

```
传统malloc/free:               Arena:
分配A ─────────────────┐       分配A ──┐
分配B ───────────────┐ │       分配B ──┤  (都在同一arena中)
分配C ─────────────┐ │ │       分配C ──┤
...                │ │ │       ...
free(A) ──────────┤ │ │       Arena_free(arena) → 全部释放!
free(B) ──────────┤ │ │       (一次调用替代 N 次 free)
free(C) ──────────┤ │ │
...               │ │ │
free(N) ──────────┘ │ │
... ────────────────┘ │
... ──────────────────┘
```

### 1.3 两个代价

Hanson 坦率承认 Arena 的两个固有缺陷:

1. **更高的内存峰值**: 由于分配是 "一气呵成" 式地推进 (bump allocation), 在当前 chunk 末尾的剩余空间将被浪费 (图 6.1 展示的碎片化)。且 Arena 无法回收单个块。
2. **悬空指针风险**: 如果将对象分配到了错误的 arena, 该 arena 提前释放后, 访问该对象将访问到未分配或已复用的内存。

但 Hanson 在实践中的观察是: "arena 管理太容易了, 这些问题很少发生"。

---

## 二、接口总览

### 2.1 `arena.h` 完整接口

```c
/* $Id$ */
#ifndef ARENA_INCLUDED
#define ARENA_INCLUDED
#include "except.h"
#define T Arena_T
typedef struct T *T;

extern const Except_T Arena_NewFailed;   // Arena 创建失败异常
extern const Except_T Arena_Failed;      // 内存分配失败异常

extern T    Arena_new    (void);                          // 创建新 Arena
extern void Arena_dispose(T *ap);                         // 销毁 Arena
extern void *Arena_alloc (T arena, long nbytes,
              const char *file, int line);                // 分配未初始化内存
extern void *Arena_calloc(T arena, long count,
              long nbytes, const char *file, int line);   // 分配并清零内存
extern void  Arena_free  (T arena);                       // 释放 Arena 内所有内存

#undef T
#endif
```

### 2.2 接口设计要点

| 设计要点 | 说明 |
|----------|------|
| **不透明指针** | `Arena_T` 是 `struct T *`, 其中 `struct T` 的定义隐藏在 `.c` 文件中。客户端永远只持有指针。 |
| **异常而非 NULL** | 与 `Mem` 接口不同, Arena 不提供 `RESIZE`。内存不足时, 通过异常机制 (`Arena_Failed`) 上报, 携带 `__FILE__` 和 `__LINE__` 信息。 |
| **独立于 Mem** | Arena 的实现直接调用 `malloc/free`, 而非 `Mem_alloc/Mem_free`。这使得 Arena 不与第5章的带检查内存分配器耦合。 |
| **`file` 参数的特殊语义** | 若 `file == NULL`, `Arena_alloc` 直接使用 `RAISE(Arena_Failed)` 简化异常; 若 `file != NULL`, 则调用 `Except_raise` 传递调用位置。 |

---

## 三、`Arena_T` 数据结构深度解析

### 3.1 核心结构体

```c
struct T {
    T    prev;    // 指向前一个 chunk (形成链表)
    char *avail;  // 当前 chunk 内下一个可用字节的地址
    char *limit;  // 当前 chunk 的结束边界 (指向末端之后一个字节)
};
```

三个字段构成了 Arena 分配器的全部状态:

- **`prev`**: 链表指针, 指向列表中的前一个 chunk。每个 chunk 的头部都是一个 `struct T` (通过 `union header` 确保对齐), `prev` 把这些 chunk 串成一条单向链表。链表末尾 (即 Arena 创建时的初始 chunk 或释放后) 的 `prev` 为 `NULL`。
- **`avail`**: 可用空间的起始指针。每次分配时, `avail` 向前推进 `nbytes`。`avail` 到 `limit` 之间的空间是空闲的。
- **`limit`**: 当前 chunk 的硬边界。一旦 `nbytes` 超过 `limit - avail`, 就触发新 chunk 分配。

### 3.2 对齐联合体 `union align`

```c
union align {
#ifdef MAXALIGN
    char pad[MAXALIGN];
#else
    int i;
    long l;
    long *lp;
    void *p;
    void (*fp)(void);
    float f;
    double d;
    long double ld;
#endif
};
```

这是 CII 中经典的对齐技巧。`union align` 包含平台上所有对齐要求最严格的基本类型。`sizeof(union align)` 等于目标平台上最严格的自然对齐粒度 (通常 8 或 16 字节)。通过条件编译 `MAXALIGN`, 用户可以在需要时手动指定对齐值。

### 3.3 `union header`: chunk 头部

```c
union header {
    struct T b;       // Arena 状态 (prev/avail/limit)
    union align a;    // 强制对齐
};
```

每个 chunk 的前 `sizeof(union header)` 字节既是 Arena 状态 (`b`), 又由其 `a` 成员保证起始地址按 `MAXALIGN` 对齐。当我们说 "chunk 的开头是一个 `struct T`" 时, 实际上是指通过 `union header` 获得的对齐版本。

**为什么要用 `union header` 而不直接用 `struct T`?** 单独使用 `struct T` 无法保证 `prev/avail/limit` 的起始地址满足所有类型的对齐要求。通过将 `struct T` 和对齐联合体并置在同一个 union 中, 编译器会按照 `union align` 的要求对齐 `union header`, 进而对齐其内的 `struct T b`。

---

## 四、内存布局: Chunk 链式分配策略

### 4.1 整体结构 ASCII 图

```
arena (Arena_T)
  │
  v
 ┌─────────────┐      ┌──────────────┐      ┌──────────────┐
 │ struct T     │      │ union header │      │ union header │
 │  prev ───────┼─────>│  struct T    │      │  struct T    │
 │  avail ──────┼──┐   │   prev ──────┼─────>│   prev = NULL│
 │  limit ──────┼──┼─┐ │   avail ─────┼──┐   │   avail ─────┼──┐
 │              │  │ │ │   limit ─────┼──┼─┐ │   limit ─────┼──┼─┐
 │==============│  │ │ │=============│  │ │ │=============│  │ │
 │  (chunk 1    │  │ │ │  (chunk 2   │  │ │ │  (chunk 3   │  │ │
 │   data)      │<-┘ │ │   data)     │<-┘ │ │   data)     │<-┘ │
 │  ^^^^^^^^^^^ │    │ │  ^^^^^^^^^^ │    │ │  ^^^^^^^^^^ │    │
 │  (unused)    │<───┘ │  (unused)   │<───┘ │  (unused)   │<───┘
 └─────────────┘      └──────────────┘      └──────────────┘
  初始 chunk           第2 chunk              第3 chunk
  (通过 malloc          (chunk1 空间不足        (chunk2 空间不足
   分配, 10K+ 大小)      时 malloc 分配)        时 malloc 分配)
```

图例:
- `====` 表示已分配的用户数据区域
- `^^^^` 表示 chunk 末尾未使用的浪费空间
- `──>` 表示 `prev` 指针方向
- 虚线箭头表示状态恢复时的"弹栈"方向

### 4.2 分配语义: push-only (bump allocator)

Arena 的分配是纯推进式的:

```
分配前:  avail ──────> [可用空间] <────── limit

  调用 Arena_alloc(arena, N):
    1. 将 N 向上对齐到 sizeof(union align) 的倍数
    2. 检查 N <= (limit - avail)?
       是 -> 直接返回 avail 的旧值, 然后 avail += N
       否 -> 分配新 chunk (见 §4.3)

分配后:  avail (新) ──> [可用空间] <────── limit

返回: avail(旧)
```

"Bump allocation" 的优点是分配的快速路径极其简单: 一次对齐算术、一次减法比较、一次指针加法。没有空闲链表的遍历, 没有分裂合并, 没有碎片整理。

```
分配前:
┌──────────────┬───────────────────────────────┐
│ header       │  已分配  │  avail -> 空闲     │ limit
└──────────────┴──────────┴───────────────────┘

Arena_alloc(arena, 64):
1. nbytes 对齐到 64
2. 剩余空间 = limit - avail = 128 >= 64? Yes
3. avail += 64
4. return avail - 64 = 原avail位置

分配后:
┌──────────────┬───────────────────────────────┬───────────┐
│ header       │  已分配  │  新分配64B  │ avail->空闲│ limit
└──────────────┴──────────┴─────────────┴───────┴───────────┘
                          ^
                   返回这个地址
```

### 4.3 新 Chunk 的分配流程

当 `nbytes > arena->limit - arena->avail` 时, 进入新 chunk 获取流程:

```
while (nbytes > arena->limit - arena->avail):

  1. 尝试从 freechunks 全局空闲链表获取 chunk:
     if (ptr = freechunks) != NULL
         freechunks = freechunks->prev  // 从链表取下
         nfree--
         limit = ptr->limit             // 记录其边界

  2. 否则调用 malloc 分配新 chunk:
     else
         m = sizeof(union header) + nbytes + 10*1024
         ptr = malloc(m)
         if (ptr == NULL) -> RAISE(Arena_Failed)
         limit = (char *)ptr + m

  3. 将当前 arena 状态 "压栈":
     *ptr = *arena              // 旧状态存入 chunk 头部
     arena->avail = (char *)((union header *)ptr + 1)
     arena->limit = limit
     arena->prev = ptr           // 链接新 chunk

  4. 回到 while 顶部, 再次尝试分配
     (因为 freechunks 提供的 chunk 可能仍不够大)
```

**关键常数 10K**:

```c
long m = sizeof (union header) + nbytes + 10*1024;
```

每次 `malloc` 分配新 chunk 时, 额外索取 10KB 空间。这个常数的选择是一个权衡:
- **过大**: chunk 末尾浪费变大 (bump allocator 不会回填)。
- **过小**: 频繁触发 `malloc` 系统调用, 降低性能。
对于 `lcc` 编译器场景 (大量小分配), 10KB 是一个经验上的甜蜜点。

**为什么要用 `while` 而不用 `if`?** 如果从 `freechunks` 获取的 chunk 仍然无法满足 `nbytes` 的要求, 循环会继续: 要么再取一个新的 free chunk, 要么调用 `malloc`。`while` 确保最终一定能获得足够大的 chunk。

### 4.4 状态栈 (State Stack) 语义

`*ptr = *arena` 这行代码是整个 Arena 设计的精髓:

```
分配新 chunk 前:
  arena -> {prev=A, avail=X, limit=Y}
  *ptr  = 新 chunk 的头部 (union header 区域)

执行 *ptr = *arena:
  *ptr  = {prev=A, avail=X, limit=Y}  <- 旧状态被 "压入" 新 chunk 头部

再设置 arena 指向新 chunk:
  arena->avail = (union header *)ptr + 1   <- 新 chunk 的数据区起点
  arena->limit = limit                      <- 新 chunk 边界
  arena->prev  = ptr                        <- prev 指向新 chunk
```

这形成了一个**隐式的状态栈**: 每个 chunk 的头部保存了分配该 chunk 时 `arena` 的完整状态。当 Arena 释放时遍历链表, 可以精确恢复每一级的状态。详见 `Arena_free` 的 "弹栈" 过程。

---

## 五、逐行中文注释

### 5.1 `Arena_new`: 创建空的 Arena

```c
T Arena_new(void) {
    // (1) 为 Arena 控制结构体本身分配内存
    //     注意是 sizeof(*arena), 即 sizeof(struct T), 通常 24 字节
    T arena = malloc(sizeof (*arena));

    // (2) 分配失败: 抛出 Arena_NewFailed 异常
    if (arena == NULL)
        RAISE(Arena_NewFailed);

    // (3) 初始化为空 Arena:
    //     prev=NULL  表示无任何 chunk
    //     limit=NULL && avail=NULL 是空 Arena 的不变量 (invariant)
    arena->prev = NULL;
    arena->limit = arena->avail = NULL;

    // (4) 返回不透明句柄
    return arena;
}
```

**延迟分配**: `Arena_new` 仅仅分配了 `sizeof(struct T)` (通常 24 字节) 的管理结构, **不预分配任何数据 chunk**。第一个数据 chunk 的分配延迟到首次 `Arena_alloc` 调用时才发生。这是一种 "allocate-on-first-use" 策略, 避免未使用的 Arena 浪费内存。

### 5.2 `Arena_dispose`: 销毁 Arena

```c
void Arena_dispose(T *ap) {
    // (1) 检查参数有效性: ap 和 *ap 都不能为 NULL
    assert(ap && *ap);

    // (2) 先释放 arena 内所有 chunk 占用的内存
    Arena_free(*ap);

    // (3) 再释放 arena 控制结构本身
    free(*ap);

    // (4) 将客户端指针置 NULL, 防止悬空指针
    //     调用后 *ap == NULL, 任何后续的 assert(ap && *ap)
    //     都会立即捕获错误使用
    *ap = NULL;
}
```

**设计要点**: `Arena_dispose` 接受 `T *ap` (二级指针), 而不是 `T ap`。这是 CII 中的惯用手法 (参见 `Mem_free`, `List_free`): 销毁函数不仅释放资源, 还将客户端的指针清零, 彻底杜绝悬空指针。

释放顺序也很关键:**先释放 chunk, 再释放控制结构**。如果反过来, chunk 还持有 `prev` 指针指向已释放的 arena, 会出问题。

### 5.3 `Arena_alloc`: 分配内存 -- 核心函数

```c
void *Arena_alloc(T arena, long nbytes,
    const char *file, int line) {
    // (1) 参数校验
    assert(arena);           // arena 指针不能为 NULL
    assert(nbytes > 0);      // 请求字节数必须为正值

    // (2) 对齐: 将 nbytes 向上取整到 sizeof(union align) 的倍数
    //     例如 align=16, nbytes=3 -> 16; nbytes=20 -> 32
    nbytes = ((nbytes + sizeof (union align) - 1)/
              (sizeof (union align)))*(sizeof (union align));

    // (3) 空间不足循环: 不断申请新 chunk 直到有空余空间
    //     注意是 while, 不是 if -- freechunks 给的 chunk 可能仍然太小
    while (nbytes > arena->limit - arena->avail) {
        T ptr;          // 新 chunk 的控制块指针
        char *limit;    // 新 chunk 的结束位置

        // --- 3a. 尝试从全局 freechunks 链表获取一个空闲 chunk ---
        if ((ptr = freechunks) != NULL) {
            freechunks = freechunks->prev;  // 从链表头部取下
            nfree--;                         // 空闲 chunk 计数减一
            limit = ptr->limit;              // 记录该 chunk 的边界
        }
        // --- 3b. 没有空闲 chunk, 调 malloc 分配新 chunk ---
        else {
            // 新 chunk 大小 = 头部 + 请求大小 + 10KB 额外空间
            long m = sizeof (union header) + nbytes + 10*1024;
            ptr = malloc(m);
            if (ptr == NULL) {
                // 分配失败, 上报异常
                // file 指针区分两种异常抛出方式:
                if (file == NULL)
                    RAISE(Arena_Failed);                 // 简化版
                else
                    Except_raise(&Arena_Failed, file, line); // 带调用点
            }
            limit = (char *)ptr + m;  // chunk 结束边界
        }

        // --- 3c. 将当前 arena 状态 "压栈" 到新 chunk 头部 ---
        //       *ptr = *arena 是整个算法的关键语句:
        //       将 (prev, avail, limit) 全部复制到新 chunk 头部的 struct T 中
        *ptr = *arena;

        // --- 3d. 更新 arena 使其指向新 chunk ---
        //       (union header *)ptr + 1 跳过 sizeof(union header) 字节,
        //       即 chunk 数据区域的起始地址 (已由 union header 对齐)
        arena->avail = (char *)((union header *)ptr + 1);
        arena->limit = limit;
        arena->prev  = ptr;
        // while 循环回到顶部, 重新检查空间是否足够
    }

    // (4) 推进 bump pointer, 返回旧值 (即分配的起始地址)
    arena->avail += nbytes;
    return arena->avail - nbytes;
}
```

**关键技巧 -- `(union header *)ptr + 1`**: 这个表达式将 `ptr` 先转换为 `union header *` 类型, 然后 `+1` 意味着增加一个 `union header` 的大小。最终的 `(char *)` 强制转换将结果表示为字节地址。这种写法的精妙之处在于:**对齐是类型驱动的** -- 编译器知道 `union header` 的大小和其对齐要求, 因此 `+1` 产生的偏移自动满足对齐约束。

### 5.4 `Arena_calloc`: 分配并清零

```c
void *Arena_calloc(T arena, long count, long nbytes,
    const char *file, int line) {
    void *ptr;

    // count 必须为正 (与标准库 calloc 行为一致)
    assert(count > 0);

    // 委托 Arena_alloc 分配 count * nbytes 的未初始化内存
    ptr = Arena_alloc(arena, count*nbytes, file, line);

    // 用 memset 清零
    memset(ptr, '\0', count*nbytes);

    return ptr;
}
```

**与标准库对比**: 标准 C 的 `calloc(count, size)` 分配 `count * size` 字节并清零; `Arena_calloc` 语义一致, 但内存来自指定 Arena。

**潜在溢出问题**: `count * nbytes` 的乘法可能导致 `long` 类型溢出。CII 的 `Arena_calloc` 未做溢出检查, 这是 C 语言自身的局限 (C11 后可以用 `__builtin_mul_overflow` 改进)。在实际使用中, Arena 的调用方通常是编译器/解析器等受控环境, 溢出风险较低。

### 5.5 `Arena_free`: 回收所有 Chunk

```c
void Arena_free(T arena) {
    assert(arena);

    // 遍历链表, 从最新 chunk 开始向旧 chunk 回收
    while (arena->prev) {
        // (1) 将当前 chunk 头部保存的状态复制到 tmp
        //     tmp 包含了更早一级的 (prev, avail, limit)
        struct T tmp = *arena->prev;

        // (2) 决定当前 chunk 的命运: 加入 freechunks 还是直接 free
        if (nfree < THRESHOLD) {
            // ---- 加入全局空闲链表 ----
            arena->prev->prev = freechunks;  // 插入 freechunks 链表头部
            freechunks = arena->prev;        //
            nfree++;                          // 空闲计数加一
            freechunks->limit = arena->limit; // 记录 chunk 边界 (关键!)
            // 注意: freechunks->limit 的更新使得将来 Arena_alloc
            //       能正确获得该 chunk 的可用空间大小
        } else {
            // ---- 池已满, 直接归还系统 ----
            free(arena->prev);
        }

        // (3) 弹栈: 恢复 arena 到上一级状态
        *arena = tmp;
        // while 回到顶部, 继续处理上一级 chunk
    }

    // (4) 不变量检查: 释放完成后, arena 应恢复到空状态
    assert(arena->limit == NULL);
    assert(arena->avail == NULL);
}
```

### 5.6 `THRESHOLD` 和 `freechunks` 全局空闲链表

```c
#define THRESHOLD 10         // 最多缓存 10 个空闲 chunk
static T freechunks;         // 全局空闲 chunk 链表头
static int nfree;            // 当前空闲 chunk 数量
```

`freechunks` 是一个跨所有 Arena 实例共享的**全局空闲 chunk 池**:

```
freechunks -> [chunk A] -> [chunk B] -> [chunk C] -> NULL
                │prev        │prev        │prev
                │limit       │limit       │limit  (记录了各自的实际大小)
              [= m1 字节]  [= m2 字节]  [= m3 字节]
```

**为什么需要 `THRESHOLD` 限制?** Hanson 指出: `freechunks` 上的 chunk 对系统中其他分配器 (例如另一个 `Mem` 实例或第三方库) 来说**仍然显示为已分配内存**。如果无限制地缓存, 可能导致其他 `malloc` 调用因内存不足而失败。`THRESHOLD = 10` 配合每个 chunk 至少 10KB 的大小, 意味着全局缓存最多持有约 100KB 的内存, 这是一个保守但安全的界限。

**`limit` 字段的双重用途**:
- 在活跃 Arena 中: `arena->limit` 指向当前 chunk 结束处, 用于判断空间是否足够。
- 在 `freechunks` 中: `freechunks->limit` 记录该空闲 chunk 的实际大小边界, 供 `Arena_alloc` 在复用该 chunk 时正确设置新 `arena->limit`。

### 5.7 释放过程的 Chunk 回收示意图

```
释放前:                          释放后 (nfree < THRESHOLD):

 arena                             arena
   │                                 │
   v                                 v
 ┌────────┐   ┌────────┐           ┌────────┐   ┌────────┐
 │ struct │   │ chunk  │           │ struct │   │ chunk  │
 │  prev──┼──>│  prev──┼──>NULL     │  prev──┼──>│  prev──┼──┐
 │  avail │   │  avail │           │  avail │   │  avail │  │
 │  limit │   │  limit │           │  limit │   │  limit │  │
 ╞════════╡   ╞════════╡           ╞════════╡   ╞════════╡  │
 │ data   │   │ data   │           │ data   │   │ data   │  │
 └────────┘   └────────┘           └────────┘   └────────┘  │
  当前chunk    下一chunk             当前chunk    下一chunk    │
                  ↑ 要释放                          ↑ 加入 freechunks
                                                      │
                                          freechunks ─┘ (指向刚回收的 chunk)
                                          nfree++    (空闲计数增一)
```

粗线箭头表示 `freechunks` 指针的更新; 被回收的 chunk 从 Arena 链表中摘下, 接到 `freechunks` 链表头部。

---

## 六、Arena vs 通用 malloc 的适用场景对比

### 6.1 对比矩阵

| 特性 | `malloc/free` (通用) | Arena (区域式) |
|------|---------------------|-----------------|
| **分配粒度** | 单个对象 | 批量 (chunk 内多次分配) |
| **释放粒度** | 单个对象 (一一对应) | 整批 (一次 `Arena_free`) |
| **分配速度** | 慢 (需查空闲链表/分裂合并) | 极快 (仅 bump pointer, 无搜索) |
| **释放速度** | 慢 (需遍历链表合并碎片) | 快 (遍历 chunk 链表, O(k) where k=chunk 数) |
| **内存碎片** | 外部碎片 (可通过合并缓解) | 内部碎片 (chunk 末尾浪费) |
| **单对象重用** | 支持 (free 后内存可被任意后续 alloc 获取) | **不支持** (单个对象无法单独回收) |
| **内存峰值** | 低 (及时释放) | 高 (延迟释放) |
| **悬空指针** | 容易出现 (free 后使用) | 更容易出现 (整个 arena 释放后, 所有指针失效) |
| **编程负担** | 高 (每处 alloc 需对应 free) | 低 (只需关心 arena 生命周期) |
| **RAII 兼容性** | 差 (C 语言无自动析构) | 好 (arena 作用于特定逻辑阶段) |

### 6.2 何时用 Arena

1. **编译器和语言处理器**: 编译一个函数/模块期间大量分配中间表示 (IR) 节点, 编译完成后一起丢弃。`lcc` C 编译器全程采用此策略。
2. **解析器 (parser)**: 解析 JSON/XML/配置文件产生 AST (抽象语法树), 解析完成或出错后整棵树回收。
3. **事务处理**: 一个数据库事务处理过程中分配的临时内存, 提交或回滚后全部回收。
4. **请求-响应服务器**: 处理单次 HTTP 请求期间的临时分配, 响应发出后回收。
5. **批处理脚本**: 按阶段批量处理数据, 每个阶段用过即弃。
6. **交互式 UI 框架**: 窗口创建时分配控件内存, 窗口销毁时释放。

### 6.3 何时不该用 Arena

1. **长生命周期对象与短生命周期对象混用**: 如果 arena 中混合了不同生命周期的对象, 要么过早释放导致悬空指针, 要么延迟释放导致内存泄漏。应使用多个 arena 分别管理。
2. **频繁的单个大对象分配**: 每次分配 1MB 且 arena 中只有一两个对象, bump allocator 的快速路径优势被抹平, chunk 末尾浪费却很大。
3. **需要对象级内存复用**: 如果程序需要频繁分配/释放特定大小的对象 (如对象池), Arena 无法提供单对象粒度复用。

### 6.4 与 Mem 接口 (第5章) 的协作关系

第5章的 `Mem` 接口和本章的 `Arena` 接口是**互补而非互斥**的关系。

| 维度 | Mem (第5章) | Arena (本章) |
|------|-------------|-------------|
| **设计目的** | 为 `malloc/free` 添加带检查的包装器 | 提供区域式内存管理 |
| **分配原语** | `Mem_alloc`, `Mem_calloc` | `Arena_alloc`, `Arena_calloc` |
| **释放原语** | `Mem_free` (单块释放) | `Arena_free` (整区释放) |
| **resize 支持** | `Mem_resize` | **无** (Arena 不支持 resize -- 见习题 6.4) |
| **除错机制** | 哨兵字节、分配日志、`Mem_leak` | 断言 (`assert`)、`RAISE` 异常 |
| **底层实现** | 调用 `malloc/free` | 调用 `malloc/free` |
| **可互操作** | 是 (Arena 内部用 `malloc`, 外部可与 Mem 共存) | 是 |

**关键设计决策**: Arena 没有使用 `Mem_alloc` 而是直接使用 `malloc`。Hanson 对此的解释是 "so that it's independent of other allocators" (使其独立于其他分配器)。这是一个品味良好的模块化决策 -- Arena 作为低级基础设施, 不应当对更高层的 Mem 接口形成循环依赖。

在同一个程序中使用两者是完全合法的: 使用 `Mem_alloc` 分配跨阶段共享的对象, 使用 Arena 管理阶段性的临时对象。

**为什么 Arena 不支持 `Resize`?** (习题 6.4):

```c
void *Arena_resize(void **ptr, long nbytes, const char *file, int line)
```

因为 Arena 是纯 bump allocator。要 resize 一个已分配块: 原地扩大需要该块恰好是最后分配的; 异地重分配需要在新位置分配、复制内容、更新指针, 但 Arena 无法 "记住" 旧位置的归属关系。Mem 接口通过 `Mem_resize` 提供此能力 (基于底层 `realloc`), 但 Arena 的设计哲学决定了它不提供此功能。

### 6.5 共享组件: `union align`

第5章的 (`mem.c` 的 checking 实现) 和第6章 (`arena.c`) 都通过相同的 `union align` 来确定对齐边界。两个模块的对齐公式也完全一致:

```c
// mem.c (checking) 和 arena.c 中相同的代码模式
nbytes = ((nbytes + sizeof(union align) - 1) /
          (sizeof(union align))) * (sizeof(union align));
```

这种重复不是疏忽, 而是模块独立的体现: 两个模块各自维护自己的对齐逻辑, 确保即使其中一个被替换, 另一个也不受影响。

### 6.6 不同的异常策略

```c
// mem.h: 统一的异常
extern const Except_T Mem_Failed;

// arena.h: 两个异常, 分别对应不同的失败语义
extern const Except_T Arena_NewFailed;   // Arena 自身创建失败
extern const Except_T Arena_Failed;      // 内存分配失败
```

`Arena_NewFailed` 和 `Arena_Failed` 的分离反映了 Hanson 对失败模式的精细思考: `Arena_new` 失败意味着 "无法创建管理结构" (极罕见, 通常是系统资源彻底耗尽); `Arena_alloc` 失败意味着 "本次分配超出可用内存" (可能通过释放其他 arena 恢复)。客户端可能需要区别对待这两种失败。

### 6.7 Mem 宏对 Arena 的适用性

第5章的 `NEW`/`NEW0`/`FREE` 等宏是为 `Mem_alloc` 设计的。但在 CII 中, Arena 没有提供等价的宏。这是有意的设计选择: Arena 的 "分配后在 arena 销毁时自动释放" 语义使得类 `ALLOC` 宏的长度参数不够明显 (arena 句柄的传递打破了简单的宏包裹模式)。

```c
// Mem 风格 (简便但隐式)
NEW(p);             // p 指向 Mem_alloc 分配的内存

// Arena 风格 (显式但冗长)
p = Arena_alloc(my_arena, sizeof(*p), __FILE__, __LINE__);
```

在实际项目中, 可以为 Arena 封装类似的宏:

```c
#define ARENA_ALLOC(arena, p) \
    ((p) = Arena_alloc((arena), (long)sizeof *(p), __FILE__, __LINE__))
```

---

## 七、应用场景深度分析

### 7.1 编译器 (lcc) 中的 Arena

`lcc` (Fraser and Hanson 1995) 是 Arena 设计灵感的直接来源。在编译器中的典型使用模式:

```c
// 伪代码: 模拟 lcc 编译一个编译单元
void compile_unit(Arena_T permanent, const char *filename) {
    // 为当前编译单元创建临时 arena
    Arena_T func_arena = Arena_new();

    // 解析 -> 中间表示 (IR) -> 优化 -> 代码生成
    // 每一步都在 func_arena 中分配临时数据
    Symbol   *symbols = parse(filename, func_arena);    // AST
    IRNode   *ir      = semantic(symbols, func_arena);  // 类型检查
    IRNode   *opt     = optimize(ir, func_arena);        // 优化
    Code     *code    = codegen(opt, permanent);         // 目标代码 -> 永久 arena

    emit(code);
    Arena_free(func_arena);  // 一次性回收所有中间数据结构
    // 所有指向 parse/semantic/optimize 中间结果的指针在此失效
}
```

关键洞察: 编译过程中的 **绝大多数内存** (AST 节点, 类型信息, 临时符号表, 优化 pass 的中间表示) 都是阶段性的。只有最终的目标代码和全局符号表需要进入永久 arena 保存。Arena 将这种 "部分永久, 大部分临时" 的模式固化为接口。

### 7.2 解析器场景

```c
// JSON 解析器中的 Arena 使用
typedef struct JsonNode_T {
    int type;
    union {
        char   *str;
        double num;
        struct { struct JsonNode_T *head; size_t len; } arr;
        struct { struct JsonNode_T *head; size_t len; } obj;
    } value;
} *JsonNode_T;

// 整个 JSON 树分配在一个 arena 中
Arena_T json_arena = Arena_new();
JsonNode_T root = json_parse(input_string, json_arena);

// 使用 JSON 树...
process(root);

// 解析完成, 整棵树回收
Arena_dispose(&json_arena);
```

**零内存泄漏保证**: 只要 `json_arena` 被正确销毁 (即使解析中途出错退出), 就不会有泄漏。这与手动 `free` 整个 AST 相比, 代码量减少显著且安全性提升。

### 7.3 HTTP 请求处理

```c
void handle_request(int client_fd) {
    Arena_T req_arena = Arena_new();

    // 解析 HTTP 请求
    Request *req = parse_http(client_fd, req_arena);

    // 业务逻辑 (所有临时分配都在 req_arena 中)
    Response *resp = process_request(req, req_arena);

    // 发送响应
    send_response(client_fd, resp);

    // 请求结束, 回收一切
    Arena_dispose(&req_arena);  // <- 只需要这一行
}
```

对应地, 如果不使用 Arena, 每个 `parse_header`, `parse_body`, `build_template` 等子函数内部都需要记得释放自己分配的内存, 出错分支的清理代码将是灾难性的。

---

## 八、Modern-C 视角与改进

### 8.1 对齐的现代化

C11 引入了 `_Alignas` 和 `_Alignof` 关键字, 以及 `max_align_t` 类型, 使得对齐探测更加标准化:

```c
// CII 的风格 (C89 兼容)
union align {
    int i;
    long l;
    long *lp;
    void *p;
    void (*fp)(void);
    float f;
    double d;
    long double ld;
};

// Modern C (C11) 的等价写法
#include <stddef.h>
#include <stdalign.h>
// max_align_t 的 _Alignof 即平台的最大基本对齐值
#define ARENA_ALIGN alignof(max_align_t)
```

CII 的做法在 C89 时代是标准技巧; 现代 C 中可以更干净地表达, 但 CII 的策略仍然兼容且可移植。

### 8.2 灵活数组成员 (Flexible Array Member, FAM)

现代 C (C99+) 中, arena chunk 的布局可以用 FAM 更清晰地表达:

```c
// CII 的原始方案 (C89)
union header {
    struct T b;
    union align a;
};
// chunk 数据起始地址: (char *)((union header *)ptr + 1)
// 需要手动指针算术, 类型不安全

// Modern C (C99+) 的替代
struct Arena_Chunk {
    struct Arena_Chunk *prev;       // 替代在 struct T 中的 prev 字段
    size_t              data_size;  // 可选: 记录数据区大小
    _Alignas(max_align_t) char data[];  // FAM: 数据从这里开始
};
// chunk 数据起始地址: chunk->data  (类型安全!)
// 分配: malloc(sizeof(struct Arena_Chunk) + data_size)
```

使用 FAM 后:
1. `data` 数组直接在结构体内偏移, 不需要手动指针加法。
2. 对齐由 `_Alignas` 显式保证, 不依赖 `union header` 的隐式技巧。
3. 类型安全性提升: `chunk->data` 是 `char[]` 类型, 表达式意义清晰。

不过, CII 坚持 C89 兼容性, 所以保留了 `union header` 方案。在理解代码时, 需要认识到这是时代约束下的最优解。

### 8.3 Zig 语言中的 Arena

Zig 将 Arena 分配器作为标准库的一等公民:

```zig
const std = @import("std");

pub fn main() !void {
    // 创建 Arena 分配器
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();  // 作用域结束时自动释放
    const allocator = arena.allocator();

    // 所有分配通过 allocator 进行
    const list = try std.ArrayList(u8).initCapacity(allocator, 1024);
    const map = try std.StringHashMap(u32).init(allocator);

    // ... 使用 list 和 map ...
    // defer arena.deinit() 在此自动回收一切
}
```

Zig 的 Arena 与 CII 的设计有相同的 DNA: 批量生命周期管理, bump pointer 快速路径, 单点释放。Zig 的关键改进在于:
1. **显式传递**: 分配器作为参数传入每个需分配的函数, 消除了隐式全局状态的耦合。
2. **`defer` 支持**: `defer arena.deinit()` 保证退出作用域时自动回收, 在错误路径上尤其有用。
3. **可组合**: Zig 的所有分配器实现统一接口, Arena 可以包装其他分配器 (如 page_allocator)。

### 8.4 Rust 语言中的 Arena

Rust 中, arena 通常由第三方 crate 提供 (如 `typed-arena`, `bumpalo`), 配合生命周期系统发挥威力:

```rust
use bumpalo::Bump;

fn parse_json(input: &str) -> Vec<&JsonValue> {
    let arena = Bump::new();      // 创建 bump allocator
    let root = arena.alloc(parse_impl(input, &arena));
    // root 的生命周期被限制在 arena 的作用域内
    vec![root]  // <- 这行在 Rust 中不会编译! root 不能逃逸 arena
}               // arena 在此析构
```

Rust 的生命周期系统 (lifetime) 天然地解决了 CII 指出的 "对象分配在错误 arena 导致悬空指针" 的问题: 编译器在编译时通过借用检查器保证从 arena 分配的引用不会逃逸 arena 的作用域。这是类型系统对运行时安全问题的静态解决方案。

Zig 和 Rust 的设计都验证了 Hanson 的核心洞见 -- 区域式分配是一种足够普遍的模式, 值得作为基础语言设施。CII 在 1996 年, 用 110 行 C 代码, 完美地捕捉了这个抽象。

### 8.5 现代Arena设计的改进方向

| 原CII | Modern-C |
|--------|----------|
| `long nbytes` | `size_t nbytes` |
| `union align` 对齐 | `alignas(max_align_t)` |
| 手动 alignment 计算 | `align_up()` 内联函数 |
| 未检查 `count*nbytes` 溢出 | `__builtin_mul_overflow` |
| 一次 10KB+ 固定 chunk | 自适应 chunk 大小 (bump allocator 2x 增长) |
| `THRESHOLD=10` 固定缓存 | 可配置或完全释放策略 |
| 无原子操作 | 线程局部 Arena (TLS) 消除锁竞争 |
| 静态空闲链表 | 大对象直通 (mmap) 避免污染 |

---

## 九、完整使用示例

```c
#include "arena.h"
#include <stdio.h>
#include <string.h>

// 模拟编译器: AST节点
typedef struct ASTNode {
    int type;
    void *left;
    void *right;
    char *value;
} ASTNode;

ASTNode *make_node(Arena_T arena, int type, const char *value) {
    ASTNode *n = Arena_alloc(arena, sizeof *n, __FILE__, __LINE__);
    n->type = type;
    n->left = n->right = NULL;
    // value 字符串也在 arena 中分配
    n->value = Arena_alloc(arena, strlen(value) + 1, __FILE__, __LINE__);
    strcpy(n->value, value);
    return n;
}

int main(void) {
    Arena_T arena = Arena_new();

    // 解析一个简单的表达式: "a + b"
    ASTNode *plus = make_node(arena, '+', "plus");
    plus->left  = make_node(arena, 'v', "a");    // 变量 a
    plus->right = make_node(arena, 'v', "b");    // 变量 b

    printf("AST: (%c %s %s)\n",
           plus->type,
           ((ASTNode *)plus->left)->value,
           ((ASTNode *)plus->right)->value);
    // 输出: AST: (+ a b)

    // 一次释放所有 AST 节点和字符串!
    Arena_dispose(&arena);

    return 0;
}
```

---

## 十、总结

Arena 分配器用 110 行 C 代码实现了一个完整的内存管理子系统, 其设计体现了以下原则:

1. **最小的接口, 最强的语义**: 五个导出函数涵盖创建/分配/清零/批量释放/销毁的所有需求。
2. **Bump allocation 的极速路径**: 大部分分配只需一次对齐和一次指针加法。
3. **全局空闲链表**: 跨 Arena 实例复用 chunk, 减少 `malloc` 系统调用。
4. **THRESHOLD 节制**: 限制缓存以避免伪内存泄漏。
5. **状态栈模式**: `*ptr = *arena` + `*arena = tmp` 构成隐式 push/pop, 优雅地管理了 chunk 链表。
6. **独立于 Mem**: 直接使用 `malloc/free`, 保持模块的可替换性。

Hanson 在 Further Reading 中指出的研究方向 -- 自动选择 arena (Barrett and Zorn 1993)、通用区域分配器 (Vo 1996, Vmalloc)、保守式垃圾收集 (Boehm and Weiser 1988) -- 至今仍是内存管理研究的活跃课题。30 年后的今天, Zig 和 Rust 标准库中的 Arena 实现, 本质上仍遵循着这本书中阐述的 "基于生命周期的内存管理" 哲学。

---

## 参考资源

- Hanson, D. R. (1990). "Fast allocation and deallocation of memory based on object lifetimes." *Software -- Practice and Experience*, 20(1), 5-12.
- Fraser, C. W. and Hanson, D. R. (1995). *A Retargetable C Compiler: Design and Implementation*. Addison-Wesley.
- Barrett, D. A. and Zorn, B. G. (1993). "Using lifetime predictors to improve memory allocation performance." *PLDI '93*.
- Vo, K.-P. (1996). "Vmalloc: A General and Efficient Memory Allocator." *Software -- Practice and Experience*, 26(3), 357-374.
- Boehm, H.-J. and Weiser, M. (1988). "Garbage Collection in an Uncooperative Environment." *Software -- Practice and Experience*, 18(9), 807-820.
- Zig Standard Library: `std.heap.ArenaAllocator` -- https://ziglang.org/documentation/master/std/#std.heap.ArenaAllocator
- `bumpalo` crate (Rust): https://crates.io/crates/bumpalo
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch05_内存管理I_Memory.html">第5章 内存管理 I</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch07_链表_List.html">第7章 链表 List</a></div>
</div>
