# 第17章: 扩展精度算术 (Extended-Precision Arithmetic)

> 原文: C Interfaces and Implementations, David R. Hanson, Chapter 17
> 源文件: `code/xp.h`, `code/xp.c`

---

## 一、设计哲学与接口概述

### 1.1 什么是扩展精度算术？

C语言的内置整数类型 (`unsigned long`, `unsigned long long`) 的位宽受限于硬件平台。当需要处理几百位甚至上千位的大整数时——例如密码学中的RSA密钥、编译器中的交叉编译常量折叠、或者科学计算中的精确有理数——内置类型远远不够。

XP (eXtended Precision) 接口提供了一个**无符号大整数**的底层运算库。与上层接口AP (Arbitrary Precision) 和 MP (Multiple Precision) 不同，XP的设计目标有三个:

1. **零内存分配**: 所有XP函数都不调用 `malloc`，内存由调用者管理。这使得XP适合作为更高层接口 (如AP/MP) 的实现基础。
2. **极简抽象**: XP直接暴露内部表示 (`unsigned char *`)，方便在性能关键路径上用汇编语言替换C实现。
3. **不检查运行时错误**: 除少数例外，XP不做运行时检查 (unchecked runtime error)，将检查责任交给上层调用者。

> 设计权衡: "XP is a dangerous interface, because it omits most checked runtime errors." — 这种设计刻意为之。XP的目标用户是更高层接口 (如AP、MP)，由它们来实现必要的运行时检查。

### 1.2 n位无符号整数的数组表示法

```
            ┌────────────────────────────────────────┐
            │          n-digit XP_T (base 256)         │
            │  小端序: digits[0] 是最低有效字节         │
            ├─────┬─────┬─────┬─────┬─────┬─────┬─────┤
            │ [0] │ [1] │ [2] │ [3] │ ... │[n-2]│[n-1]│
            ├─────┴─────┴─────┴─────┴─────┴─────┴─────┤
            │ 256⁰  │ 256¹ │ 256² │ 256³ │     │     │ 最高有效位
            └────────────────────────────────────────┘

    数值 = digits[0] * 256⁰ + digits[1] * 256¹ + ... + digits[n-1] * 256^{n-1}
```

XP将一个n位无符号整数表示为 `unsigned char` 数组，基数为 **256** (即 `2⁸`)，采用**小端序 (little-endian)** 存储——最低有效字节存放在 `digits[0]`。这种选择有深刻的工程原因:

- `unsigned char` 恰好8位，与C标准保证的最小可寻址单元一致。
- 256 是最大的2的幂，使得 `2⁸` 进制下的乘法/除法和取模运算可以被编译器优化为移位和位与操作。
- 小端序使得加减法的进位/借位传播方向与数组索引增长方向一致 (从低位到高位)。

**基数选择的约束**: 为什么选256而不是更大的2的幂如 `2¹⁶` 或 `2³²`？关键原因在于**长除法中的商位估计**。在 `XP_div` 中，需要用硬件整数除法估算商位:

```c
unsigned long y2 = y[m-1]*BASE + y[m-2];
unsigned long r3 = rem[km]*(BASE*BASE) + rem[km-1]*BASE + rem[km-2];
qk = r3 / y2;
```

`r3` 的最大值是 `(BASE-1)*BASE² + (BASE-1)*BASE + (BASE-1) = BASE³ - 1`。当 `BASE = 256` 时，`r3_max = 16,777,215`，可以放入 `unsigned long` (32位时最大值约42.9亿)。如果 `BASE = 2¹⁶ = 65,536`，则 `r3_max = 2⁴⁸ - 1 = 281万亿`，超出了32位 `unsigned long` 的范围。这是将基数限制在256以内的**核心工程约束**。详见习题17.3。

### 1.3 完整接口: `xp.h`

```c
/* xp.h — Extended-Precision Arithmetic Interface */
#ifndef XP_INCLUDED
#define XP_INCLUDED
#define T XP_T
typedef unsigned char *T;     /* XP_T = unsigned char数组, base-256小端序 */

/* === 基本算术 === */
extern int XP_add(int n, T z, T x, T y, int carry);
    /* z = x + y + carry, 返回最高位的进位 */

extern int XP_sub(int n, T z, T x, T y, int borrow);
    /* z = x - y - borrow, 返回最高位的借位 */

extern int XP_mul(T z, int n, T x, int m, T y);
    /* z = z + x*y, x有n位, y有m位, z必须有n+m位空间, 返回进位 */

extern int XP_div(int n, T q, T x, int m, T y, T r, T tmp);
    /* q = x / y, r = x mod y, tmp至少n+m+2位; y==0时返回0 */

/* === 单数字算术 (第二个操作数是单个base-256数字) === */
extern int XP_sum     (int n, T z, T x, int y);
    /* z = x + y, y是一个base-256数字 */

extern int XP_diff    (int n, T z, T x, int y);
    /* z = x - y */

extern int XP_product (int n, T z, T x, int y);
    /* z = x * y, 返回进位 (可达255) */

extern int XP_quotient(int n, T z, T x, int y);
    /* z = x / y, 返回余数 (x mod y) */

/* === 求反与比较 === */
extern int XP_neg(int n, T z, T x, int carry);
    /* z = ~x + carry; carry=0时为反码, carry=1时为补码 */

extern int XP_cmp(int n, T x, T y);
    /* 比较x和y, 返回 <0 / =0 / >0 */

/* === 移位操作 === */
extern void XP_lshift(int n, T z, int m, T x, int s, int fill);
    /* z = x << s; x有m位, z有n位; fill(0或1)填充空位 */

extern void XP_rshift(int n, T z, int m, T x, int s, int fill);
    /* z = x >> s; fill=0时逻辑右移, fill=1时算术右移 */

/* === 长度与转换 === */
extern int           XP_length (int n, T x);
    /* 返回有效位数 (去除前导零) */

extern unsigned long XP_fromint(int n, T z, unsigned long u);
    /* z = u mod 256ⁿ, 返回 u / 256ⁿ (溢出部分) */

extern unsigned long XP_toint  (int n, T x);
    /* 返回x的最低sizeof(unsigned long)字节, 转为unsigned long */

/* === 字符串转换 === */
extern int   XP_fromstr(int n, T z, const char *str, int base, char **end);
    /* 将str按base进制解析累加到z; 返回非零表示溢出; end指向停止字符 */

extern char *XP_tostr  (char *str, int size, int base, int n, T x);
    /* 将x按base进制转为字符串; x被清零; 返回str */

#undef T
#endif
```

### 1.4 接口设计要点

| 设计决策 | 理由 | 后果 |
|---------|------|------|
| `typedef unsigned char *T` | 直接暴露底层表示 | 无类型安全; 调用者负责管理内存大小 |
| 零内存分配 | 适合作为底层引擎 | 调用者必须预分配所有临时缓冲区 |
| 无const修饰符 | `const T` = "const pointer" 而非 "pointer to const" | 无法在类型层面防止别名问题 |
| 进位/借位通过返回值传递 | C语言惯用法 | 支持链式运算 (`XP_sum(..., XP_add(...))`) |
| `XP_mul` 执行 `z += x*y` 而非 `z = x*y` | 支持累加模式 | 调用者需要先将z清零 |

---

## 二、数据结构深度解析

### 2.1 核心表示: `unsigned char` 数组

```
  十进制数 751,702,468,129 的 XP_T 内部布局 (11位, 小端序):

    索引:  [0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]   [8]   [9]   [10]
    数值:   4    245   102    33   175     0     0     0     0     0     0
    权重:  256⁰  256¹  256²  256³  256⁴    ...

    验证: 4 + 245*256 + 102*65536 + 33*16777216 + 175*4294967296
        = 4 + 62720 + 6684672 + 553648128 + 751619276800
        = 751,702,468,129  ✓
```

`unsigned char` 在C标准中保证至少8位。严格来说，某些DSP平台的 `char` 可能多于8位，但 `unsigned char` 的 `CHAR_BIT` 在现代通用平台上几乎总是8位。XP以 `BASE = (1<<8) = 256` 为基数，隐式依赖 `CHAR_BIT == 8` 的前提。

### 2.2 `XP_length`: 有效位数

```c
int XP_length(int n, T x) {
    while (n > 1 && x[n-1] == 0)
        n--;
    return n;
}
```

此函数返回去除前导零后的有效位数。例如: 一个11位的XP_T，如果只有前5位非零，则 `XP_length` 返回5。注意: XP_T允许未归一化表示 (前导零不自动清除)，这减少了运行时的清理开销，但也意味着调用者需要在合适的时候自行归一化。

### 2.3 `XP_fromint` 与 `XP_toint`: 与C整数类型的互转

```c
unsigned long XP_fromint(int n, T z, unsigned long u) {
    int i = 0;
    do
        z[i++] = u % BASE;     /* 取出最低字节 */
    while ((u /= BASE) > 0 && i < n);
    for ( ; i < n; i++)
        z[i] = 0;              /* 高位填零 */
    return u;                   /* 返回溢出部分 (未能放入z的高位) */
}
```

`XP_fromint` 将一个 `unsigned long` 分解为base-256的数字序列。`do-while` 循环 (而非 `while`) 确保即使 `u == 0` 也会写入一个零字节。返回值是 `u / BASEⁿ`，即溢出部分——当输入值超过n位时，这个返回值非零。

```c
unsigned long XP_toint(int n, T x) {
    unsigned long u = 0;
    int i = (int)sizeof u;
    if (i > n) i = n;           /* 不能超出x的有效长度 */
    while (--i >= 0)
        u = BASE*u + x[i];      /* Horner方法累加 */
    return u;
}
```

`XP_toint` 是 `XP_fromint` 的逆操作: 使用**Horner方法** (嵌套乘加) 将base-256数字序列重新组装为 `unsigned long`。最多只转换 `sizeof(unsigned long)` 个字节，超出部分被截断。

---

## 三、加法与减法: 进位与借位的逐位传播

### 3.1 加法: `XP_add`

XP_add 实现了课本中的竖式加法——从最低位到最高位逐位相加，进位传递到高位。

```
  十进制示例 (base-10, n=4):          算法步骤:
                                        输入: x=9428, y=732, carry=0
     1  0  1  0    ← 进位值
                                        步骤0: S = 0+8+2=10, z[0]=0, carry=1
     9  4  2  8                          步骤1: S = 1+2+3=6,  z[1]=6, carry=0
   +    7  3  2                          步骤2: S = 0+4+7=11, z[2]=1, carry=1
   ─────────────                         步骤3: S = 1+9+0=10, z[3]=0, carry=1
   1  0  6  0  ← 各位的S值
                                         返回: carry=1 (溢出), z=0160
```

**C实现逐行注释**:

```c
int XP_add(int n, T z, T x, T y, int carry) {
    int i;
    for (i = 0; i < n; i++) {
        carry += x[i] + y[i];    /* S = carry_prev + x_i + y_i */
        z[i] = carry % BASE;     /* z_i = S mod b, 取当前位 */
        carry /= BASE;           /* 新的进位 = S / b, b=256 */
    }
    return carry;                /* 最终进位: 0=无溢出, 1=溢出 */
}
```

**变量上限分析**: 每个数字 `x[i] ≤ 255`, `y[i] ≤ 255`, `carry ≤ 1`。所以 `S ≤ 255+255+1 = 511`，安全地放入 `int` (保证至少16位，范围 ±32767)。

**伪代码**:

```
算法: 多精度加法 XP_add(n, z, x, y, carry)
  输入: n     — 数字位数
        x, y  — n位操作数 (base-256, 小端序)
        carry — 初始进位 (0 或 1)
  输出: z     — n位和
        返回  — 最终进位

  for i = 0 to n-1:
      sum  = carry + x[i] + y[i]
      z[i] = sum mod 256
      carry = sum / 256
  return carry
```

**特殊情况**: `x == z` 或 `y == z` 是允许的——读 `x[i]` 和 `y[i]` 发生在写 `z[i]` 之前，且循环内只使用当前位的 `carry`。这是该实现的一个重要设计特性: "For just these two functions, it is not an error for any of x, y, or z to be the same XP_T."

### 3.2 减法: `XP_sub`

```
  十进制示例 (base-10, n=4):          算法步骤:
                                        输入: x=9428, y=732, borrow=0
     0  1  1  0    ← 借位值
                                        步骤0: D=8+10-0-2=16, z[0]=6, borrow=1-16/10=1
     9  4  2  8                          步骤1: D=2+10-1-3=8,  z[1]=8, borrow=1-8/10=1
   -    7  3  2                          步骤2: D=4+10-1-7=6,  z[2]=6, borrow=1-6/10=1
   ─────────────                         步骤3: D=9+10-1-0=18, z[3]=8, borrow=1-18/10=0
    18  6  9  6  ← 各位的D值
                                         返回: borrow=0, z=8696
```

**公式推导**: 每一步计算 `D = x[i] + BASE - borrow_prev - y[i]`。

- `z[i] = D mod BASE`: 当前位的结果
- `borrow_new = 1 - D/BASE`: 如果 `D < BASE` (即需要借位)，则 `D/BASE = 0`，`borrow_new = 1`; 如果 `D >= BASE`，则 `borrow_new = 0`。

这个简洁的公式优雅地统一了借位逻辑。

```c
int XP_sub(int n, T z, T x, T y, int borrow) {
    int i;
    for (i = 0; i < n; i++) {
        int d = (x[i] + BASE) - borrow - y[i];
        z[i] = d % BASE;
        borrow = 1 - d/BASE;     /* d < BASE → borrow=1; d >= BASE → borrow=0 */
    }
    return borrow;               /* borrow=1 表示 x < y (结果为负) */
}
```

**伪代码**:

```
算法: 多精度减法 XP_sub(n, z, x, y, borrow)
  输入: n      — 数字位数
        x, y   — n位操作数
        borrow — 初始借位 (0 或 1)
  输出: z      — n位差
        返回   — 最终借位 (1 表示结果为负, 即 x < y)

  for i = 0 to n-1:
      diff      = x[i] + 256 - borrow - y[i]
      z[i]      = diff mod 256
      borrow    = 1 - (diff / 256)   // diff < 256 → borrow=1; else → borrow=0
  return borrow
```

### 3.3 单数字版本: `XP_sum`, `XP_diff`, `XP_product`, `XP_quotient`

这些函数是加减乘除的退化版本，第二个操作数是一个**单个base-256数字** `y` (类型为 `int`)。它们复用了与多数字版本相同的进位/借位传播结构，但`y`本身承担了进位/借位的角色。

```c
int XP_sum(int n, T z, T x, int y) {
    int i;
    for (i = 0; i < n; i++) {
        y += x[i];
        z[i] = y % BASE;
        y /= BASE;               /* y 既是加数也是进位累加器 */
    }
    return y;                    /* 返回最终进位 */
}

int XP_diff(int n, T z, T x, int y) {
    int i;
    for (i = 0; i < n; i++) {
        int d = (x[i] + BASE) - y;
        z[i] = d % BASE;
        y = 1 - d/BASE;          /* y 既是减数也是借位累加器 */
    }
    return y;
}
```

### 3.4 取反: `XP_neg`

```c
int XP_neg(int n, T z, T x, int carry) {
    int i;
    for (i = 0; i < n; i++) {
        carry += (unsigned char)~x[i];  /* 强制转换为unsigned char避免符号扩展 */
        z[i] = carry % BASE;
        carry /= BASE;
    }
    return carry;
}
```

- `carry = 0`: 实现反码 (one's complement) 取反: `z = ~x`
- `carry = 1`: 实现补码 (two's complement) 取反: `z = ~x + 1`

`(unsigned char)~x[i]` 的强制转换至关重要: 如果省略该转换，`~x[i]` 会被提升为 `int` 并进行符号扩展 (如果 `char` 是有符号的)，导致 `~0x00` 变成 `0xFFFFFFFF` 而非 `0xFF`。

---

## 四、乘法: 分配律与部分积累加

### 4.1 小学数学回顾: 部分积

```
  十进制示例: 9428 × 732

        9 4 2 8        (x[3..0])
    ×     7 3 2        (y[2..0])
    ─────────────
      1 8 8 5 6        ← 部分积 0: x × 2, 对齐位置 0
    2 8 2 8 4          ← 部分积 1: x × 3, 对齐位置 1
  6 5 9 9 6            ← 部分积 2: x × 7, 对齐位置 2
  ─────────────────
  6 9 0 1 2 9 6        ← 最终结果 (4+3=7位)
```

n位乘数 × m位乘数 → 最多 n+m 位结果。每个部分积 `x × y[j]` 有n+1位，其最低有效位对齐到位置j。

### 4.2 `XP_mul`: 边算边累加

`XP_mul` 的核心创新在于: **不显式存储部分积，而是在计算每一位部分积的同时累加到z中**。

```c
int XP_mul(T z, int n, T x, int m, T y) {
    int i, j, carryout = 0;

    for (i = 0; i < n; i++) {          /* 对于x的每一位 */
        unsigned carry = 0;

        /* 内层循环: 计算 x[i] × y[0..m-1] 并累加到 z[i..i+m-1] */
        for (j = 0; j < m; j++) {
            carry += x[i] * y[j] + z[i+j];
            z[i+j] = carry % BASE;
            carry /= BASE;
        }

        /* 扩展循环: 将进位传播到z的更高位 */
        for ( ; j < n + m - i; j++) {
            carry += z[i+j];
            z[i+j] = carry % BASE;
            carry /= BASE;
        }

        carryout |= carry;             /* 记录是否有溢出 */
    }
    return carryout;
}
```

**执行流程分解 (以x=3位, y=4位为例)**:

```
  外层循环 i=0: 计算 x[0] × y[0..3], 累加到 z[0..3]
  外层循环 i=1: 计算 x[1] × y[0..3], 累加到 z[1..4]
  外层循环 i=2: 计算 x[2] × y[0..3], 累加到 z[2..5]
  结果: z[0..6] 包含完整乘积 (3+4=7位)
```

**进位范围分析**: 在内层循环中，`carry` 的最大值是 `(b-1)(b-1) + (b-1) = b² - b = 65280`，这放入 `unsigned int` 绰绰有余。

**关键语义**: `XP_mul` 执行 `z = z + x*y` 而非 `z = x*y`。这意味着:
1. 调用者必须先初始化 `z` (通常用 `memset(z, 0, n+m)`)
2. 支持链式累加——多次调用 `XP_mul` 可以将多个乘积加到同一个 `z` 中

**伪代码**:

```
算法: 多精度乘法 XP_mul(z, n, x, m, y)
  输入: n     — x的数字位数
        m     — y的数字位数
        x, y  — 乘数 (base-256, 小端序)
  输入/输出: z — n+m位累加器 (初始应为零, 执行 z += x*y)
  输出: 返回 — 最终进位 (0或1)

  carryout = 0
  for i = 0 to n-1:                      // 对x的每一位
      carry = 0
      for j = 0 to m-1:                  // 乘以y的每一位
          carry  += x[i] * y[j] + z[i+j]
          z[i+j] = carry mod 256
          carry  = carry / 256
      for j = m to n+m-i-1:              // 传播剩余进位
          carry  += z[i+j]
          z[i+j] = carry mod 256
          carry  = carry / 256
      carryout = carryout OR carry
  return carryout
```

### 4.3 单数字乘法: `XP_product`

```c
int XP_product(int n, T z, T x, int y) {
    int i;
    unsigned carry = 0;
    for (i = 0; i < n; i++) {
        carry += x[i] * y;         /* 乘积累加 */
        z[i] = carry % BASE;
        carry /= BASE;
    }
    return carry;                  /* 返回进位 (最大为 y-1 ≤ 255) */
}
```

单数字乘法 `XP_product(n, z, x, y)` 等价于 `XP_mul(z, n, x, 1, &y)` 且 `z` 初始化为零。其返回值——进位——可达255 (当 `y=255` 且 `x[n-1]=255` 时)，因此返回值类型为 `int` 而非 `0/1`。

---

## 五、除法: 长除法 (Long Division) 的完整分解

### 5.1 除法算法的分类与选择

`XP_div` 是所有XP函数中最复杂的。它将n位被除数x除以m位除数y，得到商q和余数r。实现根据输入特征分三种情况:

| 情况 | 条件 | 算法 | 复杂度 |
|------|------|------|--------|
| 情况1 | m == 1 (单数字除数) | 逐位短除法 | O(n) |
| 情况2 | m > n (除数为零或除数>被除数) | 直接赋值 q=0, r=x | O(n+m) |
| 情况3 | 2 ≤ m ≤ n (一般长除法) | Brinch-Hansen迭代算法 | O(n*m) |

### 5.2 入口框架

```c
int XP_div(int n, T q, T x, int m, T y, T r, T tmp) {
    int nx = n, my = m;
    n = XP_length(n, x);          /* 获取x的有效位数 (去前导零) */
    m = XP_length(m, y);

    if (m == 1) {
        /* === 情况1: 单数字除法 === */
        if (y[0] == 0)
            return 0;             /* 除以零: 返回失败 */
        r[0] = XP_quotient(nx, q, x, y[0]);
        memset(r + 1, '\0', my - 1);
    } else if (m > n) {
        /* === 情况2: 除数大于被除数 === */
        memset(q, '\0', nx);      /* q = 0 */
        memcpy(r, x, n);
        memset(r + n, '\0', my - n);  /* r = x */
    } else {
        /* === 情况3: 一般长除法 (2 <= m <= n) === */
        /* ... [见5.4节] ... */
    }
    return 1;
}
```

### 5.3 单数字除法: `XP_quotient` (短除法)

```
  十进制示例: 9428 ÷ 7

        1  3  4  6      ← 商
      ┌────────────
    7 │ 9  4  2  8
        0  2  3  4      ← 进位 (部分余数)
        ─────────
        9 24 32 48      ← 部分被除数

  步骤 (从高位到低位):
    步骤0: R = 0*10 + 9 = 9,   q[3] = 9/7 = 1, R = 9%7 = 2
    步骤1: R = 2*10 + 4 = 24,  q[2] = 24/7 = 3, R = 24%7 = 3
    步骤2: R = 3*10 + 2 = 32,  q[1] = 32/7 = 4, R = 32%7 = 4
    步骤3: R = 4*10 + 8 = 48,  q[0] = 48/7 = 6, R = 48%7 = 6
    返回: R=6 (余数), q=1346 (商), 验证: 7*1346+6=9428 ✓
```

```c
int XP_quotient(int n, T z, T x, int y) {
    int i;
    unsigned carry = 0;
    for (i = n - 1; i >= 0; i--) {    /* 从最高位到最低位 */
        carry = carry * BASE + x[i];   /* R = R_prev * b + x_i */
        z[i] = carry / y;              /* q_i = R / y */
        carry %= y;                    /* R = R mod y */
    }
    return carry;                      /* 返回最终余数 */
}
```

**数值范围**: `carry` 的最大值是 `(b-1)*b + (b-1) = b² - 1 = 65,535`，放入 `unsigned int` 安全。

与加法不同，除法从**最高位向最低位**进行，因为商的最高位取决于被除数的最高位。

### 5.4 一般长除法: Brinch-Hansen算法详解

当 `2 ≤ m ≤ n` 时，XP_div 使用 Brinch-Hansen (1994) 长除法算法。这是整个XP接口中最精妙的算法。

#### 5.4.1 算法高层结构

```
算法: 长除法 (Long Division)
  输入: x — n位被除数, y — m位除数 (2 ≤ m ≤ n)
  输出: q — n-m+1位商 (存储于 q[0..n-m]), r — m位余数

  步骤:
    1. rem = x 扩展一个前导零 (n+1位)
    2. for k = n-m down to 0:          // 计算每个商位, 从最高位到最低位
         a. 估算 qk ≈ rem[k..k+m] / y  // 用3-digit/2-digit近似
         b. dq = y * qk                // m+1位部分积
         c. if (dq > rem[k..k+m]):     // 如果估计偏大
              qk--; dq = y * qk        // 修正: qk最多偏大1
         d. q[k] = qk                  // 记录商位
         e. rem[k..k+m] -= dq          // 从余数中减去 dq * b^k
    3. r = rem[0..m-1]                 // rem的前m位是最终余数
```

#### 5.4.2 数值示例: 615367 ÷ 296 = 2078 余 279

```
    n=6, m=3, 基数为10 (为便于理解，实际代码中基数为256)

             2  0  7  8     ← 商 (n-m+1=4位)
          ┌─────────────
    2 9 6 │ 6 1 5 3 6 7
            5 9 2           ← 第1步: q₃=2, rem-=2*296*10³=592000
            ─────
              2 3 3 6 7
              0 0 0 0       ← 第2步: q₂=0 (233 < 296), rem-=0
              ─────────
              2 3 3 6 7
              2 0 7 2       ← 第3步: q₁=7, rem-=7*296*10¹=20720
              ───────
                2 6 4 7
                2 3 6 8     ← 第4步: q₀=8, rem-=8*296=2368
                ───────
                  2 7 9     ← 余数 r = 279

    验证: 296 × 2078 + 279 = 615,088 + 279 = 615,367 ✓
```

以下是各迭代步骤的详细状态表:

| k | rem (有下划线的是m+1位前缀) | 估算qk | 实际qk | dq = y*qk | rem -= dq*b^k |
|---|----------------------------|--------|--------|-----------|----------------|
| 3 | 0**615**367 | 615/296=2.07 → 2 | 2 | 0592 | 0**023**367 |
| 2 | 00**233**67 | 233/296=0.78 → 0 | 0 | 0000 | 00**233**67 |
| 1 | 002**336**7 | 336/296=1.13 → 1 → 修正为... | 7 | 2072 | 00**026**47 |
| 0 | 0002**647** | 264/296=0.89 → ... | 8 | 2368 | 000**279** |

注意: 对于k=1, 简单的三数字除以两数字估计 `2336/296 ≈ 7.89` 给出了初始估值7 (实际代码中用 `rem[k+m..k+m-2] / y[m-1..m-2]`), 然后验证和修正。

#### 5.4.3 商位估计: 核心技巧

长除法中最关键的性能问题是如何快速估计每个商位 `qk`。朴素方法 (从 `BASE-1` 开始递减搜索) 在最坏情况下需要 `BASE-1 = 255` 次乘法-比较迭代，效率不可接受。

Brinch-Hansen的核心洞察是: **用被除数的三个最高数字除以除数的两个最高数字，得到的估计最多比真实商位大1**。

```c
/* qk ← y[m-2..m-1] / rem[k+m-2..k+m] */
{
    int km = k + m;
    unsigned long y2 = y[m-1]*BASE + y[m-2];          /* 除数的最高两位 */
    unsigned long r3 = rem[km]*(BASE*BASE)             /* 被除数前缀的最高三位 */
                     + rem[km-1]*BASE + rem[km-2];
    qk = r3 / y2;                                      /* 硬件整数除法估计 */
    if (qk >= BASE)
        qk = BASE - 1;                                 /* 商位必须 < BASE */
}
```

**为什么这个估计最多偏大1？** Brinch-Hansen (1994) 给出了严格的数学证明。直观理解: 此估计等价于用 `y[m-1]*BASE + y[m-2]` (二位数) 近似除数 `y` (m位数), 用 `rem[km]*BASE² + rem[km-1]*BASE + rem[km-2]` (三位数) 近似被除数的m+1位前缀。由于除数被截断 (忽略了低位 `y[0..m-3]`)，估计的商只会偏大而不会偏小。且最多偏大1。

#### 5.4.4 商位修正与减法

```c
/* 计算 dq = y * qk (m+1位), 存储在 dq[0..m] */
dq[m] = XP_product(m, dq, y, qk);    /* dq[m] 接收进位 */

/* 比较 rem[k..k+m] 与 dq[0..m], 从高位到低位 */
for (i = m; i > 0; i--)
    if (rem[i+k] != dq[i])
        break;
if (rem[i+k] < dq[i])                 /* dq > rem前缀, 需要修正 */
    dq[m] = XP_product(m, dq, y, --qk);  /* qk--, 重新计算dq */
```

比较是逐位从高位到低位进行的，找到第一个不同的位置即停止。

```c
/* 记录商位 */
q[k] = qk;

/* rem[k..k+m] -= dq (借位传播由 XP_sub 处理) */
{
    int borrow;
    borrow = XP_sub(m + 1, &rem[k], &rem[k], dq, 0);
    assert(borrow == 0);              /* 已修正保证 dq <= rem前缀 */
}
```

此处的 `assert(borrow == 0)` 是内部一致性检查: 经过修正后，`dq` 必定不大于 `rem` 的前缀，因此减法不会产生借位。

#### 5.4.5 完整长除法伪代码

```
算法: 长除法核心 (Brinch-Hansen, 1994)
  输入: n ≥ m ≥ 2
        x[0..n-1] — 被除数
        y[0..m-1] — 除数
        tmp       — 临时空间 (至少 n+m+2 字节)
  输出: q[0..n-m] — 商
        r[0..m-1] — 余数

  1. rem = tmp[0..n]           // n+1位: 拷贝x再扩展一个前导零
     dq  = tmp[n+1..n+m+1]     // m+1位: 存储 y * qk

  2. memcpy(rem, x, n)
     rem[n] = 0

  3. for k = n - m down to 0:
       // (a) 估计商位
       km = k + m
       y2 = y[m-1] * 256 + y[m-2]
       r3 = rem[km] * 65536 + rem[km-1] * 256 + rem[km-2]
       qk = r3 / y2
       if qk >= 256: qk = 255

       // (b) 计算 dq = y * qk (m+1位)
       dq[m] = XP_product(m, dq, y, qk)

       // (c) 比较 dq 与 rem[k..k+m], 必要时修正
       从高位到低位比较 rem[i+k] 与 dq[i]
       if rem[i+k] < dq[i]:
           qk = qk - 1
           dq[m] = XP_product(m, dq, y, qk)

       // (d) 记录商位
       q[k] = qk

       // (e) 减法: rem[k..k+m] -= dq
       borrow = XP_sub(m+1, &rem[k], &rem[k], dq, 0)
       assert(borrow == 0)

  4. memcpy(r, rem, m)         // rem[0..m-1] 为余数

  5. 填充 q[n-m+1..nx-1] 和 r[m..my-1] 为零
```

---

## 六、比较: `XP_cmp`

```c
int XP_cmp(int n, T x, T y) {
    int i = n - 1;
    while (i > 0 && x[i] == y[i])   /* 从最高位向最低位比较 */
        i--;
    return x[i] - y[i];              /* 第一个不同位决定大小关系 */
}
```

比较从**最高有效位向最低有效位**进行，这与人类比较两个数字的自然方式一致。`return x[i] - y[i]` 利用了无符号字符的减法语义: 如果 `x[i] > y[i]`，返回正值; 如果 `x[i] < y[i]`，由于无符号环绕，返回负值 (在 `int` 提升后)。

---

## 七、位移操作: 大数的左移与右移

### 7.1 两阶段移位策略

`XP_lshift` 和 `XP_rshift` 都采用"**字节批量移动 + 位精细移动**"的两阶段策略:

```
  移位 s 位 = 移位 (s/8) 个完整字节 + 移位 (s%8) 个剩余位

  阶段1: 按字节批量复制 (s/8 字节)
          → 用 memcpy 模式逐字节搬移

  阶段2: 按位精细调整 (s%8 位)
          → 左移: 等价于乘以 2^(s%8), OR上填充位
          → 右移: 等价于除以 2^(s%8), OR上填充位
```

### 7.2 左移: `XP_lshift`

```
  左移13位示例 (6位 → 8位, fill=0):
                                              最高位
    输入 x (6位):  [ 0xFF 0xFF 0xFF 0xFF 0xFF 0x3F ]     (44个1)
                                              ↓
    阶段1 (s/8=1字节):  整体左移1字节, 低字节填fill
    ┌────┬────┬────┬────┬────┬────┬────┬────┐
    │fill│ x0 │ x1 │ x2 │ x3 │ x4 │ x5 │ 0  │
    └────┴────┴────┴────┴────┴────┴────┴────┘
                                              ↓
    阶段2 (s%8=5位): 乘以2⁵, 最低5位 OR fill位
    ┌────┬────┬────┬────┬────┬────┬────┬────┐
    │E0=0xFF>>3│  ...... 移位后的数据 ......  │
    └────┴────┴────┴────┴────┴────┴────┴────┘
```

**阶段1: 字节批量移动**

```c
/* 伪代码等价 */
z[m + s/8 .. n-1]       = 0       // 高位清零
z[s/8 .. m + s/8 - 1]   = x[0..m-1]  // x拷贝到偏移位置
z[0 .. s/8 - 1]         = fill     // 低位填填充字节
```

C实现中，三个 `for` 循环从高位到底位依次完成这三组赋值:

```c
{
    int i, j = n - 1;
    if (n > m)
        i = m - 1;                  /* z 比 x 长: 从x的最高位开始 */
    else
        i = n - s/8 - 1;            /* z 比 x 短: 只复制能放入的部分 */

    for ( ; j >= m + s/8; j--)      /* 赋值1: 高位清零 */
        z[j] = 0;
    for ( ; i >= 0; i--, j--)       /* 赋值2: 复制x */
        z[j] = x[i];
    for ( ; j >= 0; j--)            /* 赋值3: 低位置fill */
        z[j] = fill;
}
```

**阶段2: 位精细移动**

```c
s %= 8;
if (s > 0) {
    XP_product(n, z, z, 1 << s);   /* z = z * 2^s */
    z[0] |= fill >> (8 - s);        /* 低s位用fill填充 */
}
```

`fill >> (8-s)`: 如果 `fill = 0xFF = 11111111`, `fill >> 3` (当s=5时) = `00000111`, 即低s位为1。此值 OR 到 `z[0]` 完成填充。

### 7.3 右移: `XP_rshift`

右移是左移的镜像操作:

```c
void XP_rshift(int n, T z, int m, T x, int s, int fill) {
    fill = fill ? 0xFF : 0;

    /* 阶段1: 字节批量右移 */
    {
        int i, j = 0;
        for (i = s/8; i < m && j < n; i++, j++)
            z[j] = x[i];                /* 从 x[s/8] 开始复制 */
        for ( ; j < n; j++)
            z[j] = fill;                /* 高位填 fill */
    }

    /* 阶段2: 位精细右移 */
    s %= 8;
    if (s > 0) {
        XP_quotient(n, z, z, 1 << s);   /* z = z / 2^s */
        z[n-1] |= fill << (8 - s);       /* 高s位用fill填充 */
    }
}
```

**填充字节的语义**:
- `fill = 0`: 逻辑右移 (Logical shift) —— 空位补0
- `fill = 1`: 算术右移 (Arithmetic shift) —— 空位补1 (保留负数的符号扩展效果)

---

## 八、字符串转换

### 8.1 字符到数字的映射表: `map[]`

```c
static char map[] = {
     0,  1,  2,  3,  4,  5,  6,  7,  8,  9,    /* '0'-'9' → 0-9 */
    36, 36, 36, 36, 36, 36, 36,                  /* 7个非法字符 → 36 */
    10, 11, 12, 13, 14, 15, 16, 17, 18, 19,      /* 'A'-'Z' → 10-35 */
    20, 21, 22, 23, 24, 25, 26, 27, 28, 29,
    30, 31, 32, 33, 34, 35,
    36, 36, 36, 36, 36, 36,                        /* 6个非法字符 → 36 */
    10, 11, 12, 13, 14, 15, 16, 17, 18, 19,      /* 'a'-'z' → 10-35 */
    20, 21, 22, 23, 24, 25, 26, 27, 28, 29,
    30, 31, 32, 33, 34, 35
};
```

`map[c - '0']` 给出字符 `c` 在任意进制 (2-36) 下的数字值。如果返回值 ≥ base，则该字符不是当前进制下的合法数字。值为36的条目对应ASCII表中 `'0'` 到 `'z'` 之间不表示合法数字的字符 (如 `':'`, `';'`, `'<'`, `'='`, `'>'`, `'?'`, `'@'`, `'['`, `'\'`, `']'`, `'^'`, `'_'`, `` '`' ``)。

### 8.2 `XP_fromstr`: 字符串 → XP_T

```c
int XP_fromstr(int n, T z, const char *str, int base, char **end) {
    const char *p = str;
    assert(p);
    assert(base >= 2 && base <= 36);

    /* 跳过前导空白 */
    while (*p && isspace(*p))
        p++;

    if ((*p && isalnum(*p) && map[*p - '0'] < base)) {
        int carry;
        /* 核心转换循环: z = base * z + digit */
        for ( ; (*p && isalnum(*p) && map[*p - '0'] < base); p++) {
            carry = XP_product(n, z, z, base);   /* z *= base */
            if (carry) break;                     /* 溢出: 停止 */
            XP_sum(n, z, z, map[*p - '0']);       /* z += digit */
        }
        if (end) *end = (char *)p;
        return carry;                              /* 0=成功, 非零=溢出 */
    } else {
        if (end) *end = (char *)str;
        return 0;
    }
}
```

**算法**:

```
算法: 字符串转大数 XP_fromstr
  输入: z      — 初始值 (调用者负责初始化, 通常为0)
        str    — "123ABC..." 的字符串
        base   — 2 到 36
        end    — 输出指针 (可为NULL)
  输出: z      — 累加后的值
        返回   — 0=成功, 非零=溢出

  1. 跳过空白字符
  2. for 每个合法数字字符 c in str:
       z = z * base
       if (乘法溢出): break
       z = z + map[c - '0']
  3. if end != NULL: *end = 停止位置的指针
  4. return 溢出标志
```

**关键点**: z不会被初始化为零——调用者必须先初始化z。这允许将多次 `XP_fromstr` 调用的结果累加到同一个z中。

### 8.3 `XP_tostr`: XP_T → 字符串

```c
char *XP_tostr(char *str, int size, int base, int n, T x) {
    int i = 0;
    assert(str);
    assert(base >= 2 && base <= 36);

    do {
        int r = XP_quotient(n, x, x, base);    /* r = x mod base; x = x / base */
        assert(i < size);
        str[i++] = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"[r];

        while (n > 1 && x[n-1] == 0)           /* 归一化: 去除前导零 */
            n--;
    } while (n > 1 || x[0] != 0);               /* x == 0 时停止 */

    assert(i < size);
    str[i] = '\0';

    /* 反转字符串 (digits是逆序生成的) */
    {
        int j;
        for (j = 0; j < --i; j++) {
            char c = str[j];
            str[j] = str[i];
            str[i] = c;
        }
    }
    return str;
}
```

**算法**:

```
算法: 大数转字符串 XP_tostr
  输入: x     — 待转换的XP_T
        base  — 目标进制 (2-36)
        str   — 输出缓冲区
        size  — 缓冲区大小
  输出: str   — null结尾的字符串
        x     — 被清零
        返回  — str

  1. i = 0
  2. do:
       r = x mod base        // XP_quotient 计算 x/base, 返回余数
       str[i++] = digits[r]   // "0-9A-Z" 中的字符
       x = x / base           // x变为商 (原地修改)
       从n中减去除法产生的前导零
     while (x != 0)
  3. str[i] = '\0'
  4. 反转 str[0..i-1]        // 因为我们从低位开始生成的
  5. return str
```

**注意**: `XP_tostr` 会**原地修改 (摧毁)** 输入x——这在接口文档中明确说明 ("x is set to zero")。调用者如需保留原值，应先拷贝。

---

## 九、Modern-C 改进方案

### 9.1 `__int128` 硬件加速

GCC 和 Clang 在64位平台上提供 `__int128` (有符号) 和 `unsigned __int128` (无符号) 扩展类型，可直接用于多精度运算:

```c
/* 传统方式: 使用 unsigned long 作为中间累加器 (最大65535) */
unsigned carry = 0;
carry += x[i] * y[j] + z[i+j];
z[i+j] = carry % BASE;
carry /= BASE;

/* 改进方式: 使用 unsigned __int128, BASE = 2^64 (大幅减少迭代次数) */
typedef unsigned __int128 u128;
#define BASE64 ((u128)1 << 64)

u128 carry = 0;
carry += (u128)x[i] * y[j] + z[i+j];
z[i+j] = (uint64_t)(carry % BASE64);
carry /= BASE64;
```

使用 `__int128` 的核心优势:

| 方面 | BASE=256 (原版) | BASE=2³² | BASE=2⁶⁴ |
|------|-----------------|----------|----------|
| 数字位数 (1024-bit RSA) | 128 | 32 | 16 |
| XP_mul 外层迭代 | 128 | 32 | 16 |
| 加速比 (相对BASE=256) | 1x | ~16x | ~64x |
| 商位估计可行性 | 直接用 `unsigned long` | 需要 `__int128` | 需要 `__int128` |
| 兼容性 | 100% ANSI C | GCC/Clang扩展 | GCC/Clang扩展 |

对于64位平台上的 `BASE=2⁶⁴` 实现，商位估计 `qk = r3 / y2` 需要 `unsigned __int128` 除法:

```c
/* 当 BASE=2^64 时，r3 可达 (2^64-1)^3 ≈ 2^192，需要128位除法 */
unsigned __int128 r3 = (unsigned __int128)rem[km] * BASE64 * BASE64
                     + (unsigned __int128)rem[km-1] * BASE64
                     + rem[km-2];
unsigned __int128 y2 = (unsigned __int128)y[m-1] * BASE64 + y[m-2];
qk = r3 / y2;   /* 128位除以64位, 硬件指令支持 */
```

### 9.2 无进位乘法 (Carryless Multiply)

现代x86-64处理器 (自Haswell起) 提供 `PCLMULQDQ` 指令，执行**无进位乘法** (carryless multiplication / polynomial multiplication over GF(2))。这与XP的整数乘法不同——无进位乘法对应的是二元域GF(2)上的多项式乘法，而非整数算术。

无进位乘法在以下场景中与扩展精度算术相关:
- **密码学**: GCM (Galois/Counter Mode) 认证加密中的GHASH运算
- **CRC计算**: 快速CRC-32/CRC-64
- **纠错码**: Reed-Solomon编码

在C中可通过编译器内建函数使用:

```c
#include <wmmintrin.h>  /* x86 */

/* _mm_clmulepi64_si128: 对两个64位操作数执行无进位乘法，产生128位结果 */
__m128i a = _mm_set_epi64x(0, x_i);   /* 加载64位的x_i */
__m128i b = _mm_set_epi64x(0, y_j);   /* 加载64位的y_j */
__m128i result = _mm_clmulepi64_si128(a, b, 0);
/* result 的低64位 = carryless(x_i, y_j) */
```

**注意**: `PCLMULQDQ` 执行的是GF(2)多项式乘法，而非整数乘法。两者虽然结构相似 (都需要卷积/累加)，但进位处理不同:
- 整数乘法: `z[i+j] += x[i]*y[j] + carry`, 然后 `carry = (sum) / BASE`
- 无进位乘法: `z[i+j] ^= x[i] * y[j]` (XOR替代加法, 无进位传播)

对于XP库本身 (做整数算术)，`PCLMULQDQ` 不是直接可用的替代品。但在构建**同时支持整数和GF(2)算术**的扩展精度库时，统一代码结构可以同时服务两种需求。

### 9.3 汇编语言优化

Hanson在书中明确指出:"XP's interface is as simple as possible so that some of the functions can be implemented in assembly language, if performance considerations necessitate."

汇编优化的关键收益来源:
1. **直接访问进位标志**: C语言无法直接读取处理器的进位标志 (carry flag)。`XP_add` 的 `carry /= BASE` 会被编译为多条指令，而汇编可以用单条 `ADC` (add with carry) 指令。
2. **双倍精度乘法**: x86的 `MUL` 指令产生双倍宽度的结果 (如32位×32位→64位)，C中需要两次运算 (乘积和进位) 才能取出。
3. **商位估计**: `DIV` 指令可直接用于估计，减少软件模拟开销。

以x86-64汇编为例，`XP_add` 的核心循环从:

```c
/* C版本 (约10-15条指令/迭代) */
carry += x[i] + y[i];
z[i] = carry % BASE;
carry /= BASE;
```

变为:

```asm
; x86-64 汇编 (3条指令/迭代)
; rsi=x, rdx=y, rdi=z, ecx=n, cl=carry
.loop:
    mov   al, [rsi]       ; AL = x[i]
    adc   al, [rdx]       ; AL += y[i] + CF (carry)
    mov   [rdi], al       ; z[i] = AL
    inc   rsi / rdx / rdi
    dec   ecx
    jnz   .loop
    setc  al              ; 最终进位
```

---

## 十、与 GMP 库的对比

[GNU Multiple Precision Arithmetic Library (GMP)](https://gmplib.org/) 是事实上的大整数运算标准库。以下是XP与GMP的系统性对比:

### 10.1 设计哲学

| 维度 | XP (CII) | GMP |
|------|----------|-----|
| 设计目标 | 教育性/简洁性 | 极限性能 |
| 代码规模 | ~275行 (xp.c) | ~150,000行 |
| 数字基数 | 固定 BASE=2⁸ | 自适应: 2³² 或 2⁶⁴ |
| 内存管理 | 调用者管理, 零分配 | 自动分配 (`mpz_init`) |
| 接口抽象 | 暴露裸指针 | 不透明类型 `mpz_t` |
| 错误检测 | 几乎无 (u.r.e.) | 部分检测 + abort |
| 汇编优化 | 无 | 为每个CPU微架构手写汇编 |
| 算法复杂度 | 教科书算法 O(n²) | Karatsuba, Toom-Cook, FFT |

### 10.2 算法对比

| 运算 | XP (CII) | GMP |
|------|----------|-----|
| 乘法 | 朴素 O(n²) 逐位乘 | 自适应: 朴素→Karatsuba→Toom-3→Toom-4→FFT |
| 除法 | Brinch-Hansen O(n²) | 自适应: 朴素→分治除法 |
| 幂运算 | 由AP层实现 (平方-乘) | `mpz_powm`: 滑动窗口 + Montgomery |
| GCD | 未实现 | `mpz_gcd`: 二进制GCD + Lehmer加速 |
| 素性测试 | 未实现 | Miller-Rabin 概率测试 |
| 进制转换 | O(n²) 逐位除法 | 分治转换 O(M(n) log n) |

### 10.3 性能差异 (数量级)

对于1024位 (128字节) 整数乘法:
- **XP**: O(n²) = 128² = 16,384 次乘加运算, 约 **50-100 微秒**
- **GMP**: Karatsuba + 汇编优化, 约 **1-3 微秒**
- **差异**: GMP快约 **20-100倍**

对于8192位 (RSA-8192) 乘法:
- **XP**: 朴素 O(n²), 约 **数秒**
- **GMP**: 使用FFT乘法, 约 **数毫秒**
- **差异**: GMP快约 **1000倍**

### 10.4 何时应该使用XP而非GMP

| 场景 | 推荐理由 |
|------|----------|
| 理解大整数算法 | XP的275行代码是可完整通读的教科书，GMP的15万行则不然 |
| 嵌入式系统 | XP零动态内存，GMP需要malloc |
| 教学/实验 | 修改XP的算法来实验变体很直接 |
| 固定精度/小数值 | 如果数字永远< 512位，XP的开销足够低 |
| 代码移植性 | 纯ANSI C，GMP需要特定平台的汇编文件 |

### 10.5 如何桥接XP和GMP

一个有趣的练习是用XP的接口风格实现一个GMP后端的子集。例如，GMP的 `mpn_*` 低级函数族 (`mpn_add_n`, `mpn_sub_n`, `mpn_mul_1` 等) 与XP的函数族 (`XP_add`, `XP_sub`, `XP_product`) 在概念上完全对应，只是操作数的基数更大 (GMP使用机器字长而非字节)。

```c
/* GMP低级接口 (mpn) 与 XP接口的对应 */
mpn_add_n(rp, ap, bp, n)       ≈  XP_add(n, z, x, y, 0)     // n-limb加法
mpn_sub_n(rp, ap, bp, n)       ≈  XP_sub(n, z, x, y, 0)     // n-limb减法
mpn_mul_1(rp, ap, n, vl)       ≈  XP_product(n, z, x, y)    // 单limb乘法
mpn_divrem(qp, 0, np, nn, dp, dn) ≈  XP_div(n, q, x, m, y, r, tmp)
```

从XP到GMP的学习路径: XP → AP → MP → GMP的mpn层。每一步都引入更多复杂性，但每一步也都基于前一步的核心思想。

---

## 十一、CII 架构中的定位

XP在CII库的层次结构中处于底层:

```
          ┌──────────────────────────┐
          │     应用程序 (用户代码)      │
          ├──────────────────────────┤
          │  calc (第18章)            │  ← 交互式计算器
          ├──────────────────────────┤
          │  AP (第18章)             │  ← 任意精度有符号整数 (sign-magnitude)
          │  MP (第19章)             │  ← 多精度有符号/无符号整数 (two's complement)
          ├──────────────────────────┤
          │  XP (第17章)  ← 本章     │  ← 扩展精度无符号整数 (base-256)
          ├──────────────────────────┤
          │  Mem, Arena, Assert     │  ← 内存管理与错误检测
          └──────────────────────────┘
```

AP接口使用XP实现其底层运算——AP将符号-大小表示中的"大小"部分存储为XP_T，并调用 `XP_add`, `XP_sub`, `XP_mul`, `XP_div` 等函数进行实际的多精度运算。这种分层设计使得AP可以专注于符号处理和归一化，而将位操作细节委托给XP。

---

## 十二、总结

XP接口用不到300行C代码实现了完整的多精度无符号整数运算，是CII库中"小而美"设计的典范:

| 优势 | 局限 |
|------|------|
| 零动态分配，适合嵌入式 | 无运行时错误检测 (危险接口) |
| 教科书级别的算法教学价值 | 仅 O(n²) 算法, 无Karatsuba/FFT |
| 纯ANSI C，无平台依赖 | 固定BASE=256, 无法利用64位字长 |
| 简洁接口便于汇编替代 | 无类型安全 (裸 unsigned char*) |
| 单文件, 零外部依赖 | 无GCD, 模幂, 素性测试等高级操作 |

Hanson的设计选择了 **清晰性 优于 性能**、**简洁性 优于 完备性**。这种取舍使XP成为一个理想的教学工具和学习大整数算法的起点。对于生产环境的任意精度需求，开发者应首先评估GMP等专业库——除非对代码大小、零依赖或教学目的有特殊要求。

---

## 参考文献

- Brinch-Hansen, P. (1994). "Multiple-Length Division Revisited: A Tour of the Minefield." *Software—Practice and Experience*, 24(6), 579–601.
- Hennessy, J. L. and Patterson, D. A. (1994). *Computer Organization and Design: The Hardware/Software Interface*. Morgan Kaufmann, Chapter 4.
- Knuth, D. E. (1981). *The Art of Computer Programming, Vol. 2: Seminumerical Algorithms*, 2nd ed. Addison-Wesley, Section 4.3.
- GNU Multiple Precision Arithmetic Library. https://gmplib.org/
