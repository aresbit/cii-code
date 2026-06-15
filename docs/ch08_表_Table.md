# 第8章: 表 (Table) — 泛型哈希表

> 原文: C Interfaces and Implementations, David R. Hanson, Chapter 8
> 源文件: `code/table.h`, `code/table.c`

---

## 8.0 引言: 关联表是什么

**关联表 (associative table)** 是一组键值对 (key-value pair) 的集合。它类似于数组，但其索引可以是任意类型的值，而不仅仅是整数。

关联表的应用极其广泛:

- **编译器** 维护**符号表**，将标识符映射到其类型、作用域等属性。
- **窗口系统** 维护表，将窗口标题映射到窗口相关的数据结构。
- **文档排版系统** 使用表来构建索引: 顶层的键是二十六字母，值是子表；子表的键是索引项，值是页码列表。

> 原文注: Tables are so useful that many programming languages use them as built-in data types. AWK, SNOBOL4, Icon, PostScript (as "dictionaries"), Smalltalk, and Objective-C all have table-like built-in or library types.

本章的核心问题是: **如何用 C 语言设计一个类型安全的、实现无关的泛型关联表接口?**

---

## 8.1 接口概览 (`table.h`)

Table 用一个**不透明指针类型** (opaque pointer type) 表示关联表，客户端永远看不到 `struct T` 的内部字段。

```c
/* table.h -- S8.1 Interface, line 1-20 */
#ifndef TABLE_INCLUDED
#define TABLE_INCLUDED
#define T Table_T
typedef struct T *T;          /* 不透明指针: 客户端只能持有 Table_T，看不到内部实现 */
```

这种设计带来了两个关键好处:
1. **实现无关**: 客户端代码不依赖 hash 表的内部结构，可以替换为树或其他实现而无需修改客户端。
2. **封装**: 客户端无法直接访问 `size`、`buckets`、`cmp`、`hash` 等内部字段。

### 8.1.1 创建与销毁

```c
extern T    Table_new (int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *key));
extern void Table_free(T *table);
```

**`Table_new`** 的三个参数提供了比大多数实现需要更多的信息，这是为了**支持多种底层实现**（hash 表、二叉树等）而付出的设计代价:

| 参数 | 含义 | 特殊值 |
|------|------|--------|
| `hint` | 预期条目数的估计值 | 必须 `>= 0`; 准确值可能提升性能但不影响正确性 |
| `cmp(x,y)` | 键比较函数，返回 `<0` / `==0` / `>0` | 传 `NULL` 表示键是 atom，内部用 `cmpatom` |
| `hash(key)` | 键的散列函数 | 传 `NULL` 表示键是 atom，内部用 `hashatom` |

**设计约束**: 如果 `cmp(x, y) == 0`，则必须有 `hash(x) == hash(y)`。这是 hash 表正确性的必要条件。

**`Table_free`** 释放 `*table` 并将其置为 `NULL`。**它不释放键或值** -- 如果需要释放键或值，必须先调用 `Table_map` 遍历释放。

#### 调用示例: Table_new 与 Table_free

```c
#include "table.h"
#include "atom.h"

/* cmp 和 hash 都传 NULL，使用默认 atom 比较/散列 */
Table_T atom_table = Table_new(100, NULL, NULL);

/* 自定义 strcmp 作为比较函数，hash 传 NULL (将使用默认 hashatom) */
/* 注意: 这在实践中不推荐 -- 应该为自己的 key 类型提供配套的 cmp + hash */
Table_T str_table = Table_new(200,
    (int (*)(const void *, const void *))strcmp,
    NULL);

/* 释放后指针置 NULL */
Table_free(&atom_table);
Table_free(&str_table);
/* 此刻 atom_table == NULL, str_table == NULL */
```

### 8.1.2 键值操作

```c
extern int   Table_length(T table);  /* 返回键值对数量 */
extern void *Table_put   (T table, const void *key, void *value);  /* 插入或更新 */
extern void *Table_get   (T table, const void *key);              /* 查找 */
extern void *Table_remove(T table, const void *key);              /* 删除 */
```

**`Table_put`** — 核心语义:
- 如果 `key` 不存在于表中: 添加新键值对，表增长一个条目，返回 `NULL`。
- 如果 `key` 已经存在: 用 `value` 覆盖旧值，**返回旧值**。
- 注意: 返回 `NULL` 存在语义歧义 -- 既可能表示"新插入"，也可能表示"之前的值就是 `NULL`"。

**`Table_get`** — 查找语义:
- 找到 `key`: 返回关联的 `value`。
- 未找到: 返回 `NULL`。同样存在与 `NULL` 值的歧义。

**`Table_remove`** — 删除语义:
- 找到 `key`: 移除键值对，表缩减一个条目，**返回被移除的值**。
- 未找到: 无影响，返回 `NULL`。

#### 调用示例: put / get / remove / length

```c
#include <stdio.h>
#include "table.h"
#include "atom.h"
#include "mem.h"

void basic_ops_demo(void) {
    Table_T tab = Table_new(0, NULL, NULL);  /* atom key 表 */

    /* --- Table_put: 插入 --- */
    const char *key_a = Atom_string("alpha");
    const char *key_b = Atom_string("beta");

    int *val_a; NEW(val_a); *val_a = 10;
    int *val_b; NEW(val_b); *val_b = 20;

    void *prev = Table_put(tab, key_a, val_a);
    printf("put alpha: prev=%p (expected NULL)\n", prev);  /* → NULL (新插入) */

    prev = Table_put(tab, key_b, val_b);
    printf("put beta:  prev=%p (expected NULL)\n", prev);  /* → NULL (新插入) */

    /* 覆盖已存在的 key */
    int *val_a2; NEW(val_a2); *val_a2 = 99;
    prev = Table_put(tab, key_a, val_a2);
    printf("put alpha again: prev=%p, *prev=%d (expected 10)\n",
           (void*)prev, prev ? *(int*)prev : -1);
    /* → 返回旧值 10; val_a2=99 成为新值; val_a 的旧内存泄漏了 */

    /* --- Table_get: 查找 --- */
    int *got = (int *)Table_get(tab, key_a);
    printf("get alpha: *got=%d (expected 99)\n", got ? *got : -1);

    got = (int *)Table_get(tab, key_b);
    printf("get beta:  *got=%d (expected 20)\n", got ? *got : -1);

    /* 查找不存在的 key */
    const char *key_c = Atom_string("gamma");
    got = (int *)Table_get(tab, key_c);
    printf("get gamma: %p (expected NULL)\n", (void*)got);

    /* --- Table_length: 条目数 --- */
    printf("length=%d (expected 2)\n", Table_length(tab));

    /* --- Table_remove: 删除 --- */
    int *removed = (int *)Table_remove(tab, key_a);
    printf("remove alpha: *removed=%d (expected 99)\n", removed ? *removed : -1);
    printf("length after remove=%d (expected 1)\n", Table_length(tab));

    /* 删除不存在的 key */
    removed = (int *)Table_remove(tab, key_c);
    printf("remove gamma: %p (expected NULL)\n", (void*)removed);

    /* 清理 */
    Table_free(&tab);
    FREE(val_a);  /* 被覆盖的旧值仍需手动释放 */
    FREE(val_a2); /* 因为 remove 后需要手动释放 */
    FREE(val_b);
}
```

### 8.1.3 遍历与数组导出

```c
extern void   Table_map    (T table,
    void apply(const void *key, void **value, void *cl),
    void *cl);
extern void **Table_toArray(T table, void *end);
```

**`Table_map`** — 对表中每个键值对调用 `apply`，顺序未定义。这是 **访问者模式 (Visitor Pattern)** 的 C 实现:

- `apply(key, &value, cl)` 接收 key、**指向 value 的指针**（因此 `apply` 可以修改 value）、以及客户端数据 `cl`。
- `cl` 实现了**闭包 (closure)**: 允许向 `apply` 传递上下文数据，而无需使用全局变量。
- **受检运行时错误**: `apply` 内部不得调用 `Table_put` 或 `Table_remove`（通过 `timestamp` 机制检测，见 8.3 节）。

**`Table_toArray`** — 将表中所有键值对收集到数组中:

- 如果表有 N 个键值对，返回 2N+1 元素的数组。
- 偶数索引放 key，紧随的奇数索引放 value。
- 第 2N 个元素赋值为 `end`（通常是 `NULL`），作为终止哨兵。
- 键值对的顺序是任意的 (unspecified)。

#### 调用示例: Table_map 遍历

```c
#include <stdio.h>
#include "table.h"
#include "atom.h"
#include "mem.h"

/* apply 回调: 打印每个键值对，并可选地释放 value */
static void print_kv(const void *key, void **value, void *cl) {
    const char *prefix = (const char *)cl ? (const char *)cl : "";
    printf("%skey=%s, value=%d\n", prefix,
           (const char *)key, **(int **)value);
}

/* apply 回调: 释放 value */
static void free_value(const void *key, void **value, void *cl) {
    (void)key; (void)cl;  /* 未使用 */
    FREE(*value);          /* *value 是 int* */
}

void table_map_demo(void) {
    Table_T tab = Table_new(0, NULL, NULL);

    /* 插入测试数据 */
    for (int i = 0; i < 5; i++) {
        char buf[16];
        snprintf(buf, sizeof buf, "key_%d", i);
        int *v; NEW(v); *v = i * 100;
        Table_put(tab, Atom_string(buf), v);
    }

    /* 方式1: 用 Table_map 打印所有键值对 */
    printf("=== Table_map 打印 ===\n");
    Table_map(tab, print_kv, (void *)"  ");

    /* 方式2: 用 Table_map 释放所有 value */
    Table_map(tab, free_value, NULL);

    /* 最后释放表本身 */
    Table_free(&tab);
}
```

#### 调用示例: Table_toArray 排序输出

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "table.h"
#include "atom.h"
#include "mem.h"

/* qsort 比较函数: 按 key (字符串) 排序键值对 */
static int cmp_kv(const void *x, const void *y) {
    /* x 指向键值对起始 (即 key 指针), y 同理 */
    return strcmp(*(const char **)x, *(const char **)y);
}

void table_to_array_demo(void) {
    Table_T tab = Table_new(0, NULL, NULL);

    /* 插入测试数据 */
    const char *words[] = {"zebra", "apple", "mango", "banana", NULL};
    for (int i = 0; words[i]; i++) {
        int *v; NEW(v); *v = i + 1;
        Table_put(tab, Atom_string(words[i]), v);
    }

    /* 导出为数组 + 排序 */
    void **array = Table_toArray(tab, NULL);
    int n = Table_length(tab);
    qsort(array, n, 2 * sizeof(*array), cmp_kv);

    printf("=== 按 key 字母序输出 ===\n");
    for (int i = 0; array[i]; i += 2)
        printf("  %s -> %d\n",
               (const char *)array[i],
               *(int *)array[i+1]);

    /* 清理: 先释放 value, 再释放 array 和 table */
    for (int i = 0; array[i]; i += 2)
        FREE(array[i+1]);  /* 释放 value */
    FREE(array);
    Table_free(&tab);
}
```

**预期输出** (key 按字母序排列):

```
=== 按 key 字母序输出 ===
  apple -> 2
  banana -> 4
  mango -> 3
  zebra -> 1
```

---

## 8.2 实例: 词频统计 (wf)

`wf` 程序统计每个文件（或标准输入）中各单词的出现次数，并按字母顺序输出。它是展示 `Table` 接口使用的典型案例。

### 8.2.1 完整客户端代码

```c
/* wf.c -- S8.2 Example: Word Frequencies */
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <ctype.h>
#include "atom.h"
#include "table.h"
#include "mem.h"
#include "getword.h"

/* ---------- 函数声明 ---------- */
void wf(char *, FILE *);
int first(int c);
int rest(int c);
int compare(const void *x, const void *y);
void vfree(const void *, void **, void *);

/* ---------- main ---------- */
int main(int argc, char *argv[]) {
    int i;
    for (i = 1; i < argc; i++) {
        FILE *fp = fopen(argv[i], "r");
        if (fp == NULL) {
            fprintf(stderr, "%s: can't open '%s' (%s)\n",
                    argv[0], argv[i], strerror(errno));
            return EXIT_FAILURE;
        } else {
            wf(argv[i], fp);
            fclose(fp);
        }
    }
    if (argc == 1) wf(NULL, stdin);
    return EXIT_SUCCESS;
}

/* ---------- 核心: 读词 -> atom -> 计数 ---------- */
void wf(char *name, FILE *fp) {
    Table_T table = Table_new(0, NULL, NULL);   /* atom 作 key，使用默认 cmp/hash */
    char buf[128];

    while (getword(fp, buf, sizeof buf, first, rest)) {
        const char *word;
        int i, *count;

        for (i = 0; buf[i] != '\0'; i++)    /* 统一转小写 */
            buf[i] = tolower(buf[i]);
        word = Atom_string(buf);             /* 字符串 -> atom */

        count = Table_get(table, word);      /* 查找是否已存在 */
        if (count)
            (*count)++;                       /* 已存在: 计数器+1 */
        else {
            NEW(count);                       /* 不存在: 分配计数器空间 */
            *count = 1;
            Table_put(table, word, count);   /* 插入新键值对 */
        }
    }

    if (name)
        printf("%s:\n", name);

    /* --- 排序输出 --- */
    {
        int i;
        void **array = Table_toArray(table, NULL);
        qsort(array, Table_length(table), 2*sizeof(*array), compare);
        for (i = 0; array[i]; i += 2)
            printf("%d\t%s\n", *(int *)array[i+1], (char *)array[i]);
        FREE(array);
    }

    /* --- 释放计数器和表 --- */
    Table_map(table, vfree, NULL);
    Table_free(&table);
}

/* ---------- 词识别 ---------- */
int first(int c) { return isalpha(c); }
int rest(int c)  { return isalpha(c) || c == '_'; }

/* ---------- 排序比较 (单词字母序) ---------- */
int compare(const void *x, const void *y) {
    return strcmp(*(char **)x, *(char **)y);
}

/* ---------- 释放值的回调 ---------- */
void vfree(const void *key, void **count, void *cl) {
    FREE(*count);
}
```

### 8.2.2 设计要点分析

| 设计决策 | 理由 |
|----------|------|
| 使用 atom 作 key | atom 是唯一的、不可变的字符串指针。可用默认 `cmp` (`x != y`) 和默认 `hash` (地址右移 2 位) |
| value 是指向堆上 `int` 的指针 | `Table` 不管理 value 的生命周期，客户端自行 `NEW`/`FREE` |
| 利用 `Table_map` 释放 value | `vfree` 回调释放每个计数器，不释放 key（key 是 atom，由 Atom 管理） |
| `qsort` 的 `sizeof` 参数为 `2*sizeof(*array)` | 因为每个"元素"是一个键值对（两个指针），这样 qsort 才能正确处理 |

**关键细节: `qsort` 的参数设计**

`Table_toArray` 返回一个 `void **array`，其中偶数索引 `array[0]`, `array[2]`, ... 是 key，奇数索引 `array[1]`, `array[3]`, ... 是 value。将每个键值对视为 qsort 的一个"元素"，需要:
- 元素数量 = `Table_length(table)` （即 N，而非 2N）
- 元素大小 = `2 * sizeof(*array)` （即两个指针的大小）
- 比较函数 `compare` 收到的参数是 `(void *x, void *y)`，其中 `x` 实际指向一个键值对的起始位置（即 key 指针），因此 `*(char **)x` 解引用得到 key 字符串。

### 8.2.3 wf 运行示例

```bash
$ wf table.c mem.c
table.c:
  3  apply
  7  array
 13  assert
  9  binding
 18  book
  2  break
 10  buckets
...
mem.c:
  1  allocation
  7  assert
 12  book
  1  stdlib
  9  void
...
```

---

## 8.3 实现详解 (`table.c`)

### 8.3.1 数据结构

```c
/* S8.3 Implementation, types 125 */
#define T Table_T
struct T {
    int size;                              /* buckets 数组大小 (素数)       */
    int (*cmp)(const void *x, const void *y);  /* 键比较函数指针              */
    unsigned (*hash)(const void *key);         /* 键散列函数指针              */
    int length;                            /* 当前键值对数量                 */
    unsigned timestamp;                    /* 修改计数器 (用于 Table_map 检查) */
    struct binding {
        struct binding *link;              /* 链表 next 指针                */
        const void *key;                   /* 键 (const void *)              */
        void *value;                       /* 值                            */
    } **buckets;                           /* bucket 数组指针                */
};
```

**内存布局特点**: `struct T` 和 `buckets` 数组在一次 `ALLOC` 中连续分配（见下文 `Table_new`），减少内存碎片和分配开销。

**结构可视化 (图 8.1 用文字描述)**:

```
Table_T (struct T *)
  |
  v
  struct T { size=509, cmp, hash, length, timestamp, buckets }
                                                       |
                                                       v
  buckets[0] --> [binding A] --> [binding B] --> NULL
                    | key="foo"    | key="bar"
                    | value=42     | value=99
  buckets[1] --> NULL
  buckets[2] --> [binding C] --> NULL
                    | key="baz"
                    | value=7
  ...
  buckets[508] --> NULL
```

每个 bucket 是一个链表的头指针，所有 hash 到同一 bucket 的键共享一条链表。

### 8.3.2 默认 atom 的 cmp 和 hash

```c
/* S8.3, static functions 127 */
static int cmpatom(const void *x, const void *y) {
    return x != y;                    /* atom 相等当且仅当指针相等 */
}
static unsigned hashatom(const void *key) {
    return (unsigned long)key >> 2;  /* 地址右移 2 位: 字对齐的地址低 2 位为 0 */
}
```

**`hashatom` 的分析**: atom 是 `Atom_new` 返回的堆上字符串指针，通常满足 4 字节对齐（在 32 位系统上），所以低 2 位恒为 0。右移 2 位使 hash 值分布更均匀。这是一个高度特化的优化。

**`cmpatom` 的分析**: CII 的 Table 实现只测试键的相等性，不关心相对顺序。因此 `cmpatom` 只需要返回"相等"（0）或"不等"（1），而不是 strcmp 风格的 `<0` / `==0` / `>0` 三分法。但接口仍然要求 cmp 返回三分值，因为其他可能的实现（如二叉树）需要全序比较。

### 8.3.3 `Table_new` — 创建与初始化

```c
/* S8.3, functions 126 */
T Table_new(int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *key)) {
    T table;
    int i;
    /* 素数表: 509, 509, 1021, 2053, 4093, 8191, 16381, 32771, 65521, INT_MAX */
    static int primes[] = { 509, 509, 1021, 2053, 4093,
        8191, 16381, 32771, 65521, INT_MAX };

    assert(hint >= 0);                  /* hint 不得为负 */

    /* 选择 >= hint 的最小素数对应的索引-1 */
    for (i = 1; primes[i] < hint; i++)
        ;
    /* 一次性分配 struct T + buckets 数组 */
    table = ALLOC(sizeof (*table) +
        primes[i-1] * sizeof (table->buckets[0]));
    table->size = primes[i-1];
    table->cmp  = cmp  ? cmp  : cmpatom;   /* NULL -> 默认 atom 比较 */
    table->hash = hash ? hash : hashatom;  /* NULL -> 默认 atom 散列 */
    table->buckets = (struct binding **)(table + 1); /* buckets 紧随 struct T */
    for (i = 0; i < table->size; i++)
        table->buckets[i] = NULL;          /* 所有链表头初始化为 NULL */
    table->length = 0;
    table->timestamp = 0;
    return table;
}
```

**素数表设计说明**:

| 索引 | primes[i] | 用途 |
|------|-----------|------|
| 0 | 509 | hint <= 509 时的 bucket 数 |
| 1 | 509 | 冗余项，确保 for 循环 `primes[i-1]` 合法 |
| 2 | 1021 | hint in [510, 1021] |
| 3 | 2053 | hint in [1022, 2053] |
| 4 | 4093 | hint in [2054, 4093] |
| 5 | 8191 | hint in [4094, 8191] |
| 6 | 16381 | hint in [8192, 16381] |
| 7 | 32771 | hint in [16382, 32771] |
| 8 | 65521 | hint in [32772, 65521] |
| 9 | INT_MAX | hint > 65521 时保证了 for 循环终止 |

素数大约为 2^9, 2^10, ..., 2^16，覆盖了从很小到非常大的 hash 表尺寸范围。选择素数作为 bucket 数是因为: Table 无法控制客户端提供的 hash 函数的输出分布，使用素数 bucket 数可以减少 hash 碰撞的概率。

**内存布局技巧**: `ALLOC(sizeof(*table) + primes[i-1] * sizeof(table->buckets[0]))` 将 `struct T` 和 buckets 数组分配在连续内存中。`table->buckets = (struct binding **)(table + 1)` 让 buckets 指针指向 struct T 之后的内存。这样只需要一次 `ALLOC`，对应一次 `FREE`。

```
一次 ALLOC 的内存布局:
+-------------------+-----------------------------------+
|  struct T         |  buckets[0]  buckets[1]  ...      |
|  (size/cmp/hash/  |  (初始化为 NULL)                   |
|   length/ts/      |                                    |
|   buckets指针-----+-> 指向此区域                       |
+-------------------+-----------------------------------+
```

### 8.3.4 `Table_get` — 查找键

```c
/* S8.3, functions 126 */
void *Table_get(T table, const void *key) {
    int i;
    struct binding *p;
    assert(table);
    assert(key);
    /* 计算 hash，取模定位 bucket */
    i = (*table->hash)(key) % table->size;
    /* 遍历链表查找匹配的键 */
    for (p = table->buckets[i]; p; p = p->link)
        if ((*table->cmp)(key, p->key) == 0)
            break;
    return p ? p->value : NULL;
}
```

算法流程:
1. `(*table->hash)(key) % table->size` — 通过函数指针调用 hash，取模得到 bucket 索引。
2. 遍历该 bucket 的单链表，调用 `(*table->cmp)(key, p->key)` 比较键。
3. 找到则返回 `p->value`，否则返回 `NULL`。

复杂度: 平均 O(1 + load_factor) = O(1)（当 load factor 较小时），最坏 O(N)（所有键 hash 到同一个 bucket）。

### 8.3.5 `Table_put` — 插入或更新

```c
/* S8.3, functions 126 */
void *Table_put(T table, const void *key, void *value) {
    int i;
    struct binding *p;
    void *prev;
    assert(table);
    assert(key);
    /* 定位 bucket */
    i = (*table->hash)(key) % table->size;
    /* 搜索键 */
    for (p = table->buckets[i]; p; p = p->link)
        if ((*table->cmp)(key, p->key) == 0)
            break;
    if (p == NULL) {
        /* 键不存在: 分配新 binding，头插法插入链表 */
        NEW(p);
        p->key = key;
        p->link = table->buckets[i];
        table->buckets[i] = p;
        table->length++;
        prev = NULL;             /* 新插入返回 NULL */
    } else
        prev = p->value;         /* 键已存在返回旧值 */
    p->value = value;            /* 更新值 */
    table->timestamp++;          /* 修改计数器+1 */
    return prev;
}
```

**头插法的选择**: 新的 binding 插入到链表头部 (`p->link = table->buckets[i]; table->buckets[i] = p`)。这是最简单高效的方式，避免了遍历到链表尾部。由于 hash 表中的键是唯一的（不允许重复），头插法和尾插法在语义上等价。

**Table_put 的完整流程图**:

```
Table_put(table, key, value)
  |
  v
hash(key) % size -> bucket i
  |
  v
遍历 buckets[i] 链表
  |                        \
  找到匹配的 key?            未找到
  |                        |
  v                        v
prev = p->value           NEW(p) 分配新节点
p->value = value          p->key = key
返回 prev (旧值)          p->link = buckets[i]
                          buckets[i] = p (头插)
                          length++
                          prev = NULL
                          返回 NULL (新插入)
```

### 8.3.6 `Table_remove` — 删除键值对

```c
/* S8.3, functions 126 */
void *Table_remove(T table, const void *key) {
    int i;
    struct binding **pp;    /* 指向指针的指针，实现无分支链表删除 */
    assert(table);
    assert(key);
    table->timestamp++;
    i = (*table->hash)(key) % table->size;
    /* 使用 pp 遍历，pp 指向 "指向当前 binding 的指针" */
    for (pp = &table->buckets[i]; *pp; pp = &(*pp)->link)
        if ((*table->cmp)(key, (*pp)->key) == 0) {
            struct binding *p = *pp;    /* 保存当前节点 */
            void *value = p->value;     /* 保存返回值 */
            *pp = p->link;              /* 从链表中摘除 (关键行!) */
            FREE(p);                    /* 释放节点内存 */
            table->length--;
            return value;
        }
    return NULL;
}
```

**指针的指针 (double pointer) 技法**: 这是 C 语言中优雅的链表删除方法。

- `pp` 初始化为 `&table->buckets[i]`，指向 bucket 数组元素（一个 `struct binding *`）。
- 在遍历中，`pp` 指向**前一个节点的 `link` 字段（或 bucket 头）**。
- 删除时只需 `*pp = p->link`，无需区分删除的是头节点还是中间节点。

可视化图解:

```
pp -> [bucket[i]] -> [binding A] -> [binding B] -> [binding C] -> NULL
                                             ^
                                         要删除 B

执行 *pp = (*pp)->link 后:
pp -> [bucket[i]] -> [binding A] ----------------> [binding C] -> NULL
                                    [binding B] 被 FREE 移除
```

### 8.3.7 `Table_map` — 遍历访问

```c
/* S8.3, functions 126 */
void Table_map(T table,
    void apply(const void *key, void **value, void *cl),
    void *cl) {
    int i;
    unsigned stamp;
    struct binding *p;
    assert(table);
    assert(apply);
    stamp = table->timestamp;              /* 保存修改计数 */
    for (i = 0; i < table->size; i++)
        for (p = table->buckets[i]; p; p = p->link) {
            apply(p->key, &p->value, cl);  /* 调用 apply，可修改 value */
            assert(table->timestamp == stamp); /* 检测 apply 是否修改了表 */
        }
}
```

**`timestamp` 机制**: `Table_put` 和 `Table_remove` 每次修改表时递增 `timestamp`。`Table_map` 在进入时保存 `stamp`，每次调用 `apply` 后断言 `timestamp == stamp`。如果 `apply` 内部调用了 `Table_put` 或 `Table_remove`，`timestamp` 改变，`assert` 失败，从而捕获这个受检运行时错误。

这是防御性编程的一个典范: 用极小的开销（一个 `unsigned` 字段 + 一次比较）防止了遍历过程中修改容器这一常见的 bug。

### 8.3.8 `Table_toArray` — 导出为数组

```c
/* S8.3, functions 126 */
void **Table_toArray(T table, void *end) {
    int i, j = 0;
    void **array;
    struct binding *p;
    assert(table);
    /* 分配 2N+1 个指针 */
    array = ALLOC((2*table->length + 1) * sizeof(*array));
    /* 遍历所有 binding，交替填充 key 和 value */
    for (i = 0; i < table->size; i++)
        for (p = table->buckets[i]; p; p = p->link) {
            array[j++] = (void *)p->key;  /* const 强转去掉 */
            array[j++] = p->value;
        }
    array[j] = end;      /* 终止哨兵 */
    return array;
}
```

注意 `p->key` 需要从 `const void *` 强转为 `void *`，因为数组未声明为 `const`。键值对的顺序是任意的 -- 取决于 hash 表中 binding 的遍历顺序。

**数组布局可视化** (N=3):

```
array[0] = key_3     <- 偶数索引 = key
array[1] = value_3   <- 奇数索引 = value
array[2] = key_1     <- 第2个键值对
array[3] = value_1
array[4] = key_2     <- 第3个键值对
array[5] = value_2
array[6] = end       <- 终止哨兵 (通常是 NULL)
```

### 8.3.9 `Table_free` — 释放

```c
/* S8.3, functions 126 */
void Table_free(T *table) {
    assert(table && *table);
    if ((*table)->length > 0) {
        int i;
        struct binding *p, *q;
        for (i = 0; i < (*table)->size; i++)
            for (p = (*table)->buckets[i]; p; p = q) {
                q = p->link;
                FREE(p);           /* 释放每个 binding */
            }
    }
    FREE(*table);                 /* 释放 struct T + buckets */
}
```

`Table_free` 释放表的所有内部数据结构（binding 节点 + struct T + buckets 数组）。注意: 它**不释放键和值** -- 如果需要释放，必须先通过 `Table_map` 完成。

---

## 8.4 动态 Resize 策略分析

### 8.4.1 CII 原始实现的局限

CII 的 Table 实现使用**固定大小的 hash 表**: `buckets` 分配后永不再调整。这带来两个问题:

1. **负载因子不可控**: 当条目数远超 bucket 数时，链表变长，查找退化为 O(N)。
2. **hint 的负担**: 必须通过 `Table_new` 的 `hint` 参数预估表的大小，预估值偏小导致性能下降。

### 8.4.2 负载因子与性能

> 原文: As long as the load factor, which is the number of table entries divided by the number of elements in the hash table, is reasonably small, keys can be found by looking at only a few entries. Performance suffers when the load factor gets too high.

负载因子 (load factor) λ = N / B，其中 N 是条目数，B 是 bucket 数。

- λ < 1: 平均链表长度 < 1，查找接近 O(1)
- 1 < λ < 5: 链表开始变长，但仍在可接受范围
- λ > 5: 性能显著下降，需要扩容

### 8.4.3 朴素动态扩容方案

> 原文 Exercise 8.5: Once buckets is allocated, it's never expanded or contracted. Revise the Table implementation so that it uses a heuristic to adjust the size of buckets periodically.

基本思路: 当 `length / size > threshold` (如 5) 时:
1. 分配新的、更大的 buckets 数组（选择素数列表中更大的值）。
2. 遍历旧表中的所有 binding，重新 hash 插入到新表中。
3. 释放旧表。

伪代码:

```c
static void Table_resize(T table, int new_hint) {
    /* 找到新 size */
    for (i = 1; primes[i] < new_hint; i++) ;
    int new_size = primes[i-1];

    /* 分配新 buckets */
    struct binding **new_buckets = CALLOC(new_size, sizeof(*new_buckets));

    /* 遍历旧表，重新散列 */
    for (int i = 0; i < table->size; i++) {
        struct binding *p = table->buckets[i];
        while (p) {
            struct binding *next = p->link;
            unsigned idx = (*table->hash)(p->key) % new_size;
            p->link = new_buckets[idx];
            new_buckets[idx] = p;
            p = next;
        }
    }
    /* 替换旧 buckets -- 需要独立管理 buckets 的内存 */
    FREE(table->buckets);
    table->buckets = new_buckets;
    table->size = new_size;
}
```

> **注意**: CII 原始实现将 `struct T` 和 `buckets` 分配在连续内存中，无法独立释放 `buckets`。实现动态扩容需要修改内存分配策略: 将 `buckets` 独立分配。

### 8.4.4 线性动态散列 (Larson 1988)

> 原文: Larson (1988) describes a more sophisticated approach in which the hash table is expanded (or contracted) incrementally, one hash chain at a time. Larson's approach eliminates the need for hint, and it can save storage because all tables can start small.

Larson 的线性动态散列的核心思想: **增量式扩容**，一次只重新散列一个 bucket 链，避免单次 O(N) 操作的长停顿。每次 `Table_put` 时顺便迁移一个旧 bucket 到新表。这样:

- 任何单次操作仍是 O(1)（amortized）。
- 不需要 `hint` 参数。
- 所有表可以从最小尺寸开始。

---

## 8.5 泛型实现机制深度分析

### 8.5.1 三层泛型技术

CII 的 Table 通过三层机制实现 C 语言中的泛型:

| 层次 | 机制 | 代码体现 |
|------|------|----------|
| **类型擦除** | `void *` | 所有 key 和 value 都是 `void *` / `const void *` |
| **类型不透明** | 不完整类型指针 | `typedef struct T *T` 隐藏内部 struct |
| **行为参数化** | 函数指针 | `cmp` 和 `hash` 由客户端在运行时注入 |

**类型擦除的代价**:
- 类型安全完全依赖程序员的纪律，编译器不做类型检查。
- 如果 key 是字符串却传了 `cmpatom`，不会产生编译错误，只会产生运行时错误。
- 频繁的 `void *` 与具体类型之间的强制转换。

### 8.5.2 `void *` 泛型的典型用法

```c
/* 字符串 key 的例子 */
Table_T t = Table_new(0,
    (int (*)(const void *, const void *))strcmp,  /* 传入 strcmp */
    (unsigned (*)(const void *))strhash);           /* 传入自定义 hash */

Table_put(t, "hello", (void *)42);         /* key 隐式转为 const void * */
int *val = (int *)Table_get(t, "hello");   /* 返回的 void * 需强转回 int * */
```

### 8.5.3 不透明指针的设计哲学

```c
#define T Table_T
typedef struct T *T;
```

这个惯用法贯穿整个 CII 库:
1. `#define T Table_T` 是局部宏，允许在头文件中简洁地写 `T`。
2. `typedef struct T *T` 创建不透明指针类型，struct T 的定义只在 .c 文件中。
3. `#undef T` 在头文件末尾恢复 -- 每个 CII 接口都使用相同的局部 `T` 宏模式。

**为什么不用 `void *` 直接作为 handle?** 因为 `typedef struct T *T` 保留了类型名，编译器可以区分 `Table_T` 和 `Set_T` 和 `List_T`，防止不同类型的 handle 被错误地交叉使用。

---

## 8.6 Modern-C 改进: 类型安全的泛型 Hash 表

### 8.6.1 使用 `_Generic` 实现类型安全

C11 引入的 `_Generic` 关键字允许基于类型的分派，可以用来构建编译期类型安全的 hash 表包装:

```c
/* modern_table.h -- 类型安全的泛型 hash 表包装 */
#include <string.h>
#include "table.h"

/* ---- 辅助 hash 函数 ---- */
static unsigned str_hash(const void *key) {
    const unsigned char *s = (const unsigned char *)key;
    unsigned h = 0;
    while (*s) h = h * 31 + *s++;
    return h;
}

static unsigned int_hash(const void *key) {
    return *(const unsigned *)key;
}

static int int_cmp(const void *x, const void *y) {
    return *(const int *)x - *(const int *)y;
}

/* ---- 宏: 基于类型自动选择 cmp 和 hash ---- */
#define TABLE_NEW(key_type, value_type, hint) \
    Table_new(hint, _Generic((key_type *)0,     \
        const char **: (int (*)(const void *, const void *))strcmp, \
        int *       : int_cmp,                  \
        double *    : double_cmp,               \
        default     : NULL                       \
    ),                                          \
    _Generic((key_type *)0,                     \
        const char **: str_hash,                \
        int *       : int_hash,                 \
        double *    : double_hash,              \
        default     : NULL                       \
    ))

/* ---- 宏: 类型安全的 get (自动强转) ---- */
#define TABLE_GET(table, key, type) \
    ((type)Table_get((table), (const void *)(key)))

/* ---- 宏: 类型安全的 put ---- */
#define TABLE_PUT(table, key, value, type) \
    ((type)Table_put((table), (const void *)(key), (void *)(value)))
```

### 8.6.2 使用示例

```c
/* 使用示例: _Generic 类型安全包装 */
void modern_table_demo(void) {
    /* 字符串->整数的表 */
    Table_T str_to_int = TABLE_NEW(const char *, int *, 100);
    int value1 = 42;
    Table_put(str_to_int, "answer", &value1);
    int *got = TABLE_GET(str_to_int, "answer", int *);
    printf("answer = %d\n", *got);  /* 输出 42 */
    Table_free(&str_to_int);
}
```

### 8.6.3 C23 `typeof` 的更优方案 (展望)

C23 标准引入了 `typeof`，可以写出更自然的泛型宏:

```c
/* C23 typeof 版本 (概念性代码) */
#define TABLE_GET_NEW(table, key) \
    (typeof(key) Table_get((table), (const void *)(key)))
```

目前 Clang 和 GCC 已经通过 `__typeof__` 扩展支持此功能。

### 8.6.4 C23 完整类型安全包装示例

```c
/* 概念性代码: 使用 C23 typeof / auto 特性 */
#if __STDC_VERSION__ >= 202311L
#define HASH_OF(key_expr)                       \
    _Generic((key_expr),                        \
        char *: str_hash, const char *: str_hash, \
        int: int_hash,                          \
        default: NULL)

#define TABLE_OF(key_expr, hint)                \
    Table_new(hint, NULL, HASH_OF(key_expr))

#define PUT(table, key, val)                    \
    Table_put(table, (const void *)(uintptr_t)(key), (void *)(uintptr_t)(val))

#define GET(table, key)                         \
    (typeof(key))((uintptr_t)Table_get(table, (const void *)(uintptr_t)(key)))
#endif
```

---

## 8.7 与 C++ `std::unordered_map` 对比

| 特性 | CII Table (C) | `std::unordered_map` (C++) |
|------|---------------|----------------------------|
| **泛型实现** | `void *` 类型擦除 + 函数指针 | 模板 (template)，编译期单态化 |
| **类型安全** | 无编译期检查 | 编译期完全类型安全 |
| **键/值所有权** | 客户端管理 (手动 malloc/free) | 容器拥有所有权 (RAII 自动管理) |
| **动态扩容** | 固定大小 | 自动 rehash (load factor 默认 1.0) |
| **迭代器** | `Table_map` 回调 / `Table_toArray` 数组 | 前向迭代器 + 范围 for |
| **内存开销** | 极低 (每个 binding 1 个 `link` 指针) | 每个 bucket 至少 1 个指针 + 额外簿记 |
| **接口大小** | 8 个函数 | > 30 个成员函数 |
| **编译速度** | 极快 | 模板展开显著增加编译时间 |
| **ABI 稳定性** | 稳定 (不透明指针隔离) | 不稳定 (头文件变化触发重编译) |
| **性能** | 哈希碰撞后链表遍历 O(N) | 同 O(N)，但通常用更好的 hash 策略 |
| **异常处理** | 无 (依赖 assert/返回值) | 可能抛异常 (如 bad_alloc) |
| **线程安全** | 无 (客户端自行加锁) | 无 (客户端自行加锁，标准不保证) |

**CII Table 的优势场景**:
- 嵌入式系统 / 内核编程（极小内存开销，无异常，无 RTTI）
- 需要 ABI 稳定性的库边界
- 编译速度敏感的工程
- 需要完全控制内存分配策略的场景

**C++ `std::unordered_map` 的优势场景**:
- 需要类型安全的业务逻辑开发
- 需要迭代器接口和 STL 算法兼容
- 自动内存管理和异常安全
- 性能要求高的通用场景（经过高度优化的实现）

### 性能对比示例

```c
/* CII Table 性能测试概念 */
#include <time.h>
#include "table.h"
#include "atom.h"
#include "mem.h"

void perf_test(void) {
    enum { N = 1000000 };
    clock_t start, end;

    /* 创建表: hint=N 以避免过多的哈希碰撞 */
    start = clock();
    Table_T tab = Table_new(N, NULL, NULL);

    /* 插入 100 万条记录 */
    for (int i = 0; i < N; i++) {
        char buf[32];
        snprintf(buf, sizeof buf, "key_%d", i);
        const char *key = Atom_string(buf);
        int *val; NEW(val); *val = i;
        Table_put(tab, key, val);
    }
    end = clock();
    printf("Insert %d items: %.2f sec\n", N,
           (double)(end - start) / CLOCKS_PER_SEC);

    /* 查找测试 */
    start = clock();
    for (int i = 0; i < N; i++) {
        char buf[32];
        snprintf(buf, sizeof buf, "key_%d", i);
        volatile int *v = Table_get(tab, buf); /* 注意: 非 atom 查询需自定义 cmp */
        (void)v;
    }
    end = clock();
    printf("Lookup %d items: %.2f sec\n", N,
           (double)(end - start) / CLOCKS_PER_SEC);

    /* 释放 (此处为简洁省略了逐个释放 value) */
    Table_free(&tab);
}
```

---

## 8.8 设计教训总结

CII Table 的设计体现了以下核心原则:

1. **接口与实现分离**: 不透明指针 + 函数指针使得同一接口可以有多种底层实现（hash 表、红黑树等）。
2. **最小惊奇原则**: `Table_put` 的覆盖语义、`Table_remove` 的返回值语义都是直觉性的。
3. **防御性编程**: `timestamp` 机制以极低开销防止了遍历中修改容器的 bug。
4. **客户端控制生命周期**: Table 不拥有 key 和 value，由客户端决定何时/如何分配和释放。
5. **为多种实现买单**: `cmp` 函数要求三分返回值，即使 hash 表实现只需要相等性测试 -- 这是为支持树实现而付出的接口代价。
6. **一次性连续内存分配**: `struct T` 和 `buckets` 数组在同一块内存中，减少碎片，简化释放。
7. **局部宏 `#define T` + `#undef T`**: 让每个接口头文件自成体系，风格统一，同时避免污染命名空间。

---

## 8.9 习题精选

> 原文 Exercises 8.1-8.9

**8.1** 讨论不同关联表 ADT 设计的利弊（如 `Table_get` 返回值的指针而非值本身，允许多个相同 key 的 binding 共存等）。设计并实现一种不同的表 ADT。

**8.2** 用二叉搜索树或红黑树重新实现 Table。参考 Sedgewick (1990)。

**8.3** 修改实现使 `Table_map` 按插入顺序访问，`Table_toArray` 也返回同序数组。讨论此行为的实际优势。

**8.4** 修改实现使 `Table_map` 按排序顺序访问。分析在修订实现中 `Table_put` 和 `Table_get` 的平均时间复杂度。提示: 在当前实现中，`Table_put` 平均 O(1)，`Table_get` 近乎 O(1)。

**8.5** 实现动态 hash 表扩容。编写测试程序验证启发式策略的有效性。

**8.6** 实现 Larson (1988) 的线性动态散列，与 8.5 方案对比性能。

**8.7** 修改 wf.c 测量因 atom 永不被释放造成的空间浪费。

**8.8** 修改 wf.c 的比较函数使其按计数值降序排序。

**8.9** 修改 wf.c 使其对每个文件的输出按文件名的字母顺序排列。

---

## 参考文献

- Aho, A. V., B. W. Kernighan, and P. J. Weinberger. 1988. *The AWK Programming Language*. Reading, MA: Addison-Wesley.
- Griswold, R. E. 1972. *The SNOBOL4 Programming Language*. Second edition. Englewood Cliffs, NJ: Prentice-Hall.
- Griswold, R. E. and M. T. Griswold. 1986. *The Implementation of the Icon Programming Language*. Princeton, NJ: Princeton University Press.
- Griswold, R. E. and M. T. Griswold. 1990. *The Icon Programming Language*. Second edition. Englewood Cliffs, NJ: Prentice-Hall.
- Adobe Systems. 1990. *PostScript Language Reference Manual*. Second edition. Reading, MA: Addison-Wesley.
- Larson, P. 1988. "Dynamic Hash Tables," *Communications of the ACM*, vol. 31, no. 4, pp. 446-457.
- Sedgewick, R. 1990. *Algorithms in C*. Reading, MA: Addison-Wesley.

---

> 翻译与注释完成于 2026-06-15。源代码文件: `/home/ares/yyscode/cii-code/code/table.h`, `/home/ares/yyscode/cii-code/code/table.c`
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch07_链表_List.html">第7章 链表 List</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch09_集合_Set.html">第9章 集合 Set</a></div>
</div>
