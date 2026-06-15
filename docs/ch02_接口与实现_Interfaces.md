# 第2章 接口与实现 (Interfaces and Implementations)

> 译自 David R. Hanson, *C Interfaces and Implementations: Techniques for Creating Reusable Software*, Chapter 2.

---

## 2.1 模块化与分离原则

一个大型程序若要被正确构造，必须由许多小的、可复用的模块组装而成。在C语言中，模块的组织方式遵循一个简单而深刻的原则：

> **接口 (Interface)** 定义模块*做什么* --- 即它导出的类型、函数和常量。接口放在 `.h` 头文件中。
>
> **实现 (Implementation)** 定义模块*怎么做* --- 即接口中声明的所有函数的具体代码。实现放在 `.c` 源文件中。

客户端代码只 `#include` 接口头文件，使用声明的函数，而绝对不应当依赖实现的内部细节。这就是**信息隐藏** (information hiding) 的核心。

### 2.1.1 接口文件的结构模板

本代码库中每一个接口头文件都遵循统一的模板：

```c
/* $Id$ */
#ifndef <MODULE>_INCLUDED    /* 防止重复包含的哨兵宏 */
#define <MODULE>_INCLUDED

/* 必要的系统头文件 */
#include <...>

/* 类型定义 */
#define T <Module>_T
typedef struct T *T;          /* 不透明指针 */

/* 异常声明 (可选) */
extern const Except_T <Module>_Failed;

/* 函数声明 */
extern T  <Module>_new(...);
extern void <Module>_free(T *p);
/* ... 其他操作 ... */

#undef T
#endif
```

**具体实例 --- `code/stack.h`**（全书最简单的完整接口）：

```c
#ifndef STACK_INCLUDED
#define STACK_INCLUDED
#define T Stack_T
typedef struct T *T;
extern T     Stack_new  (void);
extern int   Stack_empty(T stk);
extern void  Stack_push (T stk, void *x);
extern void *Stack_pop  (T stk);
extern void  Stack_free (T *stk);
#undef T
#endif
```

客户端只需 `#include "stack.h"`，然后即可创建和使用栈，完全不需要知道 `struct Stack_T` 的内部结构。

### 2.1.2 实现文件的结构模板

实现文件 `.c` 包含头文件中所有声明的函数体，以及接口使用者永远不应看到的内部细节：

```c
static char rcsid[] = "$Id$";
#include "assert.h"
#include "mem.h"
#include "stack.h"
#define T Stack_T

/* 结构体定义 —— 只在 .c 文件中可见 */
struct T {
    int count;
    struct elem {
        void *x;
        struct elem *link;
    } *head;
};

/* 函数实现 */
T Stack_new(void) { /* ... */ }
int Stack_empty(T stk) { /* ... */ }
/* ... */
```

关键观察：
- `struct T` 的完整定义仅在 `code/stack.c` 中出现，在 `code/stack.h` 中仅以 `typedef struct T *T;` 声明为不透明指针。
- `static` 用于内部数据（如 `code/atom.c` 中的 `buckets[]` 哈希表、`code/bit.c` 中的 `msbmask[]` 和 `lsbmask[]`），使其对其他编译单元不可见。

---

## 2.2 不透明指针 (Opaque Pointer) 技术

不透明指针是本库中实现信息隐藏的核心技术。其思想是：

1. 在 `.h` 头文件中，将类型定义为**指向不完整结构体的指针**：
   ```c
   typedef struct T *T;
   ```
   此时 `struct T` 仅被声明，未被定义 --- 编译器知道 `T` 是一个指针类型，但不知道它指向什么。

2. 在 `.c` 实现文件中，提供结构体的完整定义：
   ```c
   struct T {
       int count;
       struct elem *head;
   };
   ```

3. 客户端代码只能通过指针操作该类型的对象，无法直接访问其成员，也无法在栈上分配该类型的对象（因为编译器不知道其大小）。

### 2.2.1 不透明指针的三种变体

本书的代码库中展示了三种不同的不透明程度：

**完全透明 --- `code/list.h`**

```c
#define T List_T
typedef struct T *T;
struct T {
    T rest;
    void *first;
};
```

`struct T` 在头文件中**完全暴露**。这是一个有意为之的设计选择：链表节点极简（仅两个字段），且某些高性能代码可能希望直接遍历节点。这是书中少数透明接口之一。

**完全不透明 --- `code/stack.h`、`code/table.h`、`code/bit.h`、`code/set.h`、`code/seq.h`、`code/ring.h`、`code/ap.h`**

```c
#define T Stack_T
typedef struct T *T;
/* struct T 的定义仅在 .c 文件中 */
```

客户端完全无法看到内部结构。这是最常见的形式。例如：
- `code/stack.c` 的 `struct T` 包含 `count` 和链表头指针
- `code/bit.c` 的 `struct T` 包含 `length`、`bytes`、`words` 三个字段
- `code/arena.c` 的 `struct T` 包含 `prev`、`avail`、`limit` 三个指针

**无类型 --- `code/atom.h`**

```c
extern const char *Atom_new   (const char *str, int len);
```

Atom 模块完全不导出任何自定义类型。它返回的是 `const char *`，即普通的C字符串。其内部哈希表 (`static struct atom *buckets[2048]`) 被完全隐藏在 `code/atom.c` 内部，连类型名都不暴露。

### 2.2.2 不透明指针的编译期约束

由于客户端不知道不透明结构体的大小，以下操作在客户端代码中**编译失败**：

```c
Stack_T stk;           /* OK: 只是指针 */
Stack_T stk = malloc(sizeof(struct Stack_T));  /* 错误: 不知道大小 */
stk->count = 0;        /* 错误: 不知道成员 */
Stack_T stack_array[10]; /* 错误: 不知道元素大小 */
```

这强制客户端必须通过接口函数来创建和操作对象，保证了封装性。

---

## 2.3 接口命名约定

David R. Hanson 建立了一套严格且一致的命名约定，贯穿全书所有的接口。

### 2.3.1 T 宏模式

每一个接口头文件都定义了宏 `T` 作为模块类型名的简写：

```c
#define T Stack_T
typedef struct T *T;
/* ... 所有函数声明使用 T ... */
#undef T
```

同样，每个 `.c` 实现文件也重新定义 `T`：

```c
#define T Stack_T
struct T { /* ... */ };
/* ... 所有函数实现使用 T ... */
```

**设计意图**：
- 减少视觉噪音：用 `T` 代替冗长的 `Stack_T`，使函数签名更紧凑。
- 保持一致性：无论模块如何命名，实现代码中始终用 `T` 指代当前模块的类型。
- `#undef T` 确保宏不会污染后续代码（在 include 链的最后清理）。

**实例对比** --- `code/table.h` 中如果不使用 T 宏，声明将变为：

```c
/* 无 T 宏的冗长版本 */
extern Table_T Table_new(int hint, 
    int cmp(const void *x, const void *y),
    unsigned hash(const void *key));
extern void Table_free(Table_T *table);
```

使用 T 宏后：

```c
/* 有 T 宏的简洁版本 */
extern T    Table_new(int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *key));
extern void Table_free(T *table);
```

### 2.3.2 构造函数：`_new` 后缀

所有创建新对象的函数均以 `_new` 结尾：

| 函数 | 来源 | 说明 |
|------|------|------|
| `Stack_new(void)` | `code/stack.h` | 创建空栈 |
| `Table_new(int hint, cmp, hash)` | `code/table.h` | 创建哈希表，`hint` 为预期大小 |
| `Bit_new(int length)` | `code/bit.h` | 创建指定位长的位向量 |
| `Set_new(int hint, cmp, hash)` | `code/set.h` | 创建集合 |
| `Seq_new(int hint)` | `code/seq.h` | 创建动态序列 |
| `Ring_new(void)` | `code/ring.h` | 创建空环 |
| `Arena_new(void)` | `code/arena.h` | 创建内存池 |
| `AP_new(long int n)` | `code/ap.h` | 创建任意精度整数 |

**构造函数内部的统一模式**（以 `code/stack.c` 为例）：

```c
T Stack_new(void) {
    T stk;
    NEW(stk);           /* 通过 mem.h 的 NEW 宏分配内存 */
    stk->count = 0;     /* 初始化字段 */
    stk->head = NULL;
    return stk;         /* 返回不透明指针 */
}
```

`NEW(p)` 宏展开为 `((p) = ALLOC((long)sizeof *(p)))`，后者调用 `Mem_alloc` —— Hanson 自己封装的内存分配器，失败时抛出异常而非返回 NULL。

**特殊构造函数**：
- `List_list(void *x, ...)` --- 从变长参数列表构造链表（`code/list.c` 第15-27行），因为链表结构体定义暴露在头文件中，所以可以用这种不同寻常的方式
- `Atom_new(const char *str, int len)` --- 返回共享的不可变字符串（interned string），而非新分配的类型实例
- `AP_fromstr(const char *str, int base, char **end)` --- 从字符串解析构造

### 2.3.3 析构函数：`_free` 后缀

所有释放对象的函数均以 `_free` 结尾，并接受**指针的指针**：

| 函数 | 来源 |
|------|------|
| `Stack_free(T *stk)` | `code/stack.h` |
| `Table_free(T *table)` | `code/table.h` |
| `Bit_free(T *set)` | `code/bit.h` |
| `Set_free(T *set)` | `code/set.h` |
| `Seq_free(T *seq)` | `code/seq.h` |
| `List_free(T *list)` | `code/list.h` |
| `Ring_free(T *ring)` | `code/ring.h` |
| `AP_free(T *z)` | `code/ap.h` |

**`T *` 参数的意义**：通过接受指针的指针，`_free` 函数可以在释放内存后**将调用者的指针置为 NULL**，防止悬空指针（dangling pointer）。这是 C 语言中防御性编程的经典技巧。

以 `code/stack.c` 为例：

```c
void Stack_free(T *stk) {
    struct elem *t, *u;
    assert(stk && *stk);         /* 确保指针非空 */
    for (t = (*stk)->head; t; t = u) {
        u = t->link;
        FREE(t);                 /* 释放每个节点 */
    }
    FREE(*stk);                  /* 释放栈结构体本身 */
    /* 调用者的指针在 FREE 宏中被置为 NULL */
}
```

这里的 `FREE(ptr)` 宏（定义在 `code/mem.h` 中）展开为：

```c
#define FREE(ptr) ((void)(Mem_free((ptr), __FILE__, __LINE__), (ptr) = 0))
```

注意逗号表达式：先调用 `Mem_free`，然后将 `ptr` 置为 0（NULL）。因此 `FREE(*stk)` 不仅释放了 `*stk` 指向的内存，还将 `*stk`（即调用者的变量）设为 NULL。

**特别注意** --- `Arena_dispose(T *ap)` 是唯一的例外，它使用了不同的命名，因为 arena 的语义不是“释放”而是“处置”整个内存池。

### 2.3.4 操作函数命名

操作函数遵循 `Module_verb` 或 `Module_noun` 的格式：

| 函数 | 模块 | 模式 |
|------|------|------|
| `Stack_push` / `Stack_pop` | stack | `Module_verb` |
| `Table_put` / `Table_get` | table | `Module_verb` |
| `Seq_addlo` / `Seq_addhi` | seq | `Module_verb+dir` |
| `Bit_union` / `Bit_inter` / `Bit_diff` | bit | `Module_noun` |
| `List_reverse` / `List_copy` | list | `Module_verb` |
| `Atom_length` / `Atom_string` | atom | `Module_noun` |
| `Ring_rotate` / `Ring_remlo` / `Ring_remhi` | ring | `Module_verb` |

查询函数通常以 `_length`、`_count`、`_empty`、`_get`、`_member` 结尾，例如：
- `List_length(T list)` --- 返回链表长度
- `Bit_count(T set)` --- 返回位向量中置1的位数
- `Stack_empty(T stk)` --- 判断栈是否为空
- `Set_member(T set, const void *member)` --- 判断成员是否存在

### 2.3.5 哨兵宏：`_INCLUDED`

每个头文件的包含保护宏命名为 `<MODULE>_INCLUDED`：

```c
#ifndef STACK_INCLUDED    /* 哨兵: 防止重复包含 */
#define STACK_INCLUDED
/* ... */
#endif
```

这在本库中是完全一致的约定，没有例外。

---

## 2.4 编译单元与信息隐藏

### 2.4.1 编译单元的概念

在 C 语言中，每个 `.c` 文件连同它 `#include` 的所有头文件，构成一个独立的**编译单元** (translation unit)。编译单元是最小的独立编译粒度和信息隐藏边界。

本代码库中，每个模块的实现文件都是一个独立的编译单元：

```
code/atom.c      ---  编译单元: atom 接口的实现
code/except.c    ---  编译单元: except 接口的实现
code/stack.c     ---  编译单元: stack 接口的实现
code/list.c      ---  编译单元: list 接口的实现
code/table.c     ---  编译单元: table 接口的实现
code/bit.c       ---  编译单元: bit 接口的实现
code/mem.c       ---  编译单元: mem 接口的实现
code/arena.c     ---  编译单元: arena 接口的实现
```

### 2.4.2 `static` 关键字：文件作用域

`static` 关键字在 C 语言中有两种用法，本库中两者都被充分利用：

**`static` 函数** --- 内部辅助函数，仅在当前编译单元内可见：

- `code/bit.c` 第35-42行：`static T copy(T t)` --- 创建位向量的深拷贝，仅被 `Bit_union`、`Bit_diff` 等函数在 `setop` 宏中内部调用
- `code/str.c` 中的 `static` 辅助函数
- `code/atom.c` 中没有 static 函数，因为其所有逻辑都直接在导出的三个函数中完成

**`static` 变量** --- 模块级全局状态，对外不可见：

- `code/atom.c` 第8-12行：
  ```c
  static struct atom {
      struct atom *link;
      int len;
      char *str;
  } *buckets[2048];
  ```
  2048 个桶的哈希表被声明为 `static`，是整个 atom 模块的内部状态。客户端完全不知道它的存在。

- `code/atom.c` 第13-57行：`static unsigned long scatter[]` --- 256 个随机散列乘数，用于字符串哈希。这也是一个仅在编译单元内可见的查找表。

- `code/bit.c` 第27-34行：
  ```c
  unsigned char msbmask[] = { 0xFF, 0xFE, 0xFC, ... };
  unsigned char lsbmask[] = { 0x01, 0x03, 0x07, ... };
  ```
  注意这里缺少 `static`！这是本库中少有的没有使用 `static` 限定符的模块级数组。它们应当被声明为 `static` 以获得更好的信息隐藏（这是第 2.6.1 节将要讨论的改进点之一）。

- `code/arena.c` 第36-37行：
  ```c
  static T freechunks;
  static int nfree;
  ```
  内存池的空闲块链表和计数，仅在 arena 模块内部使用。

### 2.4.3 全局可见状态

并非所有内部状态都需要隐藏。当需要在多个编译单元之间共享状态时，使用 `extern` 声明：

- `code/except.h` 第19-20行：
  ```c
  extern Except_Frame *Except_stack;
  extern const Except_T Assert_Failed;
  ```
  异常栈是全局可见的，因为 `TRY/EXCEPT` 宏需要在客户端代码中内联展开并直接访问它。这是一个务实的设计妥协 --- 信息隐藏在与宏语法的便利性之间做了权衡。

- `code/mem.h`：`extern const Except_T Mem_Failed;`
  内存分配失败异常是全局可访问的，客户端代码可能需要在自己的异常处理器中捕获它。

### 2.4.4 `NELEMS` 宏：编译期数组大小

`code/atom.c` 第7行定义了一个在整个库中反复出现的实用宏：

```c
#define NELEMS(x) ((sizeof (x))/(sizeof ((x)[0])))
```

这个宏在编译期计算数组元素个数，避免了硬编码魔术数字。在 `code/atom.c` 中的用法：

```c
h &= NELEMS(buckets)-1;   /* 等价于 h &= 2047 */
for (i = 0; i < NELEMS(buckets); i++)  /* 等价于 i < 2048 */
```

当 `buckets` 数组的大小发生变化时，所有使用 `NELEMS` 的地方会自动调整，无需手动修改。

---

## 2.5 C 语言中的 ADT 设计模式

### 2.5.1 模式一：不透明句柄 (Opaque Handle)

这是本书最核心的 ADT 模式。接口导出指向不完整结构体的指针类型，客户端只能通过函数指针操作对象。

**模式要素**：
1. 头文件：`typedef struct T *T;`（指针抽象）
2. 构造函数：`T <Module>_new(...)` 分配并返回句柄
3. 操作函数：接受 `T` 作为第一个参数（类似 C++ 的 `this` 指针）
4. 析构函数：`void <Module>_free(T *p)` 释放并将客户端指针置 NULL

**应用实例**：
- `Stack_T` --- `code/stack.h` + `code/stack.c`
- `Table_T` --- `code/table.h` + `code/table.c`
- `Bit_T` --- `code/bit.h` + `code/bit.c`
- `Set_T` --- `code/set.h` + `code/set.c`
- `Seq_T` --- `code/seq.h` + `code/seq.c`
- `Ring_T` --- `code/ring.h` + `code/ring.c`
- `Arena_T` --- `code/arena.h` + `code/arena.c`
- `AP_T` --- `code/ap.h` + `code/ap.c`

### 2.5.2 模式二：透明结构体 (Transparent Struct)

当数据结构的简单性超过封装的需要时，结构体直接暴露在头文件中。

**应用实例** --- `code/list.h`：

```c
typedef struct T *T;
struct T {
    T rest;
    void *first;
};
```

`struct List_T` 只有两个指针字段，完全暴露。这允许客户端直接遍历链表而无需函数调用开销，但也意味着客户端可能破坏链表结构。这是一个有意为之的**性能与安全性的权衡**。

另一个类似的例子是 `code/text.h` 中的 `Text_T`：

```c
typedef struct T {
    int len;
    const char *str;
} T;
```

`Text_T` 不是指针类型，而是一个包含长度和字符串指针的结构体（**值语义**）。这是书中的另一个特殊设计选择：Text 作为小对象，按值传递效率更高，且其不变式（invariant）非常简单（`str` 指向 `len` 字节的字符串）。

还有一个极端的透明类型 --- `code/xp.h`：

```c
#define T XP_T
typedef unsigned char *T;
```

`XP_T` 仅仅是 `unsigned char *` 的类型别名，完全没有封装。这是因为扩展精度算术（XP）模块直接操作裸字节数组以实现极致性能。

### 2.5.3 模式三：静态模块 (Static Module)

不导出任何类型，只提供操作现有 C 类型的函数。

**应用实例** --- `code/arith.h`、`code/atom.h`：

`code/arith.h` 仅包含作用于 `int` 的纯函数：
```c
extern int Arith_max(int x, int y);
extern int Arith_min(int x, int y);
extern int Arith_div(int x, int y);
extern int Arith_mod(int x, int y);
extern int Arith_ceiling(int x, int y);
extern int Arith_floor  (int x, int y);
```

没有创建新类型，没有状态，所有函数都是纯数学运算。这是最简单的接口形式。

`code/atom.h` 虽有其内部的哈希表状态，但对外接口完全操作 `const char *`：
```c
extern const char *Atom_new   (const char *str, int len);
extern const char *Atom_string(const char *str);
extern const char *Atom_int   (long n);
extern int   Atom_length(const char *str);
```

### 2.5.4 模式四：泛型容器 (Generic Container via `void *`)

C 语言没有模板或泛型，Hanson 使用 `void *` 实现通用的数据容器。

**实例** --- `code/stack.h`：
```c
extern void  Stack_push(T stk, void *x);
extern void *Stack_pop (T stk);
```

任何类型的指针都可以被推入栈中。类型安全由调用者负责。

**实例** --- `code/table.h`：
```c
extern void *Table_put(T table, const void *key, void *value);
extern void *Table_get(T table, const void *key);
```

键和值都是 `void *`。由于 C 语言的类型系统限制，这是一种务实的通用方案。调用者必须自己保证键和值的类型一致性。

### 2.5.5 模式五：回调驱动的遍历 (Callback-driven Iteration)

由于没有内置的迭代器抽象，Hanson 使用**高阶函数**模式（函数指针作为参数）：

**实例** --- `code/list.h`：
```c
extern void List_map(T list,
    void apply(void **x, void *cl), void *cl);
```

`List_map` 遍历链表，对每个节点的值调用 `apply` 函数。`cl` (client closure) 是传递给 `apply` 的上下文指针，用于携带状态。

**实例** --- `code/table.h`：
```c
extern void Table_map(T table,
    void apply(const void *key, void **value, void *cl),
    void *cl);
```

**实例** --- `code/set.h`：
```c
extern void Set_map(T set,
    void apply(const void *member, void *cl), void *cl);
```

**实例** --- `code/bit.h`：
```c
extern void Bit_map(T set,
    void apply(int n, int bit, void *cl), void *cl);
```

**实例** --- `code/ring.h`：（注意 ring 没有 `_map` 函数，而是提供了 `_get`/`_put` 索引访问）

`_toArray` 转换函数是另一种“导出遍历结果”的方式：
```c
extern void **List_toArray(T list, void *end);
extern void **Table_toArray(T table, void *end);
extern void **Set_toArray(T set, void *end);
```

这些函数将容器内容复制到一个以 `end` 结尾的动态数组中，便于 C 风格的下标遍历。

### 2.5.6 模式六：异常驱动的错误处理

Hanson 摒弃了 C 语言传统的 `return NULL` 或 `return -1` 错误报告方式，而是使用**结构化异常处理**。

**实例** --- `code/mem.c`：
```c
void *Mem_alloc(long nbytes, const char *file, int line) {
    void *ptr;
    assert(nbytes > 0);
    ptr = malloc(nbytes);
    if (ptr == NULL) {
        if (file == NULL)
            RAISE(Mem_Failed);
        else
            Except_raise(&Mem_Failed, file, line);
    }
    return ptr;
}
```

内存分配失败时抛出异常而非返回 NULL。这意味着所有调用 `ALLOC`/`NEW` 的代码可以假定它们不会返回 NULL，从而简化控制流。

**实例** --- `code/arena.c`：
```c
const Except_T Arena_NewFailed = { "Arena creation failed" };
const Except_T Arena_Failed    = { "Arena allocation failed" };
```

每个模块定义自己的异常常量，提供有意义的错误消息。客户端代码可以选择性地捕获特定模块的异常。

### 2.5.7 模式七：含溯源信息的分配器

`code/mem.h` 中的分配宏在每次分配时携带源文件和行号：

```c
#define ALLOC(nbytes)  Mem_alloc((nbytes), __FILE__, __LINE__)
#define CALLOC(count, nbytes) Mem_calloc((count), (nbytes), __FILE__, __LINE__)
#define NEW(p)  ((p) = ALLOC((long)sizeof *(p)))
#define NEW0(p) ((p) = CALLOC(1, (long)sizeof *(p)))
#define FREE(ptr) ((void)(Mem_free((ptr), __FILE__, __LINE__), (ptr) = 0))
```

当分配失败时，异常信息会包含精确的源文件和行号，极大地简化了调试。`__FILE__` 和 `__LINE__` 是 C 预处理器提供的标准宏，在宏展开点被替换为调用处的文件名和行号。

---

## 2.6 Modern-C 改进

Hanson 的书出版于 1997 年（C89 时代）。自 C99、C11 和 C17 以来，C 语言获得了许多可以增强 Hanson 的接口设计模式的新特性。以下是在保持 Hanson 原始设计精神的前提下，用 Modern-C 特性进行的改进。

### 2.6.1 `const` 正确性回顾与增强

Hanson 已经在很大程度上使用了 `const`，但存在遗漏：

**已有的正确用法**：
- `code/atom.h`：`const char *Atom_new(const char *str, int len)` --- 输入和输出都是 `const` 正确
- `code/except.h`：`const Except_T Assert_Failed` --- 异常常量是不可变的
- `code/table.h`：`void *Table_put(T table, const void *key, void *value)` --- 键参数是 `const void *`
- `code/list.h`：`void List_map(T list, void apply(void **x, void *cl), void *cl)` --- 遍历不修改链表本身

**可以改进的地方**：

1. `code/bit.h` 中的 `msbmask` 和 `lsbmask`（第27-34行）：
   ```c
   /* 改进前 */
   unsigned char msbmask[] = { ... };
   unsigned char lsbmask[] = { ... };
   /* 改进后 */
   static const unsigned char msbmask[] = { ... };
   static const unsigned char lsbmask[] = { ... };
   ```
   添加 `static` 限制作用域，添加 `const` 防止意外修改。

2. 查询函数应标记为 `const` 语义 --- 虽然 C 没有 C++ 的 `const` 成员函数，但可以通过 `const T` 参数来表达“此函数不修改对象”：
   ```c
   /* 改进前 */
   extern int Table_length(T table);
   /* 改进后 --- 声明查询函数不修改参数 */
   extern int Table_length(const T table);  /* 注意: const 在 * 之后 */
   ```
   
   不过，由于 `T` 展开为 `struct T *`，`const T` 变成 `struct T *const`（指针本身是常量），而不是指向常量的指针。要实现完整的 const 正确性，需要在 typedef 层面区分 `T`（可修改指针）和 `Const_T`（指向常量的指针），但这会显著增加接口的复杂度。这是一个反映了 C 语言 const 语义局限性的设计张力。

### 2.6.2 `_Generic`：编译期类型分发

C11 引入的 `_Generic` 关键字可以在编译期根据参数类型选择不同的函数实现。这可以用来为 Hanson 的 `void *` 泛型容器提供**类型安全的包装**。

假设我们要为 `Stack` 添加类型安全的包装：

```c
/* Modern-C 改进：类型安全的 Stack 包装 */
#include <stdio.h>

/* 类型标记枚举 */
enum stack_type { STACK_INT, STACK_DOUBLE, STACK_STRING };

#define Stack_push_typed(stk, x) \
    _Generic((x),                                \
        int:    Stack_push_int(stk, x),           \
        double: Stack_push_double(stk, x),        \
        char *: Stack_push_string(stk, x),        \
        default: Stack_push_void(stk, (void*)(uintptr_t)(x)) \
    )

#define Stack_pop_typed(stk, type) \
    _Generic((type)0,                             \
        int:    Stack_pop_int(stk),               \
        double: Stack_pop_double(stk),            \
        char *: Stack_pop_string(stk),            \
        default: Stack_pop(stk)                   \
    )
```

对于 `List_map` 的回调函数，也可以使用 `_Generic` 来避免类型转换：

```c
/* 为 List_map 提供类型安全的迭代器宏 */
#define list_foreach(list, type, var)             \
    for (type *var##_ptr = (type *)(list);        \
         var##_ptr;                                \
         var##_ptr = (type *)(var##_ptr->rest))   \
        if ((var = (type)(var##_ptr->first), 1))
```

### 2.6.3 `static_assert`：编译期契约检查

C11 的 `static_assert`（C23 中变为 `static_assert`，C11 中通过 `_Static_assert` 关键字）可以在编译期强制执行接口契约，避免运行时崩溃。

**应用一**：确保不透明指针的尺寸假设

```c
/* 在 stack.c 中添加: */
#include <assert.h>  /* C11 _Static_assert 或 C23 static_assert */
_Static_assert(sizeof(struct T) <= 64, 
    "Stack_T size exceeds cache line");
```

**应用二**：确保结构体字段的对齐

```c
/* 在 arena.c 中验证对齐联合体的大小: */
_Static_assert(sizeof(union align) >= sizeof(long double),
    "Alignment union too small for long double");
_Static_assert(sizeof(union align) >= sizeof(void (*)(void)),
    "Alignment union too small for function pointer");
```

实际上，`code/arena.c` 中的 `union align`（第18-31行）的设计目的正是计算平台的最大对齐要求。使用 `static_assert` 可以确保这种平台依赖的假设在编译期被验证：

```c
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
/* Modern-C: 验证我们的最大对齐假设 */
_Static_assert(_Alignof(union align) >= _Alignof(long double),
    "Alignment union insufficient");
_Static_assert(_Alignof(union align) >= _Alignof(max_align_t),
    "Alignment union insufficient for max_align_t");
```

**应用三**：确保 `sizeof` 假设

在 `code/list.h` 中，链表节点结构体暴露在头文件中：
```c
struct T {
    T rest;
    void *first;
};
/* Modern-C: 确保与文档描述一致 */
_Static_assert(sizeof(struct T) == 2 * sizeof(void *),
    "List_T node size mismatch: expected two pointers");
```

**应用四**：确保哨兵常量与实现一致

在 `code/atom.c` 中，哈希桶数量固定为 2048：
```c
static struct atom *buckets[2048];
/* Modern-C: 编译期断言桶数量是 2 的幂（用于掩码操作） */
_Static_assert((2048 & (2048 - 1)) == 0, 
    "Bucket count must be a power of two");
```

### 2.6.4 `_Noreturn` / `noreturn`：编译器优化提示

`code/except.h` 中：
```c
extern void Except_raise(const T *e, const char *file, int line);
```

`Except_raise` 调用 `longjmp`，永远不会正常返回。在 Modern-C 中应标记为：

```c
#include <stdnoreturn.h>  /* C11 */
_Noreturn void Except_raise(const T *e, const char *file, int line);
/* 或 C23: [[noreturn]] void Except_raise(...); */
```

这使编译器能够更好地优化调用 `Except_raise` 的代码路径，并消除“函数未返回值”的警告。

### 2.6.5 `_Alignas`：精确控制内存布局

在 `code/arena.c` 中，对齐通过 `union align` 实现：

```c
union align {
    int i; long l; long *lp; void *p;
    void (*fp)(void); float f; double d; long double ld;
};
```

Modern-C 可以直接使用 `_Alignas`（或 C23 的 `alignas`）：

```c
/* Modern-C 替代方案 */
#define ARENA_ALIGN _Alignof(max_align_t)
struct T {
    T prev;
    _Alignas(max_align_t) char data[];  /* 灵活数组成员，强制对齐 */
};
```

### 2.6.6 匿名结构体与联合体 (C11)

C11 的匿名结构体/联合体可以减少命名的冗余。例如 `code/arena.c` 中的：

```c
union header {
    struct T b;
    union align a;
};
```

可以优化为：

```c
union header {
    struct T {
        T prev;
        char *avail;
        char *limit;
    };  /* 匿名结构体 */
    union {  /* 匿名联合体 */
        int i; long l; void *p; 
        double d; long double ld;
    };
};
```

### 2.6.7 复合字面量 (Compound Literals, C99)

C99 的复合字面量可以简化异常常量的定义。Hanson 手册风格的写法：

```c
/* code/mem.c: Hanson 原始写法 */
const Except_T Mem_Failed = { "Allocation failed" };

/* code/arena.c: Hanson 原始写法 */
const Except_T Arena_NewFailed = { "Arena creation failed" };
const Except_T Arena_Failed    = { "Arena allocation failed" };
```

复合字面量允许在需要时原地构造，减少全局变量：

```c
/* Modern-C: 在需要时内联构造 */
RAISE(((Except_T){ "Allocation failed" }));
```

但这种做法会使异常对象失去“按地址比较”的能力（每个复合字面量是独立的对象），因此在需要异常层次结构（如 `EXCEPT(e)` 宏中通过地址比较来匹配异常类型）的场景中，原始的全局常量定义仍然更合适。这是一个典型的“Modern-C 特性虽然可用但并非总是更好”的例子。

### 2.6.8 `bool` 类型 (C99)

Hanson 的代码中，布尔值使用 `int` 表示（0 为假，非0为真）：

```c
extern int Stack_empty(T stk);
extern int Bit_get(T set, int n);
extern int Bit_lt(T s, T t);
extern int Set_member(T set, const void *member);
```

Modern-C 中应使用 `#include <stdbool.h>` 提供的 `bool` 类型：

```c
#include <stdbool.h>
extern bool Stack_empty(T stk);
extern bool Bit_get(T set, int n);
extern bool Bit_lt(T s, T t);
extern bool Set_member(T set, const void *member);
```

这不会改变 ABI（`bool` 在大多数平台上与 `int` 大小相同），但显著提高了接口的可读性和自文档化程度。

---

## 2.7 接口设计原则总结

通过对 Hanson 的代码库的深入分析，可以归纳出以下核心设计原则：

| 原则 | 说明 | 代码实例 |
|------|------|----------|
| **分离接口与实现** | `.h` 声明做什么，`.c` 定义怎么做 | 全部 25 个模块 |
| **不透明指针** | 暴露指针类型，隐藏结构体定义 | `stack.h`/`stack.c`, `table.h`/`table.c` |
| **一致的命名约定** | `Module_verb`、`_new`、`_free`、`T` 宏 | 全部模块遵循相同的命名模式 |
| **哨兵宏** | `#ifndef X_INCLUDED` 防止重复包含 | 每个 `.h` 文件 |
| **`static` 信息隐藏** | 内部函数和数据声明为 `static` | `atom.c` 的 `buckets[]`、`scatter[]` |
| **通用性通过 `void *`** | 泛型容器使用 `void *` | `stack.h`, `list.h`, `table.h`, `set.h` |
| **异常优于返回码** | 使用 `TRY/EXCEPT` 替代 `NULL` 检查 | `mem.h`/`mem.c`, `arena.h`/`arena.c` |
| **溯源信息** | `__FILE__`/`__LINE__` 嵌入分配和异常 | `ALLOC`/`NEW`/`FREE`/`RAISE` 宏 |
| **构造/析构对称** | `_new` 分配 + 初始化，`_free(T*)` 释放 + 置空 | `stack.c`, `table.c`, `bit.c` |
| **防御性断言** | 每个公共函数入口检查前置条件 | `assert(stk)`、`assert(nbytes > 0)` |
| **最小权限原则** | 接口只导出客户端必需的内容 | `static` 用于所有内部辅助函数和数据 |

### 2.7.1 Hanson 风格的完整示例

以下是将上述所有原则综合在一起的完整示例 --- `Stack` 模块的现代重写版本：

```c
/* ============ stack.h ============ */
#ifndef STACK_INCLUDED
#define STACK_INCLUDED

#include <stdbool.h>    /* C99 bool */
#include <stddef.h>     /* size_t */

#define T Stack_T
typedef struct T *T;

/* 构造函数 */
extern T    Stack_new(void);
/* 容量与查询 */
extern bool Stack_empty(T stk);
extern int  Stack_length(T stk);
/* 操作 */
extern void  Stack_push(T stk, void *x);
extern void *Stack_pop (T stk);
extern void *Stack_peek(T stk);
/* 迭代 */
extern void  Stack_map(T stk, 
    void apply(void *x, void *cl), void *cl);
/* 销毁 --- 接收 T* 以置空调用者指针 */
extern void  Stack_free(T *stk);

#undef T
#endif /* STACK_INCLUDED */


/* ============ stack.c ============ */
#include <stddef.h>
#include <stdbool.h>
#include <stdlib.h>
#include "assert.h"
#include "mem.h"
#include "stack.h"

#define T Stack_T

/* 结构体定义 --- 仅在实现文件中可见 */
struct T {
    int count;
    struct node {
        void *data;
        struct node *next;
    } *head;
};

/* 编译期断言 */
_Static_assert(sizeof(struct T) >= sizeof(void *),
    "Stack_T must be at least pointer-sized");

T Stack_new(void) {
    T stk;
    NEW(stk);
    stk->count = 0;
    stk->head  = NULL;
    return stk;
}

bool Stack_empty(T stk) {
    assert(stk);
    return stk->count == 0;
}

int Stack_length(T stk) {
    assert(stk);
    return stk->count;
}

void Stack_push(T stk, void *x) {
    struct node *n;
    assert(stk);
    NEW(n);
    n->data = x;
    n->next = stk->head;
    stk->head = n;
    stk->count++;
}

void *Stack_pop(T stk) {
    void *x;
    struct node *n;
    assert(stk);
    assert(stk->count > 0);
    n = stk->head;
    stk->head = n->next;
    stk->count--;
    x = n->data;
    FREE(n);
    return x;
}

void *Stack_peek(T stk) {
    assert(stk);
    assert(stk->count > 0);
    return stk->head->data;
}

void Stack_map(T stk, 
    void apply(void *x, void *cl), void *cl) {
    struct node *n;
    assert(stk);
    assert(apply);
    for (n = stk->head; n; n = n->next)
        apply(n->data, cl);
}

void Stack_free(T *stk) {
    struct node *n, *next;
    assert(stk && *stk);
    for (n = (*stk)->head; n; n = next) {
        next = n->next;
        FREE(n);
    }
    FREE(*stk);  /* FREE 宏将 *stk 置为 NULL */
}
```

---

## 2.8 关键术语对照

| 英文 | 中文 | 说明 |
|------|------|------|
| Interface | 接口 | `.h` 文件定义的模块公共契约 |
| Implementation | 实现 | `.c` 文件中的具体代码 |
| Opaque Pointer | 不透明指针 | 指向未公开结构体的指针类型 |
| ADT (Abstract Data Type) | 抽象数据类型 | 通过接口访问、实现细节隐藏的数据类型 |
| Information Hiding | 信息隐藏 | 将实现细节限制在编译单元内 |
| Translation Unit | 编译单元 | `.c` 文件 + `#include` 的扩展结果 |
| Sentry Macro | 哨兵宏 | `#ifndef X_INCLUDED` 包含保护 |
| Opaque Handle | 不透明句柄 | 与不透明指针同义 |
| Client Closure | 客户端闭包 | 传递给回调函数的上下文指针 `cl` |
| Invariant | 不变式 | 数据结构的完整性约束 |
| Interned String | 驻留字符串 | 全局唯一、不可变的字符串实例 |
| Dangling Pointer | 悬空指针 | 指向已释放内存的指针 |

---

## 2.9 延伸阅读

- 原书代码仓库（官方）：<https://github.com/drh/cii>
- Hanson 全书接口快速参考：<http://cii.s3.amazonaws.com/book/pdf/quickref.pdf>
- C11 标准 `_Generic` 参考：ISO/IEC 9899:2011 Section 6.5.1.1
- C11 标准 `_Static_assert` 参考：ISO/IEC 9899:2011 Section 6.7.10
- C11 标准 `_Noreturn` 参考：ISO/IEC 9899:2011 Section 6.7.4
- P. J. Plauger, *The Standard C Library* (1992) --- C 标准库接口设计的经典著作
- Brian W. Kernighan and Rob Pike, *The Practice of Programming* (1999) --- 第 4 章专论接口设计

---

> **译者注**：本章基于 David R. Hanson *C Interfaces and Implementations* (Addison-Wesley, 1997) 第 2 章内容，结合 `/home/ares/yyscode/cii-code/code/` 目录下的实际代码进行了详细的技术分析。所有代码示例均来源于该目录中 `atom.h`、`except.h`、`list.h`、`stack.h`、`table.h`、`bit.h`、`set.h`、`seq.h`、`ring.h`、`arena.h`、`arith.h`、`ap.h`、`xp.h`、`mem.h`、`text.h`、`str.h`、`fmt.h` 等头文件及其对应的 `.c` 实现文件。Modern-C 改进部分（2.6 节）是基于 C11/C17/C23 标准的原创分析，补充了原著未涉及的新特性。
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch01_引言_Introduction.html">第1章 引言</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch03_原子_Atoms.html">第3章 原子 Atoms</a></div>
</div>
