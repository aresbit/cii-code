# 第14章 格式化输出 (Formatting)

---

## 目录

1. [设计动机：printf的四大缺陷](#1-设计动机printf的四大缺陷)
2. [核心设计思想：可扩展的printf替代品](#2-核心设计思想可扩展的printf替代品)
3. [接口全景：Fmt.h 逐行解析](#3-接口全景fmth-逐行解析)
4. [格式化状态机：Fmt_vfmt 深度剖析](#4-格式化状态机fmt_vfmt-深度剖析)
5. [转换函数注册与调用机制](#5-转换函数注册与调用机制)
6. [默认转换函数族](#6-默认转换函数族)
7. [缓冲管理与输出策略](#7-缓冲管理与输出策略)
8. [自定义格式化符：完整示例](#8-自定义格式化符完整示例)
9. [与标准printf的对比](#9-与标准printf的对比)
10. [Modern-C视角下的改进与反思](#10-modern-c视角下的改进与反思)

---

## 1. 设计动机：printf的四大缺陷

标准C库的`printf`、`fprintf`、`sprintf`、`vsprintf`系列函数无疑是C语言中使用最频繁的输出机制。它们采用格式串加变长参数的模式，通过`%c`形式的转换说明符控制参数格式化。然而，这些函数存在四个根本性缺陷：

**缺陷一：转换说明符集是封闭的。** 用户无法为自定义数据类型注册新的格式化代码。如果你的程序中定义了`struct Point { double x, y; }`，你无法通过`printf("%P", p)`来输出点坐标——`%P`并不存在，且无法扩展。

**缺陷二：输出目标是固定的。** 格式化结果只能打印到文件流(`printf`/`fprintf`)或写入字符串(`sprintf`/`vsprintf`)。无法将格式化结果发送到自定义的输出通道，如网络套接字、图形界面缓冲区或用户自定义的数据结构。

**缺陷三（最危险的缺陷）：无缓冲区边界检查。** `sprintf`和`vsprintf`不会检测输出是否会超出目标缓冲区。一旦格式化结果超过缓冲区容量，便发生栈溢出——这是C语言中最臭名昭著的安全漏洞来源之一。

**缺陷四：无参数类型检查。** 格式串中的`%d`期望`int`类型参数，但编译器无法验证实际传入的参数是否匹配。类型不匹配导致未定义行为，且在运行时难以诊断。

Fmt接口解决了前三个缺陷，并因此提供了比标准printf更为灵活、安全的格式化基础设施。

---

## 2. 核心设计思想：可扩展的printf替代品

### 2.1 策略模式：将"输出"和"格式化"解耦

Fmt的核心设计思想可以概括为一句话：**将字符的"生产"与字符的"消费"彻底分离。**

每一个转换函数不直接调用`putchar`或写入`FILE*`，而是通过一个统一的**回调接口**输出每个格式化后的字符：

```c
int put(int c, void *cl);
```

- `c`：被格式化为`unsigned char`的字符（始终为正整数）
- `cl`：客户端提供的不透明数据指针，原样透传
- 返回值：通常返回`c`本身

这个简单的抽象实现了惊人的灵活性：
- `cl`指向`FILE*`，`put`是`fputc` → 输出到文件
- `cl`指向`struct buf*`，`put`是`insert` → 写入定长缓冲区
- `cl`指向`struct buf*`，`put`是`append` → 写入自动扩容的动态字符串
- `cl`指向网络socket，`put`是`send`的包装 → 输出到网络

### 2.2 字符码到转换函数的映射表

Fmt维护一个包含256个槽位的函数指针数组`cvt[256]`：

```c
static T cvt[256];  // T = void (*)(int code, va_list_box *box,
                    //              int put(int c, void *cl), void *cl,
                    //              unsigned char flags[256],
                    //              int width, int precision);
```

每个格式代码对应的转换函数可以通过`Fmt_register`动态替换。新注册的转换函数立即对后续的`Fmt_fmt`调用生效。这种延迟绑定使得客户端可以在运行时临时覆盖标准行为（加入调试信息、更改输出格式等），并在使用后恢复原函数。

### 2.3 格式说明符的语法

Fmt支持与printf兼容的格式说明符语法：

```
% [flags] [width] [. precision] code
```

- **flags**: 合法的标志字符由`Fmt_flags`指向的字符串定义，默认为`"-+ 0"`。每个标志字符在flags数组中记录出现次数（而非简单的开/关，这为自定义转换函数提供了额外信息）。
- **width**: 字段最小宽度，可用`*`从参数列表中动态获取。省略或为`INT_MIN`时表示未指定。
- **precision**: 精度，由`.`引导。对`%s`限制最大输出字符数，对`%d`指定最少数字位数。同样支持`*`动态获取。
- **code**: 单字符格式代码（ASCII码值0-255）。

> 注意：该接口与源码实际实现之间存在一个关键差异。本书正文使用`va_list *app`直接传递`va_list`指针，而实际源码仓库（`fmt.h`/`fmt.c`）中使用了包装结构体`va_list_box`：
> ```c
> typedef struct va_list_box {
>     va_list ap;
> } va_list_box;
> ```
> 这是因为在某些平台上`va_list`被实现为数组类型，不能通过指针安全传递。`va_list_box`通过结构体包装绕过了这一限制——这是C语言中处理`va_list`跨函数传递的经典惯用法。

---

## 3. 接口全景：Fmt.h 逐行解析

以下以实际源码仓库中的`fmt.h`为基础，逐行进行中文注释。注意与书中版本的差异：书中使用`va_list *app`，实际代码中使用`va_list_box *box`。

```c
/* $Id$ */
#ifndef FMT_INCLUDED
#define FMT_INCLUDED

#include <stdarg.h>      /* va_list, va_start, va_arg, va_end */
#include <stdio.h>       /* FILE (for Fmt_fprint) */
#include "except.h"      /* Except_T, RAISE, etc. */

/* ── va_list_box: va_list 的包装结构体 ───────────────────────
 * 关键设计：有些平台上 va_list 是数组类型（如 x86-64 System V ABI），
 * 无法直接通过指针传递。将 va_list 包裹在结构体中后，结构体指针
 * 可以安全地跨越函数边界传递。这是整个Fmt接口可移植性的基石。
 */
typedef struct va_list_box {
    va_list ap;
} va_list_box;

/* ── 类型别名 T = Fmt_T ─────────────────────────────────────
 * 使用 #define 而非 typedef 的原因：CII全书统一采用这种风格，
 * 使得 ADT 类型的抽象名（Fmt_T）和实现中使用的短名（T）都能
 * 在同一个编译单元中共存——在暴露给客户端的头文件中用全名，
 * 在实现文件中用短名。最终通过 #undef T 清理。
 */
#define T Fmt_T

/* ── Fmt_T: 格式转换函数的类型签名 ─────────────────────────
 *
 * 参数列表详解：
 *   code       - 格式字符（ASCII码值，如 'd'=100, 's'=115）
 *   box        - 指向 va_list 包装的指针，用于 va_arg 取参
 *   put(c, cl) - 客户端输出回调：每输出一个字符时调用
 *   cl         - 客户端数据，原样透传给 put
 *   flags[256] - 标志计数数组，flags['-'] 表示 '-' 出现次数
 *   width      - 字段宽度；INT_MIN 表示未指定
 *   precision  - 精度；INT_MIN 表示未指定
 *
 * 无返回值（void）——所有输出通过 put 回调完成。
 */
typedef void (*T)(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[256], int width, int precision);

/* ── 全局变量与异常 ─────────────────────────────────────── */

/* Fmt_flags: 指向合法的标志字符集字符串。
 * 初始值: "-+ 0"
 * 设为 NULL 可禁用标志解析（极少使用）。
 * 这是全局可变状态——修改它会影响所有后续格式化调用。
 */
extern char *Fmt_flags;

/* Fmt_Overflow: 当 Fmt_sfmt/Fmt_vsfmt 输出超过缓冲区大小时抛出 */
extern const Except_T Fmt_Overflow;

/* ════════════════════════════════════════════════════════════
 *                   核心格式化函数
 * ════════════════════════════════════════════════════════════ */

/* Fmt_fmt: 通用格式化引擎
 *
 * 参数：
 *   put(c, cl) - 每字符回调函数
 *   cl         - 透传给 put 的客户端数据
 *   fmt        - 格式串（printf风格）
 *   ...        - 待格式化的变长参数
 *
 * 流程：
 *   1. va_start  初始化变长参数访问
 *   2. Fmt_vfmt  执行格式化（遍历格式串，分发转换函数）
 *   3. va_end    清理变长参数访问
 *
 * 示例：
 *   Fmt_fmt((int(*)(int,void*))fputc, stdout,
 *           "value = %d\n", 42);
 *   // 输出 "value = 42\n" 到标准输出
 */
extern void Fmt_fmt (int put(int c, void *cl), void *cl,
    const char *fmt, ...);

/* Fmt_vfmt: Fmt_fmt 的 va_list 版本
 *
 * 这是整个Fmt系统的心脏。所有其他格式化函数最终都调用它。
 * 参数 box 包含已初始化的 va_list，Fmt_vfmt 通过 va_arg(*box->ap)
 * 按需取出参数。
 *
 * 典型调用链：
 *   Fmt_print → Fmt_vfmt(outc, stdout, fmt, &box)
 *   Fmt_string → Fmt_vstring → Fmt_vfmt(append, &cl, fmt, box)
 *   Fmt_sfmt   → Fmt_vsfmt  → Fmt_vfmt(insert, &cl, fmt, box)
 */
extern void Fmt_vfmt(int put(int c, void *cl), void *cl,
    const char *fmt, va_list_box *box);

/* ════════════════════════════════════════════════════════════
 *                   便捷输出函数
 * ════════════════════════════════════════════════════════════ */

/* Fmt_print: 格式化输出到 stdout
 * 等价于 Fmt_fmt(outc, stdout, fmt, ...)
 */
extern void Fmt_print (const char *fmt, ...);

/* Fmt_fprint: 格式化输出到指定 FILE* 流
 * 等价于 Fmt_fmt(outc, stream, fmt, ...)
 * 其中 outc 内部调用 putc(c, f)
 */
extern void Fmt_fprint(FILE *stream,
    const char *fmt, ...);

/* Fmt_sfmt: 格式化到定长缓冲区（安全版 sprintf）
 *
 * 参数：
 *   buf  - 目标缓冲区（调用者提供）
 *   size - 缓冲区大小（必须 > 0）
 *   fmt  - 格式串
 *   ...  - 变长参数
 *
 * 返回：写入的字符数（不含结尾 '\0'）
 * RAISE Fmt_Overflow: 当格式化结果 ≥ size 时抛出异常
 *
 * 这是对 sprintf 不安全行为的直接修复。
 */
extern int Fmt_sfmt   (char *buf, int size,
    const char *fmt, ...);

/* Fmt_vsfmt: Fmt_sfmt 的 va_list 版本 */
extern int Fmt_vsfmt(char *buf, int size,
    const char *fmt, va_list_box *box);

/* Fmt_string: 格式化到动态分配的字符串（自动扩容）
 *
 * 返回：堆上分配的字符串，调用者负责 FREE
 * 内部初始分配 256 字节，超出时自动翻倍扩容。
 * 可以抛出 Mem_Failed。
 */
extern char *Fmt_string (const char *fmt, ...);

/* Fmt_vstring: Fmt_string 的 va_list 版本 */
extern char *Fmt_vstring(const char *fmt, va_list_box *box);

/* ════════════════════════════════════════════════════════════
 *                   扩展机制
 * ════════════════════════════════════════════════════════════ */

/* Fmt_register: 注册/替换格式转换函数
 *
 * 参数：
 *   code - 格式字符（1-255）
 *   cvt  - 新的转换函数
 * 返回：旧的转换函数（可用于恢复）
 *
 * 检查运行时错误：
 *   - code < 1 或 code > 255
 *   - 格式串使用未注册的转换字符时，assert(cvt[c]) 触发
 */
extern T Fmt_register(int code, T cvt);

/* ════════════════════════════════════════════════════════════
 *              供自定义转换函数使用的工具函数
 * ════════════════════════════════════════════════════════════ */

/* Fmt_putd: 输出有符号数字的字符串表示
 *
 * 按 d 转换的规则处理 str[0..len-1]（视为有符号数）：
 * - 识别前导的 '-' 或 '+' 作为符号
 * - 根据 flags 处理 +/空格/0/- 标志
 * - 处理宽度和精度（补零、左/右对齐）
 *
 * 这是自定义数值格式化器的标准输出后端。
 */
extern void Fmt_putd(const char *str, int len,
    int put(int c, void *cl), void *cl,
    unsigned char flags[256], int width, int precision);

/* Fmt_puts: 输出字符串
 *
 * 按 s 转换的规则处理 str[0..len-1]：
 * - precision 限制最大输出长度
 * - flags['-'] 控制左对齐
 * - width 指定最小字段宽度
 *
 * 这是自定义字符串格式化器的标准输出后端。
 */
extern void Fmt_puts(const char *str, int len,
    int put(int c, void *cl), void *cl,
    unsigned char flags[256], int width, int precision);

#undef T
#endif
```

---

## 4. 格式化状态机：Fmt_vfmt 深度剖析

`Fmt_vfmt`是整个系统的核心引擎。所有其他接口函数最终都将格式化任务委托给它。以下是其完整实现逻辑的逐步解说。

### 4.1 顶层循环

```c
void Fmt_vfmt(int put(int c, void *cl), void *cl,
    const char *fmt, va_list_box *box) {
    assert(put);
    assert(fmt);

    while (*fmt)
        if (*fmt != '%' || *++fmt == '%')
            put((unsigned char)*fmt++, cl);   // 普通字符或 %% → 直接输出
        else
            /* ── 处理一个格式说明符 ── */
            { ... }
}
```

主循环逐字符扫描格式串，分为两种情况：
- **非`%`字符或`%%`**：直接调用`put`输出字面量。`*++fmt == '%'`条件在检测到`%`后向前看一个字符，如果也是`%`，则输出单个`%`并跳过。
- **真正的格式说明符**：进入解析—分派流程。

### 4.2 格式说明符解析状态机

每个格式说明符的解析遵循以下状态序列：

```
状态0: 扫描 flags → 状态1
状态1: 扫描 width → 状态2
状态2: 扫描 .precision → 状态3
状态3: 读取 code → 查表 cvt[code] → 调用转换函数
```

**（1）标志扫描**

```c
unsigned char c, flags[256];
int width = INT_MIN, precision = INT_MIN;
memset(flags, '\0', sizeof flags);

if (Fmt_flags) {
    unsigned char c = *fmt;
    for ( ; c && strchr(Fmt_flags, c); c = *++fmt) {
        assert(flags[c] < 255);   // 同一标志最多出现255次
        flags[c]++;               // 累加计数而非设布尔值
    }
}
```

关键设计决策：`flags[c]`记录的是**出现次数**而非简单的0/1开关。这为自定义转换函数保留了灵活性——例如，`flags['\'']`出现次数可以控制千分位分隔符的插入频率。默认转换函数将任何正数视为"已设置"。

**（2）宽度解析**

```c
if (*fmt == '*' || isdigit(*fmt)) {
    int n;
    if (*fmt == '*') {
        n = va_arg(box->ap, int);     // 从参数列表取宽度
        assert(n != INT_MIN);          // INT_MIN 保留为哨兵值
        fmt++;
    } else
        for (n = 0; isdigit(*fmt); fmt++) {
            int d = *fmt - '0';
            assert(n <= (INT_MAX - d)/10);  // 防止溢出
            n = 10*n + d;
        }
    width = n;
}
```

溢出检测的巧妙之处：断言`n <= (INT_MAX - d)/10`等价于`10*n + d <= INT_MAX`，但避免了`10*n + d`在计算过程中溢出。这是C语言中安全整数运算的标准惯用法（David R. Hanson的特色风格）。

**（3）精度解析**

```c
if (*fmt == '.' && (*++fmt == '*' || isdigit(*fmt))) {
    int n;
    /* 同宽度解析逻辑 */
    precision = n;
}
```

注意`.`后无`*`或数字时，`.`被消费但precision保持`INT_MIN`，这被解释为"显式省略精度"。

**（4）代码分派**

```c
c = *fmt++;                     // 取格式字符（注意转为 unsigned char
                                // 以确保在0-255范围内索引）
assert(cvt[c]);                 // 检查该代码是否已注册转换函数
(*cvt[c])(c, box, put, cl,     // 调用转换函数
          flags, width, precision);
```

`c`被声明为`unsigned char`而非`char`是经过深思熟虑的——在`char`为有符号类型的平台上，字节值>=128的字符会被符号扩展为负数，导致`cvt`数组越界访问。

### 4.3 完整调用链图

```
用户代码
  │
  ├── Fmt_print(fmt, ...) ──→ Fmt_vfmt(outc, stdout, fmt, &box)
  │
  ├── Fmt_fprint(fp, fmt, ...) ──→ Fmt_vfmt(outc, fp, fmt, &box)
  │
  ├── Fmt_sfmt(buf, N, fmt, ...) ──→ Fmt_vsfmt(buf, N, fmt, &box)
  │                                       └── Fmt_vfmt(insert, &cl, fmt, box)
  │
  ├── Fmt_string(fmt, ...) ──→ Fmt_vstring(fmt, &box)
  │                                └── Fmt_vfmt(append, &cl, fmt, box)
  │
  └── Fmt_fmt(put, cl, fmt, ...) ──→ Fmt_vfmt(put, cl, fmt, &box)
                                          │
                              ┌───────────┴───────────┐
                              │  扫描格式串             │
                              │  解析 flags/width/prec  │
                              │  查表 cvt[code]         │
                              │  调用转换函数           │
                              │    └── put(c, cl) × N   │
                              └───────────────────────┘
```

---

## 5. 转换函数注册与调用机制

### 5.1 cvt[256]：函数指针调度表

```c
static T cvt[256] = {
 /*   0-  7 */ 0,     0, 0,     0,     0,     0,     0,     0,
 /*   8- 15 */ 0,     0, 0,     0,     0,     0,     0,     0,
 // ... 中间省略 ...
 /*  96-103 */ 0,     0, 0, cvt_c, cvt_d, cvt_f, cvt_f, cvt_f,
 /* 104-111 */ 0,     0, 0,     0,     0,     0,     0, cvt_o,
 /* 112-119 */ cvt_p, 0, 0, cvt_s,     0, cvt_u,     0,     0,
 /* 120-127 */ cvt_x, 0, 0,     0,     0,     0,     0,     0
};
```

ASCII码与默认转换函数的对应关系：
| ASCII | 字符 | 函数 | 说明 |
|-------|------|------|------|
| 99 | `c` | `cvt_c` | 单字符 |
| 100 | `d` | `cvt_d` | 有符号十进制整数 |
| 101 | `e` | `cvt_f` | 科学计数法浮点 |
| 102 | `f` | `cvt_f` | 定点浮点 |
| 103 | `g` | `cvt_f` | 通用浮点（自动选择e或f） |
| 111 | `o` | `cvt_o` | 无符号八进制 |
| 112 | `p` | `cvt_p` | 指针（十六进制） |
| 115 | `s` | `cvt_s` | 字符串 |
| 117 | `u` | `cvt_u` | 无符号十进制 |
| 120 | `x` | `cvt_x` | 无符号十六进制 |

未使用的槽位初始化为`0`（NULL），尝试使用未注册的格式代码将触发`assert(cvt[c])`失败。

### 5.2 Fmt_register：运行时替换

```c
T Fmt_register(int code, T newcvt) {
    T old;

    /* 边界检查：code 必须在 1-255 范围内
     * 限制 code >= 1 是因为 code 为 0（即 '\0'）不可能出现在
     * 格式说明符中（它是字符串终止符）
     */
    assert(0 < code
        && code < (int)(sizeof (cvt)/sizeof (cvt[0])));

    old = cvt[code];       // 保存旧函数（可能为 NULL）
    cvt[code] = newcvt;    // 安装新函数
    return old;            // 返回旧函数供调用者保存/恢复
}
```

典型的保存—覆盖—恢复模式：

```c
T old = Fmt_register('D', my_cvt_D);  // 安装自定义 %D
Fmt_print("result = %D\n", my_data);  // 使用新转换器
Fmt_register('D', old);               // 恢复原转换器
```

### 5.3 转换函数的完整协议

每个转换函数接收7个参数，它们是格式状态的完整快照：

| 参数 | 类型 | 含义 | 哨兵值 |
|------|------|------|--------|
| `code` | `int` | 触发本次调用的格式字符 | - |
| `box` | `va_list_box *` | 变长参数访问句柄 | - |
| `put` | `int (*)(int, void*)` | 字符输出回调 | - |
| `cl` | `void *` | 客户端数据 | - |
| `flags` | `unsigned char[256]` | 标志出现次数 | 全0=无标志 |
| `width` | `int` | 字段宽度 | `INT_MIN`=未指定 |
| `precision` | `int` | 精度 | `INT_MIN`=未指定 |

转换函数在使用`va_arg(box->ap, type)`取参后，还需负责处理`width`和`precision`为`INT_MIN`时的默认行为。`Fmt_putd`和`Fmt_puts`已经封装了这些标准化处理逻辑。

---

## 6. 默认转换函数族

### 6.1 字符串转换：cvt_s 与 Fmt_puts

`cvt_s`是最简单的默认转换函数，演示了转换函数的标准写法：

```c
static void cvt_s(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {
    char *str = va_arg(box->ap, char *);   // 取字符串参数
    assert(str);                             // NULL字符串是错误
    Fmt_puts(str, strlen(str), put, cl, flags,
        width, precision);                  // 委托给Fmt_puts
}
```

`Fmt_puts`的核心逻辑：
- 将`width == INT_MIN`标准化为0
- 负宽度 → 设置`flags['-']` + 取绝对值
- 显式precision时忽略`flags['0']`
- 若`precision < len`，截断输出长度
- 根据`flags['-']`决定前/后填充空格

### 6.2 有符号整数：cvt_d

```c
static void cvt_d(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {
    int val = va_arg(box->ap, int);
    unsigned m;
    char buf[43];          // 足够容纳任何int的十进制表示
    char *p = buf + sizeof buf;

    /* INT_MIN的绝对值不能用int表示，
     * 因此特殊处理：-(INT_MIN) = INT_MAX + 1U */
    if (val == INT_MIN)
        m = INT_MAX + 1U;
    else if (val < 0)
        m = -val;
    else
        m = val;

    /* 从低位向高位逐位生成，逆序填入buf尾部 */
    do
        *--p = m%10 + '0';
    while ((m /= 10) > 0);

    if (val < 0)
        *--p = '-';

    Fmt_putd(p, (buf + sizeof buf) - p, put, cl, flags,
        width, precision);
}
```

**INT_MIN特殊处理的原因**：在二进制补码表示中，`INT_MIN`（通常是-2147483648）的绝对值无法用`int`表示（因为正数范围到2147483647为止）。`INT_MAX + 1U`在无符号算术中精确等于`|INT_MIN|`。

**缓冲区大小43的计算**：`sizeof(int) * CHAR_BIT * 0.30103 + 3 ≈ 43`。其中`0.30103`是`log10(2)`，用于将比特数转换为十进制位数；`+3`包含符号位、`\0`和向上取整的安全边际。

### 6.3 无符号整数族：cvt_u / cvt_o / cvt_x

三者结构相同，仅在进位基数上有差别：

```
cvt_u: *--p = m%10 + '0';  m /= 10;     // 十进制
cvt_o: *--p = (m&0x7)+'0'; m >>= 3;    // 八进制（掩码+移位）
cvt_x: *--p = "0123456789abcdef"[m&0xf]; m >>= 4;  // 十六进制（查表）
```

八进制和十六进制利用位操作（`&0x7`、`>>=3`、`&0xf`、`>>=4`）避免了除法和取模运算，这在早期CPU上具有显著的性能优势。

### 6.4 指针转换：cvt_p

```c
static void cvt_p(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {
    unsigned long m = (unsigned long)va_arg(box->ap, void*);
    char buf[43];
    char *p = buf + sizeof buf;
    precision = INT_MIN;              // 强制忽略显式精度
    /* ... 同 cvt_x 的十六进制转换 ... */
}
```

指针首先被转为`unsigned long`而非`unsigned`——因为`unsigned`可能不够宽（如在64位平台上`unsigned`通常为32位，而指针为64位）。`precision = INT_MIN`强制忽略用户指定的精度值。

### 6.5 字符转换：cvt_c

```c
static void cvt_c(int code, va_list_box *box, ...) {
    if (width == INT_MIN) width = 0;       // 标准化
    if (width < 0) { flags['-'] = 1; width = -width; }
    if (!flags['-']) pad(width - 1, ' ');   // 右对齐：前导空格
    put((unsigned char)va_arg(box->ap, int), cl); // 输出字符
    if ( flags['-']) pad(width - 1, ' ');   // 左对齐：尾随空格
}
```

注意`va_arg(box->ap, int)`而非`va_arg(box->ap, char)`——C语言中，传递给可变参数的`char`和`short`会经历**默认参数提升**（default argument promotions），被自动提升为`int`。使用`char`类型取参会触发未定义行为。

### 6.6 浮点数转换：cvt_f

浮点转换是C语言库中最难正确实现的部分。Fmt采用了务实的折中方案：借用标准库`sprintf`完成实际的浮点数→字符串转换，自己只处理宽度、精度和标志。

```c
static void cvt_f(int code, va_list_box *box, ...) {
    /* 缓冲区大小计算：
     * DBL_MAX_10_EXP + 1  : 整数部分最大位数（如308+1）
     * +1                   : 小数点
     * +99                  : 小数部分最大位数（上限99）
     * +1                   : '\0'
     */
    char buf[DBL_MAX_10_EXP+1+1+99+1];

    if (precision < 0) precision = 6;   // 默认6位小数
    if (code == 'g' && precision == 0) precision = 1;  // %g特例

    {
        static char fmt[] = "%.dd?";    // 模板：%.NNc
        assert(precision <= 99);         // 精度上限99
        fmt[4] = code;                   // 填入格式字符e/f/g
        fmt[3] =      precision%10 + '0';    // 个位
        fmt[2] = (precision/10)%10 + '0';    // 十位
        sprintf(buf, fmt, va_arg(box->ap, double));
    }

    Fmt_putd(buf, strlen(buf), put, cl, flags, width, precision);
}
```

`fmt`数组的巧妙操作：通过修改字符串常量数组中的字符动态构造格式串`"%.NNc"`。精度限制为99使得该数组只需编译时已知大小。

### 6.7 pad 宏

```c
#define pad(n,c) do { int nn = (n); \
    while (nn-- > 0) \
        put((c), cl); } while (0)
```

使用`do { ... } while(0)`包装确保宏在任何上下文中表现为单条语句。`(n)`被复制到局部变量`nn`避免多次求值的副作用。

---

## 7. 缓冲管理与输出策略

Fmt巧用三个不同的`put`回调函数实现了三种截然不同的输出策略，展示了回调模式的力量。

### 7.1 outc：流输出

```c
static int outc(int c, void *cl) {
    FILE *f = cl;
    return putc(c, f);
}
```

最直接的策略——直接将字符写入`FILE*`流。`cl`就是`FILE*`指针本身。被`Fmt_print`和`Fmt_fprint`使用。

### 7.2 insert：定长缓冲（安全模式）

```c
struct buf {
    char *buf;   // 缓冲区首地址
    char *bp;    // 当前写入位置（beyond pointer）
    int size;    // 缓冲区总容量
};

static int insert(int c, void *cl) {
    struct buf *p = cl;
    if (p->bp >= p->buf + p->size)
        RAISE(Fmt_Overflow);     // ← 关键：拒绝溢出
    *p->bp++ = c;
    return c;
}
```

`insert`直接解决了`sprintf`的缓冲区溢出问题。溢出触发异常而非静默破坏内存——这在安全关键代码中是不可妥协的底线。被`Fmt_sfmt`和`Fmt_vsfmt`使用。

### 7.3 append：动态扩容

```c
static int append(int c, void *cl) {
    struct buf *p = cl;
    if (p->bp >= p->buf + p->size) {
        RESIZE(p->buf, 2*p->size);    // 容量翻倍
        p->bp = p->buf + p->size;      // bp 指向新空间起始处
        p->size *= 2;
    }
    *p->bp++ = c;
    return c;
}
```

`append`实现了类似C++ `std::string`或Java `StringBuilder`的动态扩容语义。初始容量256字节，每次翻倍，避免了多次realloc的碎片化和性能损失。被`Fmt_string`和`Fmt_vstring`使用。

格式化结束后，`Fmt_vstring`调用`RESIZE(cl.buf, cl.bp - cl.buf)`将内存收窄至精确长度，避免浪费。

---

## 8. 自定义格式化符：完整示例

### 8.1 示例一：%B —— 二进制整数输出

将无符号整数以二进制形式输出，支持宽度、`-`左对齐和`0`填充。

```c
#include "fmt.h"
#include <string.h>
#include <limits.h>

static void cvt_b(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {

    unsigned m = va_arg(box->ap, unsigned);
    /* unsigned 最大位数 = sizeof(unsigned) * CHAR_BIT */
    char buf[sizeof(unsigned) * CHAR_BIT + 1];
    char *p = buf + sizeof(buf);

    /* 生成二进制数字（从低位到高位） */
    do {
        *--p = (m & 1) + '0';
    } while ((m >>= 1) != 0);

    /* 复用 Fmt_putd 处理宽度、对齐和填充 */
    Fmt_putd(p, (buf + sizeof(buf)) - p, put, cl, flags,
             width, precision);
}

/* 注册示例 */
void register_binary(void) {
    Fmt_register('B', cvt_b);
}

/* 使用示例 */
/*
 * Fmt_print("255 in binary: %B\n", 255);
 * // 输出: 255 in binary: 11111111
 *
 * Fmt_print("%016B\n", 42);
 * // 输出: 0000000000101010
 *
 * Fmt_print("%-16B|\n", 42);
 * // 输出: 101010          |
 */
```

### 8.2 示例二：%R —— IEEE 754 浮点数内存表示

输出`float`或`double`的底层二进制表示（符号位+指数+尾数），用于教学或调试。

```c
#include "fmt.h"
#include <string.h>

/* 将32位值的二进制表示填入buf */
static int float32_bits(char *buf, float val) {
    union { float f; unsigned u; } u;
    u.f = val;
    unsigned m = u.u;
    int i;
    /* 从高位到低位生成 */
    for (i = 0; i < 32; i++) {
        buf[31 - i] = ((m >> i) & 1) + '0';
    }
    return 32;
}

static void cvt_R(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {

    float val = (float)va_arg(box->ap, double);
    char bits[35]; /* 32位 + 分隔符空格 + \0 */
    int len = float32_bits(bits, val);

    /* 插入分隔符使结果可读：
     * 格式: s eeeeeeee mmmmmmmmmmmmmmmmmmmmmmm */
    char formatted[40];
    int pos = 0;
    formatted[pos++] = bits[0];             /* 符号位 */
    formatted[pos++] = ' ';
    int i;
    for (i = 1; i <= 8; i++)
        formatted[pos++] = bits[i];         /* 指数 */
    formatted[pos++] = ' ';
    for (i = 9; i < 32; i++)
        formatted[pos++] = bits[i];         /* 尾数 */
    formatted[pos] = '\0';

    Fmt_puts(formatted, pos, put, cl, flags, width, precision);
}

/* 注册示例 */
void register_float_bits(void) {
    Fmt_register('R', cvt_R);
}

/* 使用示例 */
/*
 * Fmt_print("3.14 in IEEE 754: %R\n", 3.14f);
 * // 输出: 3.14 in IEEE 754: 0 10000000 10010001111010111000011
 */
```

### 8.3 示例三：%M —— 内存大小（人类可读）

将字节数格式化为人类可读的形式（类似`ls -lh`），支持自动选择最合适的单位。

```c
#include "fmt.h"
#include <string.h>
#include <limits.h>

static void cvt_M(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {

    unsigned long long bytes = va_arg(box->ap, unsigned long long);
    static const char *units[] = {"B", "KiB", "MiB", "GiB", "TiB", "PiB"};
    static const int num_units =
        (int)(sizeof(units) / sizeof(units[0]));
    double val = (double)bytes;
    int unit_idx = 0;

    /* 根据大小级别缩放到合适的单位 */
    while (val >= 1024.0 && unit_idx < num_units - 1) {
        val /= 1024.0;
        unit_idx++;
    }

    char buf[64];
    if (unit_idx == 0) {
        /* 字节级别：整数显示 */
        Fmt_sfmt(buf, sizeof buf, "%llu %s",
                 bytes, units[unit_idx]);
    } else {
        /* KiB及以上：保留1位小数 */
        Fmt_sfmt(buf, sizeof buf, "%.1f %s",
                 val, units[unit_idx]);
    }

    Fmt_puts(buf, (int)strlen(buf), put, cl, flags, width, precision);
}

/* 注册示例 */
void register_human_size(void) {
    Fmt_register('M', cvt_M);
}

/* 使用示例 */
/*
 * Fmt_print("File size: %M\n", 0ULL);
 * // 输出: File size: 0 B
 *
 * Fmt_print("File size: %M\n", 1024ULL);
 * // 输出: File size: 1.0 KiB
 *
 * Fmt_print("File size: %M\n", 1536000ULL);
 * // 输出: File size: 1.5 MiB
 *
 * Fmt_print("File size: %M\n", 4294967296ULL);
 * // 输出: File size: 4.0 GiB
 */
```

### 8.4 示例四：%T —— 高精度时间戳

输出带有微秒精度的时间戳格式（用于日志系统）。

```c
#include "fmt.h"
#include <string.h>
#include <time.h>

static void cvt_T(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {

    struct timespec *ts = va_arg(box->ap, struct timespec *);
    assert(ts);

    time_t sec = ts->tv_sec;
    long nsec = ts->tv_nsec;
    struct tm tm_info;

    localtime_r(&sec, &tm_info);

    char buf[64];
    Fmt_sfmt(buf, sizeof buf,
        "%04d-%02d-%02d %02d:%02d:%02d.%06ld",
        tm_info.tm_year + 1900, tm_info.tm_mon + 1,
        tm_info.tm_mday,
        tm_info.tm_hour, tm_info.tm_min, tm_info.tm_sec,
        nsec / 1000);  /* 纳秒 → 微秒 */

    Fmt_puts(buf, (int)strlen(buf), put, cl, flags, width, precision);
}

/* 注册示例 */
void register_timestamp(void) {
    Fmt_register('T', cvt_T);
}

/* 使用示例 */
/*
 * struct timespec ts;
 * clock_gettime(CLOCK_REALTIME, &ts);
 * Fmt_print("[%T] Server started\n", &ts);
 * // 输出: [2024-03-15 14:30:22.123456] Server started
 */
```

---

## 9. 与标准printf的对比

### 9.1 功能矩阵

| 特性 | 标准printf | Fmt接口 |
|------|-----------|---------|
| 基本格式化`%d %s %f %x` | 支持 | 支持（子集） |
| `%n`（写入已输出字符数） | 支持 | **不支持** |
| `%e %E %g %G`大小写 | 支持 | 仅小写（限制精度≤99） |
| 自定义格式化符 | **不支持** | 支持（`Fmt_register`） |
| 自定义输出目标 | 仅FILE*或固定缓冲区 | 任意回调函数 |
| 缓冲区溢出保护 | sprintf/vsprintf无保护 | `Fmt_sfmt`抛出`Fmt_Overflow` |
| 动态扩容字符串 | 无 | `Fmt_string`（自动翻倍） |
| 参数类型安全 | 无 | 无（继承printf的缺陷） |
| 编译期格式串检查 | 无（需编译器扩展） | 无 |
| `va_list`跨平台安全 | 不适用 | `va_list_box`包装保证 |
| 标志重复次数信息 | 丢失 | 保留（`flags[c]`为出现次数） |

### 9.2 类型安全性的根本局限

Fmt无法修复printf的第四个缺陷（类型安全），因为这是C语言变长参数机制的本质限制。任何基于`va_arg`的格式化系统在编译期都无法验证格式串与实际参数类型的匹配。解决这个问题的唯一途径是：

- **编译器扩展**：GCC的`__attribute__((format(printf, ...)))`和Clang的`-Wformat`可以在编译期检查printf族函数。
- **宏预处理**：现代C++库（如`{fmt}`、`std::format`）利用模板和`constexpr`在编译期完成格式串解析和类型验证。
- **预处理器技巧**：如在编译期展开格式串为结构化的参数列表（但这在纯C中极端困难）。

### 9.3 有意的简化

Fmt有意省略了标准printf的一些特性：

- `%n`——写入已输出字符数。这被视为安全隐患（允许写入任意内存地址）而在Fmt中被移除。
- `%e/E/G`大小写变体——Fmt仅支持小写，保持了接口的简洁性。
- `%i`（等价于`%d`）——被省略，因为冗余。
- 浮点精度上限99——使得缓冲区大小在编译时可知，避免动态分配。

---

## 10. Modern-C视角下的改进与反思

Fmt接口设计于1990年代中期（CII第一版出版于1997年）。三十年来C语言及其生态发生了显著变化。以下从Modern-C的视角审视Fmt的优势与局限。

### 10.1 snprintf：标准库的追赶

C99引入了`snprintf`——一个具有缓冲区大小参数的sprintf安全替代品：

```c
int snprintf(char *str, size_t size, const char *format, ...);
```

`Fmt_sfmt`在概念上等同于`snprintf`，但有两个关键区别：

- **溢出行为**：`snprintf`截断输出并返回"本应写入的字符数"（如果缓冲区够大），而`Fmt_sfmt`抛出`Fmt_Overflow`异常。截断语义更适合防御性编程（"尽量显示"），异常语义更适合断言式编程（"要么正确，要么失败"）。
- **size为0**：`snprintf`允许`size=0`（此时仅计算长度不写入），`Fmt_sfmt`将其视为运行时错误。

**Modern-C建议**：在生产代码中优先使用`snprintf`（它是标准库的一部分，所有C99+编译器都支持），但`Fmt_sfmt`的异常语义在内部工具和教学场景中仍有其价值。

### 10.2 {fmt} / std::format：终极替代方案

C++20的`std::format`和其底层库`{fmt}`代表了格式化系统的现代标准：

```cpp
// {fmt} / C++20
std::string s = std::format("The answer is {}.", 42);
// 编译期格式串检查（C++20 constexpr）
// 类型安全（模板推导）
// 自定义类型的特化：template<> struct std::formatter<Point> {...};
```

{fmt}相比Fmt的优势：

| {fmt} 特性 | Fmt 对应 |
|------------|---------|
| 编译期格式串验证 | 无（运行时`assert(cvt[c])`） |
| 编译期类型检查 | 无（变长参数的固有限制） |
| Python风格的`{}`占位符 | C风格的`%c` |
| 自定义类型的`formatter<>`特化 | `Fmt_register(code, cvt)` |
| 运行时性能（SIMD优化） | 依赖sprintf进行浮点转换 |
| 异常安全（无异常抛出） | 异常或断言失败 |
| Unicode支持 | 无（`unsigned char`逐字节输出） |

**Modern-C 反思**：Fmt的核心架构（回调输出 + 注册表分派）在概念上与{fmt}的扩展机制是相通的——{fmt}的`formatter<>`特化本质上也是一种"注册表"，只是它的注册表在编译期通过模板特化完成，避免了运行时的间接调用开销。

### 10.3 编译期格式串检查：GCC/Clang的format属性

GCC和Clang通过`__attribute__((format(printf, ...)))`提供了编译期格式串检查。但该机制有两个根本局限性：

1. **仅适用于printf标准格式代码**，无法检查自定义格式代码。
2. **不检查va_list版本**（如`Fmt_vfmt`），因为编译器无法静态分析va_list的内容。

要让编译器检查Fmt的自定义格式串，需要编写编译器插件或使用Clang的`__attribute__((annotate(...)))`+静态分析工具。这在实践中很少有项目去做。

### 10.4 Fmt设计中的永恒价值

尽管存在上述局限，Fmt接口中有几个设计决策经得起时间考验：

**（1）回调输出模式（put函数指针）**

这是Fmt最深远的设计贡献。现代C++中的输出迭代器（output iterator）、Rust中的`std::fmt::Write` trait、Go中的`io.Writer`接口在本质上都是同一模式的不同语法糖。回调输出使格式化器与存储策略完全解耦，这种关注点分离是任何可扩展I/O库的基础。

**（2）注册表分派表**

`cvt[256]`函数指针数组对应当今C++中的虚函数表（vtable）。它用最小的运行时开销（一次数组索引+一次间接调用）实现了开放-封闭原则：对扩展开放（注册新代码），对修改封闭（核心循环不变）。

**（3）宽度和精度的哨兵值**

用`INT_MIN`标记"未指定"避免了单独的布尔标志变量，减少了参数数量。这是C语言中处理可选参数的经典惯用法。现代C++通过`std::optional<int>`解决同一问题，但代价是额外的存储和分支。

**（4）va_list_box的包装**

`va_list`作为数组类型的可移植性问题到今天依然存在。`va_list_box`的包装方案仍然是C语言跨平台va_list传递的标准答案——它被记录在POSIX规范和多个编译器文档中。

### 10.5 改进方向

如果要重新设计Fmt以适应当代C语言实践（C11/C17），可考虑以下改进：

1. **添加`Fmt_nfmt`（带最大宽度）**：在`Fmt_fmt`的基础上增加总输出字符数上限，防止无限输出。
2. **支持`%n`的安全替代品**：通过输出参数返回字符数，而非写入内存。
3. **使用`_Generic`提供有限的类型安全**：C11的`_Generic`可以用宏包装`Fmt_fmt`，对已知类型在编译期发出警告。
4. **Unicode感知输出**：将`put(int c, void *cl)`改为`put(uint32_t codepoint, void *cl)`以支持完整的Unicode码位。
5. **移除全局可变状态**：`Fmt_flags`全局变量应改为每个`Fmt_vfmt`调用的参数或上下文结构体的一部分。

---

## 附录A：默认转换说明符速查表

| 说明符 | 参数类型 | 行为描述 |
|--------|---------|---------|
| `%c` | `int` | 输出单个字符（由`int`转为`unsigned char`） |
| `%d` | `int` | 有符号十进制。precision指定最少位数。`+`/空格控制符号。 |
| `%u` | `unsigned` | 无符号十进制 |
| `%o` | `unsigned` | 无符号八进制 |
| `%x` | `unsigned` | 无符号十六进制（小写abcdef） |
| `%f` | `double` | 定点小数 x.y。默认6位小数。精度为0时省略小数点。 |
| `%e` | `double` | 科学计数法 x.ye±pp。x恒为1位，p恒为2位。 |
| `%g` | `double` | 自动选择`%f`或`%e`。precision为有效位数，默认1。 |
| `%p` | `void *` | 指针值的十六进制表示 |
| `%s` | `char *` | 输出字符串至`\0`或precision个字符。仅`-`标志有效。 |

**标志（`Fmt_flags = "-+ 0"`）**：

| 标志 | 含义 |
|------|------|
| `-` | 左对齐（在字段宽度内）。默认右对齐。 |
| `+` | 有符号转换始终输出`+`或`-`前缀（`+`优先于空格）。 |
| ` ` | 有符号转换：正数前输出空格（若`+`未出现）。 |
| `0` | 数值转换：用`'0'`而非`' '`填充至字段宽度（若precision未指定）。 |

---

## 附录B：完整API速查

| 函数 | 用途 | 输出目标 | 溢出行为 |
|------|------|---------|---------|
| `Fmt_fmt` | 通用格式化 | 用户`put`回调 | 取决于`put` |
| `Fmt_vfmt` | `Fmt_fmt`的va_list版 | 用户`put`回调 | 取决于`put` |
| `Fmt_print` | 格式化到stdout | 标准输出 | 取决于流 |
| `Fmt_fprint` | 格式化到FILE* | 指定文件流 | 取决于流 |
| `Fmt_sfmt` | 格式化到定长缓冲区 | 用户buf | 超过size→`Fmt_Overflow` |
| `Fmt_vsfmt` | `Fmt_sfmt`的va_list版 | 用户buf | 超过size→`Fmt_Overflow` |
| `Fmt_string` | 格式化到动态字符串 | 堆分配 | 自动扩容(可能`Mem_Failed`) |
| `Fmt_vstring` | `Fmt_string`的va_list版 | 堆分配 | 自动扩容(可能`Mem_Failed`) |
| `Fmt_register` | 注册转换函数 | - | code越界→断言失败 |
| `Fmt_putd` | 工具：输出有符号数字 | 由调用者的`put`决定 | 取决于`put` |
| `Fmt_puts` | 工具：输出字符串 | 由调用者的`put`决定 | 取决于`put` |

---

> **参考文献**  
> Hanson, David R. *C Interfaces and Implementations: Techniques for Creating Reusable Software.* Addison-Wesley, 1997. Chapter 14: Formatting, pp. 215-239.
>
> Plauger, P. J. *The Standard C Library.* Prentice Hall, 1992.
>
> Steele, Guy L. and Jon L White. "How to Print Floating-Point Numbers Accurately." *Proceedings of the ACM SIGPLAN 1990 Conference on Programming Language Design and Implementation*, pp. 112-126.
>
> Clinger, William D. "How to Read Floating-Point Numbers Accurately." *Proceedings of the ACM SIGPLAN 1990 Conference on Programming Language Design and Implementation*, pp. 92-101.
>
> {fmt} library. https://github.com/fmtlib/fmt
