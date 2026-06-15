# 第5章: 内存管理 I (Memory Management)

> 原文: *C Interfaces and Implementations*, David R. Hanson, Chapter 5, pp. 67-88
> 源文件: `code/mem.h`, `code/mem.c`, `code/memchk.c`, `code/except.h`, `code/assert.h`

---

## 一、设计动机: 为什么封装 malloc/free?

所有非平凡的C程序都在运行时动态分配内存。标准C库提供四个函数: `malloc`、`calloc`、`realloc`、`free`。这四个函数虽然功能齐全，但接口设计存在多个陷阱，导致C程序中的内存管理错误极为普遍且难以调试。

### 1.1 经典内存管理错误四类

**错误1: 悬空指针 (Dangling Pointer)**

```c
p = malloc(nbytes);
/* ... 使用 p ... */
free(p);
/* 此后 p 是悬空指针 -- 指向逻辑上已不存在的内存 */
/* 解引用 p 是未定义行为，但若该块未被重新分配，错误可能长期不被发现 */
```

这种错误最危险的特征是: 发现错误的地点和时间可能与错误的源头相距甚远。

**错误2: 重复释放 (Double Free)**

```c
p = malloc(nbytes);
free(p);
free(p);  /* 释放已释放的内存 -- 通常破坏malloc内部数据结构 */
          /* 但错误可能直到下一次调用malloc/free时才会显现 */
```

**错误3: 释放非堆内存**

```c
char buf[20], *p;
if (n >= sizeof buf)
    p = malloc(n);
else
    p = buf;     /* p 指向栈上的局部数组 */
free(p);         /* 当p指向buf时，这是致命的 -- 破坏malloc内部状态 */
```

**错误4: 内存泄漏 (Memory Leak)**

```c
void itoa(int n, char *buf, int size) {
    char *p = malloc(43);    /* 分配43字节 */
    sprintf(p, "%d", n);
    if (strlen(p) >= size - 1) {
        while (--size > 0) *buf++ = '*';
        *buf = '\0';
    } else
        strcpy(buf, p);
    /* 忘记 free(p)! -- 每次调用泄漏43字节 */
    /* 如果itoa被频繁调用，程序将逐渐耗尽内存 */
}
```

此函数还包含另一个错误: 未检查 `malloc` 的返回值是否为NULL。

### 1.2 CII Mem接口的设计目标

CII (C Interfaces and Implementations) 的 `Mem` 接口针对上述问题提供了系统性的防御机制:

| 设计目标 | 实现机制 | 解决的问题 |
|---------|---------|----------|
| 永不返回NULL指针 | 分配失败时抛出 `Mem_Failed` 异常 | 消除遗漏空指针检查导致的崩溃 (错误4) |
| 拒绝零/负大小分配 | `assert(nbytes > 0)` 运行时检查 | 将实现定义行为转化为受检查的运行时错误 |
| 自动记录调用点 | 宏自动注入 `__FILE__` 和 `__LINE__` | 错误报告精确定位到调用者 |
| 消除悬空指针 | `FREE` 宏释放后将指针置为NULL | 解引用NULL产生确定性的段错误 (错误1) |
| 检测非法释放 | 检查型实现维护哈希表记录所有分配 | 捕获重复释放 (错误2) 和释放非堆内存 (错误3) |
| 使用 `long` 而非 `size_t` | 避免负数到巨大的无符号数的隐式转换 | 防御 `malloc(-1)` 这类隐蔽bug |

---

## 二、接口定义: mem.h 逐行注释

### 2.1 完整接口

```c
/* mem.h -- CII 内存管理接口 */
#ifndef MEM_INCLUDED
#define MEM_INCLUDED
#include "except.h"

/* ── 异常声明 ── */
extern const Except_T Mem_Failed;
/* 当分配失败时抛出此异常。reason字段为 "Allocation Failed"。
   若未被捕获，Except_raise 会打印:
     Uncaught exception Allocation Failed raised @parse.c:431
     aborting... */

/* ── 底层函数 (需手动传递调用点信息) ── */
extern void *Mem_alloc (long nbytes,
    const char *file, int line);
/* 分配至少nbytes字节的未初始化内存块。
   块地址满足最严格的对齐要求。
   nbytes不能为0或负数 (受检查的运行时错误)。
   永不返回NULL -- 分配失败时抛出 Mem_Failed。
   file和line用于异常报告定位调用点。 */

extern void *Mem_calloc(long count, long nbytes,
    const char *file, int line);
/* 分配足以容纳count个元素、每个nbytes字节的数组，初始化为全零。
   count和nbytes均不能为0或负数。
   ⚠ 注意: NULL指针和0.0的位模式不一定是全零! */

extern void Mem_free(void *ptr,
    const char *file, int line);
/* 释放ptr指向的内存块。ptr为NULL时无操作。
   检查型实现中，ptr必须是由Mem_alloc/Mem_calloc/Mem_resize
   返回的有效已分配地址，否则触发Assert_Failed异常。 */

extern void *Mem_resize(void *ptr, long nbytes,
    const char *file, int line);
/* 改变之前分配的块的大小为至少nbytes字节。
   ptr必须非NULL (与realloc不同!)。
   nbytes必须为正数。
   可能移动块到新地址；返回新地址（或原地址）。
   若nbytes大于原大小，超出部分未初始化。
   若nbytes小于原大小，前nbytes字节被保留。 */

/* ── 顶层宏 (自动注入调用点) ── */
#define ALLOC(nbytes) \
    Mem_alloc((nbytes), __FILE__, __LINE__)
/*
 * 用法: int *p = ALLOC(100 * sizeof(int));
 *
 * __FILE__: C预处理器标准宏，展开为当前源文件名的字符串字面量
 * __LINE__: C预处理器标准宏，展开为当前行号的十进制常量
 *
 * 这两个宏在编译时展开，使得异常报告能精确定位到调用者的源代码位置，
 * 而非Mem内部的malloc调用处。这是零开销的调试信息注入。
 *
 * (nbytes) 外加括号防止宏展开时的运算符优先级问题:
 *   ALLOC(n+1) → Mem_alloc((n+1), ...)  ✓
 *   ALLOC n+1  → Mem_alloc(n+1, ...)    ✗ (宏定义无括号时会出错)
 */

#define CALLOC(count, nbytes) \
    Mem_calloc((count), (nbytes), __FILE__, __LINE__)
/*
 * 用法: int *arr = CALLOC(100, sizeof(int));  → 400字节零初始化数组
 *
 * 语义与标准calloc一致但更严格:
 *   count=0 或 nbytes=0 → 断言失败 (受检查的运行时错误)
 *   而标准calloc(0, n)的行为是实现定义的
 */

#define  NEW(p)  ((p) = ALLOC((long)sizeof *(p)))
/*
 * 用法: struct Point *pt; NEW(pt);
 *
 * 这是CII最精巧的设计之一。逐层分析:
 *
 * (1) sizeof *(p)
 *     sizeof 是编译期运算符，其操作数在运行时不被求值。
 *     此处 *(p) 仅用于获取 p 所指对象的类型信息。
 *     因此无论 p 的值是什么 (甚至是NULL), sizeof *(p) 都能正确计算。
 *
 * (2) (long)sizeof *(p)
 *     sizeof 返回 size_t (无符号整型)。显式转换为long是为了匹配
 *     Mem_alloc 的参数类型。这种转换在绝大多数平台上安全。
 *
 * (3) ((p) = ALLOC(...))
 *     分配内存并将返回地址赋给 p。
 *     整个宏表达式返回分配后的 p 值。
 *
 * (4) 单次求值保证
 *     p 在整个宏展开中仅出现一次 (等号左侧)，
 *     因此 NEW(a[i++]) 是安全的 -- i++ 仅执行一次。
 *
 * 类型安全优势:
 *   若 p 的类型从 struct Point 变为 struct Point3D,
 *   NEW(p) 自动适配新类型的大小，无需修改。
 *   相比之下，p = Mem_alloc(sizeof(struct Point)) 必须手动更新。
 */

#define NEW0(p)  ((p) = CALLOC(1, (long)sizeof *(p)))
/*
 * 用法: struct Point *pt; NEW0(pt);
 *
 * 与NEW相同的类型推导机制，但额外将分配的内存清零。
 * 适用于包含指针字段的结构体 -- 零初始化确保所有指针为NULL。
 *
 * ⚠ 注意: CALLOC内部使用calloc实现零初始化。
 *   虽然NULL指针和0.0在所有主流平台上确实是全零位模式，
 *   但C标准并不保证这一点。这是CII文档明确指出的已知限制。
 */

#define FREE(ptr) ((void)(Mem_free((ptr), \
    __FILE__, __LINE__), (ptr) = 0))
/*
 * 用法: FREE(p);
 *
 * 这是C语言中宏编程的经典范例，利用逗号运算符实现多操作序列化:
 *
 * (1) Mem_free((ptr), __FILE__, __LINE__)
 *     首先释放内存。如果ptr为NULL，Mem_free无操作。
 *
 * (2) (ptr) = 0
 *     然后将ptr赋值为0 (NULL指针)。
 *     这是防止悬空指针的关键: 此后任何对*ptr的解引用
 *     都会产生确定性的段错误 (SIGSEGV)，而非随机的未定义行为。
 *
 * (3) (void)(..., ...)
 *     逗号表达式返回最后一个子表达式的值 (此处为0)。
 *     外层(void)转换抑制可能出现的"value computed is not used"编译器警告。
 *
 * ⚠ 多重求值警告: FREE对ptr求值两次 (传给Mem_free和赋值)，
 *   因此 FREE(*ptr++) 是错误的使用方式。
 *
 * 对比标准free: free(p) 之后 p 仍然是原来的地址值 (悬空指针)。
 *   FREE(p) 之后 p 变为NULL。
 */

#define RESIZE(ptr, nbytes)  ((ptr) = Mem_resize((ptr), \
    (nbytes), __FILE__, __LINE__))
/*
 * 用法: RESIZE(p, 200);
 *
 * 调用Mem_resize并将新地址写回ptr (因为块可能被移动)。
 *
 * Mem_resize 对比标准 realloc 的简化:
 *   ┌────────────────────┬──────────────────┬──────────────────────┐
 *   │      条件          │   realloc行为     │   Mem_resize行为     │
 *   ├────────────────────┼──────────────────┼──────────────────────┤
 *   │ ptr == NULL        │ 等价于malloc(n)  │ 断言失败 (不允许)    │
 *   │ nbytes == 0        │ 等价于free(ptr)  │ 断言失败 (不允许)    │
 *   │ ptr!=NULL,nbytes>0 │ 调整大小         │ 调整大小             │
 *   │ 分配失败            │ 返回NULL,原块保留 │ 抛出异常             │
 *   └────────────────────┴──────────────────┴──────────────────────┘
 *
 * realloc 将三种不相关的功能 (分配/释放/调整) 捆绑到一个函数中，
 * 这种过载设计是bug的温床。Mem_resize 的简化语义消除了歧义。
 */
#endif
```

### 2.2 NEW宏的设计精要: 类型安全分配

`NEW` 宏体现了CII库设计的核心哲学: 利用C语言的编译期特性消除运行时错误。

```c
/* 不良实践 -- sizeof操作数与类型耦合 */
struct Point *p;
p = Mem_alloc(sizeof(struct Point));   /* 若 p 的类型改变，此行也需要改 */
p = Mem_alloc(sizeof *p);              /* 更好: 类型推导 */
NEW(p);                                /* 最好: 一条语句完成声明+分配+赋值 */

/* 更关键的是: sizeof *p 适用于任何可解引用的指针类型 */
int    *ip;  NEW(ip);   /* 分配 sizeof(int) 字节         */
double *dp;  NEW(dp);   /* 分配 sizeof(double) 字节      */
struct Large *lp; NEW(lp); /* 分配 sizeof(struct Large) 字节 */
/* 所有调用形式完全一致，代码不随类型变化而变化 */
```

---

## 三、生产级实现 (mem.c) -- 薄封装

生产级实现 (`code/mem.c`) 是对标准库函数的薄包装，核心价值在接口规范化而非算法创新。

### 3.1 完整源码详解

```c
/* ── 头文件与数据 ── */
#include <stdlib.h>
#include <stddef.h>
#include "assert.h"
#include "except.h"
#include "mem.h"

const Except_T Mem_Failed = { "Allocation Failed" };
/*
 * Except_T 结构体定义在 except.h 中:
 *   typedef struct T { const char *reason; } T;
 * Mem_Failed.reason = "Allocation Failed"
 * 当异常未被捕获时，Except_raise 使用此字符串报告错误。
 */
```

**Mem_alloc -- 带异常处理的 malloc 包装**:

```c
void *Mem_alloc(long nbytes, const char *file, int line) {
    void *ptr;

    assert(nbytes > 0);          /* §5.1: nbytes必须为正数 */
    /*
     * CII的assert宏 (assert.h):
     *   #define assert(e) ((void)((e)||(RAISE(Assert_Failed),0)))
     *
     * 与标准assert不同: CII的assert失败时抛出 Assert_Failed 异常，
     * 而非调用abort()。这意味着异常可以被上层TRY/EXCEPT捕获处理。
     * 在NDEBUG模式下，assert(e)展开为((void)0)，零开销。
     */

    ptr = malloc(nbytes);         /* 委托给标准库 */

    if (ptr == NULL) {            /* malloc失败 */
        if (file == NULL)
            RAISE(Mem_Failed);
            /*
             * RAISE(e) 展开为 Except_raise(&(e), __FILE__, __LINE__)
             * 此处 __FILE__ 和 __LINE__ 指向 Mem_alloc 内部。
             * file==NULL 仅当Mem_alloc被直接调用(而非通过ALLOC宏)时发生。
             */
        else
            Except_raise(&Mem_Failed, file, line);
            /*
             * 使用调用者提供的 file/line，使得异常报告指向:
             *   Uncaught exception Allocation Failed raised @parse.c:431
             *   而非 Mem_alloc 内部的 malloc 调用处
             */
    }

    return ptr;
}
```

**为什么用 `long nbytes` 而非 `size_t`?**

```c
/* 如果使用 size_t (unsigned)，这个bug极难发现: */
int n = -1;
p = malloc(n);     /* -1 → size_t 变成 4294967295 (32位) 或更大的值 */
                   /* malloc 几乎必然失败，但错误信息不清晰 */

/* 使用 long (signed): */
p = ALLOC(n);      /* assert(n > 0) 立即捕获负数 → 清晰的断言失败 */
```

Hanson 选择 `long` 而非 `size_t` 是为了在有符号/无符号边界处提供明确的错误检测。这在32位时代尤为重要。在64位Linux上 `long` 是8字节，可以完整表示 `size_t` 的范围；但在64位Windows上 `long` 是4字节，存在截断风险。Modern-C应使用 `size_t` 配合显式的符号检查。

**Mem_calloc -- 双重参数验证**:

```c
void *Mem_calloc(long count, long nbytes,
    const char *file, int line) {
    void *ptr;

    assert(count > 0);           /* 不允许零计数 */
    assert(nbytes > 0);          /* 不允许零元素大小 */
    /*
     * 双重断言消除了标准calloc(0, n)的实现定义行为。
     * 在CII中，count==0或nbytes==0始终是受检查的运行时错误。
     */

    ptr = calloc(count, nbytes); /* calloc内部: count*nbytes 并清零 */

    if (ptr == NULL) {
        if (file == NULL)
            RAISE(Mem_Failed);
        else
            Except_raise(&Mem_Failed, file, line);
    }
    return ptr;
}
```

**Mem_free -- 空指针安全的free包装**:

```c
void Mem_free(void *ptr, const char *file, int line) {
    if (ptr)
        free(ptr);
    /*
     * 显式NULL检查。
     * 虽然C89及之后的标准允许 free(NULL)，但某些老旧的free实现
     * 不接受NULL参数。这行防御性代码保证了跨平台兼容性。
     *
     * file和line参数在生产级实现中未使用，但保留以保持与检查型
     * 实现的接口一致。
     */
}
```

**Mem_resize -- 简化语义的realloc包装**:

```c
void *Mem_resize(void *ptr, long nbytes,
    const char *file, int line) {
    assert(ptr);                 /* ptr必须非NULL -- 与realloc的关键区别 */
    assert(nbytes > 0);          /* nbytes必须为正数 -- 与realloc的关键区别 */

    ptr = realloc(ptr, nbytes);  /* 委托给标准库 */

    if (ptr == NULL) {
        if (file == NULL)
            RAISE(Mem_Failed);
        else
            Except_raise(&Mem_Failed, file, line);
    }
    return ptr;
}
```

`realloc` 的行为矩阵极其复杂 -- 它将分配、释放、调整三种不相关的功能捆绑在一个函数中。CII通过断言将其简化为单一职责: 调整已有块的大小。

### 3.2 性能开销分析 (生产级)

| 操作 | 额外开销 vs 直接调用malloc/free | 主要开销来源 |
|------|-------------------------------|------------|
| `ALLOC` | ~5-10条指令 | 断言(1) + 函数调用(1) + NULL检查(1) + 条件分支 |
| `CALLOC` | ~5-10条指令 | 同上，assert数量翻倍 |
| `FREE` | ~3-5条指令 | NULL检查 + 赋值NULL (逗号表达式) |
| `RESIZE` | ~5-10条指令 | 断言 + NULL检查 + 条件分支 |

生产级实现的性能开销**可忽略不计** (< 1%)。宏展开带来的额外指令在大多数CPU上仅增加几个时钟周期。如果定义了 `NDEBUG`，断言开销完全消除。

---

## 四、检查型实现 (memchk.c) -- 完整的内存跟踪系统

检查型实现是CII内存管理系统的精髓，通过完整的分配记录和验证机制来检测内存访问错误。

### 4.1 核心数据结构

#### 对齐联合体 (Alignment Union)

```c
union align {
#ifdef MAXALIGN
    char pad[MAXALIGN];          /* 平台特定的最大对齐覆盖 */
#else
    int i;                       /* 4字节对齐 */
    long l;                      /* 4或8字节对齐 */
    long *lp;                    /* 指针大小对齐 */
    void *p;                     /* 指针大小对齐 */
    void (*fp)(void);            /* 函数指针对齐 */
    float f;                     /* 4字节对齐 */
    double d;                    /* 8字节对齐 (大多数平台) */
    long double ld;              /* 10/12/16字节对齐 (平台相关) */
#endif
};
/*
 * 设计思想: union align 的大小等于其最大成员的大小，
 * 其对齐要求等于其最严格成员的对齐要求。
 *
 * 通过包含所有常见的基本类型，此联合体自然获得了
 * 平台上最严格的对齐要求。
 *
 * sizeof(union align) 在典型平台上的值:
 *   - x86-64 Linux:   16 字节 (long double)
 *   - x86-64 Windows: 16 字节
 *   - ARM64:          16 字节
 *   - x86 (32-bit):   8 字节 (double)
 *
 * MAXALIGN宏允许在编译时覆盖此默认值:
 *   cc -DMAXALIGN=64 ...   → 强制64字节对齐 (如AVX-512)
 */
```

#### 内存块描述符 (Descriptor)

```c
#define hash(p, t) (((unsigned long)(p)>>3) & \
    (sizeof (t)/sizeof ((t)[0])-1))
/*
 * 哈希函数: 将地址右移3位后取模。
 *
 * 右移3位: 由于所有分配地址都是 sizeof(union align) 的倍数，
 * 至少是 8 (或16) 的倍数，低3位不包含有用信息。
 * 移位后得到了更均匀分布的哈希输入。
 *
 * sizeof(t)/sizeof((t)[0])-1: 计算哈希表大小的掩码。
 * 此技巧仅在表大小是2的幂时有效。
 * 对于 htab[2048]: sizeof(htab)/sizeof(htab[0]) = 2048,
 * 掩码为 2048-1 = 0x7FF → 等价于 % 2048 但更高效。
 */

#define NDESCRIPTORS 512
/*
 * 描述符批量分配的单位大小。
 * dalloc一次从malloc获取512个描述符，然后逐个分发。
 * 512的选择是经验性的:
 *   - 太小 → 频繁调用malloc
 *   - 太大 → 浪费未使用的描述符空间 (512*~32B ≈ 16KB)
 */

#define NALLOC ((4096 + sizeof (union align) - 1)/ \
    (sizeof (union align)))*(sizeof (union align))
/*
 * 从系统获取新内存大块时的增量大小。
 *
 * 计算公式: 向上取整为 sizeof(union align) 的倍数。
 *   若 sizeof(union align)=16: NALLOC = ((4096+15)/16)*16 = 4096
 *   若 sizeof(union align)=8:  NALLOC = ((4096+7)/8)*8 = 4096
 *
 * 实际效果: ~4KB，对应于许多系统的页面大小。
 * 此策略减少了系统调用 (malloc) 的频率。
 */

static struct descriptor {
    struct descriptor *free;     /* 空闲链表指针; NULL = 已分配; 非NULL = 已释放 */
    struct descriptor *link;     /* 哈希桶链表指针 (处理哈希冲突) */
    const void *ptr;             /* 内存块地址 (Mem_alloc返回给用户的地址) */
    long size;                   /* 内存块大小 (字节) */
    const char *file;            /* 分配时的源文件名 */
    int line;                    /* 分配时的源行号 */
} *htab[2048];
/*
 * htab 是一个包含2048个桶的指针数组。
 * 每个桶指向一个descriptor链表 (通过link字段链接)。
 * 所有分配过的块 (包括已释放和未释放的) 都在 htab 中有记录。
 *
 * 整个结构的作用等同于数学抽象:
 *   集合 S = { (ptr, state) | state ∈ {allocated, free} }
 *   其中 ptr 是Mem_alloc/calloc/resize返回的地址
 */

static struct descriptor freelist = { &freelist };
/*
 * freelist 是空闲链表的哑元哨兵 (dummy sentinel)。
 *
 * 关键属性:
 *   - freelist 自己是链表的最后一个节点 (循环)
 *   - freelist.free 指向第一个真正的空闲描述符
 *   - 每个空闲描述符的 free 字段指向下一个空闲描述符
 *   - 遍历终止条件: bp == &freelist (回到起点)
 *
 * 初始化: freelist = { &freelist } 表示空链表:
 *   freelist.free == &freelist → 自己指向自己 → 链表为空
 */
```

### 4.2 内存块布局 (ASCII图)

```
┌──────────────────────────────────────────────────────────────────┐
│                    从系统malloc获取的大块布局                      │
│                                                                  │
│  ┌──────────────────────────────────────────────────┐            │
│  │   保留区域 (Padding / Guard Zone)                 │            │
│  │   大小 = sizeof(union align) 字节                 │            │
│  │   永不分配，充当块前护栏 (pre-block guard)        │            │
│  └──────────────────────────────────────────────────┘            │
│  ┌────────────────────┐ ┌──────────────────────────────────────┐ │
│  │  已分配块 #1       │ │  空闲空间 (可被子序列分配切分)       │ │
│  │  ┌───────────────┐ │ │                                      │ │
│  │  │ descriptor#1  │ │ │  ┌────────────────────────────────┐  │ │
│  │  │ free = NULL   │ │ │  │ 已分配块 #2                    │  │ │
│  │  │ ptr ──────────┼─┼─┼──→ (从此空闲块尾部切出)           │  │ │
│  │  │ size = s1     │ │ │  │ ┌────────────────────────────┐ │  │ │
│  │  │ file/line     │ │ │  │ │ descriptor#2               │ │  │ │
│  │  └───────────────┘ │ │  │ │ free = NULL                │ │  │ │
│  │        (独立存储)  │ │  │ │ ptr ───────────────────┐   │ │  │ │
│  └────────────────────┘ │  │ │ size = s2              │   │ │  │ │
│                         │  │ └────────────────────────┘   │ │  │ │
│  ┌────────────────────┐ │  │            (独立存储)        │ │  │ │
│  │  剩余空闲空间      │ │  └──────────────────────────────┘ │  │ │
│  │  (若再有分配请求     │ │                                  │  │ │
│  │   继续从尾部切分)   │ │                                  │  │ │
│  └────────────────────┘ │                                      │ │
│                         └──────────────────────────────────────┘ │
│  块起始地址 (ptr_start)                                          │
│  对齐于 sizeof(union align)                                      │
└──────────────────────────────────────────────────────────────────┘

关键设计决策:
  (1) 从块尾部切分 → bp->ptr 永远不会被重复使用
      (同一地址不可能被 Mem_alloc 返回两次)
  (2) 描述符与数据块物理隔离 → 缓冲区溢出不破坏描述符
  (3) 块前保留区域 → 充当护栏，越界写入不会立即覆盖相邻块


                  哈希表与空闲链表结构 (逻辑视图)
                  ═════════════════════════════════

    htab[2048]                       freelist (哑元哨兵)
    ┌────────────┐                   ┌──────────────┐
    │ htab[0]  ──┼────→ desc_A      │ free ════════╪══→ desc_X (空闲块)
    ├────────────┤      free=NULL   │ link         │    ┌──────────────┐
    │ htab[1]    │      link ──────┼──→ desc_B     │    │ free ════════╪══→ desc_Y (空闲块)
    ├────────────┤      ptr=0xA00  │   free!=NULL  │    │ link ────────┼──→ desc_Z
    │ htab[2]    │─→ NULL           │   link ──────┼──→ desc_C      │    │ ptr=0xC00    │
    ├────────────┤                  │   ptr=0xA80  │   free=NULL    │    │ size=256     │
    │    ...     │                  │   size=128   │   link ────────┼──→ NULL        │
    ├────────────┤                  └──────────────┘   ptr=0xB00    │    │ file=gen.c  │
    │ htab[1023] │─→ NULL                                 size=64   │    │ line=42     │
    └────────────┘                               file=parse.c      │    └──────────────┘
                                                 line=137         │
    ───  实线 = link 字段 (哈希桶内链表)              └──────────────┘
    ═══  虚线 = free 字段 (空闲链表)
          free == NULL  = 已分配 (allocated) 状态
          free != NULL  = 已释放 (free) 状态
```

### 4.3 指针合法性验证 -- 三层防御

```c
/* 验证代码位于 Mem_free 和 Mem_resize 中: */
if (((unsigned long)ptr) % (sizeof (union align)) != 0
    || (bp = find(ptr)) == NULL
    || bp->free)
    Except_raise(&Assert_Failed, file, line);
```

这是一个逻辑短路 (short-circuit) 的三层防御体系:

**第1层: 对齐检查 (Alignment Guard) -- O(1)**

```c
((unsigned long)ptr) % (sizeof (union align)) != 0
```

- 如果 `ptr` 不是 `sizeof(union align)` 的倍数，它**绝不可能是** `Mem_alloc` 返回的地址
- 单次取模运算，极为廉价
- 快速过滤掉绝大多数非法指针: 栈变量、全局变量、结构体成员、字符串字面量等
- 由于 `Mem_alloc` 保证所有返回地址对齐于 `sizeof(union align)`，任何非对齐地址必然非法

**第2层: 哈希表查找 (Descriptor Lookup) -- O(1) 平均**

```c
(bp = find(ptr)) == NULL
```

- 在 `htab` 哈希表中搜索该地址的描述符
- `find` 先通过哈希定位桶，再遍历桶内链表
- 如果找不到描述符，说明此地址**从未被Mem分配过**
- 每个块的描述符在分配时创建并加入 `htab`，永不删除

`find` 函数实现:

```c
static struct descriptor *find(const void *ptr) {
    struct descriptor *bp = htab[hash(ptr, htab)];
    /* hash(ptr, htab) = ((unsigned long)(ptr)>>3) & 2047
     * 即取地址的高位信息，映射到 [0, 2047] 范围的桶索引 */
    while (bp && bp->ptr != ptr)
        bp = bp->link;
    /* 遍历桶内链表，比较 ptr 字段 */
    return bp;
    /* 返回NULL表示未找到 (非法地址) */
}
```

**第3层: 重复释放检测 (Double-Free Detection) -- O(1)** (在找到描述符后)

```c
bp->free
```

- 如果描述符的 `free` 字段**非NULL**，说明该块已经被释放过
- 已释放的描述符通过 `free` 字段参与空闲链表
- 空闲链表节点: `free != NULL` (指向下一个空闲描述符)
- 已分配节点: `free == NULL` (不在空闲链表中)
- 这是通过数据结构状态而非计数器实现的重复释放检测

### 4.4 首次适配 (First-Fit) 分配算法

```c
void *Mem_alloc(long nbytes, const char *file, int line) {
    struct descriptor *bp;
    void *ptr;

    assert(nbytes > 0);

    /* ═══ 步骤1: 向上取整到对齐边界 ═══ */
    nbytes = ((nbytes + sizeof (union align) - 1) /
              (sizeof (union align))) * (sizeof (union align));
    /*
     * 确保每个返回地址都是 sizeof(union align) 的倍数。
     *
     * 例: sizeof(union align)=16, nbytes=20
     *   nbytes = ((20+15)/16)*16 = (35/16)*16 = 2*16 = 32
     *
     * 例: sizeof(union align)=16, nbytes=16
     *   nbytes = ((16+15)/16)*16 = (31/16)*16 = 1*16 = 16
     *
     * 这是经典的对齐上舍入公式。
     */

    /* ═══ 步骤2: 首次适配搜索 ═══ */
    for (bp = freelist.free; bp; bp = bp->free) {
        /*
         * 遍历空闲链表。
         * freelist.free 指向第一个空闲描述符。
         * 链表是循环的，通过 freelist 的 free 字段连接。
         */

        if (bp->size > nbytes) {
            /* ═══ 步骤2a: 从块尾部切出 nbytes 字节 ═══ */
            bp->size -= nbytes;
            ptr = (char *)bp->ptr + bp->size;
            /*
             * 从块尾部切分是关键设计:
             *   - bp->ptr (块首地址) 永不改变
             *   - 新分配的 ptr 指向块的尾部
             *   - 下次分配可能从此块的剩余空间中继续切分
             *   - 同一地址永远不会被返回两次 (因为只从尾部切)
             */

            /* ═══ 步骤2b: 创建新描述符 ═══ */
            if ((bp = dalloc(ptr, nbytes, file, line)) != NULL) {
                /* ═══ 步骤2c: 加入哈希表 ═══ */
                unsigned h = hash(ptr, htab);
                bp->link = htab[h];     /* 头插法加入哈希桶 */
                htab[h] = bp;
                return ptr;             /* 成功! */
            } else {
                /* dalloc中的malloc失败 → 抛出异常 */
                if (file == NULL)
                    RAISE(Mem_Failed);
                else
                    Except_raise(&Mem_Failed, file, line);
            }
        }

        if (bp == &freelist) {
            /* ═══ 步骤2d: 空闲链表耗尽 → 向系统申请新内存 ═══ */
            struct descriptor *newptr;

            /* 申请 nbytes + NALLOC (~4KB) 字节的大块 */
            if ((ptr = malloc(nbytes + NALLOC)) == NULL
                || (newptr = dalloc(ptr, nbytes + NALLOC,
                        __FILE__, __LINE__)) == NULL) {
                if (file == NULL)
                    RAISE(Mem_Failed);
                else
                    Except_raise(&Mem_Failed, file, line);
            }

            /* 将新大块加入空闲链表头部 */
            newptr->free = freelist.free;
            freelist.free = newptr;
            /*
             * 新的空闲描述符被插入到 freelist 之后。
             * 在下一次for循环迭代中，它将是第一个被检查的块。
             * 由于 size = nbytes + NALLOC > nbytes，
             * 它将在步骤2a中被用来满足本次分配请求。
             */
        }
    }

    /* ═══ 不可达代码 ═══ */
    assert(0);  /* 如果到达此处，空闲链表结构已损坏 */
    return NULL;
}
```

**首次适配 vs 最佳适配**:

Knuth (1973a) 的经典研究表明，首次适配在实践中通常优于最佳适配 (best fit)。原因在于:
- 首次适配的搜索时间通常更短 (找到第一个足够大的就停止)
- 最佳适配会减少大块被切分的概率，但搜索开销更大
- 在随机分配/释放模式下，首次适配的碎片化程度与最佳适配相当

### 4.5 描述符分配器 (dalloc)

```c
static struct descriptor *dalloc(void *ptr, long size,
    const char *file, int line) {
    static struct descriptor *avail;  /* 当前可用的描述符批次 */
    static int nleft;                  /* 当前批次剩余描述符数 */

    if (nleft <= 0) {
        /* 当前批次用完了，向系统申请新的512个描述符 */
        avail = malloc(NDESCRIPTORS * sizeof(*avail));
        if (avail == NULL)
            return NULL;    /* 返回NULL给调用者 → 触发 Mem_Failed */
        nleft = NDESCRIPTORS;
    }

    avail->ptr  = ptr;       /* 记录数据块地址 */
    avail->size = size;      /* 记录数据块大小 */
    avail->file = file;      /* 记录分配位置 */
    avail->line = line;      /* 记录分配行号 */
    avail->free = avail->link = NULL;  /* 初始化为已分配 + 无链表后继 */
    nleft--;
    return avail++;          /* 返回当前描述符，然后指针后移 */
    /*
     * avail++ 是后缀递增:
     *   1. 返回当前avail值 (指向刚初始化的描述符)
     *   2. avail指针前进到下一个未使用的描述符位置
     *
     * 这种模式在512个描述符用完之前不需要再次调用malloc。
     */
}
```

### 4.6 检查型 Mem_free 和 Mem_resize

```c
void Mem_free(void *ptr, const char *file, int line) {
    if (ptr) {
        struct descriptor *bp;

        /* 三层验证 (见 §4.3) */
        if (((unsigned long)ptr) % (sizeof (union align)) != 0
            || (bp = find(ptr)) == NULL || bp->free)
            Except_raise(&Assert_Failed, file, line);

        /* 将描述符加入空闲链表头部 */
        bp->free = freelist.free;
        freelist.free = bp;
        /*
         * 注意: 这里不调用标准库的 free()!
         * 内存实际上没有被归还给操作系统。
         * 而是通过描述符的free字段被标记为"可重用"。
         * 这意味着:
         *   (1) 数据仍然在内存中 (安全性考虑: 未清零)
         *   (2) 该地址不会被 Mem_alloc 再次返回 (因为只从块尾部切分)
         *   (3) 对该地址的再次 Mem_free 会被第3层检查捕获
         */
    }
}

void *Mem_resize(void *ptr, long nbytes,
    const char *file, int line) {
    struct descriptor *bp;
    void *newptr;

    assert(ptr);
    assert(nbytes > 0);

    /* 与 Mem_free 相同的三层验证 */
    if (((unsigned long)ptr) % (sizeof (union align)) != 0
        || (bp = find(ptr)) == NULL || bp->free)
        Except_raise(&Assert_Failed, file, line);

    /* 总是 分配-复制-释放 (即使是缩小操作) */
    newptr = Mem_alloc(nbytes, file, line);   /* 分配新块 */
    memcpy(newptr, ptr,
        nbytes < bp->size ? nbytes : bp->size); /* 复制数据 */
    Mem_free(ptr, file, line);                 /* 释放旧块 */
    return newptr;
    /*
     * 为什么总是分配新块? 因为检查型实现中:
     *   1. 块不能被就地扩展 (块后紧跟已分配数据)
     *   2. 块不能被就地收缩 (块首部保留区域不可变)
     *   3. "分配-复制-释放"保证了地址的唯一性不变式
     */
}
```

---

## 五、内存泄漏检测: 分配与释放的镜像记录

### 5.1 检测原理

检查型实现通过描述符的状态字段实现了分配/释放的完整镜像记录:

```
分配时 (Mem_alloc/Mem_calloc):
  → dalloc() 创建描述符
  → descriptor.free = NULL         (标记为"已分配")
  → descriptor 加入 htab 哈希表
  → descriptor 不加入 freelist     (不在空闲链表中)

释放时 (Mem_free):
  → descriptor.free = freelist.free (标记为"已释放")
  → descriptor 加入 freelist        (参与空闲链表)
  → descriptor 保留在 htab 中       (永不删除!)
  → 数据块被标记为可重用

程序结束时:
  → 遍历 htab 中的所有描述符
  → free == NULL → 已分配但未释放 → 潜在内存泄漏
  → free != NULL → 已正确释放
```

### 5.2 Mem_leak 接口 (练习5.5)

CII的练习5.5提出了一个内存泄漏报告接口:

```c
extern void Mem_leak(
    apply(void *ptr, long size, const char *file, int line, void *cl),
    void *cl
);
/*
 * 对每个已分配(未释放)的块调用 apply 回调函数。
 *
 * 参数:
 *   apply  - 回调函数，参数为:
 *     ptr   : 泄漏块的地址
 *     size  : 泄漏块的大小
 *     file  : 分配时的源文件名
 *     line  : 分配时的源行号
 *     cl    : 用户提供的上下文指针 (闭包模式)
 *   cl     - 传递给apply的上下文数据
 *
 * 使用示例:
 *   void inuse(void *ptr, long size,
 *              const char *file, int line, void *cl) {
 *       FILE *log = cl;
 *       fprintf(log, "** memory in use at %p\n", ptr);
 *       fprintf(log, "This block is %ld bytes long "
 *               "and was allocated from %s:%d\n",
 *               size, file, line);
 *   }
 *
 *   FILE *log = fopen("leak.log", "w");
 *   Mem_leak(inuse, log);  // 报告所有未释放的块
 *   fclose(log);
 */
```

这种方法比简单的"分配计数 vs 释放计数"更强大:

1. **精确定位**: 不仅知道泄漏了多少字节，还知道每个泄漏块是在哪个文件的哪一行分配的
2. **闭包模式**: 通过 `cl` 指针传递上下文 (如日志文件句柄)，回调函数可以灵活处理
3. **非侵入式**: 不需要修改程序中的分配代码

---

## 六、越界写入检测: Sentinel / Guard Bytes 模式

### 6.1 CII的多层护栏机制

检查型实现通过三种机制提供越界写入检测:

**护栏1: 块前保留区域 (Pre-block Guard)**

```c
/* 每个从系统获取的大块的前 sizeof(union align) 字节永不分配 */
if (bp->size > nbytes)    /* 严格大于: 当剩余 == sizeof(union align) 时 */
    ...                   /* 不会再满足条件，此块被永久保留 */
```

这个保留区域充当了块前护栏。任何向前越界的写入都会落入此区域。然而，CII的实现不会主动检查此区域是否被修改 -- 这只是一个被动的防护措施。

**护栏2: 描述符物理隔离**

描述符通过 `dalloc` 从独立的内存池分配，不与数据块存储在一起。这意味着:
- 数据块的缓冲区溢出不会破坏描述符的元数据 (指针、大小、文件/行号)
- 即使发生了越界写入，错误检测机制本身仍然完好

**护栏3: 块级别隔离**

```
┌──────────┬──────────────────┬──────────┬──────────────────┐
│ 保留区域  │   已分配块 #1    │ 保留区域  │   已分配块 #2    │
│ (16B)    │   (用户数据)     │ (16B)    │   (用户数据)     │
└──────────┴──────────────────┴──────────┴──────────────────┘
     ▲                             ▲
     │ 向前越界写入落入此处         │ 向后越界写入落入此处
```

由于每个块的前面都有一个保留区域，相邻块之间始终有至少 `sizeof(union align)` 字节的间隙。

### 6.2 与Purify/Valgrind的对比

CII的guard bytes方法是一种**软件层面的轻量级检测**。其局限性:

| 特性 | CII Guard Bytes | Purify | Valgrind/Memcheck |
|------|----------------|--------|-------------------|
| 检测方式 | 分配器层面 | 目标码插桩 | 二进制翻译 (JIT) |
| 块内越界 | 无法检测 | 逐指令检测 | 逐指令检测 |
| 块尾部越界 | 间接保护 (下一块的保留区域) | 直接检测 (red zone) | 直接检测 (red zone) |
| Use-after-free | 检测 (descriptor查找) | 检测 (内存染色) | 检测 (内存染色) |
| 外部依赖 | 零 | 商业软件 | 开源 (GPL) |
| 性能开销 | ~10-30% | ~2-10x | ~10-30x |
| 所需修改 | 使用ALLOC宏 | 无需 (目标码级别) | 无需 (二进制级别) |

CII的设计目标不是替代专业工具，而是提供一个**内置的、零外部依赖的调试辅助系统**。

---

## 七、包装模式: 如何实现透明跟踪

### 7.1 包装层次图

```
                    用户层  API
                 ┌─────────────────────────────────────┐
                 │  ALLOC / NEW / CALLOC / FREE / RESIZE │
                 │         (宏: 自动注入位置)            │
                 └──────────────┬──────────────────────┘
                                │
                    接口层  Mem API
                 ┌──────────────┴──────────────────────┐
                 │  Mem_alloc / Mem_calloc /            │
                 │  Mem_free  / Mem_resize              │
                 │         (函数: 参数验证)              │
                 └──────────────┬──────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                                     │
    生产级 实现层                          检查型 实现层
    ┌─────────┴──────────┐          ┌──────────┴───────────┐
    │ malloc / free /    │          │ 首次适配分配器        │
    │ calloc / realloc   │          │ (记录所有元数据)      │
    └─────────┬──────────┘          │  哈希表跟踪           │
              │                     │  空闲链表管理         │
              │                     └──────────┬───────────┘
              │                                │
    系统层  Standard C Library                 │
    ┌─────────┴────────────────────────────────┴───────┐
    │              malloc / free (系统实现)              │
    │          (ptmalloc / jemalloc / tcmalloc)         │
    └──────────────────────┬───────────────────────────┘
                           │
    操作系统层              ▼
                  sbrk / mmap 系统调用
```

### 7.2 包装的本质

CII的包装不是**替换**而是**增强**:

1. **参数空间收缩**: 将未定义/实现定义的行为转化为受检查的运行时错误
   - `malloc(0)` → 断言失败
   - `free(NULL)` → 静默无操作
   - `realloc(NULL, n)` → 断言失败 (CII)

2. **错误传播升级**: 将返回值错误码 (NULL) 升级为异常 (Mem_Failed)
   - 调用者不再需要逐层检查返回值
   - 异常沿调用栈向上传播，直到被 TRY/EXCEPT 捕获

3. **元数据注入**: 在检查型实现中，拦截所有分配/释放操作，记录完整的生命周期数据

### 7.3 与标准malloc/free共存

Mem接口的设计允许与原生 `malloc/free` 在同一程序中共存:

```c
void *p1 = malloc(100);       /* 由标准库直接管理 */
void *p2 = ALLOC(100);        /* 由Mem管理 (受检查型实现跟踪) */

free(p1);                     /* OK: 直接释放 */
FREE(p2);                     /* OK: 通过Mem释放, 检查型实现验证合法性 */

/* ⚠ 以下将触发检查型实现的Assert_Failed: */
/* Mem_free(p1, ...) */       /* p1不在htab中! */
/* free(p2) */                /* bad: p2由Mem管理, 应使用FREE */
```

---

## 八、断言在内存管理中的关键作用

### 8.1 CII的自定义assert机制

```c
/* code/assert.h -- CII版本的断言 */
#undef assert                    /* 取消标准库的assert定义 */
#ifdef NDEBUG
#define assert(e) ((void)0)     /* 发布构建: 零开销 */
#else
#include "except.h"
extern void assert(int e);      /* 声明函数形式 (见assert.c) */
#define assert(e) ((void)((e)||(RAISE(Assert_Failed),0)))
/*
 * 逻辑短路:
 *   e 为 true  → || 左侧为1, 不评估右侧 → 无操作
 *   e 为 false → || 左侧为0, 评估右侧:
 *                 1. RAISE(Assert_Failed)  → 抛出异常 (展开为
 *                    Except_raise(&Assert_Failed, __FILE__, __LINE__))
 *                 2. , 0                     → 逗号表达式返回0
 *                 3. (void)(0)               → 丢弃返回值
 *
 * 与标准assert的关键区别:
 *   标准: assert(false) → 打印消息 + abort() → 程序终止
 *   CII:  assert(false) → 抛出 Assert_Failed 异常
 *         → 可以被 TRY/EXCEPT 捕获
 *         → 服务器程序可以记录日志后优雅恢复
 */
#endif
```

`assert.c` 提供了一个函数形式以兼容函数指针:

```c
const Except_T Assert_Failed = { "Assertion failed" };
void (assert)(int e) {          /* 函数名周围的括号防止宏展开 */
    assert(e);                   /* 此处调用宏版本 */
}
```

### 8.2 assert在Mem中的部署

| 位置 | 断言条件 | 防御的错误类型 |
|------|---------|---------------|
| `Mem_alloc` | `nbytes > 0` | 零/负大小分配; 包装负数绕过size_t检查 |
| `Mem_calloc` | `count > 0` | 零计数分配; 防止calloc(0, n)的实现定义行为 |
| `Mem_calloc` | `nbytes > 0` | 零元素大小; 防止calloc(n, 0)的实现定义行为 |
| `Mem_resize` | `ptr != NULL` | NULL指针传递; 消除realloc(NULL,n)的歧义 |
| `Mem_resize` | `nbytes > 0` | 零/负新大小; 消除realloc(p,0)的歧义 |
| `Mem_alloc` (检查型) | `0` (= 不可达) | 空闲链表结构损坏检测 |

### 8.3 assert(0) 的特殊用法

在检查型 `Mem_alloc` 的 `for` 循环之后:

```c
for (bp = freelist.free; bp; bp = bp->free) {
    if (bp->size > nbytes) {
        /* ... 分配成功，return ptr */
    }
    if (bp == &freelist) {
        /* ... 申请新内存，continue (遍历新大块) */
    }
}
assert(0);    /* ← 如果程序到达此处，空闲链表结构已损坏 */
return NULL;  /* 仅用于抑制 "control reaches end of non-void function" 警告 */
```

`assert(0)` 是"活文档" -- 它声明了该代码路径在正确的程序逻辑中不可达。如果由于数据结构损坏导致程序到达此处，断言立即触发异常，防止使用损坏状态继续执行。

---

## 九、Modern-C 改进

### 9.1 AddressSanitizer (ASan): CII思想的现代化体现

AddressSanitizer (由Google开发，已集成到GCC 4.8+和Clang 3.1+) 是CII内存检测思想的系统化、自动化的现代化体现。

**ASan核心技术: 影子内存 (Shadow Memory)**

```
应用内存 (每8字节)          影子内存 (每1字节)
┌───┬───┬───┬───┐           ┌───┬───┬───┬───┐
│ A │ A │ A │ A │   ←──→    │ 0 │ 0 │ 0 │ 0 │  0 = 全部可寻址
├───┼───┼───┼───┤           ├───┼───┼───┼───┤
│ A │ A │ P │ P │   ←──→    │ 0 │ 0 │ 2 │ 2 │  k = 前k字节可寻址
├───┼───┼───┼───┤           ├───┼───┼───┼───┤
│ R │ R │ R │ R │   ←──→    │FF │FF │FF │FF │  FF = Red Zone (不可寻址)
└───┴───┴───┴───┘           └───┴───┴───┴───┘
  A = 可寻址字节              8字节→1字节的映射:
  P = 部分可寻址              影子地址 = (应用地址 >> 3) + 偏移量
  R = Red Zone (护栏)
```

**CII Guard Bytes vs ASan Red Zone**:

| 维度 | CII Guard Bytes | ASan Red Zone |
|------|----------------|--------------|
| 粒度 | 块级别 (sizeof(union align) ≈ 16B) | 字节级别 |
| 检测时机 | 分配/释放时 (通过描述符) | 每次内存访问时 (通过编译器插桩) |
| 堆缓冲区溢出 | 间接检测 (块间护栏) | 直接检测 (red zone有毒) |
| 栈缓冲区溢出 | 不检测 | 检测 (栈帧间red zone) |
| 全局缓冲区溢出 | 不检测 | 检测 (全局变量间red zone) |
| Use-after-free | 检测 (htab查找) | 检测 (隔离释放的内存到quarantine) |
| Use-after-return | 不检测 | 检测 (可选，额外开销) |
| 性能开销 | ~10-30% | ~2x 减速, ~2-3x 额外内存 |
| 代码修改 | 需要用ALLOC宏 | 不需要 (编译器自动插桩) |
| 调试信息 | 分配点文件/行号 (内置) | 分配点堆栈追踪 (需要符号表) |

**使用方式**:

```bash
# 使用ASan编译和运行 (无需修改任何代码!)
gcc -fsanitize=address -g -O1 myprogram.c -o myprogram
./myprogram

# 输出示例:
# ==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60400000
# READ of size 4 at 0x60400000 thread T0
#     #0 0x400a2b in write_out_of_bounds test.c:15
#     #1 0x400b12 in main test.c:25
# 0x60400000 is located 0 bytes to the right of 100-byte region
# [0x603fff90,0x603ffff4) allocated by thread T0 here:
#     #0 0x4c2d12 in malloc
#     #1 0x400a01 in main test.c:23
```

ASan和CII Mem的关系不是竞争关系，而是**演进关系**: CII在库层面探索了内存检测的可行性，ASan则在编译器/运行时层面将其系统化和自动化。理解CII的实现有助于深入理解ASan的工作原理。

### 9.2 C11 aligned_alloc

CII通过 `union align` + 上舍入公式手动实现对齐分配。C11引入了标准的 `aligned_alloc`:

```c
/* CII方式 (C89/C99, 手动计算) */
nbytes = ((nbytes + sizeof(union align) - 1) /
          (sizeof(union align))) * (sizeof(union align));
ptr = malloc(nbytes);

/* C11方式 (标准库支持) */
#include <stdlib.h>
#include <stdalign.h>   /* alignof, max_align_t */
ptr = aligned_alloc(alignof(max_align_t), nbytes);
```

`aligned_alloc` 的限制 (C11标准):
- `alignment` 必须是2的幂
- `size` 必须是 `alignment` 的倍数
- 返回的内存可以用 `free` 释放

在Modern-C中升级CII的 `Mem_alloc`:

```c
#include <stdlib.h>
#include <stdalign.h>

void *Mem_alloc_aligned(long nbytes, const char *file, int line) {
    void *ptr;
    size_t alignment = alignof(max_align_t);
    size_t size = (size_t)nbytes;

    assert(nbytes > 0);

    /* 确保 size 是 alignment 的倍数 (aligned_alloc要求) */
    size = ((size + alignment - 1) / alignment) * alignment;

    ptr = aligned_alloc(alignment, size);
    if (ptr == NULL) {
        if (file == NULL)
            RAISE(Mem_Failed);
        else
            Except_raise(&Mem_Failed, file, line);
    }
    return ptr;
}
```

### 9.3 智能指针模式 (Smart Pointer Pattern in C)

CII的 `FREE` 宏 -- 释放后将指针置NULL -- 是C语言中RAII思想的萌芽。Modern-C可以通过编译器扩展进一步增强:

**GCC/Clang `cleanup` 属性**:

```c
/* 清理函数: GCC/Clang在变量离开作用域时自动调用 */
void cleanup_Mem_free(void *ptr) {
    void *p = *(void **)ptr;
    if (p) Mem_free(p, __FILE__, __LINE__);
}

/* 作用域内自动释放的"智能指针"宏 */
#define scoped_ptr(T) T __attribute__((cleanup(cleanup_Mem_free)))

void process_data(void) {
    scoped_ptr(int *) arr = ALLOC(100 * sizeof(int));
    /* ... 使用 arr ... */
    /* 离开作用域时自动调用 cleanup_Mem_free(&arr) → FREE(arr) */
    /* 即使通过 return/break/goto/异常提前退出也能正确释放! */
}
```

**与C++智能指针的对应**:

| C语言模式 | C++对应 | 语义 |
|-----------|--------|------|
| `FREE(p)` 后将p置NULL | `delete p; p = nullptr;` | 手动管理 |
| `scoped_ptr(T)` + cleanup | `std::unique_ptr<T>` | 独占所有权, 自动释放 |
| 引用计数 + cleanup | `std::shared_ptr<T>` | 共享所有权 |
| CII `NEW` / `FREE` 配对 | `new` / `delete` | (不推荐在Modern C++中使用) |

### 9.4 线程安全考量

CII的Mem接口是**非线程安全**的:

- `htab[2048]` 是全局共享状态
- `freelist` 是全局共享状态
- `dalloc` 中的 `static avail` 和 `static nleft` 是全局共享状态
- 没有锁、原子操作或内存屏障

多线程程序使用CII Mem接口需要外部同步。Modern-C的改进方案:

**方案A: 粗粒度互斥锁**:

```c
#include <threads.h>   /* C11 threads */

static mtx_t mem_mutex;

/* 初始化 (在main中或通过 pthread_once / call_once) */
void mem_init(void) {
    mtx_init(&mem_mutex, mtx_plain);
}

void *Mem_alloc_ts(long nbytes, const char *file, int line) {
    mtx_lock(&mem_mutex);
    void *ptr = Mem_alloc(nbytes, file, line);
    mtx_unlock(&mem_mutex);
    return ptr;
}
```

**方案B: 线程局部存储 (TLS)**:

```c
/* 每个线程拥有独立的分配器实例 */
_Thread_local static struct descriptor *htab[2048];
_Thread_local static struct descriptor freelist = { &freelist };

/* 优点: 零锁开销, 无竞争 */
/* 缺点: 每个线程独立的内存池 (可能浪费); 不能跨线程释放 */
```

**方案C: jemalloc/tcmalloc**:

对于生产系统，使用现代内存分配器 (jemalloc, tcmalloc) 通常是最佳选择 -- 它们在内部处理了线程安全和性能优化，且与 `malloc/free` API兼容。

---

## 十、性能影响分析

### 10.1 生产级实现的量化分析

| 操作 | 额外指令数 (估算) | 额外延迟 (vs 原始 malloc/free) |
|------|-------------------|-------------------------------|
| `ALLOC(n)` | ~8 | < 5 ns (现代CPU) |
| `CALLOC(c, n)` | ~10 | < 5 ns |
| `FREE(p)` | ~5 | < 3 ns |
| `RESIZE(p, n)` | ~8 | < 5 ns |

**结论**: 生产级实现的开销在统计噪声范围内 (< 0.1%)。宏展开和额外的条件分支在分支预测器面前几乎零开销。

### 10.2 检查型实现的量化分析

| 操作 | 时间复杂度 | 瓶颈操作 |
|------|-----------|---------|
| `ALLOC(n)` (空闲链表命中) | O(搜索的块数) | 链表遍历 + `dalloc` (可能触发malloc) |
| `ALLOC(n)` (空闲链表未命中) | O(所有空闲块数 + malloc) | malloc + 描述符分配 |
| `FREE(p)` | O(1) | 哈希查找 + 链表插入 |
| `RESIZE(p, n)` | O(分配 + 释放) | ALLOC + memcpy + FREE |
| `CALLOC(c, n)` | O(ALLOC + memset) | ALLOC + 内存清零 |

**实测开销** (在典型C程序上):
- **速度降低**: 2-5x (主要来自首次适配的线性搜索和批量malloc策略)
- **内存开销**: 10-20% (描述符约32B/块 + 块首部保留16B + 碎片化)
- **系统调用频率**: 降低 (批量策略减少malloc调用次数)

### 10.3 优化策略

1. **条件编译切换**: 开发/测试时使用检查型实现，生产部署时切换到生产级实现

```makefile
# Makefile片段
ifeq ($(DEBUG),1)
    CFLAGS += -DMEM_CHECK
    SRC    += memchk.c
else
    SRC    += mem.c
endif
```

2. **哈希表大小调优**: `htab[2048]` 对大多数程序足够。分配密集型程序可考虑增大到8192或16384

3. **NDESCRIPTORS调优**: 512是经验折中。如果 `dalloc` 是热点（意味着频繁分配大量小块），增大此值可减少malloc调用

4. **空闲链表维护**: 练习5.2建议移除永远无法满足请求的最小块（`size == sizeof(union align)`），可略微减少链表遍历开销

---

## 十一、小结

David R. Hanson在CII第5章中展示了库设计的核心原则，这些原则在30年后仍然有效:

1. **接口即契约 (Interface as Contract)**: 通过断言将未定义行为转化为受检查的运行时错误。`nbytes > 0`、`ptr != NULL` 等前置条件不是注释，而是可执行的规范。

2. **渐进增强 (Progressive Enhancement)**: 提供生产级和检查型两种实现，开发者可以按需选择安全级别，而非在"快但脆弱"和"安全但慢"之间做二选一的零和游戏。

3. **防御性设计 (Defensive Design)**: 永不返回NULL (异常代替)、释放后置零 (防悬空指针)、参数范围检查 (防溢出)、描述符隔离 (防元数据损坏)。

4. **可观测性 (Observability)**: 通过 `__FILE__`/`__LINE__` 自动注入调用点信息，使得错误可追溯到源头。

5. **最小惊讶原则 (Principle of Least Astonishment)**: `Mem_resize` 的简化语义相比 `realloc` 更符合直觉 -- 一个函数只做一件事。

这些原则在现代工具中得到了延续和增强 -- ASan的red zone是CII guard bytes的系统化推广; `std::unique_ptr` 的自动释放是 `FREE` 宏零开销抽象的继承; C11的 `aligned_alloc` 和 `max_align_t` 是将 `union align` 的技巧标准化。理解CII，就是理解安全C编程的基因图谱。

---

## 参考文献

1. Hanson, D. R. (1997). *C Interfaces and Implementations: Techniques for Creating Reusable Software*. Addison-Wesley. Chapter 5, pp. 67-88.
2. Kernighan, B. W. and Ritchie, D. M. (1988). *The C Programming Language*, 2nd ed. Prentice Hall. Section 8.7.
3. Knuth, D. E. (1973). *The Art of Computer Programming*, Vol. 1: Fundamental Algorithms. Addison-Wesley.
4. Maguire, S. (1993). *Writing Solid Code*. Microsoft Press.
5. Hastings, R. and Joyce, B. (1992). Purify: Fast detection of memory leaks and access errors. *Proceedings of the Winter USENIX Conference*.
6. Serebryany, K. et al. (2012). AddressSanitizer: A fast address sanity checker. *Proceedings of USENIX ATC*.
7. Austin, T. M., Breach, S. E., and Sohi, G. S. (1994). Efficient detection of all pointer and array access errors. *Proceedings of PLDI '94*.
8. Evans, D. (1996). LCLint: A tool for statically checking C programs. *MIT Laboratory for Computer Science*.
9. Grunwald, D. and Zorn, B. (1993). CustoMalloc: Efficient synthesized memory allocators. *Software -- Practice and Experience*, 23(8).
10. Weinstock, C. B. and Wulf, W. A. (1988). Quick fit: An efficient algorithm for heap storage allocation. *ACM SIGPLAN Notices* 23(10).
