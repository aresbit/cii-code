# 第10章 动态数组

> 译自 David R. Hanson, _C Interfaces and Implementations: Techniques for Creating Reusable Software_, Chapter 10: Dynamic Arrays

---

## 10.1 引言

静态数组是 C 语言最古老的构造之一。它的致命缺陷是大小必须在编译期确定，而这种"提前承诺"在实践中几乎总是错误的——要么浪费内存，要么不够用。C99 引入了变长数组（VLA），但 VLA 的存储期受限（自动存储期，出栈即销毁），且无法在运行时安全地增长。

本章给出一个**动态泛型数组** ADT：`Array_T`。它提供以下核心能力：

- **运行时指定元素个数**（`length`）和**元素大小**（`size`）
- **可调整大小**：增加或减少元素个数
- **泛型存储**：元素以原始字节形式存放，通过 `memcpy` 读写
- **边界检查**：每次访问均校验索引合法性

Hanson 的实现只有约 80 行 C 代码，但其设计蕴含了接口/实现分离、表示暴露控制、内存异常安全等多个经典工程原则。

---

## 10.2 双层接口架构

CII 库中许多 ADT 使用了一种精巧的**双层接口**（two-level interface）模式。`Array` 模块是理解这一模式的最佳入口。

### 10.2.1 公共接口：`array.h`

```c
/* array.h — public interface */
#ifndef ARRAY_INCLUDED
#define ARRAY_INCLUDED
#define T Array_T
typedef struct T *T;

extern T    Array_new (int length, int size);
extern void Array_free(T *array);
extern int  Array_length(T array);
extern int  Array_size  (T array);
extern void *Array_get(T array, int i);
extern void *Array_put(T array, int i, void *elem);
extern void  Array_resize(T array, int length);
extern T     Array_copy  (T array, int length);

#undef T
#endif
```

关键设计决策：

1. **不完整类型（opaque pointer）**：`typedef struct T *T;` 声明了一个指向不完整结构体的指针。客户程序只能持有 `Array_T` 指针，无法窥视其内部表示。

2. **`T` 宏约定**：`#define T Array_T` 在整个 CII 库中统一使用，使得每个模块的接口函数签名看起来像一个 ADT 方法族：`Array_new`、`Array_length`、`Array_get` 等。每个模块通过 `#undef T` 恢复到干净状态。

3. **泛型通过 `void *` 和 `size` 实现**：数组不关心元素的类型。`Array_new` 接受 `size`（每个元素的字节数），`Array_get` 返回 `void *` 指针，`Array_put` 通过 `memcpy` 拷贝 `size` 个字节。这是 C 语言实现泛型的经典手法——类型擦除 + 显式大小。

4. **`Array_free` 接受 `T *`（二级指针）**：释放后将调用者的指针置为 `NULL`（通过 `FREE` 宏），防止悬挂指针。

### 10.2.2 私有表示头：`arrayrep.h`

```c
/* arrayrep.h — representation header (for privileged clients) */
#ifndef ARRAYREP_INCLUDED
#define ARRAYREP_INCLUDED
#define T Array_T
struct T {
    int length;       // 元素个数
    int size;         // 每个元素的字节数
    char *array;      // 指向 (length × size) 字节的内存
};
extern void ArrayRep_init(T array, int length,
    int size, void *ary);
#undef T
#endif
```

这是双层架构的第二层。只有需要**直接访问描述符字段**或**自行分配描述符内存**的客户程序才会 `#include "arrayrep.h"`。

设计动机（Hanson 原书）：

> 只有导入了 `ArrayRep` 的客户程序才会依赖 `Array_T` 的具体表示。只要 `ArrayRep_init` 的签名保持不变，即使向结构体中添加新字段，也不会破坏这些客户程序。

换句话说：`arrayrep.h` 将"谁依赖于表示"这一依赖关系**显式化**和**局部化**了。这是编译期可见的耦合治理手段。

---

## 10.3 表示结构：Dope Vector

`Array_T` 的本质是一个**描述符**（descriptor），或称 **dope vector**——它是一个很小的固定大小的结构体，其中 `array` 字段指向堆上分配的元素数组。

```
+------------------+         +---------------------------+
| struct Array_T   |         |  元素存储区 (堆)          |
|  length = 5      |         |  [0]  [1]  [2]  [3]  [4] |
|  size   = 8      |  ---->  |  每个元素 size 字节       |
|  array  = ptr ---|         +---------------------------+
+------------------+
```

第 `i` 个元素的内存地址为：

```
array->array + i * array->size
```

这个公式贯穿 `Array_get`、`Array_put`、`Array_copy` 的实现，是整个模块的核心不变式。

当 `length == 0` 时，`array` 字段为 `NULL`。这一约定使得零长度数组不会触发不必要的内存分配。

---

## 10.4 接口 API 逐函数分析

以下分析基于 `/code/array.c` 逐行注释。源代码外围依赖参见 [10.4.9 节](#1049-内存管理宏)。

### 10.4.1 `Array_new`

```c
T Array_new(int length, int size) {
    T array;
    NEW(array);                               // (1) 分配描述符
    if (length > 0)
        ArrayRep_init(array, length, size,
            CALLOC(length, size));             // (2) 分配元素内存 + 初始化为零
    else
        ArrayRep_init(array, length, size, NULL);  // (3) 零长度: 不分配元素内存
    return array;
}
```

**设计要点：**

- (1) `NEW(array)` 展开为 `(array = Mem_alloc((long)sizeof *(array), __FILE__, __LINE__))`。它分配描述符本身的内存。使用 `sizeof *array` 而非 `sizeof(struct T)` 是一个防御性技巧：即使 `array` 的类型变了，这行代码也无需修改。

- (2) `CALLOC` 不仅分配内存，还将所有元素初始化为零。对于整数数组（初始值 0）和指针数组（初始值 `NULL`），这是符合直觉的默认行为。

- (3) `ArrayRep_init` 是**唯一的初始化入口**——无论描述符内存从哪来（`NEW`、静态分配、嵌入其他结构），初始化逻辑始终一致。见 [10.4.2 节](#1042-arrayrep_init)。

### 10.4.2 `ArrayRep_init`

```c
void ArrayRep_init(T array, int length, int size,
    void *ary) {
    assert(array);
    assert(ary && length>0 || length==0 && ary==NULL);
    assert(size > 0);
    array->length = length;
    array->size   = size;
    if (length > 0)
        array->array = ary;
    else
        array->array = NULL;
}
```

**前置条件断言分析：**

| 断言 | 含义 |
|------|------|
| `assert(array)` | 描述符指针不能为 `NULL` |
| `assert(ary && length>0 \|\| length==0 && ary==NULL)` | 元素内存与长度的一致性：正长度必须搭配非空内存，零长度必须搭配空指针 |
| `assert(size > 0)` | 元素大小必须为正 |

**为什么单独暴露 `ArrayRep_init`？**

Hanson 的原始理由有二：

1. **标记耦合位置**：调用 `ArrayRep_init` 的代码依赖了 `Array_T` 的表示。这让维护者可以快速定位所有需要随表示变化而修改的客户文件。

2. **扩展隔离**：假设将来在 `struct T` 中添加一个新字段（如时间戳 `unsigned long seq`），只需修改 `ArrayRep_init` 为新字段赋默认值，所有通过 `ArrayRep_init` 初始化的客户程序自动获得正确的默认值，无需逐文件修改。

### 10.4.3 `Array_free`

```c
void Array_free(T *array) {
    assert(array && *array);
    FREE((*array)->array);    // 释放元素内存，并将字段置为 NULL
    FREE(*array);             // 释放描述符，并将调用者指针置为 NULL
}
```

**安全特性：**

- 参数类型是 `T *`（即 `Array_T *`），因为函数需要将调用者的指针置 `NULL`。这由 `FREE` 宏保障：`FREE(ptr)` 展开为 `(Mem_free(ptr, __FILE__, __LINE__), (ptr) = 0)`。

- 调用者传入 `&my_array` 后，`my_array` 被置为 `NULL`，防止悬挂指针被误用。这是 C 语言中实现类似 C++ `unique_ptr::reset()` 效果的手工方案。

### 10.4.4 `Array_length` 和 `Array_size`

```c
int Array_length(T array) {
    assert(array);
    return array->length;
}

int Array_size(T array) {
    assert(array);
    return array->size;
}
```

平凡但重要：这两个查询函数**不暴露任何实现细节**。客户程序只知道数组"有若干个元素"和"每个元素有多大"，而不知道元素存储在何处、是否连续、是否已分配。

### 10.4.5 `Array_get`

```c
void *Array_get(T array, int i) {
    assert(array);
    assert(i >= 0 && i < array->length);      // 边界检查
    return array->array + i*array->size;       // 计算第 i 个元素的地址
}
```

**关键行为：**

- 返回的是**指向内部缓冲区的指针**，不是元素副本。这意味着：
  - 调用者可以直接通过指针修改元素值
  - 指针的有效期持续到下一次 `Array_resize` 调用（`realloc` 可能移动内存）
  - 这在 CII 文档中有明确警告：调整大小后必须重新获取指针

- 边界检查在 DEBUG 模式下触发 `assert` 失败（通过 `RAISE(Assert_Failed)` 抛出异常），在 `NDEBUG` 模式下被编译为空操作。见 [10.4.9 节](#1049-内存管理宏) 的 `assert.h` 分析。

### 10.4.6 `Array_put`

```c
void *Array_put(T array, int i, void *elem) {
    assert(array);
    assert(i >= 0 && i < array->length);
    assert(elem);
    memcpy(array->array + i*array->size, elem,   // 目标: 第 i 个元素位置
        array->size);                               // 拷贝 size 字节
    return elem;
}
```

**设计要点：**

- **值语义**：`Array_put` 执行 `memcpy`，将 `elem` 指向的 `size` 个字节**拷贝**到数组内部。数组不持有指向外部数据的指针。这避免了悬挂引用问题。

- **返回 `elem`**（而非 `array` 或 `void`）：这是一个实用主义的选择——它允许链式表达式，如：
  ```c
  printf("%d\n", *(int *)Array_put(arr, 3, &val));
  ```

- **泛型正确性依赖调用者**：如果 `elem` 指向的数据小于 `array->size` 字节，`memcpy` 会读取越界。C 语言的类型擦除无法在编译期捕获此错误。

### 10.4.7 `Array_resize`

```c
void Array_resize(T array, int length) {
    assert(array);
    assert(length >= 0);
    if (length == 0)
        FREE(array->array);                       // (a) 收缩到零: 释放 + 置 NULL
    else if (array->length == 0)
        array->array = ALLOC(length*array->size); // (b) 从零增长: 分配新内存
    else
        RESIZE(array->array, length*array->size); // (c) 非零到非零: realloc
    array->length = length;
}
```

**三种路径分析：**

| 路径 | 条件 | 操作 | 新元素内容 |
|------|------|------|-----------|
| (a) | `length == 0` | `FREE` → 释放并置 `NULL` | N/A |
| (b) | `array->length == 0` | `ALLOC` → `malloc` | 未初始化（垃圾值） |
| (c) | `length > 0 && array->length > 0` | `RESIZE` → `realloc` | 原有元素保留；新元素未初始化 |

**注意：与 C++ `std::vector::resize` 的区别**

Hanson 的 `Array_resize` **不初始化新元素**。在路径 (b) 和 (c) 中，增长部分的元素内容是未定义的。这反映了 C 语言的性能优先哲学。与之对比，`std::vector<T>::resize(n)` 会对新元素执行值初始化（对基本类型为零初始化，对类类型为默认构造）。

### 10.4.8 `Array_copy`

```c
T Array_copy(T array, int length) {
    T copy;
    assert(array);
    assert(length >= 0);
    copy = Array_new(length, array->size);         // (1) 创建目标数组
    if (copy->length >= array->length
        && array->length > 0)                       // (2a) 目标 >= 源: 全量拷贝
        memcpy(copy->array, array->array,
            array->length*array->size);
    else if (array->length > copy->length
        && copy->length > 0)                        // (2b) 目标 < 源: 截断拷贝
        memcpy(copy->array, array->array,
            copy->length*array->size);
    return copy;
}
```

**拷贝语义：**

```
源数组: [A][B][C][D][E]  (length=5)
                          ↓ Array_copy(src, 3)
副本:   [A][B][C]         (length=3, 只拷贝前3个)
                          ↓ Array_copy(src, 8)
副本:   [A][B][C][D][E][?][?][?]  (length=8, 后3个未初始化)
```

这里有一个巧妙的实现细节：`Array_copy` 在实现中**通过 `copy->array` 和 `copy->length` 访问了 `copy` 的内部字段**。这意味着 `array.c` 本身是 `arrayrep.h` 的合法客户。它通过 `#include "arrayrep.h"`（见 `array.c` 第 6 行）获得了直接访问表示的权限。

四个分支的决策表：

| `copy->length` vs `array->length` | `memcpy` 行为 | 理由 |
|-----------------------------------|---------------|------|
| `copy` >= `src` && `src` > 0 | 拷贝 `src->length` 个 | 全量拷贝 |
| `src` > `copy` > 0 | 拷贝 `copy->length` 个 | 截断到目标容量 |
| `copy` == 0 | 不拷贝（条件不满足） | 零长度目标 |
| `src` == 0 | 不拷贝（条件不满足） | 零长度源 |

### 10.4.9 内存管理宏

`array.c` 使用的内存管理原语来自 `/code/mem.h`：

```c
#define ALLOC(nbytes) \
    Mem_alloc((nbytes), __FILE__, __LINE__)
#define CALLOC(count, nbytes) \
    Mem_calloc((count), (nbytes), __FILE__, __LINE__)
#define NEW(p)  ((p) = ALLOC((long)sizeof *(p)))
#define FREE(ptr) ((void)(Mem_free((ptr), \
    __FILE__, __LINE__), (ptr) = 0))
#define RESIZE(ptr, nbytes) \
    ((ptr) = Mem_resize((ptr), (nbytes), __FILE__, __LINE__))
```

核心设计：

- 每个宏向底层函数传递 `__FILE__` 和 `__LINE__`，使得内存分配失败时可以通过 CII 的异常机制报告精确的源位置。
- `FREE` 使用逗号表达式：先 `Mem_free`，再 `(ptr) = 0`。返回值被 `(void)` 丢弃，防止在条件表达式中误用。
- `RESIZE` 重新赋值 `ptr`，因为 `realloc` 可能返回新地址。

断言系统来自 `/code/assert.h`：

```c
#undef assert
#ifdef NDEBUG
#define assert(e) ((void)0)
#else
#include "except.h"
extern void assert(int e);
#define assert(e) ((void)((e)||(RAISE(Assert_Failed),0)))
#endif
```

CII 用自己的 `assert` 替代标准库版本。当断言失败时，它通过异常机制 `RAISE(Assert_Failed)` 传播错误，而非简单 `abort()`。`NDEBUG` 定义时，断言被完全移除，与标准行为一致。

---

## 10.5 扩容策略：倍增算法与内存浪费分析

### 10.5.1 机制与策略分离

Hanson 的 `Array_resize` 只提供**机制**（mechanism）——按指定长度调整内存。它不内置**策略**（policy）——何时调整、调整到多大。这种分离是 Unix 哲学的体现。

典型的客户端使用模式：

```c
/* 客户端增长策略: 倍增 */
void ensure_capacity(Array_T arr, int needed) {
    if (needed > Array_length(arr)) {
        int new_len = Array_length(arr) == 0 ? 1 : Array_length(arr) * 2;
        while (new_len < needed) new_len *= 2;
        Array_resize(arr, new_len);
    }
}
```

### 10.5.2 倍增策略的摊销分析

倍增策略（capacity doubling）是动态数组的经典优化。假设从空数组开始，每次空间不足时将容量翻倍，连续插入 `n` 个元素的总时间复杂度分析：

**模型：** 每次扩容时，所有已有元素都要拷贝到新内存区域。

| 扩容次数 | 扩容前容量 | 拷贝开销（元素数） |
|---------|-----------|-------------------|
| 第 1 次 | 0 → 1 | 0 |
| 第 2 次 | 1 → 2 | 1 |
| 第 3 次 | 2 → 4 | 2 |
| 第 4 次 | 4 → 8 | 4 |
| ... | ... | ... |
| 第 k 次 | 2^(k-2) → 2^(k-1) | 2^(k-2) |

**总拷贝开销：**

```
T(n) = 1 + 2 + 4 + ... + n/2
     = n/2 + n/4 + n/8 + ... + 1
     < n
```

加上 `n` 次元素放置操作，总操作数 < 2n。因此：

$$T_{amortized}(n) = \frac{O(n)}{n} = O(1)$$

**结论：** 倍增策略将单次插入的均摊时间降为常数级。

### 10.5.3 内存浪费分析

**最坏情况：** 刚完成扩容后，数组的使用率只有 50%，浪费 50% 的已分配空间。

**长期平均：** 数组大小在 `[capacity/2 + 1, capacity]` 间随机分布时，平均使用率为 75%，平均浪费 25%。

**增长因子对比：**

| 增长因子 | 均摊插入 | 平均内存浪费 | 评价 |
|---------|---------|-------------|------|
| ×2.0 | O(1) | ~25% | 教科书方案，简单 |
| ×1.5 | O(1) | ~17% | GCC `std::vector` 使用；内存友好 |
| +c (固定) | O(n) | 极小 | 不适合通用场景 |

**收缩策略：** 为避免在边界处反复分配/释放（thrashing），常见的做法是只在数组填充率降至 **1/4** 时将容量收缩至 **1/2**，保留增长缓冲。

### 10.5.4 Hanson 的设计选择

Hanson 选择将增长策略**完全留给客户**。这一决定的权衡：

| 优势 | 劣势 |
|------|------|
| 灵活性最大化：客户可以选择倍增、固定增量、指数增长等任意策略 | 每个客户需要手动编写增长逻辑 |
| 接口简洁：`Array_resize` 只做一件事 | 缺少类似 `push_back` 的便捷函数 |
| 内存精确控制：客户精确知道每次 `realloc` 的时机 | 新手容易写出 O(n^2) 的逐元素增长 |

---

## 10.6 `arrayrep.h` 的封装模式

### 10.6.1 模块内部文件依赖图

```
客户程序 (只 #include "array.h")
    │
    ├── Array_new() ──────┐
    ├── Array_free()      │
    ├── Array_length()    │  依赖 arrayrep.h (需要访问表示)
    ├── Array_size()      │
    ├── Array_get()       ├── array.c (#include "array.h" + "arrayrep.h")
    ├── Array_put()       │              │
    ├── Array_resize()    │              ├── mem.h (NEW, ALLOC, CALLOC, FREE, RESIZE)
    └── Array_copy() ─────┘              └── assert.h
    │
    └── 客户只看到不完整类型 Array_T, 不能解引用

特权客户 (同时 #include "array.h" + "arrayrep.h")
    │
    ├── 可以直接读取 array->length, array->size, array->array
    ├── 可以自行分配描述符内存 (静态、栈上、嵌入其他结构体)
    └── 必须通过 ArrayRep_init() 初始化
```

### 10.6.2 嵌入描述符的用法

`arrayrep.h` 存在的主要原因之一是支持**将描述符嵌入其他数据结构**：

```c
#include "array.h"
#include "arrayrep.h"

struct my_vector {
    Array_T data;        /* 嵌入的描述符，非指针！ */
    /* ... 其他字段 */
};

void my_vector_init(struct my_vector *v, int elem_size) {
    /* 描述符作为 my_vector 的一部分已分配，
       只需初始化其字段 */
    ArrayRep_init(&v->data, 0, elem_size, NULL);
}
```

这种模式避免了为描述符单独分配堆内存，将描述符和容器写在同一个缓存行中，提升了局部性。

---

## 10.7 完整使用示例

```c
/* demo.c — 演示 Array_T 的典型用法 */
#include <stdio.h>
#include "array.h"

int main(void) {
    /* 创建容纳 4 个 double 的动态数组 */
    Array_T arr = Array_new(4, sizeof(double));

    /* 写入元素 */
    double vals[] = {3.14, 2.718, 1.414, 0.577};
    for (int i = 0; i < 4; i++)
        Array_put(arr, i, &vals[i]);

    /* 扩展为 8 个 */
    Array_resize(arr, 8);
    double extra[] = {1.0, 2.0, 3.0, 4.0};
    for (int i = 0; i < 4; i++)
        Array_put(arr, i + 4, &extra[i]);

    /* 通过指针直接读取 */
    for (int i = 0; i < Array_length(arr); i++) {
        double *p = Array_get(arr, i);
        printf("arr[%d] = %g\n", i, *p);
    }

    /* 创建缩短的副本 */
    Array_T small = Array_copy(arr, 3);
    printf("small length = %d\n", Array_length(small)); /* 输出: 3 */

    Array_free(&small);
    Array_free(&arr);  /* arr 被置为 NULL */
    return 0;
}
```

---

## 10.8 Modern-C 改进

Hanson 的代码写于 C89 时代。以下是现代 C（C11/C17/C23）视角下的改进建议。

### 10.8.1 灵活数组成员（FAM）

C99 引入的灵活数组成员（flexible array member）可以将描述符和元素存储合并为**单次堆分配**：

```c
/* modern_array.h — C11 FAM 版本 */
struct ModernArray {
    int length;
    int size;
    char data[];       /* FAM: 编译时不占空间，运行时附加分配 */
};

ModernArray *ModernArray_new(int length, int size) {
    ModernArray *a = malloc(sizeof(ModernArray) + (long)length * size);
    if (!a) return NULL;
    a->length = length;
    a->size   = size;
    return a;
}
```

**优势：**
- 描述符和元素在一次 `malloc` 中分配，减少内存碎片
- 更好的缓存局部性（描述符和数据相邻）
- 一次 `free` 即可释放全部资源

**代价：**
- 无法将描述符栈分配或嵌入其他结构
- 无法单独 `realloc` 元素区（必须 `realloc` 整个结构体）

Hanson 的分离设计选择了**灵活性**而非**紧凑性**。两种方案各有适用场景，不存在绝对优劣。

### 10.8.2 `reallocarray` — 整数溢出安全

OpenBSD 引入的 `reallocarray`（现已被 glibc 2.26+ 和 musl 1.1.20+ 采纳）解决了 `length * size` 的整数溢出风险：

```c
/* 原始代码 — 存在溢出风险 */
RESIZE(array->array, length * array->size);

/* 现代改进 — 溢出安全 */
array->array = reallocarray(array->array, length, array->size);
```

`reallocarray` 内部检查乘法溢出，若 `length * size` 超过 `SIZE_MAX` 则返回 `NULL` 并设置 `errno = ENOMEM`，而非静默分配一块过小的内存。

### 10.8.3 `memcpy` vs 类型化访问

C 语言的严格别名规则（strict aliasing）允许编译器假设不同类型的指针不指向同一内存。Hanson 的 `char *array` + `memcpy` 模式天然遵守这一规则（`char *` 可以别名任何类型）。

现代 C 代码中，如果已知元素类型，可以使用 `_Generic` 提供类型安全的包装：

```c
#define Array_get_typed(arr, i, type) \
    (*(type *)Array_get((arr), (i)))
```

但这只是语法糖——底层的泛型机制仍然是 `void *` + `size`。

---

## 10.9 与 C++ `std::vector` 对比

Hanson 的 `Array_T` 初看像是 C 语言版的 `std::vector`，但二者的设计哲学有根本差异。

### 10.9.1 关键差异对照表

| 特性 | CII `Array_T` | C++ `std::vector<T>` |
|------|--------------|---------------------|
| 泛型机制 | `void *` + `size`（类型擦除） | 模板（编译期单态化） |
| 增长策略 | **无内置策略**，客户端自行管理 | 自动倍增（通常 ×1.5 或 ×2） |
| `resize` 语义 | 只调整 `length`，新元素**未初始化** | 新元素**值初始化**（基本类型→0） |
| `push_back` | 不存在 | 内置，自动扩容 |
| `size()` / `capacity()` | 只有 `length`，无 `capacity` 概念 | 区分 `size()` 和 `capacity()` |
| 内存释放 | `Array_free` 释放描述符 + 元素 | `vector` 析构自动释放；`shrink_to_fit` 可主动收缩 |
| 迭代器安全 | 返回裸指针；`resize` 后**失效**，无自动检测 | 迭代器失效规则明确；DEBUG 模式有检测 |
| 复制语义 | `Array_copy` 创建独立副本 | 拷贝构造/赋值运算符可被调用 |
| 异常安全 | 内存不足时通过 CII 异常机制处理 | RAII + 异常安全保证（strong guarantee） |
| 自定义分配器 | 不支持 | 通过模板参数支持 |
| 存储方式 | 描述符 + 数据**分离分配**（两次 `malloc`） | 连续内存块（描述符在栈上，数据在堆上） |

### 10.9.2 设计哲学分歧

**CII 的哲学：** "提供最小可用机制，策略由客户决定。" `Array_T` 只有 8 个函数，每个只做一件明确的事。增长策略、内存分配器、错误处理策略全部外置。

**STL 的哲学：** "提供完整解决方案，99% 的用例开箱即用。" `std::vector` 有 30+ 个成员函数，覆盖从 `push_back` 到 `emplace_back`（完美转发）、`shrink_to_fit`、`data()` 等细节。

对于系统编程场景，CII 哲学更优：你不想为一个你不使用的 `push_back` 自动扩容策略付出代码膨胀的代价。对于应用编程场景，STL 哲学更优：你不想每次增长数组都手写 5 行代码。

### 10.9.3 为什么 `vector` 有 `capacity` 而 `Array_T` 没有？

这是两者最深刻的设计差异。`std::vector` 维护两个大小：

```
vector<int> v;
v.size()     = 5   ← 逻辑元素个数
v.capacity() = 8   ← 物理分配的空间
```

`push_back` 的自动扩容正是基于 `capacity` 与 `size` 的差值判断。`Array_T` 只有一个 `length`，它既是逻辑大小也是物理大小。这意味着：
- `Array_T` 不需要维护额外状态（更简单）
- `Array_T` 无法在内部自动优化增长（更透明）
- `Array_T` 的 `length` 改变立刻触发 `realloc`（更可预测）

---

## 10.10 小结

Hanson 的动态数组实现只用约 80 行代码，但展示了多个超越代码本身的工程原则：

1. **接口/实现分离**：`array.h` 只暴露行为，`arrayrep.h` 只对需要表示访问的客户暴露结构定义。这是编译期封装治理的教科书案例。

2. **机制与策略分离**：`Array_resize` 提供重分配机制，增长策略由客户决定。这是 Unix 传统在 ADT 设计中的体现。

3. **描述符模式**：`Array_Rep_init` 集中控制字段赋值，使得未来添加新字段时只需修改一处。

4. **泛型通过类型擦除**：`void *` + `size` + `memcpy` 是 C 语言在没有模板的情况下最好的泛型实现方式，代价是类型安全完全由调用者负责。

5. **边界检查与性能的折中**：`assert` 在 DEBUG 模式提供安全网，在 RELEASE 模式编译为无操作。不妥协安全性，也不牺牲性能。

这些原则在 CII 库的后续章节（Table、Set、Seq 等）中反复出现，形成了一套一致且优雅的 C 语言 ADT 设计语言。

---

**源代码引用：**
- `/code/array.h` — 公共接口
- `/code/arrayrep.h` — 私有表示头
- `/code/array.c` — 实现
- `/code/mem.h` / `/code/mem.c` — 内存管理
- `/code/assert.h` — 断言系统

**参考文献：**
- Hanson, David R. _C Interfaces and Implementations: Techniques for Creating Reusable Software_. Addison-Wesley, 1997. Chapter 10.
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch09_集合_Set.html">第9章 集合 Set</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch11_序列_Seq.html">第11章 序列 Seq</a></div>
</div>
