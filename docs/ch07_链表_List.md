# 第7章: 链表 (List)

> 原文: C Interfaces and Implementations, David R. Hanson, Chapter 7
> 源文件: `code/list.h`, `code/list.c`

## 一、设计哲学与接口概述

### 1.1 最简单的链表实现

CII的List是一种极其精简的**单向链表**, 每个节点只有两个字段:

```c
#define T List_T
typedef struct T *T;
struct T {
    T rest;         // 指向下一个节点 (Lisp风格的cdr)
    void *first;    // 节点数据 (Lisp风格的car)
};
```

**设计特点:**
- 单向, 非循环
- `void *first` 实现泛型 (可存储任意指针)
- `rest` 命名来自Lisp/Scheme的cdr传统
- 无独立的头节点 — 空链表就是NULL
- 不可变风格: 大部分操作返回新链表, 不修改原链表

### 1.2 API一览

```c
T      List_push   (T list, void *x);     // 头插 O(1)
T      List_pop    (T list, void **x);    // 头删 O(1)
T      List_append (T list, T tail);     // 尾部拼接 O(n)
T      List_copy   (T list);             // 复制 O(n)
T      List_reverse(T list);             // 反转 O(n)
T      List_list   (void *x, ...);       // 从变参构建 O(n)
int    List_length (T list);             // 长度 O(n)
void   List_free   (T *list);            // 释放 O(n)
void   List_map    (T list, void apply(void **x, void *cl), void *cl);
void **List_toArray(T list, void *end);  // 转数组 O(n)
```

---

## 二、数据结构

### 2.1 内存布局

```
空链表: list = NULL

非空链表:
list ──→ ┌──────────┐    ┌──────────┐    ┌──────────┐
         │ first: ●─┼──→│ "data1"  │    │ "data2"  │    │ "data3"  │
         │ rest:  ●─┼──→│          │    │          │    │          │
         └──────────┘    └──────────┘    └──────────┘    └──────────┘
              ↑               ↑               ↑               ↑
           节点1           节点2           节点3           NULL
```

每个节点8+8=16字节 (64位系统), 数据通过void*间接引用。

---

## 三、原始CII代码逐行中文注释

### 3.1 List_push — 前置插入

```c
T List_push(T list, void *x) {
    T p;
    NEW(p);            // 分配新节点
    p->first = x;      // 数据放在first
    p->rest  = list;   // 新节点的rest指向原链表头
    return p;          // 返回新链表头 (新节点)
}
// 用法: list = List_push(list, data);
// 注意返回值必须重新赋值! 因为头部变了
```

### 3.2 List_pop — 头部删除

```c
T List_pop(T list, void **x) {
    if (list) {                   // 非空链表
        T head = list->rest;      // 保存新的链表头
        if (x)
            *x = list->first;     // 通过**x返回被删除的数据
        FREE(list);               // 释放节点内存
        return head;              // 返回新的链表头
    } else
        return list;              // 空链表: 原样返回NULL
}
// 用法: list = List_pop(list, &popped_data);
```

### 3.3 List_list — 可变参数构造

```c
T List_list(void *x, ...) {
    va_list ap;
    T list, *p = &list;           // p是指向"下一个节点的rest字段"的指针

    va_start(ap, x);
    for ( ; x; x = va_arg(ap, void *)) {
        NEW(*p);                  // *p = 新节点 (通过rest指针间接赋值)
        (*p)->first = x;          // 设置数据
        p = &(*p)->rest;          // p指向新节点的rest字段
    }
    *p = NULL;                    // 最后一个节点的rest = NULL
    va_end(ap);
    return list;
}
// 用法: T names = List_list("Alice", "Bob", "Charlie", NULL);
//                                         最后一个必须为NULL!
```

**关键技巧**: `T *p = &list` — p指向的是"链表的根指针"或"前一节点的rest字段", 从而用统一的代码处理第一个节点和后续节点。

### 3.4 List_append — 尾部追加

```c
T List_append(T list, T tail) {
    T *p = &list;               // 从链表根指针开始

    while (*p)                  // 遍历到链表末尾 (*p == NULL)
        p = &(*p)->rest;        // ↓
                                // 第一次: p = &list
                                // 循环后: p指向最后一个节点的rest (即NULL的位置)

    *p = tail;                  // 将最后一个节点的rest指向tail
    return list;
}
// 用法: list = List_append(list, List_list("d", "e", NULL));
// O(n) — 需要遍历到尾部, 无尾指针缓存
```

### 3.5 List_copy — 深拷贝

```c
T List_copy(T list) {
    T head, *p = &head;

    for ( ; list; list = list->rest) {
        NEW(*p);                      // 分配新节点
        (*p)->first = list->first;    // 复制数据指针 (浅拷贝数据!)
        p = &(*p)->rest;              // 移动到新节点的rest
    }
    *p = NULL;
    return head;
}
// 注意: 只复制节点, 不复制first指向的数据 (浅拷贝)
```

### 3.6 List_reverse — 经典反转

```c
T List_reverse(T list) {
    T head = NULL, next;

    for ( ; list; list = next) {
        next = list->rest;      // 保存下一个节点
        list->rest = head;      // 当前节点指向前一个 (反转!)
        head = list;            // head前进到当前节点
    }
    return head;
}
// 原地反转, O(n)时间, O(1)额外空间
```

反转过程图示:

```
初始:  A→B→C→NULL    head=NULL
Step1: B→C→NULL       A→NULL        head=A
Step2: C→NULL          B→A→NULL      head=B
Step3: NULL             C→B→A→NULL   head=C
```

### 3.7 List_map — 高阶函数

```c
void List_map(T list, void apply(void **x, void *cl), void *cl) {
    assert(apply);
    for ( ; list; list = list->rest)
        apply(&list->first, cl);     // 传入first的指针! 可以修改元素
}
// apply的第一个参数是void **, 允许修改节点数据
// cl (closure) 用于传递额外上下文
```

使用示例:

```c
void print_element(void **x, void *cl) {
    printf("%s ", (char *)*x);
}

void toupper_element(void **x, void *cl) {
    char *s = (char *)*x;
    for (int i = 0; s[i]; i++)
        s[i] = toupper(s[i]);
}

List_map(names, print_element, NULL);   // 打印所有名字
List_map(names, toupper_element, NULL); // 转为大写
```

### 3.8 List_toArray — 转为数组

```c
void **List_toArray(T list, void *end) {
    int i, n = List_length(list);
    void **array = ALLOC((n + 1)*sizeof (*array)); // n+1: 留一个位置给哨兵

    for (i = 0; i < n; i++) {
        array[i] = list->first;
        list = list->rest;
    }
    array[i] = end;      // 最后一个元素作为哨兵 (类似argv的NULL结尾)
    return array;
}
// 返回的数组需手动FREE
```

### 3.9 List_free — 释放

```c
void List_free(T *list) {
    T next;
    assert(list);
    for ( ; *list; *list = next) {
        next = (*list)->rest;  // 保存下一个节点
        FREE(*list);           // 释放当前节点 (FREE宏同时置NULL)
    }
    // 循环结束后 *list == NULL (链表根指针被清零)
}
// 参数是T* (二级指针), 所以可以修改外部的链表变量
// 用法: List_free(&list); // list变量在调用后变为NULL
```

---

## 四、Modern-C 改进方案

### 4.1 侵入式链表 (Linux内核风格)

```c
// 侵入式: 链表节点嵌入在数据结构中
struct list_node {
    struct list_node *next, *prev;
};

struct Student {
    int id;
    char *name;
    struct list_node node;  // 嵌入的链表节点
};

// 优点: 零额外分配, 缓存友好
// 缺点: 一个对象只能在一个链表中
```

### 4.2 编译时泛型 (C11 _Generic)

```c
#define List_of(T) struct { List_of(T) *rest; T first; }

// 使用:
List_of(int) *intlist = NULL;
List_of(double) *dlist = NULL;
// intlist->first 类型是int, 无需void*转换!
```

### 4.3 现代替代

| CII List | Modern替代 |
|----------|-----------|
| `void *first` | C11 `_Generic` 宏或C++ template |
| 单向链表 | ST (Single Tail) list (带尾指针) |
| `List_length O(n)` | 缓存长度在结构体中 O(1) |
| 手动FREE | RAII (C++) 或 defer (Zig/Go) |

---

## 五、完整使用示例

```c
#include "list.h"
#include "mem.h"
#include <stdio.h>

// List_map的回调: 打印整数
void print_int(void **x, void *cl) {
    printf("%d ", *(int *)*x);
}

// List_map的回调: 翻倍
void double_it(void **x, void *cl) {
    *(int *)*x *= 2;
}

int main() {
    // 构建链表 [1, 2, 3, 4, 5]
    int *data;
    T list = NULL;
    for (int i = 5; i >= 1; i--) {
        NEW(data);
        *data = i;
        list = List_push(list, data);
    }

    printf("Original: ");  // 1 2 3 4 5
    List_map(list, print_int, NULL);
    printf("\n");

    printf("Length: %d\n", List_length(list));  // 5

    // 翻倍所有元素
    List_map(list, double_it, NULL);
    printf("Doubled:  ");  // 2 4 6 8 10
    List_map(list, print_int, NULL);
    printf("\n");

    // 反转
    list = List_reverse(list);
    printf("Reversed: ");  // 10 8 6 4 2
    List_map(list, print_int, NULL);
    printf("\n");

    // 弹出头部
    void *popped;
    list = List_pop(list, &popped);
    printf("Popped: %d\n", *(int *)popped);  // 10

    // 转为数组
    void **arr = List_toArray(list, NULL);
    for (int i = 0; arr[i]; i++)
        printf("arr[%d] = %d\n", i, *(int *)arr[i]);
    FREE(arr);

    // 释放链表
    List_free(&list);  // list现在为NULL
    printf("list after free: %p\n", list);  // (nil)

    return 0;
}
```
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch06_内存管理II_Arena.html">第6章 内存管理 II Arena</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch08_表_Table.html">第8章 表 Table</a></div>
</div>
