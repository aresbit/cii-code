# 第1章 引言 (Introduction)

> 译自 David R. Hanson, *C Interfaces and Implementations: Techniques for Creating Reusable Software*, Chapter 1.
>
> 本章为全书的纲领性引言：阐明CII库的设计哲学、Literate Programming 方法论、接口与实现的分离原则、代码命名约定，并从 Modern-C (C11/C17) 的视角审视改进方向。

---

## 1.1 一个动机示例：`double.c`

Hanson 以一段简洁但存在设计缺陷的程序开篇，用以展示"随意编写的C代码"与"基于接口精心设计的C代码"之间的鸿沟。

`code/double.c` 是一个检测文本文件中连续重复单词的工具：

```c
/* code/double.c -- 检测文件中连续重复的单词 */
static char rcsid[] = "$Id: double.c 6 2007-01-22 00:45:22Z drhanson $";
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <ctype.h>
#include <string.h>
int linenum;
int getword(FILE *, char *, int);
void doubleword(char *, FILE *);

int main(int argc, char *argv[]) {
    int i;
    for (i = 1; i < argc; i++) {
        FILE *fp = fopen(argv[i], "r");
        if (fp == NULL) {
            fprintf(stderr, "%s: can't open '%s' (%s)\n",
                argv[0], argv[i], strerror(errno));
            return EXIT_FAILURE;
        } else {
            doubleword(argv[i], fp);
            fclose(fp);
        }
    }
    if (argc == 1) doubleword(NULL, stdin);
    return EXIT_SUCCESS;
}

int getword(FILE *fp, char *buf, int size) {
    int c;
    c = getc(fp);
    for ( ; c != EOF && isspace(c); c = getc(fp))
        if (c == '\n')
            linenum++;
    {
        int i = 0;
        for ( ; c != EOF && !isspace(c); c = getc(fp))
            if (i < size - 1)
                buf[i++] = tolower(c);
        if (i < size)
            buf[i] = '\0';
    }
    if (c != EOF)
        ungetc(c, fp);
    return buf[0] != '\0';
}

void doubleword(char *name, FILE *fp) {
    char prev[128], word[128];
    linenum = 1;
    prev[0] = '\0';
    while (getword(fp, word, sizeof word)) {
        if (isalpha(word[0]) && strcmp(prev, word) == 0)
            {
                if (name)
                    printf("%s:", name);
                printf("%d: %s\n", linenum, word);
            }
        strcpy(prev, word);
    }
}
```

这段代码能工作，但 Hanson 指出了若干结构性问题：

- **全局变量污染**：`linenum` 是全局变量，任何函数都可以修改它，形成了隐式的、脆弱的耦合。
- **固定大小的缓冲区**：`prev[128]` 和 `word[128]` 是硬编码的魔数，遇到超长单词时行为未定义。
- **无接口抽象**：`getword` 函数直接操作裸 `FILE*`，与文件I/O紧耦合，无法复用到其他输入源（如字符串、网络流）。
- **职责混杂**：`doubleword` 函数同时承担了单词读取、重复检测和输出格式化三项职责。

Hanson 的目标是展示：通过适当的数据抽象和接口设计，同样的功能可以用更清晰、更安全、更可复用的代码实现。全书 20 章正是构建这一套**可复用C组件库**的完整旅程。

---

## 1.2 CII 全书概览与设计哲学

### 1.2.1 全书二十章结构

CII 全书 20 章可分为五个逻辑区块：

| 区块 | 章节 | 主题 |
|------|------|------|
| **基础设施** | Ch 1-6 | 引言、接口/实现原则、原子、异常与断言、内存管理 |
| **数据结构** | Ch 7-13 | 链表、表、集合、动态数组、序列、环形缓冲区、位向量 |
| **字符串** | Ch 14-16 | 格式化、低级字符串、高级字符串 |
| **算术** | Ch 17-19 | 扩展精度、任意精度、多精度算术 |
| **并发** | Ch 20 | 线程与同步原语 |

本代码库涵盖了这个完整的接口体系，共计 **25 个 `.h` 头文件**，每个文件定义一个独立模块的公共接口。

### 1.2.2 核心设计哲学

CII 的设计哲学可以用以下原则概括：

**原则一：接口就是承诺 (An interface is a contract)**

每个 `.h` 头文件定义一份明确的契约：它声明了模块导出的类型、函数和常量，并隐含了调用这些函数的先决条件（preconditions）、后置条件（postconditions）和不变式（invariants）。客户端代码只通过接口访问模块，绝不触及实现细节。

**原则二：不透明指针即封装 (Opaque pointers are encapsulation)**

这是CII库中最核心的封装技术。在头文件中，类型被声明为指向不完整结构体的指针；在实现文件中，结构体的完整定义才被给出。客户端永远无法直接访问结构体成员，从而强制了信息的单向流动。

以 `code/table.h` 为例：

```c
/* code/table.h -- 哈希表接口 */
#define T Table_T
typedef struct T *T;

extern T    Table_new (int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *key));
extern void Table_free(T *table);
extern void *Table_put   (T table, const void *key, void *value);
extern void *Table_get   (T table, const void *key);
extern void *Table_remove(T table, const void *key);
extern int   Table_length(T table);
extern void  Table_map   (T table,
    void apply(const void *key, void **value, void *cl), void *cl);
extern void **Table_toArray(T table, void *end);
#undef T
```

客户端声明 `Table_T t = Table_new(500, cmp, hash)` —— 但 `struct Table_T` 的内部结构（桶数组大小、装载因子、冲突解决策略）对客户端完全透明。这使得实现者可以在不破坏客户端二进制兼容性的前提下，自由地更换哈希函数、调整重哈希策略、甚至将链地址法替换为开放寻址法。

**原则三：每一个模块对自己分配的资源负责**

CII 统一的资源管理模式是：`M_new()` 分配并返回不透明指针，`M_free(T *p)` 释放并将指针置NULL。不存在"部分释放"或"需要客户端手动管理内部缓冲区"的情况。这是对标准C库——`malloc/free` 必须配对、`fopen/fclose` 必须配对——这种碎片化管理方式的系统性改进。

**原则四：错误通过异常传播，而非返回值**

Hanson 在 CII 中设计了一套基于 `setjmp/longjmp` 的异常处理宏：`TRY/EXCEPT/ELSE/FINALLY/END_TRY`。这是全书最具争议的设计决策之一，我们将在 1.6 节从 Modern-C 视角讨论其得失，并在第4章详细分析其内部机制。

**原则五：可复用组件应形成正交的语言层**

CII 本质上是在标准C之上构造了一个微型的可复用抽象层。它的模块之间具有清晰的分层依赖关系：

- 底层：`mem.h`（内存分配）、`except.h`（异常机制）、`assert.h`（断言）
- 中层：`atom.h`（不可变字符串）、`arena.h`（基于区域的内存）、`list.h`（链表）、`table.h`（哈希表）
- 高层：`set.h`（基于Table的集合）、`seq.h`（动态序列）、`text.h`（高级字符串）
- 应用层：`ap.h`（任意精度算术，依赖 `xp.h`）、`fmt.h`（格式化）、`thread.h`（线程）

这种分层结构使得每个模块可以作为独立组件被复用，而不是被锁定在一个单片框架中。

### 1.2.3 一份典型的模块清单

为帮助读者建立整体印象，以下是本代码库中全部接口模块的简要说明：

| 模块 | 头文件 | 功能 |
|------|--------|------|
| Arith | `arith.h` | 整数算术辅助（min/max/div/mod 的标准化版本） |
| Atom | `atom.h` | 不可变字符串原子表，基于哈希表实现，用于字符串驻留 |
| Except | `except.h` | 基于 `setjmp/longjmp` 的结构化异常处理 |
| Assert | `assert.h` | 断言机制，与Except集成 |
| Mem | `mem.h` | 带文件/行号追踪的内存分配器 |
| Arena | `arena.h` | 基于区域的内存管理，支持批量释放 |
| List | `list.h` | 单链表，支持 push/pop/append/reverse/map |
| Table | `table.h` | 通用哈希表，支持任意键类型 |
| Set | `set.h` | 集合抽象，基于 Table 实现 |
| Array | `array.h` | 动态数组，支持任意元素大小 |
| Seq | `seq.h` | 动态序列（双端可增长的数组） |
| Ring | `ring.h` | 环形双端队列 |
| Bit | `bit.h` | 位向量，支持集合操作 |
| Fmt | `fmt.h` | 可扩展的格式化输出框架 |
| Str | `str.h` | 带有明确位置索引的低级字符串操作 |
| Text | `text.h` | 带长度编码的高级字符串（类似现代C中的 `string_view`） |
| XP | `xp.h` | 扩展精度无符号整数算术（底层） |
| AP | `ap.h` | 任意精度有符号整数算术（基于 XP） |
| MP | `mp.h` | 多精度无符号整数算术 |
| Thread | `thread.h` | 协作式用户态线程 |
| Sem | `sem.h` | 计数信号量 |
| Chan | `chan.h` | 同步通道（CSP风格的线程通信） |

---

## 1.3 面向接口编程 vs 面向对象编程

### 1.3.1 方法论的本质差异

Hanson 的 CII 展示了一种与面向对象编程（OOP）平行的抽象范式。二者的根本区别在于**数据的归属**：

| 维度 | 面向对象编程 (OOP) | 面向接口编程 (CII风格) |
|------|-------------------|----------------------|
| **数据的拥有者** | 对象（数据+方法绑定在一起） | 接口（数据属于调用者，操作由模块提供） |
| **封装机制** | `private` 成员 + 方法 | 不透明指针（`struct` 定义仅在 `.c` 中可见） |
| **多态机制** | 虚函数表 (vtable) | 函数指针参数（`cmp`/`hash`/`apply`） |
| **继承** | 类继承/接口继承 | 一组独立模块的**组合使用** |
| **析构** | 析构函数（自动或手动） | 显式 `M_free(T *p)`，指针置NULL |
| **内存模型** | 对象内聚（一次分配） | 不透明指针包装内部结构（内部可能多次分配） |

### 1.3.2 CII 中的"多态"——函数指针即协议

在 C++/Java 中，多态通过类和接口实现。在 CII 中，多态通过**函数指针参数**实现。

以 `code/table.h` 为例：

```c
extern T Table_new(int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *key));
```

`cmp` 和 `hash` 这两个函数指针是客户端必须提供的**回调协议**。`Table` 模块定义了"如何哈希/如何比较"的接口约定，但不规定"整数"还是"字符串"——由客户端传入的函数指针决定。

这种设计在 `code/set.h` 中也得到了精确的复现：

```c
extern T Set_new(int hint,
    int cmp(const void *x, const void *y),
    unsigned hash(const void *x));
```

`Table` 和 `Set` 共享了相同的函数指针签名约定，但又彼此独立 —— 这正是"面向接口"而非"面向继承"的设计。

在运行时行为的定制方面，CII通过 `apply` 回调实现遍历：

```c
/* code/list.h */
extern void List_map(T list,
    void apply(void **x, void *cl), void *cl);
```

这相当于 OOP 中的 `visitor` 模式或函数式编程中的 `map`，但通过原始函数指针实现。`void *cl` 是一个通用的闭包指针（closure pointer），用于向回调函数传递上下文——这是C语言中缺乏真正闭包时的标准工法。

### 1.3.3 为什么不直接用 C++ 或 Java？

Hanson 在第1章中明确回答了这个问题——不是出于对C的固执偏好，而是基于工程现实：

1. **移植性**：在1997年原书写作时，C++ 编译器在不同平台上的行为差异很大。C89 是真正意义上的通用标准。
2. **运行时开销**：C++ 的虚函数表、RTTI、异常展开都带来不可忽视的间接成本。嵌入式系统和内核代码往往禁止使用这些特性。
3. **互操作性**：C ABI 是所有语言的通用分母。一个设计良好的C接口可以被Python（`ctypes`/`cffi`）、Rust（`extern "C"`）、Lua（`luaL_Reg`）直接调用，而 C++ 的 name mangling 构成了一道人为的壁垒。
4. **极简主义**：Hanson 展示了**仅仅依靠结构体、函数指针和预处理器宏**，C语言完全能做到与OOP语言同等的表达能力。

---

## 1.4 Literate Programming 方法论

### 1.4.1 什么是 Literate Programming

*Literate Programming*（文学化编程）是 Donald E. Knuth 在 1984 年提出的编程范式。其核心主张是：**程序应当被写给人阅读，只是顺便也能被机器编译执行**。

传统编程中，代码的结构由编译器决定（函数、文件、模块）；Literate Programming 中，代码的结构由**人类的思维展开顺序**决定。一个 Literate Program 是一篇按叙事逻辑组织的散文，源代码片段则被嵌入到叙述中。

这一方法论在本书中的体现是双重的：

1. **原书本身的格式**：Hanson 的原书使用 `noweb` 工具编写。每一章节既是技术论述，又包含了可以直接 `tangle`（抽取）为可编译 `.c`/`.h` 文件的代码片段。而 `weave`（编织）则生成可供印刷的 TeX 文档。

2. **代码的注释质量**：观察任一 `.c` 文件中的代码，你会注意到注释极少。这是 Literate Programming 的哲学——**代码本身即是最终的产物，解释代码的文字写在书里，不写在代码里**。`.c` 文件中的注释仅保留了版本标识（`static char rcsid[] = "$Id$"`）。

### 1.4.2 Tangle 和 Weave

在 Knuth 的系统中：

- **Tangle**（解绕）：从文学化源文件中抽取代码片段，按定义顺序拼接成可编译文件。
- **Weave**（编织）：将同一源文件转换为带有交叉引用、索引和排版的格式化文档。

`noweb` 管道示例：

```sh
notangle -Rdouble.c ch01.nw > double.c
noweave -index ch01.nw > ch01.tex
```

这意味着本书中出现的每行代码都不是"摘录"——它们是直接由源码文件驱动的。Hanson 保证了书中展示的代码与实际编译运行的代码完全一致。

### 1.4.3 在 CII 代码库中的体现

本代码库中的文件结构直接反映了这一方法论：

- `.h` 文件是模块的"文学化声明"部分
- `.c` 文件是模块的"文学化实现"部分

以 `code/atom.c` 为例，其中 `Atom_new` 函数（79-104行）实现了原子字符串的驻留逻辑，它使用了一个2048槽的哈希表（`buckets[2048]`）和一个预计算的散射表（`scatter[]`）。这段代码本身不使用行内注释解释算法，因为它假定读者已经通过阅读书中的对应章节（第3章）理解了其设计意图和数据结构的选择理由。

### 1.4.4 Literate Programming 的当代回声

虽然 `noweb` / `CWEB` 工具链在今天已不常见，但 Literate Programming 的核心思想在多个现代实践中以不同形式存活下来：

- **Jupyter Notebook**：代码 + Markdown叙述的交替编排
- **Rust doc-tests**：文档注释中的代码示例会被自动编译和测试
- **Quarto / R Markdown**：面向数据科学的文学化工作流
- **mdBook / Doxygen**：从注释生成文档

---

## 1.5 CII的代码风格与命名约定

### 1.5.1 唯一标识符策略

C语言没有命名空间机制。在大型项目中，不同模块极有可能定义同名的函数或类型，导致链接冲突。CII 采用了一套简单而高效的**前缀命名约定**来模拟命名空间：

```
<Module>_<function>        -- 导出的公共函数
<Module>_<Constant>        -- 导出的公共常量/异常
<T>                        -- 在头文件内部使用的局部类型别名
```

每个模块名是其所属 `.h` 文件名的首字母大写形式：

| 文件名 | 模块名 | 类型名 | 示例函数 |
|--------|--------|--------|----------|
| `atom.h` | Atom | `Atom_T` | `Atom_new`, `Atom_string` |
| `list.h` | List | `List_T` | `List_push`, `List_append` |
| `table.h` | Table | `Table_T` | `Table_new`, `Table_get` |
| `stack.h` | Stack | `Stack_T` | `Stack_new`, `Stack_pop` |
| `seq.h` | Seq | `Seq_T` | `Seq_new`, `Seq_addhi` |

### 1.5.2 哨兵宏 (`_INCLUDED`)

每个头文件都使用唯一的哨兵宏来防止重复 `#include`：

```c
#ifndef <MODULE>_INCLUDED
#define <MODULE>_INCLUDED
/* ... declarations ... */
#endif
```

例如：
- `atom.h` → `ATOM_INCLUDED`
- `list.h` → `LIST_INCLUDED`
- `except.h` → `EXCEPT_INCLUDED`

### 1.5.3 缩写规则

CII 的函数命名追求简洁而不牺牲可读性。下列缩写贯穿整个库：

| 缩写 | 全称 | 含义 |
|------|------|------|
| `lo` | low | 低位/前端/最小值 |
| `hi` | high | 高位/后端/最大值 |
| `addlo` | add low | 在前端/左侧添加元素 |
| `addhi` | add high | 在后端/右侧添加元素 |
| `remlo` | remove low | 从前端/左侧移除元素 |
| `remhi` | remove high | 从后端/右侧移除元素 |
| `sub` | substring | 子字符串提取 |
| `cmp` | compare | 比较函数 |
| `fmt` | format | 格式化 |

这些缩写在整个代码库中保持了令人惊叹的一致性。一旦学会 `addlo/addhi/remlo/remhi` 这四个词，你就掌握了 `seq.h`、`ring.h` 和其他所有双端数据结构的命名模式。

### 1.5.4 统一的资源管理模式

所有CII模块遵循相同的资源管理生命周期：

```c
/* 创建 (allocate) */
extern T M_new(/* parameters */);

/* 销毁 (deallocate, sets pointer to NULL) */
extern void M_free(T *p);
```

注意 `M_free` 接受的是 `T *`（即指向指针的指针），而不是 `T`。这使得释放后指针自动置NULL成为可能，有效防止悬垂指针（use-after-free）的错误。对于像 `Table_free` 这样的操作，当指针置NULL时，其内部的所有资源也在同一调用中被递归释放。

### 1.5.5 函数命名模式

浏览本代码库中所有的 `.h` 文件可以归纳出以下规律：

- **构造函数**：`Module_new(...)` — 返回新实例
- **析构函数**：`Module_free(T *p)` — 释放资源并置NULL指针
- **查询函数**：`Module_length(T)`, `Module_get(T, i)` — 查询属性或元素
- **操作函数**：`Module_put(T, ...)`, `Module_push(T, ...)` — 修改实例状态
- **遍历/映射**：`Module_map(T, apply, cl)` — 基于回调函数的高阶遍历
- **转换函数**：`Module_toArray(T, end)` — 将内部数据结构转换为数组

### 1.5.6 示例：`Sem_T` 中的 `LOCK/END_LOCK` 宏

`code/sem.h` 提供了一个简洁而优雅的临界区保护模式：

```c
#define LOCK(mutex) do { Sem_T *_yymutex = &(mutex); \
    Sem_wait(_yymutex);
#define END_LOCK Sem_signal(_yymutex); } while (0)
```

使用时无需显式调用 `wait/signal`：

```c
LOCK(mutex)
    /* critical section -- automatically protected */
    shared_counter++;
END_LOCK
```

这种 `do { ... } while (0)` 的宏包装是C语言中构造块级作用域作用宏的经典技法，既安全（可以在任何需要语句的地方使用），又不产生悬挂的 `else` 问题。

---

## 1.6 Modern-C (C11/C17) 视角的改进方向

CII 原书出版于 1997 年，基于 C89 标准。此后C语言经历了 C99、C11、C17 三次标准化迭代，引入了大量语言和标准库的改进。从今天的眼光审视，CII 的很多设计选择既有永恒的美感，也暴露出一些时代局限性。

### 1.6.1 可自然取代的设计

**`Arith_max` / `Arith_min` → 泛型宏**  
C11 引入了 `_Generic` 关键字，可以直接写出类型安全的泛型 `max`/`min` 宏，无需为每种类型编写不同函数：

```c
// Modern-C 风格（C11+）
#define max(a, b) _Generic((a), int: max_int, double: max_double)(a, b)
```

CII 的 `Arith_max(int x, int y)` 仅支持 `int` 类型。

**`Atom_new` → C11 字符串字面量合并**  
C11 没有直接提供字符串驻留的语法糖，但编译器的字符串池优化已经隐式完成了 `Atom_new` 的部分功能。不过 `Atom_new` 的动态驻留（运行时构造字符串的共享）至今没有标准语言替代方案。

**`Mem.h` 内存追踪 → Address Sanitizer (ASan)**  
`mem.h` 在 `Mem_alloc` 中记录了 `__FILE__` 和 `__LINE__` 以辅助内存泄漏调试。这在1997年是先进的。如今 ASan 和 Valgrind 提供了更全面且零代码侵入的内存错误检测，包括堆缓冲区溢出、栈溢出、use-after-free 和 use-after-return。`mem.h` 的堆追踪器在调试场景下已被这些工具覆盖。

### 1.6.2 值得保留但需要适配的设计

**`TRY/EXCEPT` 异常宏**  
这是 CII 中最具争议的设计。基于 `setjmp/longjmp` 的异常机制在 C89 中是一个创造性的突破，但在现代C实践中有几个已知局限：

1. **与 C++ 异常的交互缺失**：在 C/C++ 混合项目中，`longjmp` 穿越 C++ 栈帧时不会触发析构函数，容易导致资源泄漏。
2. **线程安全性差**：`setjmp/longjmp` 依赖全局/线程局部的跳转缓冲区，在信号处理器和线程环境中行为不总是可预测。
3. **现代替代方案**：C11 的 `thread_local` 存储类可以显著简化 `Except_stack` 的线程安全实现。如果重新设计，可以将 TRY/EXCEPT 转而构建于 C11 `<threads.h>` 之上。

尽管如此，TRY/EXCEPT 的接口契约——如 `RAISE`、`RERAISE`、`FINALLY`——仍然是资源安全释放的有效保障，其设计理念在 Rust 的 `?` 操作符和 Go 的 `defer` 中都有回响。

**不透明指针的 const-correctness**  
CII 的函数声明几乎不使用 `const`。例如 `Table_get` 的声明是：

```c
extern void *Table_get(T table, const void *key);
```

按现代C的最佳实践，应该写成：

```c
extern const void *Table_get(const T table, const void *key);
//     ^^^^^                ^^^^^
//     返回值不可修改         table 本身不会被修改
```

C89 对 `const` 的支持受限，但在 C11/C17 中，`const` 修饰符应当被系统性地应用到所有不会被修改的参数和返回值上。这既是一种文档化手段（自说明的契约），也允许编译器做更多优化。

### 1.6.3 全新可能

**`_Atomic` 与无锁数据结构**  
C11 引入了 `_Atomic` 类型限定符和 `<stdatomic.h>`。CII 的 `thread.h` 和 `sem.h` 使用用户态线程和协程切换，没有涉及真正的多核并发。在现代C中，可以将 `Chan.h` 重新实现为无锁的单生产者单消费者（SPSC）队列：

```c
// Modern-C 风格，使用 C11 atomics
#include <stdatomic.h>
struct chan {
    _Atomic size_t head;
    _Atomic size_t tail;
    void *buffer[CHAN_SIZE];
};
```

`_Atomic` 消除了对繁重的互斥锁的需求，使得跨平台的高性能并发通道成为可能，且不需要依赖 `setjmp/longjmp` 风格的上下文切换。

**`static_assert` 编译期断言**  
C11 的 `static_assert`（在 `<assert.h>` 中定义）可以在编译时验证常量表达式。对于CII中大量依赖运行时 `assert()` 检查的不变式——如 `sizeof(int) == 4` 或结构体对齐——`static_assert` 可以更早地捕获错误：

```c
// Modern-C: 编译期检查类型大小假设
static_assert(sizeof(struct atom) <= 64,
    "atom structure exceeds cache line");
```

CII 的 `assert.h` 中只有运行时的 `assert(e)`，没有任何编译期检查。

**`_Alignas` 对齐控制**  
CII 的 `arena.h` 在分配内存时需要手动确保对齐。C11 的 `_Alignas` 和 `alignof` 提供了标准的、可移植的对齐控制，可以消除 `arena.c` 中手动的对齐计算。

### 1.6.4 不被C标准直接覆盖的部分

即使是在 2026 年，CII 中仍有大量组件在现代C中找不到**直接的**标准替代方案。这部分恰好是CII最有价值、最值得学习的部分：

- **`Arena.h`**：C11没有提供基于区域的内存分配器。基于区域的分配在编译器和游戏引擎中仍被广泛使用。
- **`Table.h` / `Set.h`**：标准C库仍没有哈希表。每个新的C项目都在重复发明 `dict`。
- **`Text.h`**：带长度编码的字符串（`{len, str}`），这是 `std::string_view` 的C语言版本，C标准库至今不提供。
- **`Fmt.h`**：C的标准 `printf` 族无法扩展自定义格式说明符。`Fmt_register` 提供了一种闭源世界没有的标准扩展机制。
- **`AP.h` / `MP.h`**：任意精度整数运算没有进入C标准，只有GMP这样的第三方库。
- **`Except.h`**：虽然其实现方式有待商榷，但结构化异常处理这一需求在C语言中仍然真实存在，且标准库至今无解。

这些组件恰恰是CII为什么在今天仍然具有生命力的原因：它填补了C标准库中最显眼的空白。

---

## 1.7 如何使用本书

### 1.7.1 建议的阅读路径

- **第1-6章**（基础设施）应顺序阅读。异常机制和内存管理是全书其余所有模块的依赖基础。
- **第7-13章**（数据结构）可以选择与你的需求相关的章节。每一章独立讲解一个数据结构接口，可以单独阅读而不必按顺序。
- **第14-16章**（字符串）推荐顺序阅读。`Str.h` 提供了函数式风格的字符串操作基础，`Text.h` 在此基础上构建了带长度编码的安全字符串。
- **第17-19章**（算术）可以独立阅读，但推荐先读第17章（XP）再读第18章（AP），因为AP直接构建于XP之上。
- **第20章**（线程）自成一体，可以在任何阶段阅读。

### 1.7.2 代码在何处

本代码库中的 `code/` 目录包含了全书所有接口头文件（`.h`）和实现文件（`.c`）。每章的相关代码在 `README.md` 中有详细的索引。你可以：

```bash
# 编译所有模块
cd code && make

# 运行 double.c 示例
gcc -o double double.c && ./double test.txt

# 查看某个接口的定义
cat code/stack.h
```

---

## 1.8 章结

CII 不是一本关于C语言语法的书。它是一本关于**在C语言中用最小的机制实现最大的抽象**的书。Hanson没有试图将C变成C++，而是展示了：一个设计良好的接口、一个完全隐藏的实现、和一个基于不透明指针的封装约定，足以构成软件复用的坚实基础。

在接下来的章节中，我们将逐模块剖析这个体系。第2章从"接口与实现"的形式化定义开始，正式进入CII的世界。

---

> **下一章**: [第2章 接口与实现 (Interfaces and Implementations)](ch02_接口与实现_Interfaces.md)
>
> **原书信息**: David R. Hanson, *C Interfaces and Implementations: Techniques for Creating Reusable Software*, Addison-Wesley Professional, 1997. ISBN: 0-201-49841-3.
>
> **代码库**: 本章代码对应 `code/double.c`，亦见原书第1章 pp. 1-14。
