# 第18/19章 多精度算术 (MP: Multiple-Precision Arithmetic)

> 译注: 本书第18章名为 "Arbitrary-Precision Arithmetic"，介绍 **AP** 接口 (符号-幅值表示、透明指针封装)；第19章名为 "Multiple-Precision Arithmetic"，介绍 **MP** 接口 (补码表示、定长 n 位、暴露底层存储)。二者均构建于 XP (第17章) 之上。本章融合两章内容，以 MP 为主线，辅以 AP 的对比分析。

---

## 18/19.1 概述

MP 是 CII 库三个算术接口中的最后一个。前两个分别是：

| 接口 | 章节 | 表示 | 精度模型 | 适用场景 |
|------|------|------|----------|----------|
| **XP** | Ch17 | unsigned char 数组, 无符号 | 定长 n 位，无符号 | 底层算术原语 |
| **AP** | Ch18 | 符号-幅值, 不透明指针 `struct T*` | 任意精度, 自动扩展 | 通用大整数计算 |
| **MP** | Ch19 | `unsigned char*`, 补码 | 定长 n 位, 全局设定 | 编译器、密码学 |

MP 的设计定位是**不需要频繁分配内存**、**需要精确控制位宽**、**同时支持有符号和无符号运算**的应用场景。典型用例包括：

1. **交叉编译器**: 在 32 位机器上编译生成 64 位目标代码，需要操作 64 位整数常量。
2. **密码学**: RSA、ECDSA 等算法需要数百位的定长整数运算。
3. **浮点常量转换**: 将十进制浮点字面量精确转换为 IEEE 754 浮点数时，可能需要多精度整数 (Clinger 1990)。

MP 直接暴露其底层表示——`MP_T` 即 `unsigned char*`，与 XP 相同。一个 n 位整数由 `nbytes = (n-1)/8 + 1` 个字节组成，**小端序**存储。

---

## 18/19.2 MP 在 XP 之上构建

### 层次关系

```
MP (有符号/无符号 n 位整数, 补码, 溢出检测, 字符串转换)
 ↑
XP (无符号 n 位整数, 位宽以字节计, 无溢出检测, 字符串转换)
 ↑
Arith (可移植的有符号整数除法和取模)
```

MP 的几乎所有算术运算最终都委托给 XP 函数。MP 层增加的职责是：

1. **符号处理**: 在补码表示下进行符号感知的运算和溢出检测。
2. **溢出检测**: 检查结果是否超出 n 位宽的限制。
3. **位宽管理**: 通过 `MP_set(n)` 全局设定位宽，避免每次调用传递 n。
4. **临时空间管理**: 在 `MP_set` 中一次性分配 `3*nbytes + 2*nbytes+2` 的临时空间，避免算术函数反复分配。

### MP_set: 全局精度设定

```c
int MP_set(int n);
```

MP 默认初始化为 32 位算术。调用 `MP_set(n)` 后，所有后续 MP 函数调用均使用 n 位精度。该函数返回之前的精度值。内部计算三个关键常量：

```c
nbits = n;                               // 位宽
nbytes = (n - 1) / 8 + 1;                // 所需字节数
shift  = (n - 1) % 8;                    // 符号位在最高字节中的偏移
msb    = ones(n);                         // 低 (shift+1) 位为 1 的掩码
```

其中 `ones(n)` 宏生成低 `(n-1)%8+1` 位全 1 的掩码。当 n=32 时: `nbytes=4`, `shift=7`, `msb=0xFF`。

**临时空间布局** (由 `MP_set` 一次性分配):

```
tmp[0]: nbytes       ← 用于存放取负后的 x
tmp[1]: nbytes       ← 用于存放取负后的 y 或 y 的副本
tmp[2]: nbytes       ← 用于存放转换后的立即数操作数
tmp[3]: 2*nbytes+2   ← 用于存放双倍长乘积或 XP_div 的临时空间
```

---

## 18/19.3 MP_T 数据结构: 补码表示 (非符号-幅值)

### 重要澄清: MP 使用补码, AP 使用符号-幅值

| 性质 | AP (Ch18) | MP (Ch19) |
|------|-----------|-----------|
| 类型定义 | `typedef struct T *T;` (不透明指针) | `typedef unsigned char *T;` (暴露存储) |
| 有符号表示 | **符号-幅值** (`int sign; XP_T digits;`) | **二进制补码** (bit n-1 为符号位) |
| 零的表示 | 唯一正零 (`sign=1, ndigits=1, digits[0]=0`) | 唯一全零 |
| 分配模型 | 每次运算分配新对象 | 同对象复用 (z 可等于 x 或 y) |
| 精度模型 | 自动扩展，仅受内存限制 | 固定 n 位，由 `MP_set` 设定 |

**MP 的补码布局**:

```
地址递增 →
+-----------+-----------+-----+-----------+-----------+
| byte      | byte      | ... | byte      | byte      |
| nbytes-1  | nbytes-2  |     | 1         | 0         |
+-----------+-----------+-----+-----------+-----------+
  ↑                                   ↑
  符号位 = bit n-1              最低有效字节 (LSB)
  (第 nbytes-1 字节的 shift 位)
```

符号位提取宏:

```c
#define sign(x) ((x)[nbytes-1]>>shift)
// 当 nbits=32 时等价于 x[3]>>7
```

这个表示选择带来了一个关键优势：**有符号加法和无符号加法可以使用完全相同的底层 XP 运算**，区别仅在于溢出检测逻辑。AP 的符号-幅值表示则需要加法去判断操作数符号、可能转化为实际减法。

### 零的表示

在补码表示中，零只有一种表示: 所有位（所有 nbytes 个字节）全为 0。规范化条件是自然的：最高有效字节在 msb 掩码之外的位应为符号位的扩展。

---

## 18/19.4 接口 API 逐行详解

MP 接口共 49 个函数和 2 个异常，分为以下类别：

### 4.1 初始化与分配

#### MP_new

```c
T MP_new(unsigned long u);
// 功能: 分配 nbytes 字节的 MP_T, 初始化为 u, 返回之
// 实现: 调用 ALLOC(nbytes) 然后委托给 MP_fromintu
// 异常: 若 u 超出 n 位无符号范围 → MP_Overflow
// 注意: 溢出时仍会先设置结果 (保留低 nbits), 然后抛出异常
```

实现 (from Ch19, mp.c):
```c
T MP_new(unsigned long u) {
    return MP_fromintu(ALLOC(nbytes), u);
}
```

#### MP_fromint / MP_fromintu

```c
T MP_fromint (T z, long v);           // 有符号, v 可为负
T MP_fromintu(T z, unsigned long u);  // 无符号, u ≥ 0
// 功能: 将 C 的 long / unsigned long 值写入 z
// 策略:
//   MP_fromintu → XP_fromint (设置字节) + 溢出检测 (~msb 位是否非零)
//   MP_fromint  → 处理 LONG_MIN 特例 + 负数取补码 + 符号范围检查
// 异常: 溢出 → MP_Overflow (但 z 已被设置)
```

**MP_fromint 处理 LONG_MIN 的关键逻辑** (from Ch19, set z to v):

```c
if (v == LONG_MIN) {
    // LONG_MIN = -LONG_MAX-1, 不能直接取负
    XP_fromint(nbytes, z, LONG_MAX + 1UL);  // 先设幅值
    XP_neg(nbytes, z, z, 1);                 // 再取补码
} else if (v < 0) {
    XP_fromint(nbytes, z, -v);              // 先设绝对值
    XP_neg(nbytes, z, z, 1);                // 再取补码
} else {
    XP_fromint(nbytes, z, v);               // 正数直接设
}
z[nbytes-1] &= msb;  // 清除扩展位
```

有符号溢出检查:
```c
// v 是否超出 nbits 位有符号整数的范围?
(nbits < 8*sizeof(v) &&
 (v < -(1L<<(nbits-1)) || v >= (1L<<(nbits-1))))
```

#### MP_free

MP 不提供独立的 `MP_free` 函数 (AP 提供了 `AP_free`)。分配由 `ALLOC`/`FREE` 宏直接管理，体现 MP 的低级定位。

---

### 4.2 转换函数

#### MP_toint / MP_tointu

```c
long          MP_toint (T x);  // 有符号转换
unsigned long MP_tointu(T x);  // 无符号转换
// 功能: 将 MP_T 转为 C 整型
// 策略: 内部先调用 MP_cvt/MP_cvtu 转为 sizeof(long) 位宽, 再 XP_toint
// 异常: x 超出目标类型范围 → MP_Overflow (无法捕获结果)
```

#### MP_cvt / MP_cvtu

```c
T MP_cvt (int m, T z, T x);  // 有符号位宽转换
T MP_cvtu(int m, T z, T x);  // 无符号位宽转换
// 功能: 将 x 转为 m 位宽的 MP_T 存入 z
// 参数: m 为新的位宽 (≥ 2)
// 策略:
//   m < nbits → 缩窄: 检查高位是否均为符号扩展, 溢出则抛异常
//   m ≥ nbits → 扩展: 无符号填 0, 有符号填符号位
// 异常: 缩窄时超出范围 → MP_Overflow (z 已设置)
```

**缩窄时的关键检查** (from Ch19, narrow signed x):

```c
int fill = sign(x) ? 0xFF : 0;
// 检查 bits [m..nbits-1] 是否全是符号扩展
int carry = (x[mbytes-1] ^ fill) & ~ones(m);
for (i = mbytes; i < nbytes; i++)
    carry |= x[i] ^ fill;
// carry != 0 表示溢出
```

#### MP_fromstr / MP_tostr / MP_fmt / MP_fmtu

```c
T      MP_fromstr(T z, const char *str, int base, char **end);
char  *MP_tostr  (char *str, int size, int base, T x);
void   MP_fmt    (int code, va_list *app, ...);  // 有符号格式化
void   MP_fmtu   (int code, va_list *app, ...);  // 无符号格式化
```

- `MP_fromstr`: 类似 `strtoul`, 从字符串解析无符号整数。base ∈ [2, 36]。溢出时设置 `*end` 为终止字符后抛出 `MP_Overflow`。
- `MP_tostr`: 将 MP_T 转为字符串。若 `str == NULL`, 内部自动分配空间 (调用者负责释放)。使用 `nbits/k + 2` 估算空间 (k 为 ≤ base 的最大 2 的幂次)。
- `MP_fmt` (有符号): 若 x 为负, 先取补码再用 `MP_tostr` 转换, 最后在字符串前添加 `-`。
- `MP_fmtu` (无符号): 直接调用 `MP_tostr`。

---

### 4.3 有符号算术运算 (补码, 含溢出检测)

#### MP_add — 有符号加法

```c
T MP_add(T z, T x, T y);
// 功能: z = x + y (补码加法)
// 策略: 直接调用 XP_add (补码加法与无符号加法硬件一致)
// 溢出: x 和 y 同号 且 结果符号不同于操作数符号 → MP_Overflow
```

溢出判断逻辑 (from Ch19):
```c
sx = sign(x);
sy = sign(y);
XP_add(nbytes, z, x, y, 0);  // 直接加, 补码自动处理符号
z[nbytes-1] &= msb;
if (sx == sy && sy != sign(z))
    RAISE(MP_Overflow);
// 原理: 同号相加结果符号必同号, 否则溢出
```

#### MP_sub — 有符号减法

```c
T MP_sub(T z, T x, T y);
// 功能: z = x - y (补码减法)
// 溢出: x 和 y 异号 且 结果符号等于 y 的符号 → MP_Overflow
```

溢出判断逻辑:
```c
// x > 0, y < 0 → 期望结果 > 0; x < 0, y > 0 → 期望结果 < 0
if (sx != sy && sy == sign(z))
    RAISE(MP_Overflow);
```

#### MP_neg — 取负

```c
T MP_neg(T z, T x);
// 功能: z = -x
// 策略: XP_neg (逐字节取反 + 1)
// 溢出: 仅当 x 为负且结果仍为负 (即 x = 0x80...00, 最小负数)
```

```c
sx = sign(x);
XP_neg(nbytes, z, x, 1);  // carry=1 实现 "取反加一"
if (sx && sx == sign(z))
    RAISE(MP_Overflow);
// 最小负数的负数仍是其自身 → 溢出
```

#### MP_mul / MP_mul2 — 有符号乘法

```c
T MP_mul  (T z, T x, T y);  // 单倍长: z = x*y mod 2^n
T MP_mul2 (T z, T x, T y);  // 双倍长: z = x*y (2n 位), 永不溢出
// 策略:
//   1. 记录符号 sx, sy
//   2. 将负操作数取补码变为正数 (存入 tmp[0], tmp[1])
//   3. 调用 XP_mul 计算无符号 2n 位乘积 (存入 tmp[3])
//   4. 若 sx != sy, 对结果取补码
//   5. MP_mul: 截取低 nbits + 溢出检测;  MP_mul2: 全部 2nbits
```

**关键注意**: `MP_mul2` 需要 `z` 有 `(2*nbits-1)/8+1` 字节空间，不能由 `MP_new` 分配 (它只分配 nbytes)。

#### MP_div — 有符号除法 (截断策略)

```c
T MP_div(T z, T x, T y);
// 功能: z = x / y
// 截断策略: 向负无穷截断 (floor division)
//   同号 → q ≥ 0, 直接无符号除法即得
//   异号 → q < 0, 需调整: q = -(|x|/|y|), 若余数非零则 q -= 1
// 溢出: 仅当 x 为最小负数且 y = -1 (商超出正数范围)
// 异常: y = 0 → MP_Dividebyzero
```

**符号相异时的商调整** (from Ch19, adjust the quotient):
```c
XP_neg(nbytes, z, z, 1);      // q = -(|x|/|y|)
if (!iszero(tmp[2]))           // 若有非零余数
    XP_diff(nbytes, z, z, 1);  // q = q - 1 (向负无穷截断)
```

#### MP_mod — 有符号取模 (余数永为正)

```c
T MP_mod(T z, T x, T y);
// 功能: z = x mod y
// 定义: x mod y = x - y * floor(x/y)
// 结果: 余数始终非负
// 策略:
//   同号 → 余数即无符号除法余数
//   异号 → 若余数非零, 余数 = |y| - 余数
```

**符号相异时的余数调整**:
```c
if (!iszero(z))
    XP_sub(nbytes, z, y, z, 0);  // 余数 = |y| - 余数
```

---

### 4.4 无符号算术运算

```c
T MP_addu(T z, T x, T y);  // z = x + y
T MP_subu(T z, T x, T y);  // z = x - y (x < y 时溢出)
T MP_mulu(T z, T x, T y);  // z = x * y mod 2^n
T MP_divu(T z, T x, T y);  // z = x / y
T MP_modu(T z, T x, T y);  // z = x mod y
T MP_mul2u(T z, T x, T y); // z = x*y 双倍长 (永不溢出)
```

无符号加法溢出: 进位非零 或 最高字节超出 msb 掩码 → `MP_Overflow`。

无符号减法溢出: `XP_sub` 返回非零借位 (即 x < y) → `MP_Overflow`。

无符号乘法溢出 (单倍长):
```c
// 检查 tmp[3] 的高 nbytes 中是否有非零字节
if (tmp[3][nbytes-1] & ~msb)
    RAISE(MP_Overflow);
for (i = 0; i < nbytes; i++)
    if (tmp[3][i + nbytes] != 0)
        RAISE(MP_Overflow);
```

---

### 4.5 便捷函数 (立即数操作数)

```c
T   MP_addi (T z, T x, long y);           // z = x + y
T   MP_subi (T z, T x, long y);           // z = x - y
T   MP_muli (T z, T x, long y);           // z = x * y
T   MP_divi (T z, T x, long y);           // z = x / y
long MP_modi (T x, long y);               // 返回 x mod y
// 无符号版本: _addui, _subui, _mului, _divui, _modui
```

这些函数避免分配：当 `y` 是单字节 (小于 256) 时，直接调用 XP 的单数字函数 (`XP_sum`, `XP_diff`, `XP_product`, `XP_quotient`)；否则通过内部 `apply`/`applyu` 函数将 `y` 转换为临时 `MP_T` 后调用通用版本。

**`applyu` 辅助函数** (from Ch19):
```c
static int applyu(T op(T, T, T), T z, T x, unsigned long u) {
    unsigned long carry;
    { T z = tmp[2]; /* set z to u 的代码 */ }
    op(z, x, tmp[2]);                // 调用通用函数
    return carry != 0;               // 返回 u 是否溢出
}
// applyu 在 u 溢出时仍先计算结果再返回 1
```

**`apply` 辅助函数** (有符号版):
```c
static int apply(T op(T, T, T), T z, T x, long v) {
    { T z = tmp[2]; /* set z to v (处理负数/LONG_MIN) */ }
    op(z, x, tmp[2]);
    return /* v 是否超出 nbits 位有符号范围 */;
}
```

---

### 4.6 比较运算

```c
int MP_cmp  (T x, T y);    // 有符号比较
int MP_cmpi (T x, long y); // x vs. 立即数
int MP_cmpu (T x, T y);    // 无符号比较
int MP_cmpui(T x, unsigned long y);
// 返回: <0 (x<y), =0 (x=y), >0 (x>y)
```

**有符号比较的关键逻辑**:

```c
sx = sign(x);
sy = sign(y);
if (sx != sy)
    return sy - sx;        // 符号不同: 正 > 负
else
    return XP_cmp(nbytes, x, y);  // 符号相同: 按无符号比
```

`MP_cmpi` 和 `MP_cmpui` 不要求 `y` 能放入 `MP_T`；当 `y` 超出范围时该事实反映在比较结果中。

---

### 4.7 位运算

```c
T MP_and (T z, T x, T y);   // z = x & y
T MP_or  (T z, T x, T y);   // z = x | y
T MP_xor (T z, T x, T y);   // z = x ^ y
T MP_not (T z, T x);        // z = ~x
T MP_andi(T z, T x, unsigned long y);  // z = x & y
T MP_ori (T z, T x, unsigned long y);
T MP_xori(T z, T x, unsigned long y);
```

位运算永不会抛出异常。`MP_not` 后需用 `z[nbytes-1] &= msb` 清除扩展位。立即数版本内部调用 `applyu` 但忽略其返回值 (位运算不产生溢出)。

实现极其简洁 (来自 Ch19 的 `bitop` 宏):
```c
#define bitop(op) \
    int i; assert(z); assert(x); assert(y); \
    for (i = 0; i < nbytes; i++) z[i] = x[i] op y[i]; \
    return z
```

---

### 4.8 移位运算

```c
T MP_lshift(T z, T x, int s);  // 逻辑左移, 空位填 0
T MP_rshift(T z, T x, int s);  // 逻辑右移, 空位填 0
T MP_ashift(T z, T x, int s);  // 算术右移, 空位填符号位
// 约束: s ≥ 0 (checked runtime error)
```

实现模式 (`shft` 宏):
```c
#define shft(fill, op) \
    if (s >= nbits) memset(z, fill, nbytes); \
    else op(nbytes, z, nbytes, x, s, fill);  \
    z[nbytes-1] &= msb;
```

- `MP_lshift`: `shft(0, XP_lshift)` — 左移 s 位，等价于乘以 2^s
- `MP_rshift`: `shft(0, XP_rshift)` — 逻辑右移
- `MP_ashift`: `shft(sign(x), XP_rshift)` — 算术右移，`fill = sign(x) ? 0xFF : 0`

---

### 4.9 异常

```c
extern const Except_T MP_Overflow;       // "Overflow"
extern const Except_T MP_Dividebyzero;   // "Division by zero"
```

**关键的溢出语义**: 所有 MP 函数**先计算结果再抛出异常**。超出的位被简单丢弃，结果保留低 `nbits` 位。这允许客户端用 `TRY-EXCEPT` 忽略溢出:

```c
MP_T z = MP_new(0);
TRY
    MP_fromintu(z, 0xFFF);   // z = 0xFF (取低 8 位)
EXCEPT(MP_Overflow);         // 忽略溢出, z 有效
END_TRY;
```

---

## 18/19.5 符号处理技巧: 补码 vs 符号-幅值

### 为什么 MP 选择补码而 AP 选择符号-幅值

| 考量 | 符号-幅值 (AP) | 补码 (MP) |
|------|----------------|-----------|
| 加法实现 | 需要区分 4 种符号组合 | 统一: 补码加法 = 无符号加法 |
| 零的表示 | 两套 (+0 和 -0), 需规范化 | 唯一零 |
| 扩展/缩窄 | 自然 (`ndigits` 变化) | 需显式符号扩展/掩码 |
| 分配模型 | 每次运算分配新对象, 无别名问题 | 同对象复用, z 可为 x 或 y |
| 运算速度 | 符号判断分支开销 | 无分支, 但溢出检测有开销 |
| 适用场景 | 通用大整数 (Python int) | 编译器常量、密码学 |

### MP 的符号位定位

```c
#define sign(x) ((x)[nbytes-1] >> shift)
// 当 nbits=32: 取 x[3] 的最高位 (bit 7)
// 当 nbits=31: 取 x[3] 的 bit 6
```

### 溢出检测的核心模式

溢出检测是 MP 实现中最复杂的部分。核心模式总结如下：

**无符号溢出**:
```c
carry  = XP_...(...);
carry |= z[nbytes-1] & ~msb;   // 检查超出 nbits 的高位是否非零
z[nbytes-1] &= msb;             // 截断
if (carry) RAISE(MP_Overflow);
```

**有符号加法溢出**:
```c
// 同号相加, 结果符号必须与操作数相同
if (sx == sy && sy != sign(z))
    RAISE(MP_Overflow);
```

**有符号减法溢出**:
```c
// 异号相减, 结果符号不能等于减数符号
if (sx != sy && sy == sign(z))
    RAISE(MP_Overflow);
```

**有符号乘法溢出** (先转为无符号计算):
```c
// (1) 先检查无符号溢出: 高半部分非零
// (2) 再检查符号语义: 同号相乘结果必为非负
if (sx == sy && sign(z))
    RAISE(MP_Overflow);
```

---

## 18/19.6 AP 接口 (第 18 章) 补充: 符号-幅值实现

为完整起见，补充 AP 接口的关键实现细节 (与 MP 对比)。

### AP_T 结构

```c
struct T {
    int sign;      // 1 或 -1, 零永远是 +0
    int ndigits;   // 有效位数
    int size;      // 分配的位数
    XP_T digits;   // 指向紧邻结构体的字节数组
};
```

结构体和 `digits` 数组通过单次 `CALLOC` 分配 (结构体后的灵活数组):
```c
static T mk(int size) {
    T z = CALLOC(1, sizeof(*z) + size);
    z->sign = 1;         // 默认为正零
    z->size = size;
    z->ndigits = 1;
    z->digits = (XP_T)(z + 1);  // 指向结构体后的空间
    return z;
}
```

### AP 加法的符号处理

与 MP 不同, AP 的加法必须区分符号:

| y < 0 | y ≥ 0 |
|-------|-------|
| x < 0: -( \|x\| + \|y\| ) | x < 0: \|y\| - \|x\| |
| x ≥ 0: \|x\| - \|y\| | x ≥ 0: \|x\| + \|y\| |

实际计算 \|x\| - \|y\| 时需判断大小以确保结果正确符号。

### AP 乘法的符号处理

```c
z->sign = iszero(z) || ((x->sign ^ y->sign) == 0) ? 1 : -1;
// 同号为 +, 异号为 -, 零为正
```

### AP 除法与取模的截断策略

AP 的除法**向负无穷截断** (与 MP 相同, 与 Python `//` 相同):

- q = floor(x / y), 即满足 q * y <= x 的最大整数
- 余数 r = x - y*q, 始终 ≥ 0

实现 (摘录):
```c
// 截断策略与 Arith_div 相同 (见 Ch2)
// 验证: -13/5 = -3 (floor), 余数 = 2
//        13/-5 = -3 (floor), 余数 = -2... 不, 余数为正 = 2
```

---

## 18/19.7 Karatsuba 乘法的实现分析

Karatsuba 算法 (1962) 仅在本书的**习题 18.4** 中被提及, 用于加速 AP 乘法, 并未在 MP 或 AP 的正文实现中出现。正文中的乘法使用小学算法 (复杂度 O(n^2))。

### 算法原理

对于两个 2n 位数 x 和 y, 将每个数拆分为高 n 位和低 n 位:

```
x = a * B^n + b
y = c * B^n + d

x * y = ac * B^(2n) + (ad + bc) * B^n + bd
```

传统方法需要 4 次 n 位乘法: `ac, ad, bc, bd`。

**Karatsuba 技巧**: 注意到 `ad + bc = ac + bd + (a-b)(d-c)`。只需要 3 次乘法: `ac, bd, (a-b)(d-c)`, 加上 2 次减法和 2 次加法。大 O 复杂度从 O(n^2) 降至 O(n^log2(3)) ≈ O(n^1.585)。

### 递归实现框架

```c
// 伪代码: 基于 XP_mul 的递归 Karatsuba
T XP_karatsuba(T z, int n, T x, T y) {
    if (n <= KARATSUBA_THRESHOLD)
        return XP_mul(z, n, x, n, y);   // 小学算法用于小规模
    int m = n / 2;
    // 拆分: x = a*B^m + b, y = c*B^m + d
    // 递归: ac, bd, (a-b)(d-c)
    // 组合: z = ac*B^(2m) + (ac+bd+(a-b)*(d-c))*B^m + bd
}
```

**门限值**: 习题要求确定 n 为何值时 Karatsuba 显著快于小学算法。实践中门限值通常在 20-100 个数字之间, 取决于硬件和数字基数。

**为什么 MP 没有内置 Karatsuba**: MP 定位于定长 n 位运算, 典型 n 为 32-128 位。对于这种规模, 小学乘法 (使用硬件整数乘法指令计算逐字节乘积) 已经足够高效。Karatsuba 的递归开销在 n 较小时抵消了其优势。

---

## 18/19.8 整数除法与取模的截断策略

### 三种截断方案对比

C 语言中整数除法的截断方向是**实现定义**的 (C89/C90), 尽管大多数现代平台采用向零截断。本书的三个算术接口采用了不同的策略:

| 接口 | 除法截断 | 余数符号 | 数学定义 |
|------|----------|----------|----------|
| **Arith** (Ch2) | 向负无穷 (floor) | 余数 ≥ 0 | `Arith_div(x,y) = floor(x/y)` |
| **AP** | 向负无穷 (floor) | 余数 ≥ 0 | q = max {k \| k*y ≤ x} |
| **MP** | 向负无穷 (floor) | 余数 ≥ 0 | 同 AP |

三者统一采用 **"向负无穷截断" (floor division)**，余数始终非负。这是有意为之的设计决策，与 Python 的 `//` 和 `%` 行为一致，但与 C (C99 起规定向零截断) 和大多数 CPU 不同。

### 为什么选择向负无穷

1. **余数非负** 消除了多种特殊情况的处理。
2. **数学一致性**: `(x/y)*y + x%y = x` 对所有整数 x, y 成立。
3. **密码学需求**: 模运算通常要求余数在 [0, m-1] 范围内。
4. **可移植性**: C89 中除法截断方向是实现定义的，向负无穷截断提供了跨平台一致的行为。

### Arith 接口的可移植实现

Arith 接口 (Ch2) 在编译时检测平台的除法截断方向:

```c
int Arith_div(int x, int y) {
    if (-13/5 == -2        // 平台使用向零截断
    &&  (x < 0) != (y < 0) // x 和 y 符号不同
    &&  x % y != 0)        // 且不能整除
        return x/y - 1;    // 修正: -13/5 = -2 → -3
    else
        return x/y;
}

int Arith_mod(int x, int y) {
    if (-13/5 == -2
    &&  (x < 0) != (y < 0)
    &&  x%y != 0)
        return x%y + y;    // 修正: -13%5 = -3 → 2
    else
        return x%y;
}
```

`(-13/5 == -2)` 这个编译时常量表达式在**向零截断**的平台上为真, 在**向负无穷截断**的平台上为假。据此选择不同的修正路径。

### MP 除法中的异号调整

如 4.3 节所述, MP_div 在操作数符号相异时必须手动调整商:

```c
if (sx != sy) {
    XP_neg(nbytes, z, z, 1);      // 取负
    if (!iszero(tmp[2]))           // 余数非零
        XP_diff(nbytes, z, z, 1);  // 减 1
}
```

这是因为无符号除法的余数 `|x| mod |y|` 在有符号场景下需要补调整。

---

## 18/19.9 Modern-C 改进方向

若用 Modern C (C11/C17/C23) 重写 MP, 可以考虑以下改进:

### 1. 移动语义与所有权

当前 MP 的同对象复用设计 (z 可以是 x 或 y) 是一种手动内存管理。Modern C 可以引入所有权标记:

```c
// 使用 const 和 restrict 帮助编译器优化
T MP_add(T *restrict z, const T *restrict x, const T *restrict y);
```

### 2. 更高效的乘法算法

对于 n 位乘法:
- **小学算法 (O(n^2))**: 当前实现, 适合 n ≤ 128 位
- **Karatsuba (O(n^1.585))**: 适合 n ≈ 200-2000 位
- **Toom-Cook 3-way (O(n^1.465))**: 适合 n ≈ 500-10000 位
- **FFT/NTT (O(n log n log log n))**: 适合 n > 10000 位

对于 MP 的典型应用场景 (32-128 位), 小学乘法足够。但如果扩展位宽到密码学规模 (2048+ 位), 应当引入组合策略:

```c
T MP_mul(T z, T x, T y) {
    if (nbits <= 128)        return mp_mul_schoolbook(z, x, y);
    else if (nbits <= 2000)  return mp_mul_karatsuba(z, x, y);
    else                     return mp_mul_toom3(z, x, y);
}
```

### 3. SIMD 加速

现代 CPU 的 SIMD 指令 (AVX2, AVX-512, NEON) 可以并行处理多个字节的加法和乘法。XP 的逐字节循环 (BASE=256) 可以改造为使用 64 位 "数字" (BASE=2^64) 并利用 `__int128` 或 `_addcarry_u64` 等内置函数。

### 4. 编译期特化

当前 `nbits` 是运行时常量, 导致大量 `if (nbits < 8)` 等判断。可以用 C11 `_Generic` 或 C++ 模板为常用位宽 (32, 64, 128, 256) 生成特化版本:

```c
#define MP_ADD(z, x, y) _Generic((z), \
    mp32_t*: mp32_add,                 \
    mp64_t*: mp64_add,                 \
    default:  mp_generic_add)(z, x, y)
```

### 5. const 限定

习题 19.5 探讨了使用 const 限定参数的几种类型定义方式:

```c
// 方案 A: typedef unsigned char T[];  (数组类型)
//   → const T 表示 const unsigned char[]
//   缺点: T 不能用作返回类型

// 方案 B: typedef unsigned char T;
//   → T *MP_add(T z[], const T x[], const T y[]);
//   或 T *MP_add(T *z, T *x, T *y);
```

### 6. 错误处理: errno 或 Result 类型

当前 MP 使用 `TRY-EXCEPT` 机制 (setjmp/longjmp) 处理溢出。Modern C 可考虑:
- `errno` 风格: 设置全局错误码
- Result 类型: 返回包含值和状态码的结构体
- `_Optional` (C23): 在溢出时返回空值

---

## 18/19.10 对比: MP vs Python int vs GMP mpz

| 特性 | CII MP | Python int | GMP mpz_t |
|------|--------|------------|-----------|
| **精度模型** | 定长 n 位 (全局设定) | 任意精度, 自动扩展 | 任意精度, 自动扩展 |
| **有符号表示** | 补码 | 符号-幅值 (30 位/数字) | 符号-幅值 (limb 数组) |
| **核心算法** | 小学算法 (O(n^2)) | Karatsuba + Toom-Cook 3-way | Toom-Cook 3/4/6/8-way + FFT |
| **内存管理** | 手动: ALLOC/FREE | GC (引用计数) | 手动: `mpz_init`/`mpz_clear` |
| **溢出行为** | 截断 + 异常 | 永不溢出 (自动扩展) | 永不溢出 (自动扩展) |
| **除法截断** | 向负无穷 (floor) | 向负无穷 (floor, `//`) | 向零截断 (C 风格) |
| **位运算** | 原生支持 | 原生支持 | 原生支持 |
| **字符串转换** | base 2-36 | base 2-36, 带符号 | base 2-62 |
| **代码量** | ~400 行 (mp.c) + ~300 行 (xp.c) | >2000 行 (longobject.c) | >100K 行 |
| **适用场景** | 教学、编译器、嵌入式 | 通用编程 | 高性能计算、密码学 |

### 关键区别

1. **除法语义**: MP 和 Python 均向负无穷截断 (`-13/5 = -3`), 而 GMP 向零截断 (`-13/5 = -2`)。这是有意的设计分歧: MP/Python 注重数学一致性, GMP 注重硬件效率。

2. **分配策略**: MP 鼓励复用已有变量 (z 可为 x 或 y), 避免了频繁分配。Python int 是不可变的, 每次运算创建新对象。GMP 使用类似 MP 的复用模式。

3. **溢出哲学**: MP 的定长溢出语义实际上**模拟了硬件行为**——这对于交叉编译器至关重要。Python int 和 GMP 则永不溢出。

4. **内部基数**: MP 使用基 256 (逐字节), Python 使用基 2^30 (30 位数字), GMP 使用基 2^64 (64 位 limb)。更大的基数意味着更少的数字操作次数, 但需要双倍字长来存储中间结果。

### mpcalc: MP 的计算器演示

`mpcalc` 是使用 MP 接口的波兰后缀计算器 (类似 `dc`), 展示了 MP 的完整用法模式:

```
% mpcalc
10 i 16 o   # 设置输入基=10, 输出基=16
255         # 推入 255
16 *        # 乘 16
p           # 打印: FF0
```

关键模式 (from mpcalc.c):
```c
volatile MP_T x = NULL, y = NULL, z = NULL;
TRY
    switch (c) {
    case '+': y = pop(); x = pop();
              z = MP_new(0); (*f->add)(z, x, y); break;
    ...
    }
EXCEPT(MP_Overflow)
    Fmt_fprint(stderr, "?overflow\n");
EXCEPT(MP_Dividebyzero)
    Fmt_fprint(stderr, "?divide by 0\n");
END_TRY;
if (z) Seq_addhi(sp, z);  // 结果入栈
FREE(x); FREE(y);          // 操作数释放
```

---

## 18/19.11 AP vs MP: 选型指南

```
                 ┌─────────────────────────────────────┐
                 │       需要任意精度 (无界增长)?        │
                 └──────────────┬──────────────────────┘
                        Yes     │     No
                ┌───────────────┴───────────────┐
                │                               │
        ┌───────┴───────┐               ┌───────┴───────┐
        │   使用 AP      │               │ 需要定长位宽?   │
        │ (符号-幅值,     │               │ (交叉编译器等)  │
        │  自动扩展)      │               └───────┬───────┘
        └───────────────┘                 Yes    │    No
                                ┌─────────┴──────┴─────────┐
                        ┌───────┴───────┐           ┌───────┴───────┐
                        │   使用 MP      │           │ 使用 C 原生    │
                        │ (补码, n 位定长,│           │ long / int    │
                        │  复用友好)      │           │               │
                        └───────────────┘           └───────────────┘
```

---

## 18/19.12 小结

MP 是一个精心设计的**低级多精度整数库**, 其核心设计哲学体现在:

1. **分层清晰**: MP (符号+溢出) → XP (无符号算术) → Arith (可移植除法)。每层职责单一。
2. **分配最小化**: 临时空间在 `MP_set` 中一次性分配, 算术函数零分配。
3. **补码的选择**: 使有符号和无符号加法共享底层实现 (XP_add), 简化了硬件级推理。
4. **先计算再抛异常**: 所有函数即使溢出也先设置结果 (保留低 nbits), 使 `TRY-EXCEPT` 优雅地处理截断。
5. **同对象复用**: `z` 可以与 `x` 或 `y` 是同一个 `MP_T`, 由 `tmp[]` 临时空间保证正确性。
6. **统一的 floor division**: 与 Python 和数学教材一致, 余数永为非负。

MP 的 27 行 `mp.c` 文件仅包含一个 `memmove` shim (用于不支持该函数的平台); 真正的 MP 实现在本书的 literate programming 源码中, 通过代码块 (chunks) 从各章节提取合成。完整的 MP 实现约 400 行 C 代码, 加上其依赖的 XP (约 275 行) 和 Arith (约 30 行), 总计不到 1000 行, 却提供了工业级的定长多精度算术能力。

---

## 参考文献

- Hanson, David R. *C Interfaces and Implementations*. Addison-Wesley, 1997. Chapters 17-19.
- Clinger, William D. "How to Read Floating Point Numbers Accurately." *PLDI*, 1990.
- Karatsuba, A. "Multiplication of Multidigit Numbers on Automata." *Soviet Physics Doklady*, 1962.
- Schneier, Bruce. *Applied Cryptography*. Wiley, 1996.
- Press, William H. et al. *Numerical Recipes in C*, 2nd ed. Cambridge, 1992. Section 20.6.
- Goldberg, David. "What Every Computer Scientist Should Know About Floating-Point Arithmetic." *ACM Computing Surveys*, 1991.
- Granlund, Torbjorn. *The GNU Multiple Precision Arithmetic Library (GMP)*. https://gmplib.org/
- Python Software Foundation. *CPython longobject.c*. https://github.com/python/cpython

---

*本章译自 *C Interfaces and Implementations* Chapters 18-19, 结合源代码 `/code/xp.h`, `/code/xp.c`, `/code/mp.h`, `/code/mp.c`, `/code/arith.h`, `/code/arith.c`, `/code/mpcalc.c` 综合分析。*
