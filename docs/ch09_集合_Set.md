# 第9章 集合 (Sets)

> 原文: David R. Hanson, *C Interfaces and Implementations*, Chapter 9
> 翻译与扩展: 基于 set.h / set.c 精读 + Modern-C 补充分析

---

## 9.0 引言

**集合 (Set)** 是一种无序、互异的成员 (member) 集合。基本操作包括: 成员测试、添加成员、删除成员。高阶操作包括集合的并 (union)、交 (intersection)、差 (difference) 和对称差 (symmetric difference)。

给定两个集合 s 和 t:

| 运算 | 记号 | 含义 |
|------|------|------|
| 并 (Union) | s + t | s 和 t 中所有成员的集合 |
| 交 (Intersection) | s * t | 同时出现在 s 和 t 中的成员的集合 |
| 差 (Difference) | s - t | 出现在 s 中但不在 t 中的成员的集合 |
| 对称差 (Symmetric Diff) | s / t | 仅在 s 或仅在 t 中出现的成员的集合 |

集合通常关联一个**全集 (universe)** —— 所有可能成员的集合。例如，字符集合的全集是 256 个 8-bit 字符编码。有了全集 U，就可以定义补集 U - s。

**CII 的 Set 接口不依赖全集概念。** Set 导出的函数只操纵集合成员，但从不直接检查成员本身。与 Table 接口一样，Set 要求客户端提供比较函数和哈希函数来决定两个成员是否相等以及如何定位它们。

应用场景: Set 和 Table 常一起使用。例如本章的 `xref` 程序就是典型的 Set + Table 联合使用案例 —— Table 建立标识符到文件名的映射，Set 在每个文件中存储不重复的行号。

---

## 9.1 接口 (Interface)

```c
/* set.h */
#ifndef SET_INCLUDED
#define SET_INCLUDED
#define T Set_T
typedef struct T *T;

// ============ 分配与释放 ============
extern T    Set_new   (int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *x));
extern void Set_free  (T *set);

// ============ 基本集合操作 ============
extern int   Set_length(T set);
extern int   Set_member(T set, const void *member);
extern void  Set_put   (T set, const void *member);
extern void *Set_remove(T set, const void *member);

// ============ 集合遍历 ============
extern void   Set_map    (T set,
    void apply(const void *member, void *cl), void *cl);
extern void **Set_toArray(T set, void *end);

// ============ 集合运算 (返回新集合) ============
extern T Set_union(T s, T t);   // s + t  并集
extern T Set_inter(T s, T t);   // s * t  交集
extern T Set_minus(T s, T t);   // s - t  差集
extern T Set_diff (T s, T t);   // s / t  对称差

#undef T
#endif
```

### 9.1.1 分配与释放

**`Set_new(int hint, cmp, hash)`**

分配、初始化并返回一个新的 `T`。`hint` 是对集合预期包含成员数量的估计值，精确的 `hint` 可以提升性能，但任何非负值均可接受。

- `cmp(x, y)`: 比较两个成员 x 和 y。如果 x < y 返回负整数，x == y 返回 0，x > y 返回正整数。**当 `cmp(x,y)` 返回 0 时，x 和 y 中只有一个会出现在集合中**，且 `hash(x)` 必须等于 `hash(y)`。
- `hash(x)`: 将成员映射到无符号整数 (哈希值)。
- 若 `cmp` 为 `NULL`，则成员假定为 **原子 (atom)**，`x == y` 表示 x 和 y 相同。若 `hash` 为 `NULL`，则使用默认的原子哈希函数。
- 可能抛出 `Mem_Failed` 异常。

**`Set_free(T *set)`**

释放 `*set` 并将其置为 `NULL`。它**不会**释放成员本身——需要客户端先使用 `Set_map` 手动释放成员。传入 `NULL` 集合或 `*set` 是检查级运行时错误。

### 9.1.2 基本操作

**`Set_length(T set)`** — 返回 set 的基数 (cardinality)，即包含的成员数量。

**`Set_member(T set, const void *member)`** — 若 member 在 set 中返回 1，否则返回 0。

**`Set_put(T set, const void *member)`** — 将 member 添加到 set (若已存在则更新为最新的 member 指针)。可能抛出 `Mem_Failed`。

**`Set_remove(T set, const void *member)`** — 若 member 在 set 中，则移除并返回被移除的成员 (可能不同于传入的 member 指针)；否则返回 `NULL`。

> 传入 `NULL` set 或 member 是检查级运行时错误。

### 9.1.3 遍历操作

**`Set_map(T set, void apply(const void *member, void *cl), void *cl)`**

对 set 中的每个成员调用 `apply(member, cl)`，cl 是客户端特定的闭包指针。注意与 `Table_map` 的区别: `apply` 的 member 参数是 `const void *` 而非 `void **`——**apply 不能修改集合中存储的成员指针**。不过，apply 仍可通过强制类型转换修改成员所指向的值，这可能影响集合的语义。

> apply 调用 `Set_put` 或 `Set_remove` 是检查级运行时错误。

**`Set_toArray(T set, void *end)`**

返回指向 N+1 元素数组的指针，包含 set 的所有 N 个成员 (顺序未指定)，第 N+1 个元素赋值为 `end` (通常为 `NULL`)。客户端负责释放返回的数组。可能抛出 `Mem_Failed`。

### 9.1.4 集合运算

四个函数均创建并返回**新的** T，可能抛出 `Mem_Failed`。所有函数都将 `NULL` s 或 t 解释为空集，但总是返回非空的新 T。

- **`Set_union(s, t)`** — 返回 s + t
- **`Set_inter(s, t)`** — 返回 s * t
- **`Set_minus(s, t)`** — 返回 s - t
- **`Set_diff(s, t)`** — 返回 s / t

> 检查级运行时错误: (1) s 和 t 均为 NULL；(2) s 和 t 都非 NULL 但比较/哈希函数不同。这意味着同一对的集合必须通过相同 cmp/hash 的 `Set_new` 创建。

### 9.1.5 与 Table 接口的对比

| 特征 | Set | Table |
|------|-----|-------|
| 核心抽象 | 无序成员集合 | 键-值对映射 |
| 基础数据 | member | key + value |
| 添加操作 | `Set_put(set, member)` | `Table_put(table, key, value)` |
| 查找操作 | `Set_member(set, member)` — 返回 0/1 | `Table_get(table, key)` — 返回 value 指针 |
| 删除操作 | `Set_remove(set, member)` — 返回被删除的 member | `Table_remove(table, key)` — 返回被删除的 value |
| map 回调 | `apply(member, cl)` — member 是 `const void *` | `apply(key, &value, cl)` — value 是 `void **` 可修改 |
| 高阶运算 | union / inter / minus / diff | 无 (客户端自行实现) |
| toArray | N+1 元素 (N个成员 + end) | 2N+1 元素 (键值交替 + end) |

**核心关系**: Set 实质上是 **值被忽略的 Table**。成员 = 键，忽略值。这也是习题 9.1 "用 Table 实现 Set" 和习题 9.2 "用 Set 实现 Table" 的思路基础。

---

## 9.2 示例: 交叉引用列表 (xref)

`xref` 程序输出输入文件中标识符的交叉引用列表，帮助定位程序中所有标识符的使用位置。

```
% xref xref.c getword.c
...
FILE    getword.c: 6
        xref.c: 18 43 72
...
c       getword.c: 7 8 9 10 11 16 19 22 27 34 35
        xref.c: 141 142 144 147 148
```

### 9.2.1 数据结构设计

xref 使用 **Table + Set 嵌套** 的数据结构:

```
identifiers (Table_T)
  |-- key: "FILE" (Atom)
  |   value: Table_T (files)
  |     |-- key: "getword.c" (Atom)
  |     |   value: Set_T {6}
  |     |-- key: "xref.c" (Atom)
  |         value: Set_T {18, 43, 72}
  |-- key: "c" (Atom)
      value: Table_T (files)
        ...
```

- **顶层 Table**: 键 = 标识符原子, 值 = 文件 Table
- **文件 Table**: 键 = 文件名原子, 值 = 行号 Set
- **行号 Set**: 成员 = 指向整数的指针 (int*)

### 9.2.2 关键代码片段

```c
// 导航到指定文件和标识符的行号集合
files = Table_get(identifiers, id);
if (files == NULL) {
    files = Table_new(0, NULL, NULL);
    Table_put(identifiers, id, files);
}
set = Table_get(files, name);
if (set == NULL) {
    set = Set_new(0, intcmp, inthash);
    Table_put(files, name, set);
}

// 仅在行号不在集合中时分配内存并添加
int *p = &linenum;
if (!Set_member(set, p)) {
    NEW(p);
    *p = linenum;
    Set_put(set, p);
}
```

**设计要点**: 先测试 `Set_member` 再分配，避免内存泄漏。如果直接 `NEW(p)` 然后 `Set_put`，当行号已存在时，新分配的内存不会被添加到集合，造成泄漏。

### 9.2.3 遍历打印

打印流程展示了 Table 和 Set 的联合使用:

```c
// 1. 顶层: Table_toArray 获取所有标识符, qsort 排序
void **array = Table_toArray(identifiers, NULL);
qsort(array, Table_length(identifiers), 2*sizeof(*array), compare);
for (i = 0; array[i]; i += 2) {
    printf("%s", (char *)array[i]);
    print(array[i+1]);  // 打印该标识符对应的文件表
}

// 2. 中层: 遍历文件表, 对每个文件调用 Set_toArray
void print(Table_T files) {
    void **array = Table_toArray(files, NULL);
    for (i = 0; array[i]; i += 2) {
        printf("\t%s:", (char *)array[i]);
        void **lines = Set_toArray(array[i+1], NULL);  // 获取行号数组
        qsort(lines, Set_length(array[i+1]), sizeof(*lines), cmpint);
        for (j = 0; lines[j]; j++)
            printf(" %d", *(int *)lines[j]);
    }
}
```

---

## 9.3 实现

Set 的实现与 Table 非常相似——**基于哈希表**，使用客户端提供的比较函数和哈希函数定位成员。

### 9.3.1 结构体设计

```c
/* set.c */
struct T {
    int length;                                    // 集合的基数 (成员数量)
    unsigned timestamp;                            // 用于 Set_map 的运行时错误检查
    int (*cmp)(const void *x, const void *y);     // 成员比较函数指针
    unsigned (*hash)(const void *x);              // 成员哈希函数指针
    int size;                                      // buckets 数组的大小 (质数)
    struct member {
        struct member *link;                       // 链表指针 (分离链接法)
        const void *member;                        // 指向实际成员的指针
    } **buckets;                                   // 哈希桶数组
};
```

**设计分析**:

1. **分离链接法 (Separate Chaining)**: 每个 `buckets[i]` 是一个 `member` 结构的单向链表的头指针。哈希值相同的成员存储在同一个链表中。

2. **内存布局优化**: `Set_new` 使用一次 `ALLOC` 同时分配 `struct T` 和 `buckets` 数组，两者在内存中连续放置。`buckets` 指针指向 `struct T` 之后的区域:
   ```c
   set = ALLOC(sizeof(*set) + primes[i-1] * sizeof(set->buckets[0]));
   set->buckets = (struct member **)(set + 1);  // 指向 struct T 末尾
   ```
   这种单次分配的技巧减少了内存碎片，并提升了 CPU 缓存局部性。

3. **timestamp 机制**: 用于强制 `Set_map` 遍历期间集合不能被修改的运行时检查。每次 `Set_put` 或 `Set_remove` 都会递增 `timestamp`。`Set_map` 在入口处保存初始 `timestamp` 值，并在每次调用 `apply` 后断言其未变——若变了，说明 apply 回调中错误地修改了集合。

4. **cmp 和 hash 函数指针**: 每个集合携带自己的比较和哈希函数。`NULL` 函数指针表示成员是原子，自动启用默认的 `cmpatom` 和 `hashatom`。这种设计允许对不同集合使用不同的比较/哈希策略。

### 9.3.2 默认原子比较与哈希

```c
static int cmpatom(const void *x, const void *y) {
    return x != y;                              // 原子相等 ≡ 指针相同
}

static unsigned hashatom(const void *x) {
    return (unsigned long)x >> 2;               // 右移2位: 跳过字对齐的低2位零
}
```

- `cmpatom`: 原子 x 和 y 相等当且仅当 `x == y` (指针相同)。返回 0 表示相等，1 表示不等。注意此实现只测试相等性，不关心相对顺序——这比 `strcmp` 那样的三值比较简单。
- `hashatom`: 原子本身就是地址，可以直接作为哈希值。右移 2 位是因为原子通常起始于字边界 (word boundary)，最低 2 位可能总是 0，跳过它们能更好地分散哈希值到不同的桶中。

### 9.3.3 哈希表大小选择

```c
static int primes[] = { 509, 509, 1021, 2053, 4093,
    8191, 16381, 32771, 65521, INT_MAX };
```

这些质数是 2^n 附近 (n 从 9 到 16) 的素数，提供广泛的哈希表大小范围。使用质数的原因: Set 无法控制客户端哈希函数的输出分布，质数取模可以减轻哈希值的规律性带来的聚集效应。相比之下，Atom 本身控制哈希值，所以可以使用更简单的算法。

```c
for (i = 1; primes[i] < hint; i++)  // 找到第一个 ≥ hint 的质数索引
    ;
set->size = primes[i-1];            // 使用前一个质数 (避免过度分配)
```

循环从索引 1 开始 (不是 0)，这意味着当 `hint = 0` 或 `hint` 很小时，`primes[0] = 509` 被选中。这种设计保证了最小的合理哈希表大小。

> **与 Table 的完全复用**: 这个质数表、选择逻辑、以及 `cmpatom`/`hashatom` 与 Table 实现完全相同。Set 和 Table 共享了哈希表的底层基础设施。

### 9.3.4 `Set_new` — 完整实现

```c
T Set_new(int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *x)) {
    T set;
    int i;
    static int primes[] = { 509, 509, 1021, 2053, 4093,
        8191, 16381, 32771, 65521, INT_MAX };

    assert(hint >= 0);                                        // hint 必须非负

    // 质数选择: 找到第一个 ≥ hint 的质数, 使用前一个作为实际大小
    for (i = 1; primes[i] < hint; i++)
        ;

    // 单次分配: struct T + buckets 数组连续存储
    set = ALLOC(sizeof(*set) +
        primes[i-1] * sizeof(set->buckets[0]));

    set->size = primes[i-1];                                  // 桶数量
    set->cmp  = cmp  ? cmp  : cmpatom;                        // NULL cmp → 默认原子比较
    set->hash = hash ? hash : hashatom;                       // NULL hash → 默认原子哈希
    set->buckets = (struct member **)(set + 1);               // buckets 紧跟在 struct T 之后

    // 初始化所有桶为空链表
    for (i = 0; i < set->size; i++)
        set->buckets[i] = NULL;

    set->length = 0;
    set->timestamp = 0;
    return set;
}
```

### 9.3.5 成员操作详解

#### Set_member — 成员测试

```c
int Set_member(T set, const void *member) {
    int i;
    struct member *p;

    assert(set);
    assert(member);

    // 步骤1: 哈希 → 取模 → 定位桶号
    i = (*set->hash)(member) % set->size;

    // 步骤2: 遍历该桶的链表, 用 cmp 比较每个成员
    for (p = set->buckets[i]; p; p = p->link)
        if ((*set->cmp)(member, p->member) == 0)     // cmp 返回 0 = 相等
            break;

    // 步骤3: p != NULL 表示找到了匹配的成员
    return p != NULL;
}
```

**时间复杂度**: O(1) 平均，O(n) 最坏 (所有元素哈希碰撞在同一链表中，即退化到链表查找)。

#### Set_put — 添加成员

```c
void Set_put(T set, const void *member) {
    int i;
    struct member *p;

    assert(set);
    assert(member);

    // 步骤1: 搜索 member 是否已存在于集合中
    i = (*set->hash)(member) % set->size;
    for (p = set->buckets[i]; p; p = p->link)
        if ((*set->cmp)(member, p->member) == 0)
            break;

    if (p == NULL) {
        // 步骤2a: 不存在 → 分配新节点, 采用头插入法 (prepend) 添加到链表
        NEW(p);                                 // MEM 宏: 分配 sizeof(*p) 字节
        p->member = member;                     // 存储成员指针
        p->link = set->buckets[i];              // 新节点指向原链表头
        set->buckets[i] = p;                    // 桶头指针指向新节点
        set->length++;                          // 基数递增
    } else
        // 步骤2b: 已存在 → 更新成员指针 (覆盖旧指针, 即使 "相等")
        p->member = member;

    set->timestamp++;                           // 变更标记: 用于 Set_map 检查
}
```

**设计要点**:
- 成员已存在时，**更新 `p->member`** 而非忽略。这允许客户端用新的成员指针替换旧的，即使两个成员被 `cmp` 判定为 "相同"。例如，客户端可能想更新成员指向的底层数据。
- 新节点采用**头插入** (prepend): 将新节点放在链表头部。这是最简单的插入策略，O(1) 且不需要遍历到链表尾部。
- `timestamp++` 是 `Set_map` 的运行时错误检查基础。

#### Set_remove — 删除成员

```c
void *Set_remove(T set, const void *member) {
    int i;
    struct member **pp;                            // 二级指针!

    assert(set);
    assert(member);

    set->timestamp++;                              // 变更标记

    // 步骤1: 哈希定位桶
    i = (*set->hash)(member) % set->size;

    // 步骤2: 使用二级指针 pp 遍历链表
    // pp 指向 "指向当前节点的那根指针" (可能是桶头指针, 也可能是前驱节点的 link 字段)
    for (pp = &set->buckets[i]; *pp; pp = &(*pp)->link)
        if ((*set->cmp)(member, (*pp)->member) == 0) {
            struct member *p = *pp;                // p = 待删除的节点
            *pp = p->link;                         // 从链表中摘除: 让前驱指针跳过 p
            member = p->member;                    // 保存被删除的成员指针
            FREE(p);                               // 释放 member 结构体
            set->length--;                         // 基数递减
            return (void *)member;                 // 返回被删除的成员
        }

    return NULL;                                   // 未找到
}
```

**二级指针 (`struct member **pp`) 技巧** 是 C 语言中链表删除的经典模式 (参见第 8 章 Table_remove):

```
pp 的演进过程:
初始: pp → &buckets[i] → [A] → [B] → NULL
       *pp = [A]

检查A: pp → buckets[i] → [A] → [B] → NULL
       (*pp)->member = A.member
       如果A不匹配: pp = &A.link

现在: pp → &A.link → [B] → NULL
       *pp = [B]

检查B: (*pp)->member = B.member
       如果B匹配: *pp = B.link  (即 A.link = B.link, 跳过B)
```

关键优势: **无论删除的是头节点还是中间节点，代码完全相同**——不需要特殊判断 `if (p == buckets[i])`。

#### Set_length 和 Set_free

```c
int Set_length(T set) {
    assert(set);
    return set->length;                            // 直接返回 length 字段
}

void Set_free(T *set) {
    assert(set && *set);                           // set 和 *set 都不能为 NULL

    if ((*set)->length > 0) {                      // 优化: 非空才遍历释放
        int i;
        struct member *p, *q;
        for (i = 0; i < (*set)->size; i++)         // 遍历每个桶
            for (p = (*set)->buckets[i]; p; p = q) {
                q = p->link;                       // 保存下一个节点
                FREE(p);                            // 释放当前 member 结构体
            }
    }
    FREE(*set);                                    // 释放 struct T (包含 buckets 数组)
    // FREE 宏自动将 *set 置为 NULL
}
```

关键细节: `Set_free` **只释放 `member` 结构体和 `struct T`，不释放成员指向的数据**。释放成员数据是客户端的责任 (通常通过 `Set_map` + `FREE` 完成)。这是 C 语言手动内存管理中常见的职责分离。

#### Set_map — 遍历成员

```c
void Set_map(T set,
    void apply(const void *member, void *cl), void *cl) {
    int i;
    unsigned stamp;
    struct member *p;

    assert(set);
    assert(apply);

    stamp = set->timestamp;                        // 入口: 保存当前时间戳

    for (i = 0; i < set->size; i++)                // 遍历所有桶
        for (p = set->buckets[i]; p; p = p->link) {
            apply(p->member, cl);                  // 调用回调 (传入成员指针)
            assert(set->timestamp == stamp);       // 断言: apply 未调用 Set_put/remove
        }
    // 遍历结束时 stamp 未被修改 = apply 回调没有改动集合
}
```

**遍历顺序**: 哈希桶顺序 (0 到 size-1)，链表顺序。这是不确定的 (取决于哈希分布)。

**timestamp 断言**: 如果 apply 回调中错误地调用了 `Set_put` 或 `Set_remove`，`set->timestamp` 会变化，断言立即失败，给出明确的错误提示。

**与 Table_map 的关键区别**: `apply(member, cl)` 中 member 是 `const void *`，而 `Table_map` 中 value 是 `void **` (可修改)。这意味着 `Set_map` 的回调不能修改集合中存储的成员指针——但要注意，apply 仍可通过强制类型转换修改成员指向的数据，这可能改变集合的语义。

#### Set_toArray — 转换为数组

```c
void **Set_toArray(T set, void *end) {
    int i, j = 0;
    void **array;
    struct member *p;

    assert(set);

    // 分配 N+1 个指针的数组 (额外的1个用于 end 哨兵)
    array = ALLOC((set->length + 1) * sizeof(*array));

    // 遍历所有桶, 收集成员到数组中
    for (i = 0; i < set->size; i++)
        for (p = set->buckets[i]; p; p = p->link)
            array[j++] = (void *)p->member;        // const 强制转非 const (数组未声明 const)

    array[j] = end;                                // 哨兵: 标记数组结束

    return array;
}
```

`p->member` 被强制从 `const void *` 转为 `void *`，因为返回的数组未声明为 `const`——客户端可能需要修改通过数组访问到的成员。

---

## 9.4 集合运算实现与算法分析

所有四个集合运算都返回**新创建**的集合，不修改 s 或 t。它们使用一个内部辅助函数 `copy`。

### 9.4.1 内部函数: copy

```c
static T copy(T t, int hint) {
    T set;

    assert(t);
    set = Set_new(hint, t->cmp, t->hash);          // 创建具有相同 cmp/hash 的新集合

    // 遍历 t 的所有桶和链表
    { int i;
      struct member *q;
      for (i = 0; i < t->size; i++)
          for (q = t->buckets[i]; q; q = q->link)
        {
            // 直接插入到 set (绕过 Set_put, 避免其无意义的搜索)
            struct member *p;
            const void *member = q->member;
            int i = (*set->hash)(member) % set->size;
            NEW(p);
            p->member = member;
            p->link = set->buckets[i];
            set->buckets[i] = p;
            set->length++;
        }
    }
    return set;
}
```

**copy 的核心优化**: 它没有调用 `Set_put`，而是**直接操作新集合的内部结构**，跳过了 `Set_put` 中 "先搜索再插入" 的步骤。因为新集合刚创建，必然是空的，每个成员都肯定不在其中——搜索是纯浪费。这是通过访问 "特权信息" (内部表示) 换取性能的典型做法。

### 9.4.2 并集: Set_union (s + t)

```c
T Set_union(T s, T t) {
    // 边界情况: 空集处理
    if (s == NULL) {
        assert(t);
        return copy(t, t->size);                   // NULL + t = t 的副本
    } else if (t == NULL)
        return copy(s, s->size);                   // s + NULL = s 的副本
    else {
        // 核心逻辑: 复制较大的集合, 然后将另一个的所有成员插入
        T set = copy(s, Arith_max(s->size, t->size));
        assert(s->cmp == t->cmp && s->hash == t->hash);  // 同一 cmp/hash 的要求
        { int i;
          struct member *q;
          for (i = 0; i < t->size; i++)            // 遍历 t 的所有桶
              for (q = t->buckets[i]; q; q = q->link)
                Set_put(set, q->member);           // 插入到副本 (自动去重)
        }
        return set;
    }
}
```

**算法分析**:
1. 复制 s (使用 copy, 直接插入, 跳过搜索)
2. 遍历 t 的每个成员，用 `Set_put` 添加到副本
3. `Set_put` 自动处理重复 (t 中已在 s 中的成员不会被重复添加)

**时间复杂度**: O(|s| + |t|) 平均 (假设每个 Set_put 是 O(1))

**hint 策略**: 使用 `Arith_max(s->size, t->size)`——结果集合至少与较大参数一样大。

### 9.4.3 交集: Set_inter (s * t)

```c
T Set_inter(T s, T t) {
    if (s == NULL) {
        assert(t);
        return Set_new(t->size, t->cmp, t->hash);  // 空 ∩ t = 空集
    } else if (t == NULL)
        return Set_new(s->size, s->cmp, s->hash);  // s ∩ 空 = 空集
    else if (s->length < t->length)
        return Set_inter(t, s);                    // 关键优化: 确保遍历较小的集合
    else {
        T set = Set_new(Arith_min(s->size, t->size), s->cmp, s->hash);
        assert(s->cmp == t->cmp && s->hash == t->hash);
        { int i;
          struct member *q;
          for (i = 0; i < t->size; i++)            // 遍历较小的集合 t
              for (q = t->buckets[i]; q; q = q->link)
                if (Set_member(s, q->member))       // 只添加同时出现在 s 中的成员
                    {
                        // 直接插入 (新集合为空, 无需搜索)
                        struct member *p;
                        const void *member = q->member;
                        int i = (*set->hash)(member) % set->size;
                        NEW(p);
                        p->member = member;
                        p->link = set->buckets[i];
                        set->buckets[i] = p;
                        set->length++;
                    }
        }
        return set;
    }
}
```

**算法优化**:

1. **参数交换策略**: `if (s->length < t->length) return Set_inter(t, s)` — 总是确保遍历**较小的集合**。对于交集，结果集 ≤ min(|s|, |t|)，所以遍历小集合并在大集合中查找更高效。
   - 大集合遍历 + 小集合查找: O(|s| * 1), s 是大集合
   - 小集合遍历 + 大集合查找: O(|t| * 1), t 是小集合
   - 对于 Set_member 平均 O(1), 遍历次数 = 结果集上限, 所以总是遍历小集合更优

2. **直接插入**: 与 copy 一样，新集合为空，直接插入成员而不调用 `Set_put`。

3. **hint 策略**: 使用 `Arith_min(s->size, t->size)`——结果集不会超过较小集合的大小。

**时间复杂度**: O(min(|s|, |t|)) 平均。

### 9.4.4 差集: Set_minus (t - s)

> 注意函数签名: `Set_minus(T t, T s)` 返回 t - s。参数名与数学记号保持一致。

```c
T Set_minus(T t, T s) {
    if (t == NULL) {
        assert(s);
        return Set_new(s->size, s->cmp, s->hash);  // 空 - s = 空集
    } else if (s == NULL)
        return copy(t, t->size);                   // t - 空 = t 的副本
    else {
        T set = Set_new(Arith_min(s->size, t->size), s->cmp, s->hash);
        assert(s->cmp == t->cmp && s->hash == t->hash);
        { int i;
          struct member *q;
          for (i = 0; i < t->size; i++)            // 遍历被减数 t
              for (q = t->buckets[i]; q; q = q->link)
                if (!Set_member(s, q->member))      // 只添加不在 s 中的成员
                    {
                        struct member *p;
                        const void *member = q->member;
                        int i = (*set->hash)(member) % set->size;
                        NEW(p);
                        p->member = member;
                        p->link = set->buckets[i];
                        set->buckets[i] = p;
                        set->length++;
                    }
        }
        return set;
    }
}
```

**参数语义**: 函数参数 `(T t, T s)` 命名上与数学记号 t - s 一致——第二参数 s 是 "被减去的" 集合。函数内部遍历 t 中所有成员，只将不在 s 中的成员加入结果集。

**时间复杂度**: O(|t|) 平均 (每个成员执行一次 Set_member 查找)。

### 9.4.5 对称差: Set_diff (s / t)

对称差 = (s - t) + (t - s)，即出现在 s 或 t 中但**不同时**出现在两者中的成员。

```c
T Set_diff(T s, T t) {
    if (s == NULL) {
        assert(t);
        return copy(t, t->size);                   // 空 / t = t
    } else if (t == NULL)
        return copy(s, s->size);                   // s / 空 = s
    else {
        T set = Set_new(Arith_min(s->size, t->size), s->cmp, s->hash);
        assert(s->cmp == t->cmp && s->hash == t->hash);

        // 第一遍: 添加 t 中有但 s 中没有的成员
        { int i;
          struct member *q;
          for (i = 0; i < t->size; i++)
              for (q = t->buckets[i]; q; q = q->link)
                if (!Set_member(s, q->member))
                    {
                        struct member *p;
                        const void *member = q->member;
                        int i = (*set->hash)(member) % set->size;
                        NEW(p);
                        p->member = member;
                        p->link = set->buckets[i];
                        set->buckets[i] = p;
                        set->length++;
                    }
        }

        // 交换 s 和 t 的引用 → 复用相同代码块
        { T u = t; t = s; s = u; }

        // 第二遍: 添加 (原先的) s 中有但 t 中没有的成员
        { int i;
          struct member *q;
          for (i = 0; i < t->size; i++)
              for (q = t->buckets[i]; q; q = q->link)
                if (!Set_member(s, q->member))
                    {
                        struct member *p;
                        const void *member = q->member;
                        int i = (*set->hash)(member) % set->size;
                        NEW(p);
                        p->member = member;
                        p->link = set->buckets[i];
                        set->buckets[i] = p;
                        set->length++;
                    }
        }
        return set;
    }
}
```

**实现技巧分析**:

两遍遍历使用**完全相同的代码块** (都是 `for each member q in t: if (!Set_member(s, q->member)) 添加`)，通过交换 s 和 t 的引用来实现代码复用:

```
第一遍: s=原s, t=原t  →  添加 原t-原s
交换:   {T u=t; t=s; s=u;}   →  s=原t, t=原s
第二遍: s=原t, t=原s  →  添加 原s-原t
总计: (原t-原s) + (原s-原t) = s / t
```

这个交换技巧避免了写两个不同的代码块或提取函数，在源代码层面保持了简洁性。

**时间复杂度**: O(|s| + |t|) 平均 (两遍完全遍历)。

### 9.4.6 集合运算算法总结

| 运算 | 策略 | 平均时间复杂度 | 新集哈希表大小 |
|------|------|--------------|-------------|
| Union (s + t) | 复制 s，遍历 t 添加 | O(\|s\| + \|t\|) | max(s.size, t.size) |
| Inter (s * t) | 遍历较小集，仅添加共有成员 | O(min(\|s\|, \|t\|)) | min(s.size, t.size) |
| Minus (t - s) | 遍历 t，仅添加不在 s 中的成员 | O(\|t\|) | min(s.size, t.size) |
| Diff (s / t) | 两遍遍历: (t-s) + (s-t) | O(\|s\| + \|t\|) | min(s.size, t.size) |

**共同模式**:
1. 处理 NULL 参数 (空集): 返回空集或副本
2. 断言 `s->cmp == t->cmp && s->hash == t->hash` (运行时检查一致性)
3. 使用 copy 或直接插入 (绕过 `Set_put` 的无意义搜索)

---

## 9.5 与 Table 的代码复用关系

Set 和 Table 的实现有**高度结构同构性 (structural isomorphism)**，这是 CII 库中最显著的代码复用案例之一。

### 9.5.1 共享组件清单

| 组件 | Set | Table |
|------|-----|-------|
| 哈希表结构 | `struct member { link, member } **buckets` | `struct binding { link, key, value } **buckets` |
| 质数表 | `{509, 509, 1021, ..., INT_MAX}` | 完全相同 |
| size 选择逻辑 | `for (i=1; primes[i] < hint; i++)` | 完全相同 |
| 默认原子比较 | `cmpatom: return x != y` | 完全相同 |
| 默认原子哈希 | `hashatom: return (unsigned long)x>>2` | 完全相同 |
| 内存布局 | 单次 `ALLOC(sizeof(*X) + primes*sizeof(bucket))` | 完全相同 |
| buckets 指针 | `buckets = (struct X**)(X + 1)` | 完全相同 |
| timestamp 机制 | 用于 Set_map 检查 | 用于 Table_map 检查 |
| length 字段 | 成员计数 | 键值对计数 |
| 链表的头插入 | `p->link = buckets[i]; buckets[i] = p` | 完全相同 |
| 二级指针删除 | `pp = &buckets[i]; *pp = p->link` | 完全相同 |
| toArray 分配模式 | `ALLOC((length+1) * sizeof(*array))` | `ALLOC((2*length+1) * sizeof(*array))` |
| free 遍历释放 | 遍历桶链表释放节点 | 完全相同 |

### 9.5.2 差异点

| 方面 | Set | Table |
|------|-----|-------|
| 节点数据结构 | 只存 member 指针 | 存 key + value 两个指针 |
| put 行为 | 存在则更新 member 指针, 无返回值 | 存在则更新 value, 返回旧 value |
| remove 返回值 | 返回被删除的 member 指针 | 返回被删除的 value 指针 |
| get/查询 | `Set_member` 返回 int (0/1) | `Table_get` 返回 value 指针或 NULL |
| map 回调签名 | `apply(member, cl)` — member 不可改 | `apply(key, &value, cl)` — value 可改 |
| 集合运算 | union/inter/minus/diff | 无 (客户端自行实现) |
| toArray 元素数 | N + 1 (N members + end哨兵) | 2N + 1 (键值交替 + end哨兵) |

### 9.5.3 为什么选择独立实现而不是继承/抽象

C 语言不支持继承，但 CII 选择了**复制 + 调整 (copy-and-adapt)** 而非以下替代方案:

1. **宏抽象**: 用 `#define` 生成代码——丧失类型安全，调试困难
2. **void* 泛型**: 统一的哈希表 + 运行时 tag——增加间接调用开销
3. **提取公共模块**: 如习题 9.3 建议——增加层级，C 中缺乏语言支持

选择复制的原因:

1. **类型安全**: `Set_T` 和 `Table_T` 是不同的不透明指针类型，编译器可检查误用
2. **语义差异**: `Set_map` 的 `apply` 签名不同 (member 不可修改)，`Set_remove` 返回值含义不同——这些差异使得简单的代码共享反而增加复杂度
3. **性能**: 各自独立实现避免了额外的间接调用层
4. **教学价值**: 习题 9.1 (用 Table 实现 Set)、9.2 (用 Set 实现 Table)、9.3 (抽取公共接口) 故意让学生探索这种关系

实际上，CII 选择了**在具体文件级别复制代码**而非在接口级别抽象。这导致了 set.c ~280 行和 table.c ~250 行的大量结构重复——但换来了每个接口的独立清晰性和类型安全。这是 C 语言中 "duplication is cheaper than the wrong abstraction" 哲学的体现。

---

## 9.6 小集合优化: 直接存储 vs 哈希存储

### 9.6.1 原始设计的空间效率问题

CII 的 Set 实现**没有针对小集合做特殊优化**——即使是只有 1 个成员的集合，也要分配至少 509 个桶 (primes[0]) 的哈希表。每个桶是一个指针 (4 或 8 字节)，总开销 ≥ 2036 字节。

对于许多实际场景，这造成了巨大的空间浪费:
- xref 中每个文件的标识符通常只有几十个行号
- 编译器的符号表可能有大量名空间，每个只有几个符号
- 图的邻接表中，每个顶点的度数通常很小

### 9.6.2 直接存储模式

对 n < 阈值 (如 8) 的小集合，可以用**线性数组**替代哈希表:

```
      哈希表模式 (大集合)              小集合模式 (直接存储)
   buckets[509]                       members[8]  (顺序存储)
   ├── [0] → NULL                     [0] = ptr_to_member_1
   ├── [1] → (ptr1, link)             [1] = ptr_to_member_2
   ├── [2] → NULL                     [2] = ptr_to_member_3
   └── ...                            ...
```

**小集合阈值的经验选择**:
- 成员 ≤ 8 时，线性搜索 O(n) 比哈希 O(1) 的常数开销更小
- 内存: 8 个指针的连续数组 (64 字节) << 509 个桶 + 链表节点 (数 KB)
- 缓存: 连续数组访问的 cache miss 远少于指针跳转的哈希链表

### 9.6.3 可能的实现思路

```c
#define SMALL_SET_THRESHOLD 8

struct T {
    int length;
    unsigned timestamp;
    int (*cmp)(const void *x, const void *y);
    unsigned (*hash)(const void *x);
    int size;           // >0: 哈希表桶数; <0: 直接存储容量 (绝对值)
    union {
        struct member {
            struct member *link;
            const void *member;
        } **buckets;     // 哈希表模式 (size > 0)
        const void **members;  // 直接存储模式 (size < 0, 容量 = -size)
    } storage;
};
```

转换策略 (升级为哈希表):
1. 当 `length` 超过 `SMALL_SET_THRESHOLD` 时触发升级
2. 分配新的哈希表 (质数表选择)
3. 将直接存储中的所有成员重新哈希插入到哈希表中
4. 释放旧的直接存储数组

这种 **small-size optimization (SSO)** 在现代容器库 (如 LLVM 的 SmallVector, C++ std::string) 中广泛使用。

---

## 9.7 Modern-C 改进

### 9.7.1 bitset 优化已知全集的小集合

当成员来自一个**确定的、大小适中的全集**时，可以用 **bitset (位图)** 来表示集合，将空间复杂度从 O(n) 降至 O(U/8) 字节，其中 U 是全集大小。

```c
// 经典 bitset 实现 (适用于全集大小 ≤ 4096)
#include <stdint.h>
#include <stdbool.h>

typedef struct {
    uint64_t bits[64];   // 64 * 64 = 4096 bits
    int length;          // 当前集合基数
} SmallSet;

static inline bool smallset_member(const SmallSet *s, int id) {
    return (s->bits[id / 64] >> (id % 64)) & 1;
}

static inline void smallset_put(SmallSet *s, int id) {
    if (!smallset_member(s, id)) {
        s->bits[id / 64] |= (1ULL << (id % 64));
        s->length++;
    }
}

static inline void smallset_remove(SmallSet *s, int id) {
    if (smallset_member(s, id)) {
        s->bits[id / 64] &= ~(1ULL << (id % 64));
        s->length--;
    }
}

// bitwise 集合运算 → O(num_words) 而非 O(n)
static inline SmallSet smallset_inter(SmallSet a, SmallSet b) {
    SmallSet r = {0};
    for (int i = 0; i < 64; i++)
        r.bits[i] = a.bits[i] & b.bits[i];
    return r;
}
```

**适用场景**: 字符集合 (全集 256, 只需 32 字节)、寄存器编号、IR 中的变量编号、AST 节点类型集合等。

**性能对比** (对 256-元素全集):

| 操作 | 哈希表 Set | bitset SmallSet |
|------|-----------|----------------|
| member 测试 | ~1 次哈希 + 链表遍历 | 1 次位测试 |
| 添加成员 | ~1 次哈希 + 链表遍历 + malloc | 1 次位设置 |
| 并集 | O(\|s\|+\|t\|) 遍历 + 插入 | O(1) 按位 OR (4 words) |
| 交集 | O(min(\|s\|,\|t\|)) 遍历 + 查找 | O(1) 按位 AND (4 words) |
| 空间 (空集) | ~2KB (509 桶) | 32 字节 (256 bits) |

### 9.7.2 SIMD 加速集合运算

当两个集合大小相同时，bitset 的集合运算可以直接映射到 SIMD 指令:

```c
#include <immintrin.h>  // AVX2 intrinsics

// 256-bit SIMD 并集 (一次性处理 256 个成员)
void bitset_union_avx2(uint64_t *dst, const uint64_t *a,
                       const uint64_t *b, int nwords) {
    for (int i = 0; i < nwords; i += 4) {
        __m256i va = _mm256_loadu_si256((const __m256i *)&a[i]);
        __m256i vb = _mm256_loadu_si256((const __m256i *)&b[i]);
        __m256i vr = _mm256_or_si256(va, vb);
        _mm256_storeu_si256((__m256i *)&dst[i], vr);
    }
}

// 交集: _mm256_and_si256
// 差集: _mm256_andnot_si256
// 对称差: _mm256_xor_si256
```

性能估算: 对于 4096-bit 的集合 (512 字节)，AVX2 一次性处理 256 bits，全集运算只需 16 次 SIMD 操作。相比逐 `uint64_t` 的 64 次普通操作，理论加速比约为 4x (整数 SIMD 单元)。

### 9.7.3 哈希表的动态扩展

CII 的 Set 和 Table 实现使用**固定大小的哈希表**——`buckets` 分配后永不扩展或收缩。传统哈希表的负载因子 (load factor = n / size) 超过阈值时需要扩展以维持 O(1) 查找。

动态扩展的实现概要:

```c
static void set_expand(T set) {
    if (set->length / set->size > 5) {     // 负载因子 > 5 → 触发扩展
        int oldsize = set->size;
        struct member **old = set->buckets;

        // 查找更大的质数
        int i;
        for (i = 1; primes[i] <= oldsize; i++)
            ;
        int newsize = primes[i];

        // 重新分配 struct T + 新桶数组
        // 遍历旧桶的每条链表, 将每个 member 重新哈希到新桶中
        // 释放旧结构体 ...
    }
}
```

**Larson (1988)** 描述了一种**增量式线性动态哈希**算法，每次只扩展一个哈希链，将重新哈希的成本分摊到多次操作中，消除了初始 `hint` 参数的需要，且所有表可以从最小的 size 开始，逐步增长。

### 9.7.4 缓存友好的开放寻址法

分离链接法 (chaining) 需要额外的链表节点分配，带来:
- **指针追踪 (pointer chasing)** 开销: 每次跟随一个 link 字段都可能触发 cache miss
- **内存碎片**: 每个节点独立分配，散布在堆中

对于现代 CPU (L1 cache 延迟 ~4 cycles, 主存 ~200 cycles):

**开放寻址法 (open addressing)** 配合 **Robin Hood hashing** 或 **SwissTable** (Google Abseil 中使用的算法) 可能是更好的选择:
- 所有成员存储在连续的数组中 (无指针跳转)
- SIMD 加速的探测 (一次比较 16 个槽)
- 负载因子可达 0.875 (vs. 分离链接法的 ~1)
- 内存效率更高 (无 link 指针开销)

但开放寻址法要求成员大小已知且可移动，这与 CII 的 `const void *` 指针语义不直接兼容——需要更深层的接口重构。

### 9.7.5 综合改进方案: Tagged Union 设计

```c
enum SetImpl { SET_BITSET, SET_DIRECT, SET_HASH };

struct T {
    enum SetImpl impl;     // 实现类型标签
    int length;
    union {
        struct {
            uint64_t *bits;
            int universe;         // 全集大小
        } bitset;
        struct {
            const void **members;
            int capacity;
        } direct;
        struct {
            struct member **buckets;
            int size;
            int (*cmp)(const void *, const void *);
            unsigned (*hash)(const void *);
        } hash;
    } u;
};
```

每种操作 (`Set_member`, `Set_put`, `Set_remove`, 集合运算) 根据 `impl` 分发到对应的实现。小集合自动使用 bitset 或直接存储，达到阈值后透明升级到哈希表。

---

## 9.8 习题要点

| 题号 | 主题 | 核心思想 |
|------|------|---------|
| 9.1 | 用 Table 实现 Set | member 作为 key，value 忽略 (可存 NULL 或 dummy) 。需额外处理 Set_map 签名差异 |
| 9.2 | 用 Set 实现 Table | key-value pair 作为 member，定制 cmp/hash 只看 key。Table_get 需要遍历 Set 返回 value |
| 9.3 | 抽取公共接口 | 设计 ScatterTable/HashTable 作为 Set 和 Table 的基类。提炼哈希、查找、插入、删除的公共逻辑 |
| 9.4 | 设计 Bag (多重集) | 成员可重复出现。存储方案: 添加计数 count 字段，或允许多个相同成员节点共存 |
| 9.5 | copy 批量分配优化 | 已知 count，一次 ALLOC 分配所有 member 结构体，然后逐个填充到桶中。减少 malloc 调用次数 |
| 9.6 | 缓存哈希值 | 在 member 结构体中添加 hash 字段，避免重复计算。先比较 hash 值，相同时才比较成员本身 |
| 9.7 | 同大小桶优化 | s.size == t.size 时，逐桶 union/inter 而非逐成员: 合并相同哈希值的链表，避免跨桶查找 |
| 9.8 | xref 行号范围合并 | 对连续行号 "7,8,9,10,11" 显示为 "7-11"。需要排序后检测连续区间 |
| 9.9 | xref 内存完全释放 | 增量释放策略: 在打印过程中顺便释放 Table 和 Set 节点，避免遍历完再单独释放 |
| 9.10 | 整数比较溢出 | `return **(int**)x - **(int**)y` 在极端值下可能整数溢出 (如 INT_MIN - INT_MAX)。正确做法是显式 if-else 比较 |

---

## 9.9 总结

Set 接口是 CII 库中与 Table 最紧密相关的组件:

1. **相同的底层机制**: 分离链接哈希表、质数表 size 选择、原子默认 cmp/hash 函数、timestamp 运行时检查
2. **简化的抽象**: member 即 key，无关联 value。`Set_member` 返回 0/1 而非返回指针
3. **额外的高阶运算**: union / inter / minus / diff 四个集合运算返回新集——这是 Table 没有直接提供的
4. **保守的代码复用策略**: 复制而非继承。Set 和 Table 的 .c 文件各自 ~280 行和 ~250 行，大量结构重复但换来了类型安全和接口清晰

**Set 的核心设计决策**:
- 分离链接法 (chaining) 作为碰撞解决策略——简单且删除操作自然
- 头插入 (prepend) 作为插入策略——O(1) 且代码简单
- 固定大小哈希表——对 hint 参数的依赖是主要局限
- 集合运算的 copy + 遍历策略——正确但效率有优化空间 (见习题 9.5-9.7)

理解 Set 实现的核心在于理解它与 Table 的结构同构性——两者本质上是同一哈希表引擎的不同接口封装。习题 9.1-9.3 精确地引导学生探索这种关系，这也是 CII 全书的核心教学哲学: **通过对比和重构来理解接口与实现的分离**。

---

*本章基于 David R. Hanson 的《C Interfaces and Implementations》第 9 章，结合 `set.h` / `set.c` 源代码精读分析及 Modern-C 优化建议编写。*
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch08_表_Table.html">第8章 表 Table</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch10_动态数组_Array.html">第10章 动态数组 Array</a></div>
</div>
