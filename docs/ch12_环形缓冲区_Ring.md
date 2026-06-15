# 第12章 环形结构 (Rings)

> 译注: 本章基于 David R. Hanson 著《C Interfaces and Implementations》第12章 "Rings" 内容，
> 结合源码 `ring.h`/`ring.c` 进行逐行精读与中文注释。
> 本章在忠实翻译原文的基础上，补充了固定大小环形缓冲区的变体讨论、无锁实现以及
> 与 Linux 内核 `kfifo` 的对比分析。

---

## 12.1 引言: 什么是 Ring?

在 CII 库中，**Ring** 是一种**环形双向链表** (circular doubly-linked list)。与常见的
单向链表不同，环形链表没有"起点"和"终点"，每个节点都同时是前驱和后继。这种对称结构
使得在任意位置插入和删除元素的代价接近。

### 12.1.1 环形结构的两个维度

这里需要先澄清一个容易混淆的概念。在计算机科学中，"Ring" 这个词可以指两种不同的
数据结构:

| 维度 | CII Ring (本章主体) | 固定大小环形缓冲区 |
|------|---------------------|---------------------|
| 底层结构 | 双向链表，节点动态分配 | 固定数组 + 头尾索引 |
| 容量 | 无上限 (依赖堆内存) | 预分配，固定 |
| 满/空判断 | 不适用 (动态伸缩) | 经典 off-by-one 问题 |
| 典型用途 | 通用容器、双端队列 | 生产者-消费者模型、内核数据路径 |
| 并发模型 | 需要外部锁 | 适合无锁 (lock-free) 实现 |

本章主体内容围绕 **CII Ring (环形双向链表)** 展开。第12.6节起将讨论固定大小环形缓冲区
及其现代演进方向。

---

## 12.2 数据结构

### 12.2.1 接口定义 (`ring.h`)

```c
/* $Id$ */
#ifndef RING_INCLUDED
#define RING_INCLUDED
#define T Ring_T
typedef struct T *T;

extern T        Ring_new   (void);              // 创建空环
extern T        Ring_ring  (void *x, ...);      // 用可变参数列表创建环
extern void     Ring_free  (T *ring);           // 释放环及所有节点
extern int      Ring_length(T  ring);           // 返回环中元素个数
extern void    *Ring_get   (T ring, int i);     // 获取第 i 个元素
extern void    *Ring_put   (T ring, int i, void *x); // 替换第 i 个元素，返回旧值
extern void    *Ring_add   (T ring, int pos, void *x); // 在指定位置插入元素
extern void    *Ring_addlo (T ring, void *x);   // 在低端 (头部) 插入
extern void    *Ring_addhi (T ring, void *x);   // 在高端 (尾部) 插入
extern void    *Ring_remove (T ring, int i);    // 删除第 i 个元素
extern void    *Ring_remlo (T ring);            // 删除低端 (头部) 元素
extern void    *Ring_remhi (T ring);            // 删除高端 (尾部) 元素
extern void     Ring_rotate(T ring, int n);     // 旋转: 移动头部指针 n 个位置

#undef T
#endif
```

`Ring_T` 是一个**不透明指针类型** (opaque pointer)，客户端代码只能通过上述函数操作环，
而不能直接访问其内部字段。这是 CII 库贯彻始终的 ADT (Abstract Data Type) 设计原则。

### 12.2.2 内部实现结构 (`ring.c`)

```c
struct T {
    struct node {
        struct node *llink, *rlink;  // 左指针 (逆时针) 和右指针 (顺时针)
        void *value;                 // 存储的泛型指针
    } *head;                         // 指向"第 0 个"元素
    int length;                      // 环中元素个数
};
```

核心设计要点:

1. **双向链接**: 每个节点有 `llink` (left link, 逆时针方向) 和 `rlink` (right link, 顺时针方向) 两个指针。
2. **循环闭合**: 对于非空环，`head->llink` 指向最后一个元素，最后一个元素的 `rlink` 指向 `head`，形成闭合回路。
3. **`head` 语义**: `head` 指向环的"逻辑起点" (第 0 个元素)。旋转环就是改变 `head` 的指向。
4. **泛型存储**: `void *value` 可以存储任意类型的指针，体现了 C 语言中泛型编程的惯用手法。

### 12.2.3 ASCII 结构示意图

```
空环 (length == 0):
    ring->head == NULL
    ring->length == 0


单节点环 (length == 1):
           +----------+
           |  value   |<------ ring->head
           +----------+
           |  llink   |---+
           |  rlink   |---+
           +----------+   |
               ^          |
               |          |
               +----------+  (自环: llink == rlink == 自身)


多节点环 (length == 4, 以 A 为 head):

          +--------+    +--------+    +--------+    +--------+
  head--->| value  |--->| value  |--->| value  |--->| value  |
          |  (A)   |    |  (B)   |    |  (C)   |    |  (D)   |
          +--------+    +--------+    +--------+    +--------+
          | llink|<----| llink|<----| llink|<----| llink|
          | rlink|---->| rlink|---->| rlink|---->| rlink|
          +--------+    +--------+    +--------+    +--------+
              ^                                       |
              |                                       |
              +---------------------------------------+
                         (D.rlink == A, A.llink == D)

顺时针方向 (rlink):  head=A -> B -> C -> D -> (回到 A)
逆时针方向 (llink):  head=A -> D -> C -> B -> (回到 A)

通过 rlink 遍历即正向遍历 (0, 1, 2, ..., length-1)
通过 llink 遍历即反向遍历 (0, length-1, length-2, ..., 1)
```

---

## 12.3 接口 API 逐行注释

### 12.3.1 创建与销毁

#### `Ring_new` -- 创建空环

```c
T Ring_new(void) {
    T ring;
    NEW0(ring);            // NEW0: 调用 CALLOC (calloc) 分配并零初始化
    ring->head = NULL;     // 空环: 头指针为 NULL
    return ring;
}
```

`NEW0` 宏展开为 `Mem_calloc(1, sizeof(struct T), __FILE__, __LINE__)`，
分配内存并将所有字节置零，因此 `ring->length` 自动初始化为 0。

#### `Ring_ring` -- 批量创建环

```c
T Ring_ring(void *x, ...) {      // x 是第一个元素，... 是可变参数，以 NULL 终结
    va_list ap;
    T ring = Ring_new();          // 先创建空环
    va_start(ap, x);
    for ( ; x; x = va_arg(ap, void *))  // 遍历可变参数，遇 NULL 停止
        Ring_addhi(ring, x);     // 每个元素添加到高端 (尾部)
    va_end(ap);
    return ring;
}
```

这是一种便捷的构造方法。调用示例:

```c
Ring_T r = Ring_ring("alpha", "beta", "gamma", NULL);
// r 包含: alpha -> beta -> gamma, head 指向 "alpha"
```

#### `Ring_free` -- 释放环

```c
void Ring_free(T *ring) {
    struct node *p, *q;
    assert(ring && *ring);           // 断言: ring 指针及其指向的环都非 NULL
    if ((p = (*ring)->head) != NULL) {
        int n = (*ring)->length;
        for ( ; n-- > 0; p = q) {    // 精确遍历 length 次
            q = p->rlink;            // 先保存下一个节点
            FREE(p);                 // 释放当前节点
        }
    }
    FREE(*ring);                     // 最后释放环结构体本身
}
```

两个重要细节:
1. **释放前保存后继**: `q = p->rlink` 必须在 `FREE(p)` 之前执行，否则访问已释放内存是未定义行为。
2. **按 `length` 计数释放**: 不依赖 `p == head` 作为终止条件，因为 `FREE(p)` 后 `p` 的内容已不可信。
3. **`FREE(*ring)` 后将 `*ring` 置零**: `FREE` 宏展开后会执行 `(*ring) = 0`，防止悬挂指针。

---

### 12.3.2 存取与替换

#### `Ring_length` -- 获取环长度

```c
int Ring_length(T ring) {
    assert(ring);
    return ring->length;    // O(1) 操作，直接返回 length 字段
}
```

#### `Ring_get` -- 按索引获取元素 (带路径优化)

```c
void *Ring_get(T ring, int i) {
    struct node *q;
    assert(ring);
    assert(i >= 0 && i < ring->length);  // 检查的运行时错误 (索引越界)
    {
        int n;
        q = ring->head;
        if (i <= ring->length/2)           // 优化: 选择较短路径
            for (n = i; n-- > 0; )
                q = q->rlink;              // 顺时针走 i 步
        else
            for (n = ring->length - i; n-- > 0; )
                q = q->llink;              // 逆时针走 (length - i) 步
    }
    return q->value;
}
```

这段代码体现了环形结构的一个关键优化: **双向遍历取捷径**。对于长度为 N 的环:

- 如果 `i <= N/2`: 从 `head` 出发，沿 `rlink` 顺时针走 `i` 步到达目标。
- 如果 `i > N/2`: 从 `head` 出发，沿 `llink` 逆时针走 `N - i` 步到达目标。

最坏情况下的步数不超过 `floor(N/2)`，对于随机访问来说，时间复杂度
为 O(N)，但常数因子比单向链表小一半。

#### `Ring_put` -- 替换指定位置的元素

```c
void *Ring_put(T ring, int i, void *x) {
    struct node *q;
    void *prev;
    assert(ring);
    assert(i >= 0 && i < ring->length);
    {
        int n;
        q = ring->head;
        if (i <= ring->length/2)           // 同样的捷径选择
            for (n = i; n-- > 0; )
                q = q->rlink;
        else
            for (n = ring->length - i; n-- > 0; )
                q = q->llink;
    }
    prev = q->value;    // 保存旧值
    q->value = x;       // 写入新值
    return prev;        // 返回旧值 (CII 惯例: 替换操作返回被替换的旧值)
}
```

返回旧值是 CII 库的设计惯例。这样做的好处是客户端可以决定是否需要释放旧值指向的资源。

---

### 12.3.3 插入操作

#### `Ring_addhi` -- 在高端 (尾部) 插入

"高端" (high end) 指的是紧邻 `head` 之前的位置，即逻辑上的最后一个元素之后。

```c
void *Ring_addhi(T ring, void *x) {
    struct node *p, *q;
    assert(ring);
    NEW(p);                              // 为新节点分配内存, p 指向新节点
    if ((q = ring->head) != NULL) {      // 非空环
        p->llink = q->llink;             // 新节点的左指针指向原尾节点
        q->llink->rlink = p;             // 原尾节点的右指针指向新节点
        p->rlink = q;                    // 新节点的右指针指向 head
        q->llink = p;                    // head 的左指针指向新节点
    } else                               // 空环
        ring->head = p->llink = p->rlink = p;  // 单节点自环
    ring->length++;
    return p->value = x;                 // 存入数据并返回
}
```

**非空环插入过程 (在 head=D 之前插入新节点 N):**

```
插入前:  A <-> B <-> C <-> D (head=A, 尾=D, D.llink=C, C.rlink=D, D.rlink=A, A.llink=D)

步骤1: p->llink = q->llink;     // p->llink = D (因为 q=head=A, q->llink=D)
步骤2: q->llink->rlink = p;     // D.rlink = p  (原尾节点 D 的 rlink 从 A 改为 p)
步骤3: p->rlink = q;            // p->rlink = A
步骤4: q->llink = p;            // A.llink = p

插入后:  A <-> B <-> C <-> D <-> N (head=A, 尾=某个...)

等等，让我更仔细地跟踪:

设原环 head=A, 且: A.llink=D, D.rlink=A

步骤1: p->llink = q->llink = A.llink = D
步骤2: q->llink->rlink = p 即 D.rlink = p (D的右指针指向新节点)
步骤3: p->rlink = q = A (新节点右指针指向head)
步骤4: q->llink = p 即 A.llink = p (head的左指针指向新节点)

结果: D.rlink=p, p.rlink=A, p.llink=D, A.llink=p
数据流: head=A -> ... -> D -> p -> (回到A)
即新节点 p 成为新的尾节点。
```

**空环插入过程:**

```c
ring->head = p->llink = p->rlink = p;
```

单节点自环: `head` 指向新节点，新节点的 `llink` 和 `rlink` 都指向自己。
遍历: `p -> p -> p -> ...` (无限自循环)。

#### `Ring_addlo` -- 在低端 (头部) 插入

```c
void *Ring_addlo(T ring, void *x) {
    assert(ring);
    Ring_addhi(ring, x);                   // 先在尾部插入
    ring->head = ring->head->llink;        // 然后将 head 指向新插入的节点
    return x;
}
```

巧妙之处: 利用 `Ring_addhi` 将新节点插入尾部，然后通过 `ring->head = ring->head->llink`
将 `head` "回退"一步，使新节点成为第 0 个元素。

```
步骤1 (addhi 后): head=A -> B -> C -> N (head 仍为 A)
步骤2 (head = head->llink): head 变为 N (因为 A.llink = N)
最终: head=N -> A -> B -> C
```

这样新节点 N 就处于低端 (索引 0) 的位置。

#### `Ring_add` -- 在任意位置插入

```c
void *Ring_add(T ring, int pos, void *x) {
    assert(ring);
    assert(pos >= -ring->length && pos <= ring->length + 1);
    if (pos == 1 || pos == -ring->length)
        return Ring_addlo(ring, x);       // 插入到低端
    else if (pos == 0 || pos == ring->length + 1)
        return Ring_addhi(ring, x);       // 插入到高端
    else {
        struct node *p, *q;
        int i = pos < 0 ? pos + ring->length : pos - 1;
        {
            int n;
            q = ring->head;
            if (i <= ring->length / 2)    // 定位到插入点的前驱
                for (n = i; n-- > 0; )
                    q = q->rlink;
            else
                for (n = ring->length - i; n-- > 0; )
                    q = q->llink;
        }
        // q 指向插入位置的前一个节点 (即新节点将插入在 q 和 q->rlink 之间)
        NEW(p);
        p->llink = q->llink;
        q->llink->rlink = p;
        p->rlink = q;
        q->llink = p;
        ring->length++;
        return p->value = x;
    }
}
```

位置语义说明:

```
head            位置 0
head->rlink     位置 1
head->rlink->rlink 位置 2
...
head->llink     位置 length-1

插入位置 pos 的含义:
  pos = 0 或 pos = length+1              -> 高端 (尾部) = addhi
  pos = 1 或 pos = -length               -> 低端 (头部) = addlo
  pos = k (1 < k <= length)              -> 插入后成为第 k 个元素 (索引 k)
  pos = -k (1 < k < length)              -> 插入后成为倒数第 k+1 个元素
```

负索引支持是 Ring 的一大特色，类似于 Python 的负索引语义:
- `pos = -1` 等价于 `pos = length-1`
- `pos = -length` 等价于 `pos = 0`

---

### 12.3.4 删除操作

#### `Ring_remove` -- 删除指定位置元素

```c
void *Ring_remove(T ring, int i) {
    void *x;
    struct node *q;
    assert(ring);
    assert(ring->length > 0);             // 环不能为空
    assert(i >= 0 && i < ring->length);
    {
        int n;
        q = ring->head;
        if (i <= ring->length / 2)        // 捷径定位
            for (n = i; n-- > 0; )
                q = q->rlink;
        else
            for (n = ring->length - i; n-- > 0; )
                q = q->llink;
    }
    if (i == 0)                            // 删除的是 head
        ring->head = ring->head->rlink;    // head 后移
    x = q->value;                          // 保存返回值
    q->llink->rlink = q->rlink;           // 将 q 的前驱的 rlink 指向 q 的后继
    q->rlink->llink = q->llink;           // 将 q 的后继的 llink 指向 q 的前驱
    FREE(q);                               // 释放被删除的节点
    if (--ring->length == 0)              // 如果环变空
        ring->head = NULL;                 // head 置空
    return x;
}
```

关键步骤分析:

1. **`q->llink->rlink = q->rlink`**: 使 q 的前驱节点"跳过" q，直接指向 q 的后继。
2. **`q->rlink->llink = q->llink`**: 使 q 的后继节点"跳过" q，直接指向 q 的前驱。
3. 两步完成后，q 被从环中摘除，但 q 的字段仍可能持有有效地址 (不过 FREE 后不可再访问)。

#### `Ring_remhi` -- 删除高端 (尾部) 元素

```c
void *Ring_remhi(T ring) {
    void *x;
    struct node *q;
    assert(ring);
    assert(ring->length > 0);
    q = ring->head->llink;               // 尾部节点 = head 的左邻
    x = q->value;
    q->llink->rlink = q->rlink;          // 断开 q 的前后连接
    q->rlink->llink = q->llink;
    FREE(q);
    if (--ring->length == 0)
        ring->head = NULL;
    return x;
}
```

`Ring_remhi` 是 O(1) 操作: 直接通过 `head->llink` 定位尾部节点，无需遍历。
这得益于环形双向链表的对称结构。

#### `Ring_remlo` -- 删除低端 (头部) 元素

```c
void *Ring_remlo(T ring) {
    assert(ring);
    assert(ring->length > 0);
    ring->head = ring->head->rlink;      // head 后移一位
    return Ring_remhi(ring);             // 此时原 head 成为尾节点，调用 remhi 删除
}
```

精妙的实现: 先将 `head` 后移，使原 `head` 变成尾节点，然后复用 `Ring_remhi` 完成删除。
这也是 O(1) 操作。

---

### 12.3.5 旋转操作

#### `Ring_rotate` -- 旋转环

```c
void Ring_rotate(T ring, int n) {
    struct node *q;
    int i;
    assert(ring);
    assert(n >= -ring->length && n <= ring->length);
    if (n >= 0)
        i = n % ring->length;             // 正向旋转步数
    else
        i = n + ring->length;             // 负向旋转转化为等价的正向步数
    {
        int n;
        q = ring->head;
        if (i <= ring->length / 2)        // 捷径定位新 head
            for (n = i; n-- > 0; )
                q = q->rlink;
        else
            for (n = ring->length - i; n-- > 0; )
                q = q->llink;
    }
    ring->head = q;                       // 更新 head
}
```

旋转操作只改变 `head` 指针，不移动任何节点数据。语义如下:

```
初始: head = A, 环 = A -> B -> C -> D

Ring_rotate(r, 1):  head = B, 环 = B -> C -> D -> A
Ring_rotate(r, -1): head = D, 环 = D -> A -> B -> C
Ring_rotate(r, 3):  head = D, 环 = D -> A -> B -> C
```

`n % ring->length` 处理了旋转超过一圈的情况。但对于负数 `n`，C 标准中 `%` 运算符的结果
可能是负的 (实现定义)，因此代码使用了显式的 `n + ring->length` 来处理负旋转。

---

## 12.4 环形指针运算技巧

### 12.4.1 双指针遍历取捷径

CII Ring 中最核心的优化思路: **利用双向链接，总是选择较短的遍历路径**。这个技巧
在 `Ring_get`、`Ring_put`、`Ring_add`、`Ring_remove` 和 `Ring_rotate` 中反复出现:

```c
q = ring->head;
if (i <= ring->length/2)
    for (n = i; n-- > 0; )
        q = q->rlink;           // 顺时针: i 步
else
    for (n = ring->length - i; n-- > 0; )
        q = q->llink;           // 逆时针: (length - i) 步
```

对长度为 N 的环:
- 最坏情况步数: `floor(N/2)`
- 平均步数: `N/4`
- 与单向链表相比: 单向链表总是走 i 步 (最坏 N-1 步)，平均 `N/2` 步。

### 12.4.2 自环单节点

当环中只有一个元素时:
```
p->llink == p &&
p->rlink == p
```
此时任何方向的遍历都会无限循环回到自身。在编写遍历代码时，必须用计数器而非比较指针
来控制循环终止:

```c
// 正确: 用计数控制
struct node *p = ring->head;
for (int n = 0; n < ring->length; n++, p = p->rlink) {
    // 处理 p->value
}

// 错误: 用指针比较控制 (单节点环会死循环)
struct node *p = ring->head;
do {
    // 处理 p->value
    p = p->rlink;
} while (p != ring->head);
```

### 12.4.3 HEAD 即索引原点

CII Ring 将 `head` 视为 "第 0 个元素"，所有索引计算以 `head` 为参考点。
`Ring_rotate` 通过改变 `head` 来重新定义索引原点，这使得环形结构
天然支持"旋转视图"的操作模式。

### 12.4.4 链接操作的原子性

环形链表中的插入和删除需要同时更新 4 个指针 (新节点的 2 个 + 相邻节点的 2 个)。
正确的操作顺序至关重要:

```
插入 p 到 q 之前 (p 将成为 q 的 llink):
  p->llink = q->llink;      // (1) 新节点指向左邻
  q->llink->rlink = p;      // (2) 左邻指向新节点
  p->rlink = q;              // (3) 新节点指向 q
  q->llink = p;              // (4) q 指向新节点

删除 q:
  q->llink->rlink = q->rlink;  // (1) 前驱跳过 q
  q->rlink->llink = q->llink;  // (2) 后继跳过 q
  FREE(q);                      // (3) 释放 q
```

---

## 12.5 满/空判断条件 (经典 Off-by-One 问题)

> 本节讨论的是**固定大小环形缓冲区** (array-based ring buffer) 的满/空问题，
> 而非 CII 的动态链表 Ring。CII Ring 通过 `length` 字段和 NULL 检查来判断空，
> 不涉及容量上限。

### 12.5.1 问题描述

基于数组的环形缓冲区使用两个索引 `head` (读指针) 和 `tail` (写指针) 来管理数据:

```
    +---+---+---+---+---+---+---+---+
    |   | D | E | F | G |   |   |   |
    +---+---+---+---+---+---+---+---+
          ^               ^
         head            tail

head: 下一个要读取的位置
tail: 下一个要写入的位置 (或最后一个已写入位置之后)
```

核心矛盾: 在容量为 N 的缓冲区中，`head` 和 `tail` 各有 N 种取值，组合有 `N * N` 种状态。
但是我们需要区分的状态数量是 `N + 1` (0 到 N 个元素)。无论如何设计，都有一个状态
是"模糊"的 -- 这就是经典的 off-by-one 问题。

### 12.5.2 三种常见方案

**方案 A: 牺牲一个槽位**

```
head == tail  -> 空
(tail + 1) % N == head -> 满

可用容量: N - 1
```

这是最常见的实现方式，也是 Linux 内核 `kfifo` 采用的方法。

```c
// 方案 A 的实现
#define BUF_SIZE 256  // 实际可用容量 = 255

int ring_buf_empty(int head, int tail) {
    return head == tail;
}

int ring_buf_full(int head, int tail) {
    return ((tail + 1) % BUF_SIZE) == head;
}
```

**方案 B: 使用独立计数器**

```c
// 维护单独的 count 变量
int count;

int ring_buf_empty(void) {
    return count == 0;
}

int ring_buf_full(void) {
    return count == BUF_SIZE;
}
```

优点: 直观清晰，可用全部 N 个槽位。
缺点: 需要一个额外的计数器变量，在并发环境下增加了一个竞争变量。

**方案 C: 使用特殊标志位**

```c
int overflow;  // 当 tail 追上 head 时置位

int ring_buf_empty(int head, int tail, int overflow) {
    return (head == tail) && !overflow;
}

int ring_buf_full(int head, int tail, int overflow) {
    return (head == tail) && overflow;
}
```

优点: 可用全部 N 个槽位。
缺点: 需要维护标志位，逻辑复杂，在并发环境中更难保证正确性。

### 12.5.3 环形缓冲区的标准实现

```c
#include <stdbool.h>

#define RING_CAPACITY 256   /* MUST be power of 2 for bitmask optimization */

typedef struct {
    void *buf[RING_CAPACITY];
    unsigned int head;      /* 读指针 */
    unsigned int tail;      /* 写指针 */
} RingBuffer;

/* 初始化 */
void rb_init(RingBuffer *rb) {
    rb->head = 0;
    rb->tail = 0;
    /* buf 内容不需要初始化 */
}

/* 满/空判断 */
static inline bool rb_empty(const RingBuffer *rb) {
    return rb->head == rb->tail;
}

static inline bool rb_full(const RingBuffer *rb) {
    return ((rb->tail + 1) & (RING_CAPACITY - 1)) == rb->head;
    /* 使用 & 而非 % 的前提: RING_CAPACITY 是 2 的幂 */
}

/* 写入 (生产者) -- 返回 false 表示缓冲区已满 */
bool rb_put(RingBuffer *rb, void *item) {
    if (rb_full(rb))
        return false;
    rb->buf[rb->tail] = item;
    rb->tail = (rb->tail + 1) & (RING_CAPACITY - 1);
    return true;
}

/* 读取 (消费者) -- 返回 NULL 表示缓冲区已空 */
void *rb_get(RingBuffer *rb) {
    void *item;
    if (rb_empty(rb))
        return NULL;
    item = rb->buf[rb->head];
    rb->head = (rb->head + 1) & (RING_CAPACITY - 1);
    return item;
}
```

### 12.5.4 关键实现细节

**1. 容量必须是 2 的幂**

当 `RING_CAPACITY` 为 2 的幂时:
```c
/* 这两种写法等价，但位运算更快 */
index = (index + 1) % RING_CAPACITY;        // 通用写法
index = (index + 1) & (RING_CAPACITY - 1);  // 2的幂优化
```

**2. `head` 和 `tail` 的溢出处理**

使用 `unsigned int` 类型，当 `head` 或 `tail` 递增超过 `UINT_MAX` 时，自动回绕 (wrap around)
是 C 标准定义的合法行为。只要容量是 2 的幂且索引计算使用位掩码，这个溢出是无害的。

**3. 可存储元素数量**

实际可存储元素数 = `RING_CAPACITY - 1` (保留一个槽位区分空/满)。

---

## 12.6 生产者-消费者模式应用

### 12.6.1 经典模式

环行缓冲区最常见的应用场景是**生产者-消费者模型**:

```
    Producer Thread              Consumer Thread
    ===============              ===============
    
    while (producing) {          while (consuming) {
        item = produce();            item = rb_get(&rb);
        rb_put(&rb, item);           consume(item);
    }                           }
```

在多线程环境中，最基本的实现需要加锁:

```c
#include <pthread.h>

typedef struct {
    void *buf[RING_CAPACITY];
    unsigned int head;
    unsigned int tail;
    pthread_mutex_t lock;
    pthread_cond_t  not_empty;
    pthread_cond_t  not_full;
} BlockingRing;

void blocking_put(BlockingRing *rb, void *item) {
    pthread_mutex_lock(&rb->lock);
    while (rb_full(rb))                           // 等待缓冲区非满
        pthread_cond_wait(&rb->not_full, &rb->lock);
    rb->buf[rb->tail] = item;
    rb->tail = (rb->tail + 1) & (RING_CAPACITY - 1);
    pthread_cond_signal(&rb->not_empty);           // 通知消费者
    pthread_mutex_unlock(&rb->lock);
}

void *blocking_get(BlockingRing *rb) {
    void *item;
    pthread_mutex_lock(&rb->lock);
    while (rb_empty(rb))                           // 等待缓冲区非空
        pthread_cond_wait(&rb->not_empty, &rb->lock);
    item = rb->buf[rb->head];
    rb->head = (rb->head + 1) & (RING_CAPACITY - 1);
    pthread_cond_signal(&rb->not_full);            // 通知生产者
    pthread_mutex_unlock(&rb->lock);
    return item;
}
```

### 12.6.2 单生产者单消费者 (SPSC) 优化

当只有一个生产者线程和一个消费者线程时，可以实现更高效的同步:

```c
/* SPSC: 不需要锁，只需要适当的内存屏障 */
bool spsc_put(RingBuffer *rb, void *item) {
    unsigned int next_tail = (rb->tail + 1) & (RING_CAPACITY - 1);
    if (next_tail == rb->head)   // 本地检查，无需锁
        return false;            // 满
    rb->buf[rb->tail] = item;
    /* 内存屏障: 确保数据写入对消费者可见 */
    atomic_thread_fence(memory_order_release);
    rb->tail = next_tail;
    return true;
}
```

原因: 生产者只写 `tail`，消费者只写 `head`，两者不互相写入对方的变量，因此不存在数据竞争。

---

## 12.7 Modern-C 改进: 无锁 Ring Buffer (C11 原子操作)

### 12.7.1 设计原理

C11 标准引入了 `<stdatomic.h>`，为无锁数据结构提供了语言级支持。
无锁环形缓冲区的核心思想:

- 生产者和消费者各自拥有独立的"游标" (cursor)
- 使用 `memory_order_acquire` / `memory_order_release` 保证 happens-before 关系
- 没有互斥锁，没有上下文切换开销

### 12.7.2 完整实现

```c
#include <stdatomic.h>
#include <stdbool.h>
#include <stdlib.h>

#define LFQ_CAPACITY 256  /* MUST be power of 2 */
#define LFQ_MASK     (LFQ_CAPACITY - 1)

typedef struct {
    _Atomic unsigned int head;        /* 消费者游标 */
    _Atomic unsigned int tail;        /* 生产者游标 */
    void *buf[LFQ_CAPACITY];
} LockFreeQueue;

LockFreeQueue *lfq_create(void) {
    LockFreeQueue *q = calloc(1, sizeof(*q));
    return q;
}

void lfq_destroy(LockFreeQueue *q) {
    free(q);
}

/* 生产者: 单生产者安全 */
bool lfq_enqueue(LockFreeQueue *q, void *item) {
    unsigned int tail = atomic_load_explicit(&q->tail, memory_order_relaxed);
    unsigned int next_tail = (tail + 1) & LFQ_MASK;

    /* 检查是否已满 */
    unsigned int head = atomic_load_explicit(&q->head, memory_order_acquire);
    if (next_tail == head)
        return false;  /* full */

    q->buf[tail] = item;

    /* release: 保证 item 写入在 tail 更新之前对消费者可见 */
    atomic_store_explicit(&q->tail, next_tail, memory_order_release);
    return true;
}

/* 消费者: 单消费者安全 */
void *lfq_dequeue(LockFreeQueue *q) {
    unsigned int head = atomic_load_explicit(&q->head, memory_order_relaxed);

    /* 检查是否已空 */
    unsigned int tail = atomic_load_explicit(&q->tail, memory_order_acquire);
    if (head == tail)
        return NULL;  /* empty */

    void *item = q->buf[head];
    unsigned int next_head = (head + 1) & LFQ_MASK;

    /* release: 保证 item 读取在 head 更新之前完成 */
    atomic_store_explicit(&q->head, next_head, memory_order_release);
    return item;
}
```

### 12.7.3 内存序选择分析

| 操作 | 内存序 | 原因 |
|------|--------|------|
| `head` 读取 (生产者) | `acquire` | 确保看到消费者已消费的槽位 |
| `tail` 读取 (消费者) | `acquire` | 确保看到生产者已写入的数据 |
| `tail` 写入 (生产者) | `release` | 确保数据写入在 tail 更新之前 |
| `head` 写入 (消费者) | `release` | 确保数据读取在 head 更新之前 |
| 本地变量读取 | `relaxed` | 自身写入的值，不需要同步 |

### 12.7.4 多生产者 / 多消费者 (MPMC) 扩展

对于真正的 MPMC 场景，需要使用 `atomic_compare_exchange_strong` (CAS) 来原子地
"认领"槽位:

```c
/* MPMC: 使用 CAS 竞争槽位 */
bool mpmc_enqueue(LockFreeQueue *q, void *item) {
    unsigned int tail, next_tail;
    do {
        tail = atomic_load_explicit(&q->tail, memory_order_relaxed);
        next_tail = (tail + 1) & LFQ_MASK;
        unsigned int head = atomic_load_explicit(&q->head, memory_order_acquire);
        if (next_tail == head)
            return false;  /* full */
    } while (!atomic_compare_exchange_weak_explicit(
        &q->tail, &tail, next_tail,
        memory_order_release, memory_order_relaxed));
    /* 成功认领槽位 tail */
    q->buf[tail] = item;
    return true;
}
```

CAS 循环 (又称 "自旋") 保证了多生产者之间互不覆盖，每个生产者原子地推进 `tail`。

---

## 12.8 与 Linux 内核 `kfifo` 对比

Linux 内核中的 `kfifo` (kernel FIFO) 是环形缓冲区在工业级 C 代码中的典范实现。
以下是 CII Ring 与内核 `kfifo` 的全面对比:

### 12.8.1 设计哲学对比

| 维度 | CII Ring | Linux kfifo |
|------|----------|-------------|
| 设计目标 | 通用 ADT，教学与库使用 | 内核数据路径，极致性能 |
| 底层结构 | 动态分配双向链表 | 固定大小数组 |
| 容量 | 无硬性上限 | 预分配，固定 |
| 内存分配 | 逐节点 `malloc`/`free` | 一次分配，永不释放 |
| 并发安全 | 不内置 (需外部锁) | 内置无锁 SPSC 支持 |
| 元素类型 | `void *` (指针) | 原始字节流 |
| 索引计算 | 指针追踪 | `& (size - 1)` 位掩码 |
| 适用场景 | 通用应用层双端队列 | 内核驱动、高速数据路径 |

### 12.8.2 内存模型对比

**CII Ring** (动态链表):
```
每个节点: 3 个指针 (llink, rlink, value) = 24 bytes (64-bit)
每个节点独立 malloc: 可能的内存碎片化
10000 个元素: ~240KB + malloc 开销
```

**Linux kfifo** (数组):
```
预分配: sizeof(element) * capacity 连续内存
无 malloc 开销，缓存友好
但是: 只能存储值类型或固定大小结构体，不能存储 void* 泛型
```

### 12.8.3 kfifo 的关键实现技巧

Linux 内核 `kfifo` 的核心数据结构 (简化版):

```c
struct kfifo {
    unsigned char *buffer;     /* 缓冲区起始地址 */
    unsigned int   size;       /* 缓冲区大小 (2 的幂) */
    unsigned int   in;         /* 写指针 (生产者) */
    unsigned int   out;        /* 读指针 (消费者) */
};
```

关键技巧:

**1. 巧妙的长度计算**

```c
/* 已使用字节数 */
#define kfifo_len(fifo) ((fifo)->in - (fifo)->out)
```

利用 `unsigned int` 的回绕特性，无需任何条件判断或取模。例如:
- `in = 5, out = 2` -> `len = 3`
- `in = 2, out = UINT_MAX - 2` (回绕后) -> `len = 2 - (UINT_MAX-2)` = `5` (无符号数减法)

**2. 无分支的满/空判断**

```c
/* 空 */
#define kfifo_is_empty(fifo) ((fifo)->in == (fifo)->out)

/* 可用空间 (不必牺牲槽位) */
#define kfifo_avail(fifo) ((fifo)->size - kfifo_len(fifo))
```

**3. 批量操作**

kfifo 支持批量入队/出队，使用 `min()` 计算单次最大连续复制长度:

```c
unsigned int kfifo_in(struct kfifo *fifo, const void *from, unsigned int len) {
    len = min(len, kfifo_avail(fifo));
    unsigned int l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
    /* 第一部分: 从 in 到缓冲区末尾 */
    memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), from, l);
    /* 第二部分: 从缓冲区开头 (如果回绕) */
    memcpy(fifo->buffer, from + l, len - l);
    fifo->in += len;
    return len;
}
```

### 12.8.4 适用场景总结

```
场景决策树:

需要随机访问任意位置?
  YES -> CII Ring (Ring_get/Ring_put 支持索引访问)
  NO  -> 继续

需要动态增减容量?
  YES -> CII Ring (链表天然支持)
  NO  -> 继续

需要极高吞吐、低延迟?
  YES -> Linux kfifo (缓存友好，无锁)
  NO  -> CII Ring

元素是变长或需要频繁插入/删除?
  YES -> CII Ring (链表 O(1) 插入/删除)
  NO  -> Linux kfifo
```

---

## 12.9 实现决策与权衡

### 12.9.1 为什么 CII 选择链表而非数组?

Hanson 在设计 CII Ring 时选择了动态双向链表，这个选择背后的考量:

1. **通用性**: `void *` 指针可以存储任意类型，而数组需要预知元素大小。
2. **无容量限制**: 链表天然不受预分配容量的约束，适合元素数量不可预测的场景。
3. **插入/删除友好**: O(1) 的任意位置插入和删除，这是数组环形缓冲区无法做到的。
4. **教学价值**: 环形双向链表是一种基础而重要的数据结构，其实现清晰地展示了
   指针操作的各项技巧。

### 12.9.2 为什么生产环境中更多使用数组环形缓冲区?

1. **缓存局部性**: 连续内存访问在 CPU cache 中的命中率远高于链表跳转。
2. **内存效率**: 每个 `void *` 元素只需 8 字节 (64-bit)，而链表节点需要 24 字节 + 分配器开销。
3. **可预测性**: 固定大小意味着没有 `malloc` 失败的风险，适合嵌入式/内核场景。
4. **无锁友好**: 数组上的原子操作远比链表上的指针 CAS 简单可靠。

---

## 12.10 完整示例: 日志环形缓冲区

以下示例结合了两者的优点: 使用固定大小数组实现环形缓冲区，
对外提供类似 CII Ring 的接口风格。

```c
/* log_ring.h */
#ifndef LOG_RING_INCLUDED
#define LOG_RING_INCLUDED

typedef struct LogRing_T *LogRing_T;

extern LogRing_T LogRing_new(int capacity);
extern void      LogRing_free(LogRing_T *ring);
extern int       LogRing_length(LogRing_T ring);
extern int       LogRing_capacity(LogRing_T ring);
extern int       LogRing_put(LogRing_T ring, const char *msg);
extern char     *LogRing_get(LogRing_T ring, int index);
extern int       LogRing_rotate(LogRing_T ring, int n);

#endif
```

```c
/* log_ring.c */
#include <stdlib.h>
#include <string.h>
#include "log_ring.h"

struct LogRing_T {
    char   **buf;
    int      capacity;     /* 必须是 2 的幂 */
    int      mask;
    int      head;         /* 逻辑 0 号位置 */
    int      length;       /* 当前元素数 */
};

LogRing_T LogRing_new(int capacity) {
    /* 向上取整到 2 的幂 */
    int cap = 1;
    while (cap < capacity) cap <<= 1;

    LogRing_T ring = malloc(sizeof(*ring));
    ring->buf      = calloc(cap, sizeof(char *));
    ring->capacity = cap;
    ring->mask     = cap - 1;
    ring->head     = 0;
    ring->length   = 0;
    return ring;
}

void LogRing_free(LogRing_T *ring) {
    if (ring && *ring) {
        /* 释放所有存活的字符串 */
        for (int i = 0; i < (*ring)->length; i++) {
            int idx = ((*ring)->head + i) & (*ring)->mask;
            free((*ring)->buf[idx]);
        }
        free((*ring)->buf);
        free(*ring);
        *ring = NULL;
    }
}

int LogRing_length(LogRing_T ring) {
    return ring->length;
}

int LogRing_capacity(LogRing_T ring) {
    return ring->capacity - 1;  /* 有效容量 = 总容量 - 1 */
}

/* 生产者: 写入新日志，满了则丢弃最旧的 */
int LogRing_put(LogRing_T ring, const char *msg) {
    if (ring->length == ring->capacity - 1) {
        /* 满: 丢弃最旧的元素 (覆盖) */
        int old_idx = ring->head;
        free(ring->buf[old_idx]);
        ring->head = (ring->head + 1) & ring->mask;
        ring->length--;
    }
    int idx = (ring->head + ring->length) & ring->mask;
    ring->buf[idx] = strdup(msg);
    ring->length++;
    return idx;
}

/* 随机访问: 类似 Ring_get 的接口 */
char *LogRing_get(LogRing_T ring, int index) {
    if (index < 0 || index >= ring->length)
        return NULL;
    int idx = (ring->head + index) & ring->mask;
    return ring->buf[idx];
}

/* 旋转: 类似 Ring_rotate, 改变逻辑起点 */
int LogRing_rotate(LogRing_T ring, int n) {
    if (ring->length == 0) return 0;
    n = n % ring->length;
    if (n < 0) n += ring->length;
    ring->head = (ring->head + n) & ring->mask;
    return n;
}
```

这个实现融合了 CII Ring 的 API 风格 (`Ring_get`、`Ring_rotate`) 和数组环形缓冲区的
性能优势 (O(1) 索引、无 per-element malloc、缓存友好)。

---

## 12.11 延伸阅读

- **CII 链表 (第11章)**: 单向链表 `List_T` 与环形链表 `Ring_T` 的设计对比。
- **Linux kfifo 源码**: `include/linux/kfifo.h` 和 `lib/kfifo.c`，环形缓冲区工业级实现的标杆。
- **Dmitry Vyukov 的无锁队列**: [Single-Producer/Single-Consumer Queue](http://www.1024cores.net/home/lock-free-algorithms/queues)，无锁数据结构的经典参考资料。
- **C11 原子操作**: ISO/IEC 9899:2011 Section 7.17 `<stdatomic.h>`。
- **Parnas (1972)**: "On the Criteria To Be Used in Decomposing Systems into Modules" -- ADT 设计理念的奠基性论文。

---

## 12.12 核心要点回顾

1. **CII Ring 是环形双向链表**，核心数据结构为 `{llink, rlink, value} *head` + `length`。
2. **捷径遍历**是最关键的优化: 总是选择顺时针或逆时针中更短的路径，最坏情况下仅需 `N/2` 步。
3. **旋转操作**只改变 `head` 指针，不移动数据，O(N) 的时间复杂度来自定位新 `head`。
4. **`addlo`/`remlo` 复用 `addhi`/`remhi`** 是代码复用的典范: `addlo = addhi + head回退`, `remlo = head前移 + remhi`。
5. **固定大小环形缓冲区**使用 `head == tail` 判空，`(tail+1) % N == head` 判满 -- 牺牲一个槽位解决 off-by-one 歧义。
6. **无锁 SPSC 环形缓冲区**只需 `acquire`/`release` 内存序，无需 CAS，也无需互斥锁。
7. **Linux kfifo** 通过 `unsigned int` 自然溢出和位掩码索引实现了零分支的极致性能。

---

> **译后记**: CII 的 Ring 实现是一份优雅的教学代码。Hanson 通过不到 200 行的 C 代码，
> 完整展示了环形双向链表的设计、实现与优化思路。在现代工程实践中，大多数场景下的
> "ring" 指的是固定大小数组环形缓冲区，但理解 CII Ring 的设计仍然具有重要价值:
> 它教会我们如何思考指针操作的正确性、对称性，以及如何在通用性和性能之间做出
> 有意识的权衡。
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch11_序列_Seq.html">第11章 序列 Seq</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch13_位向量_Bit.html">第13章 位向量 Bit</a></div>
</div>
