# 第19章 多精度计算器 (Calc)

> **原书**: _C Interfaces and Implementations_, David R. Hanson  
> **本章**: 多精度算术接口 (MP) 及其在终端计算器中的应用  
> **源代码文件**: `arith.h`, `arith.c`, `calc.c`, `mpcalc.c`, `mp.h`, `mp.c`, `xp.h`, `seq.h`, `except.h`

---

## 19.1 引言：从 calc 到 mpcalc

CII 全书以三个梯度的算术接口作为"集大成"的展示：第17章的 **XP** (eXtended Precision) 提供裸字节序列的多精度运算原语，第18章的 **AP** (Arbitrary Precision) 在 XP 之上封装了符号-绝对值表示、自动内存管理和 Fmt 格式化，而第19章的 **MP** (Multiple Precision) 则用 n 位二进制补码表示，同时导出**有符号和无符号**两套完整的算术函数族。

本章以**终端多精度计算器** `mpcalc` 为应用范例，展示 MP 接口在实际程序中的用法。`mpcalc` 的前身是 `calc`（基于 AP 的 RPN 计算器），二者均采用**波兰后缀表示法**（Reverse Polish Notation, RPN）：

```
100 99 1 - p
  =>  1
```

值被压入栈，运算符从栈中弹出操作数并将结果压回。没有括号，没有优先级——这正是 RPN 的核心魅力：**语法分析器简化为一个字符级的 switch-case 分发**。

本章将按照以下路径展开：

1. 先梳理 `arith.h` 提供的**可移植整数语义基础**——除法和取模在负数上的行为在 C89 中是实现定义的，`Arith_div` 和 `Arith_mod` 将其标准化为**向负无穷截断**，这恰好是 MP 除法语义所需。
2. 逐段剖析 `mpcalc.c` 的 REPL 交互循环、词法数字收集和运算符分发。
3. 从概念上对比 RPN 直接求值与中缀表达式解析的区别，在 "Modern-C 改进" 一节中详细讨论 **Pratt 解析**（运算符绑定强度）和**递归下降**如何将中缀输入转换为 AST 再求值。
4. 最后，从模块组合的视角审视 `mpcalc` 如何成为 CII 全书各章接口的汇合点——Seq、Mem、Fmt、Except、XP、MP 六大模块协同工作，展现了 Hanson 接口设计哲学的完整兑现。

---

## 19.2 基础篇：可移植整数运算 — arith.h

`arith.h` (原书第2章) 只导出六个函数，代码合计不足三十行，但它的存在解决了 C 语言一个历史遗留问题：**除法和取模在涉及负数操作数时，C89 标准允许实现选择向零截断或向负无穷截断**。

### 19.2.1 接口定义

```c
/* arith.h */
extern int Arith_max(int x, int y);
extern int Arith_min(int x, int y);
extern int Arith_div(int x, int y);
extern int Arith_mod(int x, int y);
extern int Arith_ceiling(int x, int y);
extern int Arith_floor  (int x, int y);
```

### 19.2.2 核心实现：向负无穷的除法和取模

```c
/* arith.c — 关键逻辑 */
int Arith_div(int x, int y) {
    if (-13/5 == -2
    &&  (x < 0) != (y < 0) && x%y != 0)
        return x/y - 1;
    else
        return x/y;
}

int Arith_mod(int x, int y) {
    if (-13/5 == -2
    &&  (x < 0) != (y < 0) && x%y != 0)
        return x%y + y;
    else
        return x%y;
}
```

设计要点：

1. **编译期检测**：`-13/5 == -2` 在编译期即可求值。若为真，说明当前 C 实现的除法**向零截断**（如 x86），此时 Arith_div 需要在 `x` 和 `y` 异号且不能整除时，将商减 1，从而得到向负无穷的截断。
2. **不变式保证**：`Arith_div(x, y) * y + Arith_mod(x, y) == x` 始终成立。
3. **Arith_mod 的余数始终非负**：`0 <= Arith_mod(x, y) < |y|`。

### 19.2.3 与 MP 的关系

MP 接口的带符号除法 `MP_div` 和取模 `MP_mod` 明确规定**截断方向为负无穷**，余数始终非负。这恰好与 `Arith_div` / `Arith_mod` 的语义对齐。`mpcalc` 虽然直接使用的是 `MP_div` / `MP_mod`（大整数层），但它们的语义约定来自同一个数学传统，而 `arith.h` 在 C 语言层面为这个传统提供了可移植的实现。

> **关键认知**：`arith.h` 是"语义基础设施"。它不提供解析、不构建 AST、不管理优先级——它只保证**算术本身的定义是明确且可移植的**。任何构建在其上的表达式求值器，都因此获得跨平台一致性。

---

## 19.3 MP 接口：多精度算术概览

MP 接口是 CII 中最大的一章：**49 个函数 + 2 个异常**。

### 19.3.1 表示

```
MP_T ≡ unsigned char *

n-bit 整数存储于 n/8 字节，小端序（最低有效字节在前）。
比特 n-1 为符号位，采用二进制补码表示。
```

关键内部变量（由 `MP_set(n)` 设定）：

| 变量 | 含义 | 32-bit 示例 |
|------|------|------------|
| `nbits` | 位宽 | 32 |
| `nbytes` | `(nbits-1)/8 + 1` | 4 |
| `shift` | `(nbits-1) % 8` | 7 |
| `msb` | `ones(nbits)`, 即低 `shift+1` 位为 1 的掩码 | 0xFF |

符号位的访问宏：

```c
#define sign(x) ((x)[nbytes-1] >> shift)
```

### 19.3.2 两类异常

```c
extern const Except_T MP_Overflow;      // 溢出
extern const Except_T MP_Dividebyzero;  // 除零
```

关键约定：**所有 MP 函数在抛出异常前先完成结果的计算和赋值**。这意味着调用者可以用 `TRY-EXCEPT` 捕获异常并安全地使用截断后的结果：

```c
TRY
    MP_fromintu(z, 0xFFF);   // nbits=8 时溢出
EXCEPT(MP_Overflow) ;
END_TRY;
// z 中保留了低 8 位的 0xFF
```

### 19.3.3 函数族一览

| 类别 | 有符号 | 无符号 | 说明 |
|------|--------|--------|------|
| 创建/转换 | `MP_new`, `MP_fromint`, `MP_fromintu`, `MP_toint`, `MP_tointu`, `MP_cvt`, `MP_cvtu` | | 初始化和类型转换 |
| 加减乘除取模 | `MP_add`, `MP_sub`, `MP_mul`, `MP_div`, `MP_mod`, `MP_neg` | `MP_addu`, `MP_subu`, `MP_mulu`, `MP_divu`, `MP_modu` | 基本算术 |
| 双倍乘法 | `MP_mul2` | `MP_mul2u` | 2n-bit 积（不溢出）|
| 立即数版本 | `MP_addi`, `MP_subi`, `MP_muli`, `MP_divi`, `MP_modi` | `MP_addui`, `MP_subui`, `MP_mului`, `MP_divui`, `MP_modui` | 第二个操作数为 long/unsigned long |
| 比较 | `MP_cmp`, `MP_cmpi` | `MP_cmpu`, `MP_cmpui` | -1/0/+1 |
| 位运算 | `MP_and`, `MP_or`, `MP_xor`, `MP_not` | `MP_andi`, `MP_ori`, `MP_xori` | 逐位 |
| 移位 | `MP_lshift`, `MP_rshift`, `MP_ashift` | | 逻辑/算术移位 |
| 字符串 | `MP_fromstr`, `MP_tostr`, `MP_fmt`, `MP_fmtu` | | 与 base 2..36 的相互转换 |

---

## 19.4 calc.c：基于 AP 的 RPN 计算器

在深入 `mpcalc` 之前，理解其前身 `calc.c` 的结构至关重要。

### 19.4.1 整体架构

`calc` 的核心是一个**单字符驱动的 REPL 循环**：

```
                 ┌──────────────┐
  stdin ──────►  │   getchar()  │
                 └──────┬───────┘
                        │ c
                        ▼
            ┌──────────────────────┐
            │   switch(c) 分发      │
            ├──────────────────────┤
            │ '0'..'9': 收集数字   │──► AP_fromstr() ──► Stack_push()
            │ '+':  pop, pop, add  │──► Stack_push()
            │ '-':  pop, pop, sub  │──► Stack_push()
            │ 'p':  打印栈顶       │──► Fmt_print()
            │ 'f':  打印全栈       │
            │ 'c':  清空栈         │
            │ 'q':  退出           │
            └──────────────────────┘
```

### 19.4.2 关键代码片段

`pop()` 函数始终返回一个 `AP_T`，即使栈为空也返回 `AP_new(0)`，从而简化了运算符处理中的错误检查：

```c
AP_T pop(void) {
    if (!Stack_empty(sp))
        return Stack_pop(sp);
    else {
        Fmt_fprint(stderr, "?stack underflow\n");
        return AP_new(0);
    }
}
```

数字收集逻辑：

```c
case '0': case '1': ... case '9': {
    char buf[512];
    {
        int i = 0;
        for ( ; c != EOF && isdigit(c); c = getchar(), i++)
            if (i < (int)sizeof(buf) - 1)
                buf[i] = c;
        if (i > (int)sizeof(buf) - 1) {
            i = (int)sizeof(buf) - 1;
            Fmt_fprint(stderr,
                "?integer constant exceeds %d digits\n", i);
        }
        buf[i] = 0;
        if (c != EOF)
            ungetc(c, stdin);
    }
    Stack_push(sp, AP_fromstr(buf, 10, NULL));
    break;
}
```

**词法分析**在这里退化为主循环中的字符级硬编码：数字字符触发数字收集，空白字符被忽略，其他字符按运算符处理。没有独立的 tokenizer，没有符号表，没有 AST——RPN 的计算模型使得这一切都变得多余。

### 19.4.3 RPN 语义

`calc` 的操作符集合：

| 字符 | 语义 |
|------|------|
| `~` | 取负 |
| `+ - * / %` | 基本算术 |
| `^` | 幂运算 |
| `d` | 复制栈顶 |
| `p` | 打印栈顶 |
| `f` | 打印全栈 |
| `c` | 清空栈 |
| `q` | 退出 |

一条典型的 `calc` 会话：

```
$ calc
100 99 - p
1
10 20 + 3 * p
90
^D
```

---

## 19.5 mpcalc.c：多精度 REPL 计算器

`mpcalc` 的结构与 `calc` 几乎同构，但在以下方面进行了扩展：

1. **多精度**：使用 `MP_T` 替代 `AP_T`，支持可配置的 n 位宽度
2. **有符号/无符号双模式**：根据输出基数自动切换
3. **异常处理**：通过 `TRY-EXCEPT` 捕获 `MP_Overflow` 和 `MP_Dividebyzero`
4. **动态栈**：用 `Seq_T` 替代 `Stack_T`，支持 `Seq_get` 直接索引
5. **可变输入/输出基数**：`i` 和 `o` 命令支持 2..36 进制
6. **位运算和移位**：新增 `& | ^ ! < >` 操作符

### 19.5.1 REPL 主循环的完整结构

```
main()
│
├── Seq_new(0)                      ← 初始化栈
├── Fmt_register('D', MP_fmt)      ← 注册有符号打印格式
├── Fmt_register('U', MP_fmtu)     ← 注册无符号打印格式
│
└── while ((c = getchar()) != EOF)
    │
    ├── volatile MP_T x=NULL, y=NULL, z=NULL
    │
    ├── TRY ───────────────────────────────────────┐
    │   switch(c)                                   │
    │   ├─ ' ' '\t' '\n' ...  : break  (忽略空白)    │
    │   ├─ '0'..'9'           : 收集数字, MP_fromstr│
    │   ├─ '+' '-' '*' '/' '%': pop×2, MP_xxx()     │
    │   ├─ '&' '|' '^'        : pop×2, 逐位运算     │
    │   ├─ '!' '~'            : pop×1, MP_not/neg   │
    │   ├─ '<' '>'            : pop×2, MP_l/rshift  │
    │   ├─ 'p' 'f'            : Fmt_print 打印      │
    │   ├─ 'i' 'o'            : 设置输入/输出基数    │
    │   ├─ 'k'                : MP_set 精度          │
    │   ├─ 'd'                : 复制栈顶             │
    │   ├─ 'c'                : 清空栈               │
    │   ├─ 'q'                : 清理并退出           │
    │   └─ default            : 报错 "is unimplemented"
    │
    ├── EXCEPT(MP_Overflow)    ─── Fmt_fprint("?overflow\n")
    ├── EXCEPT(MP_Dividebyzero)─── Fmt_fprint("?divide by 0\n")
    └── END_TRY
        │
        ├── if (z) Seq_addhi(sp, z)  ← 结果压栈
        ├── FREE(x), FREE(y)         ← 释放操作数
        └── loop
```

### 19.5.2 有符号/无符号模式切换

`mpcalc` 最精巧的设计之一是 `f` 指针——它指向一个函数指针结构体，根据输出基数 (`obase`) 自动选择有符号或无符号的算术函数族：

```c
struct {
    const char *fmt;
    MP_T (*add)(MP_T, MP_T, MP_T);
    MP_T (*sub)(MP_T, MP_T, MP_T);
    MP_T (*mul)(MP_T, MP_T, MP_T);
    MP_T (*div)(MP_T, MP_T, MP_T);
    MP_T (*mod)(MP_T, MP_T, MP_T);
} s = { "%D\n",
    MP_add,  MP_sub,  MP_mul,  MP_div,  MP_mod  },
  u = { "%U\n",
    MP_addu, MP_subu, MP_mulu, MP_divu, MP_modu },
 *f = &s;  // 默认有符号
```

当用户执行 `oi` 命令改变基数时：

```c
case 'i': case 'o': {
    long n;
    x = pop();
    n = MP_toint(x);
    if (n < 2 || n > 36)
        Fmt_fprint(stderr, "?%d is an illegal base\n", n);
    else if (c == 'i')
        ibase = n;
    else
        obase = n;
    if (obase == 2 || obase == 8 || obase == 16)
        f = &u;   // 切换为无符号模式
    else
        f = &s;   // 切换为有符号模式
    break;
}
```

这解释了为什么在 `obase` 为 2/8/16 时，操作符 `+ - * / %` 和 `p` / `f` 打印执行的是无符号运算——因为二进制/八进制/十六进制通常用于表示位模式而非有符号数值。

### 19.5.3 任意进制数字收集

与 `calc` 只用 `isdigit()` 不同，`mpcalc` 支持 2..36 进制的数字收集：

```c
/* strchr 查询字符串的第 (36-ibase) 个字符起：
   对于 ibase=10 → 从下标 26 开始 → "9876543210"
   对于 ibase=16 → 从下标 20 开始 → "fedcba9876543210"
   对于 ibase=36 → 从下标 0 开始  → "zyxwvutsrqponmlkjihgfedcba9876543210"
*/
strchr(&"zyxwvutsrqponmlkjihgfedcba9876543210"[36-ibase],
       tolower(c))
```

这是一个精巧的查找表技巧：字符串字面量包含所有 36 个有效数字字符，通过偏移 `36 - ibase` 跳过超出当前基数的前缀，`strchr` 返回非 NULL 表示 `c` 是有效的数字字符。

### 19.5.4 shift 操作符的特殊处理

```c
case '<': { long s;
    y = pop(); z = pop();
    s = MP_toint(y);
    if (s < 0 || s > INT_MAX) {
        Fmt_fprint(stderr,
            "?%d is an illegal shift amount\n", s);
        break;  // 不修改 z，z 将被重新压栈
    }
    MP_lshift(z, z, s);
    break;
}
```

注意：如果移位量非法，`z` 未被修改，而主循环末尾的 `if (z) Seq_addhi(sp, z)` 会将**原始值**重新压栈——这实现了一种优雅的错误恢复：非法操作被静默忽略，操作数不受影响。

---

## 19.6 词法分析与语法分析的协作

### 19.6.1 RPN 下的"免解析"模型

在 RPN 计算器中，**词法分析和语法分析是同一件事**。原因如下：

1. **没有中缀歧义**：`1 2 + 3 *` 在 RPN 中只有一种解释：`(1 + 2) * 3`。操作符总是紧跟在操作数之后，不需要括号，不需要优先级表。
2. **不需要前瞻**：每个字符确定一个操作。数字字符触发数字收集（循环读直到非数字），非数字字符立即作为操作符分发。最大前瞻距离 = 1 个字符。
3. **不需要构建 AST**：计算在执行时即时发生。栈本身就是"活着的 AST"——栈上的值代表了所有尚未被消费的子表达式结果。

**RPN 求值等价于对表达式树的后序遍历时即时计算**：

```
中缀: (1 + 2) * 3
语法树:        RPN: 1 2 + 3 *
    (*)               步骤1: push(1)    栈: [1]
   /   \              步骤2: push(2)    栈: [1, 2]
 (+)    3             步骤3: pop(2), pop(1), push(1+2=3)  栈: [3]
/   \                 步骤4: push(3)    栈: [3, 3]
1   2                 步骤5: pop(3), pop(3), push(3*3=9)  栈: [9]
```

### 19.6.2 主循环作为"统一驱动"

```
┌──────────────────────────────────────────────────┐
│                   while(getchar())                │
│                       │                           │
│         ┌─────────────┼─────────────┐             │
│         ▼             ▼             ▼             │
│    ┌─────────┐  ┌──────────┐  ┌─────────┐        │
│    │ 空白字符 │  │ 数字字符  │  │ 操作符   │        │
│    │ →忽略   │  │ →数字收集 │  │ →分发    │        │
│    └─────────┘  └──────────┘  └─────────┘        │
│                                    │              │
│                             ┌──────┴──────┐       │
│                             ▼             ▼       │
│                        ┌────────┐   ┌─────────┐   │
│                        │ pop×2  │   │ pop×1   │   │
│                        │ 计算   │   │ 计算    │   │
│                        │ push   │   │ push    │   │
│                        └────────┘   └─────────┘   │
└──────────────────────────────────────────────────┘
```

这种设计的本质是 **lexer 和 parser 融合在同一个字符级 switch 中**。从编译器设计的角度看，这是一个**无前瞻、无回溯**的确定性有限自动机（DFA）驱动的即时求值器。

---

## 19.7 从 RPN 到中缀：运算符绑定强度与 Pratt 解析

虽然 `calc` / `mpcalc` 使用 RPN 是极简主义设计的自然结果——简单的字符分发就能工作——但在实际应用中，用户更习惯中缀表达式 `1 + 2 * 3` 而非 `1 2 3 * +`。

本节作为 **Modern-C 改进** 讨论，设计一个中缀表达式解析器作为 `mpcalc` 的替代前端。关键问题：**如何正确解析运算符优先级？**

### 19.7.1 运算符优先级与结合性表

首先定义一张优先级表（借鉴 C 语言的惯例，但可以根据需要自定义）：

```
优先级 (binding power)      运算符              结合性
──────────────────────────────────────────────────
 1 (最低)                   =                   右结合
 2                          ||                  左结合
 3                          &&                  左结合
 4                          |                   左结合
 5                          ^                   左结合
 6                          &                   左结合
 7                          == !=               左结合
 8                          < <= > >=           左结合
 9                          << >>               左结合
10                          + -                 左结合
11                          * / %               左结合
12                          ** (幂)             右结合
13 (最高)                   ! ~ - (一元)        右结合
14                          () [] .             后缀
```

在 C 代码中，这可以用一个结构体数组表示：

```c
typedef enum {
    ASSOC_NONE,  // 非运算符
    ASSOC_LEFT,  // 左结合
    ASSOC_RIGHT  // 右结合
} Associativity;

typedef struct {
    const char *op;         // 运算符字符串
    int         precedence;  // 绑定强度 (数值越大越强)
    Associativity assoc;
    int         arity;      // 1=一元, 2=二元
    MP_T (*eval)(MP_T, MP_T); // [UNSPECIFIED: 实际实现签名]
} Operator;

static Operator operators[] = {
    {"=",   1, ASSOC_RIGHT, 2, NULL},
    {"||",  2, ASSOC_LEFT,  2, NULL},
    {"&&",  3, ASSOC_LEFT,  2, NULL},
    {"|",   4, ASSOC_LEFT,  2, NULL},
    {"^",   5, ASSOC_LEFT,  2, NULL},
    {"&",   6, ASSOC_LEFT,  2, NULL},
    {"==",  7, ASSOC_LEFT,  2, NULL},
    {"!=",  7, ASSOC_LEFT,  2, NULL},
    {"<",   8, ASSOC_LEFT,  2, NULL},
    {"<=",  8, ASSOC_LEFT,  2, NULL},
    {">",   8, ASSOC_LEFT,  2, NULL},
    {">=",  8, ASSOC_LEFT,  2, NULL},
    {"<<",  9, ASSOC_LEFT,  2, NULL},
    {">>",  9, ASSOC_LEFT,  2, NULL},
    {"+",  10, ASSOC_LEFT,  2, NULL},
    {"-",  10, ASSOC_LEFT,  2, NULL},
    {"*",  11, ASSOC_LEFT,  2, NULL},
    {"/",  11, ASSOC_LEFT,  2, NULL},
    {"%",  11, ASSOC_LEFT,  2, NULL},
    {"**", 12, ASSOC_RIGHT, 2, NULL},
    {"!",  13, ASSOC_RIGHT, 1, NULL},  // 一元
    {"~",  13, ASSOC_RIGHT, 1, NULL},  // 一元
    {"-",  13, ASSOC_RIGHT, 1, NULL},  // 一元负号 (需上下文区分)
    {NULL,  0, ASSOC_NONE,  0, NULL}   // 哨兵
};
```

### 19.7.2 术语精确定义

在深入 Pratt 解析前，先厘清几个关键术语，它们常被混用：

- **词法分析 (Lexing/Tokenization)**：将字符流转换为 token 流。每个 token 有**类型**（NUMBER, PLUS, MINUS, LPAREN, RPAREN, EOF 等）和可选的**字面值**。
- **语法分析 (Parsing)**：将 token 流转换为 AST。Pratt 解析是一种**自顶向下**的解析算法。
- **AST 求值 (Evaluation)**：遍历 AST，对每个节点执行其代表的操作，返回计算结果。
- **Pratt 解析**：由 Vaughan Pratt 于 1973 年提出，又称为 **top-down operator precedence (TDOP) parsing**。核心思想：每个 token 有两种绑定强度——**左绑定强度 (left binding power, lbp)** 和 **右绑定强度 (right binding power, rbp)** [UNSPECIFIED: Pratt 论文中使用的术语略有不同]。

**LL 递归下降 vs. Pratt 解析**：
- 经典递归下降 (LL) 为每个优先级写一个函数（`expr()` 调用 `term()` 调用 `factor()`），这在运算符较多时代码冗长。
- Pratt 解析用一个统一的循环替代多层嵌套调用，由优先级数值驱动。

### 19.7.3 Pratt 解析的核心思想

Pratt 解析将每个 token 视为一个对象，它知道：
1. `nud()` (null denotation)：当 token 出现在表达式**开头**时做什么（如数字、一元负号、左括号）
2. `led(left)` (left denotation)：当 token 出现在**中缀/后缀**位置时做什么（如二元运算符、右括号）
3. `lbp`：该 token 的左绑定强度

核心循环 `expr(rbp)` 的伪代码：

```
function expr(rbp):
    t = next_token()
    left = t.nud()          // 处理前缀形式
    while rbp < peek().lbp: // 只要右侧运算符绑定更强
        t = next_token()
        left = t.led(left)  // 将左侧作为子表达式传入
    return left
```

**关键直觉**：`rbp` 是"我右侧的运算符要有多强才能把我抢走"。如果下一个运算符的 `lbp` 大于当前 `rbp`，则它绑定得更紧，当前表达式应成为它的左操作数。

### 19.7.4 具体解析示例

解析 `1 + 2 * 3`：

```
        调用 expr(0)
        │
        ├─ token="1", nud() → 返回 AST节点 [1]
        │  left = [1]
        │
        ├─ peek() = "+", lbp=10 > rbp=0? 是
        │   token="+", led(left=[1])
        │   │
        │   │  rbp = 10 (左结合: lbp=10, 右结合: lbp-1=9)
        │   │  调用 expr(10)
        │   │  │
        │   │  ├─ token="2", nud() → [2]
        │   │  │  left = [2]
        │   │  │
        │   │  ├─ peek() = "*", lbp=11 > rbp=10? 是
        │   │  │   token="*", led(left=[2])
        │   │  │   │
        │   │  │   │  rbp = 11
        │   │  │   │  expr(11)
        │   │  │   │  ├─ token="3", nud() → [3]
        │   │  │   │  │  left = [3]
        │   │  │   │  │
        │   │  │   │  │  peek()=EOF, lbp=0 > rbp=11? 否
        │   │  │   │  │  return [3]
        │   │  │   │  │
        │   │  │   │  return [*]
        │   │  │   │        /  \
        │   │  │   │      [2]  [3]
        │   │  │
        │   │  ├─ peek() = EOF, lbp=0 > rbp=10? 否
        │   │  └─ return [*]
        │   │            /  \
        │   │          [2]  [3]
        │   │
        │   return [+]
        │         /  \
        │       [1]  [*]
        │            /  \
        │          [2]  [3]
        │
        ├─ peek() = EOF, lbp=0 > rbp=0? 否
        └─ return AST根节点
```

**最终 AST**：

```
          (+)
         /   \
       [1]   (*)
             /   \
           [2]   [3]
```

### 19.7.5 AST 求值

有了 AST 后，求值是一个简单的递归后序遍历：

```
function eval_ast(node):
    if node.type == NUMBER:
        return node.value
    elif node.type == UNARY_OP:
        operand = eval_ast(node.right)
        return apply_unary(node.op, operand)
    elif node.type == BINARY_OP:
        left  = eval_ast(node.left)
        right = eval_ast(node.right)
        return apply_binary(node.op, left, right)
```

对比 RPN 和 AST 方案：

| 维度 | RPN (calc/mpcalc) | AST + Pratt Parser |
|------|-------------------|-------------------|
| 输入形式 | 后缀表达式 | 中缀表达式 |
| 解析复杂度 | O(1) 字符分发 | Pratt 解析 O(n) |
| 内存 | 栈存中间值 | AST 节点 + 栈 |
| 用户友好度 | 低（需训练） | 高（自然书写） |
| 错误恢复 | 简单（忽略非法） | 需要设计同步策略 |
| 代码行数 | ~180 行 | ~500+ 行 |
| 扩展性 | 添加操作符 = 加 case | 添加操作符 = 加一行优先级表 + eval |

### 19.7.6 错误恢复策略 [Modern-C 改进]

当前 `mpcalc` 的错误处理简单但脆弱：遇到不识别的字符打印警告并继续。对于中缀解析器，需要更好的错误恢复：

1. **Panic Mode**：遇到语法错误时，跳过后续 token 直到遇到同步 token（如 `;` `\n` `)`）
2. **错误产生式**：在语法中显式添加常见错误的产生式规则（如缺失操作数 → 插入默认值 0）
3. **括号匹配诊断**：记录左右括号数量，在表达式结束时报告不匹配
4. **除零保护**：在 AST 求值阶段而非解析阶段，当除数为零时输出 `NaN` 标识并继续

示例错误恢复骨架：

```c
typedef struct {
    int         errors;
    const char *source;
    int         pos;
    Token       current;
    Token       next;
} Parser;

static void parse_error(Parser *p, const char *msg) {
    Fmt_fprint(stderr, "?parse error at position %d: %s\n",
               p->pos, msg);
    p->errors++;
    /* panic mode: skip to next sync token */
    while (p->current.type != TOK_EOF
           && p->current.type != TOK_SEMI
           && p->current.type != TOK_NEWLINE)
        advance(p);
}

AST_Node *expr(Parser *p, int rbp) {
    Token t = p->current;
    advance(p);
    AST_Node *left = nud(p, &t);
    if (!left) {
        parse_error(p, "expected expression");
        left = ast_number(0);  /* 错误恢复: 插入 0 */
    }
    while (rbp < lbp(p->current)) {
        t = p->current;
        advance(p);
        left = led(p, left, &t);
    }
    return left;
}
```

---

## 19.8 CII 全书模块的集大成组合

`mpcalc` 不是孤立程序。它以极简的代码（180 行）展示了 CII 六大模块的协同工作方式。下图展示了模块间的依赖关系：

```
                    mpcalc.c (180行)
                   /     |      \      \
                  /      |       \      \
               Seq      Mem      Fmt     MP
              (序列)   (内存)   (格式化)  (多精度)
                |                 |        |
                |                 |    ┌───┴───┐
                |                 |    XP    Except
                |                 |  (扩展精度)(异常)
                |                 |
                └─────────────────┘
                    (都依赖 Mem)
```

### 19.8.1 逐模块调用分析

| 模块 | 在 mpcalc 中的用途 | 关键调用 |
|------|-------------------|---------|
| **Seq** | 操作数栈（动态数组） | `Seq_new(0)`, `Seq_addhi()`, `Seq_remhi()`, `Seq_length()`, `Seq_get()`, `Seq_free()` |
| **Mem** | 内存分配/释放 | `ALLOC()` (通过 `MP_new`), `FREE()` |
| **Fmt** | 格式化输出 | `Fmt_register()`, `Fmt_print()`, `Fmt_fprint()` |
| **MP** | 大整数运算 | 加减乘除、位运算、移位、字符串转换 |
| **XP** | (被 MP 调用) 字节级多精度原语 | `XP_add`, `XP_mul`, `XP_div`, `XP_fromstr` 等 |
| **Except** | 异常处理 | `TRY-EXCEPT-END_TRY`, `RAISE()`, `MP_Overflow`, `MP_Dividebyzero` |

### 19.8.2 Seq 替代 Stack 的设计考量

`calc` 使用 `Stack_T`（基于链表），`mpcalc` 改用 `Seq_T`（基于动态数组）。这个变化使得 `f` 命令（打印全栈）的实现变得极其简单：

```c
/* mpcalc 中的 f 命令: O(1) 实现 */
case 'f': {
    int n = Seq_length(sp);
    while (--n >= 0)
        Fmt_print(f->fmt, Seq_get(sp, n), obase);
    break;
}

/* calc 中的 f 命令: 需要借助临时栈反转 */
case 'f':
    if (!Stack_empty(sp)) {
        Stack_T tmp = Stack_new();
        while (!Stack_empty(sp)) { /* 弹出到临时栈 */
            AP_T x = pop();
            Fmt_print("%D\n", x);
            Stack_push(tmp, x);
        }
        while (!Stack_empty(tmp)) /* 从临时栈恢复 */
            Stack_push(sp, Stack_pop(tmp));
        Stack_free(&tmp);
    }
    break;
```

`Seq_get(sp, n)` 提供了**随机访问**能力——可以直接通过索引读取栈中任意位置的值而不需要弹出。这是 `Seq` 相对于 `Stack` 的关键优势：它既是栈（通过 `addhi`/`remhi` 在末尾操作），也是可索引的数组。

### 19.8.3 异常处理层的优雅性

`mpcalc` 在主循环中只写了一个 `TRY-EXCEPT` 块，却保护了所有运算符操作。这个设计的关键在于变量约定：

```c
volatile MP_T x = NULL, y = NULL, z = NULL;
TRY
    switch (c) {
        /* ... 运算符设置 x, y, z ... */
    }
EXCEPT(MP_Overflow)
    Fmt_fprint(stderr, "?overflow\n");
EXCEPT(MP_Dividebyzero)
    Fmt_fprint(stderr, "?divide by 0\n");
END_TRY;
if (z)
    Seq_addhi(sp, z);  // 异常发生时 z 可能已设置（截断后的值）
FREE(x);
FREE(y);
```

- `volatile` 关键字确保 `x`, `y`, `z` 在 `setjmp`/`longjmp` 跨越时不被寄存器优化破坏。
- `z` 负责持有**结果**：正常情况是计算结果，异常情况可能是截断后的值（符合 MP 接口"先设值再抛异常"的约定）。
- `x` 和 `y` 持有**操作数**引用，负责在循环末尾释放。

### 19.8.4 CII 设计哲学的最终体现

`mpcalc` 作为全书终章的应用示例，体现了 Hanson 的以下接口设计原则：

1. **接口即契约**：MP 接口的 49 个函数各自有精确的溢出语义和截断约定。调用者（`mpcalc`）无需了解 MP 内部使用 XP 的细节。
2. **层叠抽象**：XP（裸字节）→ MP（n 位补码整数）→ mpcalc（交互式计算器），每层只关心自己层级的语义。
3. **异常优于错误码**：`MP_Overflow` 和 `MP_Dividebyzero` 使得 `mpcalc` 的主循环不需要在每个运算符调用后检查返回值——所有错误统一在 `EXCEPT` 子句中处理。
4. **最小化分配**：`MP_set(n)` 一次性分配所有临时缓冲区，后续 48 个函数中只有 4 个会触发分配。这与 AP 接口（每次运算都分配新对象）形成对比——MP 面向的是需要细粒度控制分配的应用场景（如编译器、加密算法）。
5. **命名约定驱逐符号冲突**：`MP_add`、`Seq_addhi`、`Fmt_print`——每个标识符以模块缩写为前缀，在 C 的平面全局命名空间中建立人工命名空间。

---

## 19.9 本章小结

`mpcalc` 用 180 行 C 代码实现了一个功能完备的多精度终端计算器，具备：

- 可变精度（`k` 命令，2..INT_MAX 位）
- 可变输入/输出基数（`i`/`o` 命令，2..36 进制）
- 有符号/无符号双模式自动切换
- 基本算术 + 位运算 + 移位
- 异常安全的溢出和除零处理
- 栈下溢检测与错误恢复
- 全栈查看（`f` 命令，利用 Seq 的随机访问）

从软件工程角度看，`mpcalc` 是 CII 全书的缩影：RP (A Tour of the Code) 揭示的不是某个孤立的"多精度库"，而是一套**可组合的接口设计方法论**——每一个接口做好一件事，接口之间通过清晰的依赖关系组合成更强大的系统。

`mpcalc` 的极简设计（RPN + 字符级 switch）在实用角度是一个缺陷——用户需要适应后缀表示法——但正是这种极简使得代码成为教学典范：**在 180 行中完整演示了 REPL 循环、词法分析、栈求值、异常处理和内存管理的全部要素**。对于任何想要理解 C 语言模块化设计的读者，`mpcalc` 都是值得反复精读的范本。

---

## 19.10 延伸阅读

1. **Pratt, V. R. (1973).** "Top down operator precedence." _Proceedings of the 1st Annual ACM SIGACT-SIGPLAN Symposium on Principles of Programming Languages (POPL '73)._ — Pratt 解析的原论文。
2. **Crockford, D. (2007).** "Top Down Operator Precedence." — 将 Pratt 解析通俗化的经典博客文章，用 JavaScript 实现了一个完整的 Pratt 解析器。
3. **Nystrom, R. (2021).** _Crafting Interpreters._ Chapter 6 (Parsing Expressions) 和 Chapter 17 (Compiling Expressions) — 分别用 Java 和 C 实现了 Pratt 解析的两种变体，是理解运算符优先级解析的最佳实践读物。
4. **Hanson, D. R. (1997).** _C Interfaces and Implementations._ 第 2 章 (Arith)、第 17 章 (XP)、第 18 章 (AP)、第 19 章 (MP) — 原书中的四个算术相关章节，构成一条从可移植除法到多精度计算器的完整技术路线。
5. **Hanson, D. R. (1996).** _A Retargetable C Compiler: Design and Implementation._ — David Hanson 与 Christopher Fraser 合著的 lcc 编译器，MP 接口的设计动机之一即为支持交叉编译器的常量折叠。
