# 第13章 位向量 (Bit Vectors)

> **原文**: David R. Hanson, *C Interfaces and Implementations*, Chapter 13  
> **译注**: 本章将原书英文内容翻译为中文，并结合源代码逐行注释，补充现代C语言改进方案

---

## 13.0 引言

第9章描述的`Set`可以保存任意元素——元素仅由客户端提供的函数操作。整数集合不如`Set`灵活，但使用频率高到值得单独设计一个ADT。**位向量(bit vector)** 可用于表示从0到N-1的整数集合。例如，256位向量足以高效地表示字符集。

位向量提供了一种**位级(bit-level)集合实现**：用一个比特(bit)表示一个整数是否属于集合。若第n位为1则表示整数n在集合中，为0则表示不在。

`Bit`接口提供`Set`的大部分集合操作函数，外加一些位向量特有的函数。与`Set`不同，由位向量表示的集合具有**明确定义的全集(universe)**，即[0, N-1]范围内的整数。因此`Bit`可以实现`Set`所不能的操作，例如求补集(complement)。

---

## 13.1 接口 (Interface)

位向量这个名字本身就揭示了整数集合的表示方式——本质上是一个比特序列。然而，`Bit`接口仅导出一个不透明类型来表示位向量：

```c
// bit.h -- 位向量接口
#ifndef BIT_INCLUDED
#define BIT_INCLUDED
#define T Bit_T
typedef struct T *T;    // Bit_T 是不透明指针类型

// ... 导出函数 ...

#undef T
#endif
```

### 13.1.1 创建与查询

位向量的长度在创建时通过`Bit_new`固定：

| 函数 | 签名 | 功能 |
|------|------|------|
| `Bit_new` | `extern T Bit_new(int length);` | 创建长度为`length`的向量，所有位初始化为0。`length`为负是受检运行时错误。可能引发`Mem_failed`。 |
| `Bit_length` | `extern int Bit_length(T set);` | 返回位向量的位数。 |
| `Bit_count` | `extern int Bit_count(T set);` | 返回位向量中置1的位数（即位集合的基数/population count）。 |

传递给任何`Bit`接口函数的`T`为null属于受检运行时错误，唯一的例外是四个集合运算函数(`Bit_union`, `Bit_inter`, `Bit_minus`, `Bit_diff`)。

```c
extern void Bit_free(T *set);
```
释放`*set`并清空`*set`。`set`或`*set`为null是受检运行时错误。

### 13.1.2 单元素操作

```c
extern int Bit_get(T set, int n);  // 获取第n位，返回0或1
extern int Bit_put(T set, int n, int bit);  // 设置第n位为bit，返回旧值
```

`Bit_get`返回第n位的值，即测试整数n是否在集合中。`Bit_put`将第n位设置为`bit`(0或1)并返回该位的前一个值。n为负、n >= length、或bit非0非1均为受检运行时错误。

### 13.1.3 范围操作

以下函数操作**连续位段**——即集合的子集：

```c
extern void Bit_clear(T set, int lo, int hi);  // 清除 bits [lo, hi]
extern void Bit_set  (T set, int lo, int hi);  // 置位 bits [lo, hi]
extern void Bit_not  (T set, int lo, int hi);  // 翻转 bits [lo, hi]
```

- `Bit_clear`: 将位lo到hi（含两端）清零
- `Bit_set`: 将位lo到hi置1
- `Bit_not`: 将位lo到hi取反

`lo > hi`、或`lo`/`hi`为负、或超出长度范围均为受检运行时错误。

### 13.1.4 集合比较

```c
extern int Bit_lt (T s, T t);   // s ⊂ t (s是t的真子集)
extern int Bit_eq (T s, T t);   // s = t (s等于t)
extern int Bit_leq(T s, T t);   // s ⊆ t (s是t的子集)
```

三个函数均要求s和t长度相同，否则为受检运行时错误。

### 13.1.5 遍历

```c
extern void Bit_map(T set,
    void apply(int n, int bit, void *cl), void *cl);
```

对集合中每一位调用`apply`，按位号从0开始递增。n是位号(0到length-1)，bit是第n位的值，cl是客户端提供的指针。

**关键语义**: `apply`**可以修改集合**。若对位n的`apply`调用修改了位k（k > n），后续调用将看到新值。这要求`Bit_map`必须**就地处理**各位，而不能先复制向量再遍历。

### 13.1.6 四大集合运算

```c
extern T Bit_union(T s, T t);  // s + t  并集 (按位OR)
extern T Bit_inter(T s, T t);  // s * t  交集 (按位AND)
extern T Bit_minus(T s, T t);  // s - t  差集 (s AND NOT t)
extern T Bit_diff (T s, T t);  // s / t  对称差 (按位XOR)
```

四个函数都返回一个**新创建的集合**。

**特殊规则**: 这些函数接受s或t为null（但不能同时为null），将null解释为空集。例如`Bit_union(s, NULL)`返回s的副本。s和t同时为null、或长度不同于为受检运行时错误。

---

## 13.2 实现 (Implementation)

### 13.2.1 数据结构: 双视图设计

```c
// bit.c -- 核心数据结构
#define T Bit_T
struct T {
    int length;             // 位向量的长度（位数）
    unsigned char *bytes;   // 按字节访问的视图
    unsigned long *words;   // 按机器字访问的视图（并行运算）
};
```

这是CII位向量设计的核心：**单一内存块的双重视图**。

- `length`: 位向量的总位数
- `bytes`: 指向至少`length/8`个字节。位在字节内的编号规则：**第0位是最低有效位(LSB)，第7位是最高有效位(MSB)**。`bytes[i]`持有位`8*i`到`8*i+7`。
- `words`: 指向同一块内存，用于**逐字并行运算**（并/交/差/对称差）

使用`unsigned char`数组而非`unsigned long`数组的主要动机是：允许用**查表法**实现`Bit_count`、`Bit_set`、`Bit_clear`和`Bit_not`的范围操作。

### 13.2.2 核心宏

```c
// 每个 unsigned long 包含的位数
#define BPW (8*sizeof (unsigned long))

// 容纳 len 位需要多少个 unsigned long（向上取整到 BPW 边界）
#define nwords(len) ((((len) + BPW - 1)&(~(BPW-1)))/BPW)

// 容纳 len 位需要多少个字节（向上取整到 8 边界）
#define nbytes(len) ((((len) + 8 - 1)&(~(8-1)))/8)
```

`nwords`和`nbytes`使用经典的**位运算向上取整**：
- `(len + BPW - 1)` 将len加上(BPW-1)实现向上取整
- `&(~(BPW-1))` 清除低位使结果对齐到BPW的倍数
- 最后除以BPW得到字数

例如`nwords(65)`在64位机器上(BPW=64)：`(65+63)&(~63) = 128&~63 = 128/64 = 2`，即65位需要2个`unsigned long`。

### 13.2.3 内存分配: Bit_new

```c
T Bit_new(int length) {
    T set;
    assert(length >= 0);           // §13.2: 长度非负检查
    NEW(set);                      // 分配 struct T 本身
    if (length > 0)
        set->words = CALLOC(nwords(length),
            sizeof (unsigned long));   // 分配位数组（初始化为0）
    else
        set->words = NULL;
    set->bytes = (unsigned char *)set->words;  // bytes指向同一块内存
    set->length = length;          // 记录长度
    return set;
}
```

**关键实现细节**: `CALLOC`确保所有位初始化为0。`bytes`被设置为`words`指针的强制转换，两块指针共享同一内存区域。这种**联合视图(union view)** 技术使得：

- **逐字节**访问（用于`Bit_get`、`Bit_set`、`Bit_clear`、`Bit_not`、`Bit_count`等需要位粒度或字节粒度的操作）
- **逐字**访问（用于`Bit_union`、`Bit_inter`、`Bit_minus`、`Bit_diff`、`Bit_eq`、`Bit_leq`、`Bit_lt`等需要批量并行操作）

`Bit_new`可能额外分配最多`sizeof(unsigned long) - 1`个字节。这些多余字节必须保持为0，否则后续函数将产生错误结果。

### 13.2.4 释放与查询

```c
void Bit_free(T *set) {
    assert(set && *set);
    FREE((*set)->words);   // 释放位数组
    FREE(*set);             // 释放结构体本身，并置*set=NULL
}

int Bit_length(T set) {
    assert(set);
    return set->length;     // 直接返回长度字段
}
```

---

## 13.3 位运算核心技巧

### 13.3.1 位索引: n/8 与 n%8

位向量的位访问核心公式：

```
第n位  →  字节索引 = n / 8,  字节内偏移 = n % 8
```

在字节内部，**位0是最低有效位，位7是最高有效位**。位号从左到右递增：

```
字节内的位布局:
  [位7] [位6] [位5] [位4] [位3] [位2] [位1] [位0]
   MSB                                   LSB
```

### 13.3.2 Bit_get: 读位

```c
int Bit_get(T set, int n) {
    assert(set);
    assert(0 <= n && n < set->length);           // §边界检查
    return ((set->bytes[n/8]>>(n%8))&1);         // 右移后取最低位
}
```

操作分解：
1. `n/8`：定位目标字节
2. `n%8`：位在字节内的偏移
3. `bytes[n/8] >> (n%8)`：将目标位右移到最低位
4. `& 1`：屏蔽其余位，只保留目标位

### 13.3.3 Bit_put: 写位

```c
int Bit_put(T set, int n, int bit) {
    int prev;
    assert(set);
    assert(bit == 0 || bit == 1);                // §bit只能是0或1
    assert(0 <= n && n < set->length);
    prev = ((set->bytes[n/8]>>(n%8))&1);         // 保存旧值
    if (bit == 1)
        set->bytes[n/8] |=   1<<(n%8);           // 置位: OR 掩码
    else
        set->bytes[n/8] &= ~(1<<(n%8));          // 清零: AND NOT 掩码
    return prev;                                  // 返回旧值
}
```

- **置1**: 使用`OR`运算 — `bytes[k] |= (1 << offset)`，不影响其他位
- **清零**: 使用`AND NOT`运算 — `bytes[k] &= ~(1 << offset)`，只清除目标位

这是教科书级别的原子位操作模式。

### 13.3.4 掩码查找表: msbmask 和 lsbmask

范围操作(`Bit_set`, `Bit_clear`, `Bit_not`)的核心是两个静态查找表：

```c
// MSB掩码: 保留从某位开始的最高有效位
unsigned char msbmask[] = {
    0xFF, 0xFE, 0xFC, 0xF8,
    0xF0, 0xE0, 0xC0, 0x80
};

// LSB掩码: 保留从某位开始的最低有效位
unsigned char lsbmask[] = {
    0x01, 0x03, 0x07, 0x0F,
    0x1F, 0x3F, 0x7F, 0xFF
};
```

**msbmask的作用**: `msbmask[k]`覆盖从位k到位7的位。例如`msbmask[3] = 0xF8 = 11111000`，保留第3-7位。

**lsbmask的作用**: `lsbmask[k]`覆盖从位0到位k的位。例如`lsbmask[3] = 0x0F = 00001111`，保留第0-3位。

**两表合用的妙处**: 当lo和hi位于同一字节内时，`msbmask[lo%8] & lsbmask[hi%8]`自动产生覆盖[lo, hi]范围的掩码。例如lo=1, hi=5：`msbmask[1] & lsbmask[5] = 0xFE & 0x3F = 0x3E = 00111110`，恰好覆盖位1到5。

---

## 13.4 范围操作实现

### 13.4.1 Bit_set: 范围置位

```c
void Bit_set(T set, int lo, int hi) {
    assert(set);
    assert(0 <= lo && hi < set->length);          // §边界检查
    assert(lo <= hi);                              // §lo不能大于hi

    if (lo/8 < hi/8) {                             // 跨字节情况
        // 区域1: 设置 byte[lo/8] 的 MSB 部分
        set->bytes[lo/8] |= msbmask[lo%8];
        // 区域2: 设置中间整字节全为 0xFF
        {
            int i;
            for (i = lo/8+1; i < hi/8; i++)
                set->bytes[i] = 0xFF;
        }
        // 区域3: 设置 byte[hi/8] 的 LSB 部分
        set->bytes[hi/8] |= lsbmask[hi%8];
    } else {                                       // 同字节情况
        set->bytes[lo/8] |= (msbmask[lo%8] & lsbmask[hi%8]);
    }
}
```

**三段式处理图解**（以`Bit_set(set, 3, 54)`为例，60位集合）：

```
字节:    [0]      [1]      [2]      [3]      [4]      [5]      [6]    [7]
     +--------+--------+--------+--------+--------+--------+--------+---+
     |00000111|11111111|11111111|11111111|11111111|11111111|11111110|000|
     +--------+--------+--------+--------+--------+--------+--------+---+
      ^^^^^^^^                                     ^^^^^^^^
      区域1: msbmask[3]=0xF8                      区域3: lsbmask[6]=0x7F
      lo/8 = 0                                    hi/8 = 6
                    区域2: 整字节 0xFF (字节1-5)
```

### 13.4.2 Bit_clear: 范围清零

```c
void Bit_clear(T set, int lo, int hi) {
    assert(set);
    assert(0 <= lo && hi < set->length);
    assert(lo <= hi);

    if (lo/8 < hi/8) {
        int i;
        set->bytes[lo/8] &= ~msbmask[lo%8];        // AND NOT: 清零MSB部分
        for (i = lo/8+1; i < hi/8; i++)
            set->bytes[i] = 0;                     // 整字节清零
        set->bytes[hi/8] &= ~lsbmask[hi%8];        // AND NOT: 清零LSB部分
    } else {
        set->bytes[lo/8] &= ~(msbmask[lo%8] & lsbmask[hi%8]);
    }
}
```

与`Bit_set`对称：使用`AND NOT`代替`OR`，中间区域用`0`代替`0xFF`。

### 13.4.3 Bit_not: 范围取反

```c
void Bit_not(T set, int lo, int hi) {
    assert(set);
    assert(0 <= lo && hi < set->length);
    assert(lo <= hi);

    if (lo/8 < hi/8) {
        int i;
        set->bytes[lo/8] ^= msbmask[lo%8];         // XOR: 翻转MSB部分
        for (i = lo/8+1; i < hi/8; i++)
            set->bytes[i] ^= 0xFF;                 // 整字节翻转
        set->bytes[hi/8] ^= lsbmask[hi%8];         // XOR: 翻转LSB部分
    } else {
        set->bytes[lo/8] ^= (msbmask[lo%8] & lsbmask[hi%8]);
    }
}
```

取反使用`XOR`运算：`x ^ 1 = ~x`，`x ^ 0 = x`。因此用掩码进行XOR就能翻转掩码覆盖的位。

### 13.4.4 Bit_map: 逐位遍历

```c
void Bit_map(T set,
    void apply(int n, int bit, void *cl), void *cl) {
    int n;
    assert(set);
    for (n = 0; n < set->length; n++)
        apply(n, ((set->bytes[n/8]>>(n%8))&1), cl);
}
```

每一个位都调用`apply`。虽然从性能角度看，`n/8`每隔8次循环才变化一次，可以将当前字节缓存到局部变量中以减少重复的除法和内存访问，但**这违反接口规范**：接口明确规定，若`apply`修改了尚未遍历的位，后续调用必须看到新值。因此必须每次都重新从内存读取，不能缓存。

---

## 13.5 Bit_count: 种群计数 (Population Count)

### 13.5.1 查表法实现

`Bit_count`返回集合中1的个数（即population count或Hamming weight）。本书使用经典的**半字节查表法**：

```c
int Bit_count(T set) {
    int length = 0, n;
    static char count[] = {
        0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4
    };                                              // 16个半字节的popcount表
    assert(set);
    for (n = nbytes(set->length); --n >= 0; ) {
        unsigned char c = set->bytes[n];
        length += count[c&0xF] + count[c>>4];      // 低半字节 + 高半字节
    }
    return length;
}
```

**工作原理**:
1. 每个字节拆分为两个半字节（4位），每个半字节有16种可能值
2. 查表`count[16]`获取每个半字节中1的个数
3. 累加所有字节的两个半字节计数
4. 额外字节由于`Bit_new`初始化为0，不影响结果

**半字节查表法的优雅之处**：表只有16个条目，却能O(1)计算任意字节的popcount。这是时间与空间的绝佳平衡。在嵌入式或small-footprint场景下，16字节的查找表可以放入L1 cache甚至寄存器中。

---

## 13.6 比较运算的字级实现

三个比较函数都在**unsigned long级别**（`BPW`位一次）进行比较，获得显著的性能加速。核心在于`s & ~t`这个位运算恒等式。

### 13.6.1 Bit_eq: 相等性比较

```c
int Bit_eq(T s, T t) {
    int i;
    assert(s && t);
    assert(s->length == t->length);
    for (i = nwords(s->length); --i >= 0; )
        if (s->words[i] != t->words[i])            // 逐字比对，不等即返回0
            return 0;
    return 1;
}
```

一旦发现任何字不相等就立即返回0，这是**提前退出(early exit)**优化。

### 13.6.2 Bit_leq: 子集判断

```c
int Bit_leq(T s, T t) {
    int i;
    assert(s && t);
    assert(s->length == t->length);
    for (i = nwords(s->length); --i >= 0; )
        if ((s->words[i] & ~t->words[i]) != 0)     // s_i & ~t_i != 0 意味着 s 不是 t 的子集
            return 0;
    return 1;
}
```

**核心逻辑**: s ⊆ t 当且仅当 s ∩ ~t = ∅。在位级别，这意味着对于每一位，若s中为1则t中也必须为1。表达式`(s->words[i] & ~t->words[i]) != 0`检测是否存在"s中为1但t中为0"的位。这是一种优雅的**无需逐位测试**的子集判断方法。

### 13.6.3 Bit_lt: 真子集判断

```c
int Bit_lt(T s, T t) {
    int i, lt = 0;
    assert(s && t);
    assert(s->length == t->length);
    for (i = nwords(s->length); --i >= 0; )
        if ((s->words[i] & ~t->words[i]) != 0)     // s ⊄ t → 返回0
            return 0;
        else if (s->words[i] != t->words[i])        // s_i ≠ t_i → 标记存在差异
            lt |= 1;
    return lt;                                      // s ⊆ t 且 至少一个字不同 → s ⊂ t
}
```

`Bit_lt`实现s ⊂ t的判断：s ⊆ t 且 s ≠ t。该函数遍历所有字，先用`(s->words[i] & ~t->words[i]) != 0`检查是否违反s ⊆ t，再用`s->words[i] != t->words[i]`检查是否存在差异。完成后若`lt==1`则s是真子集。

---

## 13.7 集合运算的字级并行优化

### 13.7.1 setop宏: 统一四则运算

并/交/差/对称差四个函数共享相同的结构，仅在以下三处不同：
1. s和t相同集合时的返回值
2. s为null时的返回值
3. t为null时的返回值
4. 核心位运算操作符

CII用`setop`宏统一抽象这些差异：

```c
#define setop(sequal, snull, tnull, op)             \
    if (s == t) { assert(s); return sequal; }       \  // 情况1: 同一集合
    else if (s == NULL) { assert(t); return snull; } \  // 情况2: s为空
    else if (t == NULL) return tnull;                \  // 情况3: t为空
    else {                                           \
        int i; T set;                                \
        assert(s->length == t->length);              \
        set = Bit_new(s->length);                    \  // 创建结果集
        for (i = nwords(s->length); --i >= 0; )      \
            set->words[i] = s->words[i] op t->words[i]; \  // 逐字并行运算
        return set;                                  \
    }
```

**关键设计**: 循环遍历`nwords`个`unsigned long`，每个字上执行`op`运算。这意味着在64位机器上，一次循环迭代处理64位——与逐字节处理相比速度提升8倍。

### 13.7.2 Bit_union: 并集 (s + t)

```c
T Bit_union(T s, T t) {
    setop(copy(t), copy(t), copy(s), |)
}
```

`|` 操作符在宏展开后成为 `set->words[i] = s->words[i] | t->words[i]`。

| 情形 | 返回值 | 含义 |
|------|--------|------|
| s == t | `copy(t)` | 并集等于自身 |
| s == NULL | `copy(t)` | 空集并t等于t |
| t == NULL | `copy(s)` | s并空集等于s |
| 一般 | new set | 逐字按位OR |

### 13.7.3 Bit_inter: 交集 (s * t)

```c
T Bit_inter(T s, T t) {
    setop(copy(t),
        Bit_new(t->length), Bit_new(s->length), &)
}
```

| 情形 | 返回值 | 含义 |
|------|--------|------|
| s == t | `copy(t)` | 交集等于自身 |
| s == NULL | `Bit_new(t->length)` | 空集：空集交任何集为空 |
| t == NULL | `Bit_new(s->length)` | 空集：任何集交空集为空 |
| 一般 | new set | 逐字按位AND |

### 13.7.4 Bit_minus: 差集 (s - t)

```c
T Bit_minus(T s, T t) {
    setop(Bit_new(s->length),
        Bit_new(t->length), copy(s), & ~)
}
```

`s - t = s ∩ ~t`。宏使用`& ~`操作符，展开为`set->words[i] = s->words[i] & ~t->words[i]`。

| 情形 | 返回值 | 含义 |
|------|--------|------|
| s == t | `Bit_new(s->length)` | 空集：集合减去自身为空 |
| s == NULL | `Bit_new(t->length)` | 空集：空集减去任何集为空 |
| t == NULL | `copy(s)` | s减空集等于s |
| 一般 | new set | 逐字按位 s & ~t |

### 13.7.5 Bit_diff: 对称差 (s / t)

```c
T Bit_diff(T s, T t) {
    setop(Bit_new(s->length), copy(t), copy(s), ^)
}
```

对称差 `s / t = (s - t) + (t - s)`，在位级别等价于按位XOR。

| 情形 | 返回值 | 含义 |
|------|--------|------|
| s == t | `Bit_new(s->length)` | 空集：自身对称差为空 |
| s == NULL | `copy(t)` | 空集对称差t等于t |
| t == NULL | `copy(s)` | s对称差空集等于s |
| 一般 | new set | 逐字按位XOR |

### 13.7.6 辅助函数: copy

```c
static T copy(T t) {
    T set;
    assert(t);
    set = Bit_new(t->length);
    if (t->length > 0)
        memcpy(set->bytes, t->bytes, nbytes(t->length));
    return set;
}
```

`copy`创建一个位向量的深拷贝。使用`memcpy`逐字节复制，性能接近理论极限。

---

## 13.8 设计要点与权衡

### 13.8.1 为什么用 unsigned char 数组而不是 unsigned long 数组？

1. **查表法的可行性**: `Bit_set`、`Bit_clear`、`Bit_not`的范围操作依赖`msbmask[8]`和`lsbmask[8]`两个8条目查找表。如果使用32位或64位单元，查找表的大小将爆炸（2^32条目不可行）。

2. **位粒度操作的自然匹配**: `Bit_get`和`Bit_put`需要访问单个位，字节级的`n/8`和`n%8`索引比字级更直观。

3. **通过双指针实现最佳两全**: `bytes`和`words`指向同一块内存，字节操作和字操作可以共存。比较运算和集合运算走`words`快速路径，单元素/范围操作走`bytes`精确路径。

### 13.8.2 位号约定: 右起LSB

位向量采用**最低有效位(LSB)为位0**的约定。这意味着：
- 在一个字节内，位0在最右边，位7在最左边
- 这与底层硬件的位编号一致，可以利用CPU原生的移位指令

### 13.8.3 setop宏 vs 代码重复

`setop`宏是CII中**逻辑复用**的典范。四个集合运算函数在结构上高度相似，用宏抽象差异（四种不同的位运算符 + 三种不同的边界情况处理），在保持代码紧凑的同时不失可读性。

这种宏技巧的代价是：调试时无法单步进入宏展开的代码，但在此处利大于弊。

---

## 13.9 现代C改进: 硬件加速与SIMD

### 13.9.1 硬件popcount: `__builtin_popcountl`

原书实现的`Bit_count`使用半字节查表法，每次处理一个字节。现代CPU（自Intel Nehalem/AMD Barcelona起）提供了**硬件popcount指令**（`POPCNT`），可以在单个CPU周期内统计一个机器字的位数。

改进后的`Bit_count`:

```c
#include <stdint.h>

int Bit_count_hw(T set) {
    int length = 0;
    for (int n = nwords(set->length); --n >= 0; )
        length += __builtin_popcountl(set->words[n]);
    return length;
}
```

在64位机器上，这会带来约**8倍至20倍**的性能提升，因为：
- 每次迭代处理64位（vs 原实现处理8位）
- 每个字使用单条`POPCNT`指令（vs 查表+加法+移位）
- 编译器可能自动向量化循环

对于GCC/Clang, 使用`__builtin_popcountl`; 对于MSVC, 使用`__popcnt64`; 对于C2x标准, 可用`stdc_count_ones`。

### 13.9.2 SIMD/SSE位运算加速

集合运算(`Bit_union`, `Bit_inter`, `Bit_minus`, `Bit_diff`)可受益于SIMD指令：

```c
#include <emmintrin.h>  // SSE2

void Bit_union_sse2(T dst, T s, T t) {
    int n = nwords(s->length);
    for (int i = 0; i < n; i += 2) {
        __m128i vs = _mm_loadu_si128((__m128i*)&s->words[i]);
        __m128i vt = _mm_loadu_si128((__m128i*)&t->words[i]);
        __m128i vr = _mm_or_si128(vs, vt);
        _mm_storeu_si128((__m128i*)&dst->words[i], vr);
    }
}
```

**关键技术选择**:
- **SSE2** (`_mm_or_si128`): 每个指令处理128位（2个`unsigned long`），是最广泛支持的选择
- **AVX2** (`_mm256_or_si256`): 每指令处理256位（4个`unsigned long`），Haswell+
- **AVX-512** (`_mm512_or_si512`): 每指令处理512位（8个`unsigned long`），Skylake-X+

**适用场景**: SIMD加速在**大位向量**（>1024位）上效果显著。对于小位向量（<256位），纯字级操作已经足够，SIMD的额外开销（对齐检查、函数调用）反而得不偿失。

**对齐注意事项**: 原实现使用`CALLOC`分配，不保证SIMD所需的对齐。生产级实现应使用`aligned_alloc`或在遍历开始时使用未对齐加载(`_mm_loadu_si128`)，如上例所示。

### 13.9.3 条件编译: 可移植硬件加速

```c
int Bit_count_modern(T set) {
    int length = 0, n;
#if defined(__GNUC__) || defined(__clang__)
    for (n = nwords(set->length); --n >= 0; )
        length += __builtin_popcountl(set->words[n]);
#elif defined(_MSC_VER)
    #include <intrin.h>
    for (n = nwords(set->length); --n >= 0; )
        length += __popcnt64(set->words[n]);
#else
    // 回退：原书查表法
    static char count[] = {0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4};
    for (n = nbytes(set->length); --n >= 0; ) {
        unsigned char c = set->bytes[n];
        length += count[c&0xF] + count[c>>4];
    }
#endif
    return length;
}
```

### 13.9.4 使用`restrict`优化指针别名

在现代C中，`restrict`关键字告知编译器指针之间不存在别名关系：

```c
T Bit_union_restrict(T restrict s, T restrict t) {
    // 编译器可以更激进地向量化和重排内存访问
    // ...
}
```

对于集合运算这类密集内存访问的操作，`restrict`可以让编译器生成更高效的SIMD代码。

---

## 13.10 与 C++ std::bitset 的对比

| 维度 | CII Bit_T | C++ std::bitset\<N\> |
|------|-----------|---------------------|
| **长度** | 运行时动态确定 (`Bit_new(N)`) | 编译时模板参数固定 |
| **内存分配** | 堆分配 (`CALLOC`)，需要手动`Bit_free` | 栈分配（小位集）或作为类成员内联，RAII自动管理 |
| **运算符重载** | 函数调用: `Bit_union(s, t)` | 自然语法: `s \| t`, `s & t`, `s ^ t`, `~s` |
| **类型安全** | 运行时assert检查长度一致 | 编译期类型检查，不同长度是不同的类型 |
| **迭代** | `Bit_map` 回调（遍历所有位） | 无内置迭代器，需手动循环 |
| **计数** | `Bit_count` 查表法 O(N/8) | `count()` 通常编译器优化为硬件popcount |
| **子集判断** | `Bit_leq` 逐字比较 O(N/BPW) | `(s & t) == s` 表达式，具相同复杂度 |
| **null语义** | 集合运算接受null（解释为空集） | 无null概念，默认构造为空集 |
| **标准库集成** | 无 | 与`<algorithm>`, ranges等无缝配合 |
| **可移植性** | ANSI C，无依赖 | C++98起标准库一部分 |

**选择建议**:
- **`Bit_T`**: 适用于纯C项目、嵌入式系统、需要运行时确定位长度的场景、教学目的（清晰展示底层实现）
- **`std::bitset`**: 适用于C++项目、编译期已知长度的场景、需要运算符自然语法的场景
- 对于**超大位集**（>10^6位），两者都不理想，应考虑`boost::dynamic_bitset`或专门设计的压缩位集结构

---

## 13.11 习题与拓展思考

1. **稀疏位集的压缩**: 大多数位为0的"稀疏"位集可通过行程编码(RLE)或不存储长零串来节省空间。一种方案是将位集分块，仅分配非零块。

2. **空间多路复用集合(Spatially Multiplexed Sets)** (Gimpel, 1974): 每个位存储在**跨一个字距离**的位置，使得一个N字的数组可以存储32个N位集合。一个32位掩码仅设置第i位即可选中第i列。该表示法的优势在于某些操作可以**常数时间完成**（通过操作掩码）。

3. **硬件加速的范围操作**: 原书习题13.3建议在lo/8和hi/8跨越足够多的字节时，改用`unsigned long`字级操作取代逐字节循环。在带`memset`的平台上，中间区域的填充可以用`memset`一步完成。

4. **缓存popcount**: 维护一个位计数缓存，使得`Bit_count`可以O(1)返回。代价是每次修改位时更新缓存。在频繁查询、不频繁修改的场景下收益显著。

---

## 13.12 小结

CII的`Bit`模块示范了以下C程序设计技术：

1. **双视图设计**: 同一内存块通过`bytes`和`words`两个指针提供字节和字两种视图，按需选择访问粒度
2. **查表法优化**: `msbmask[8]`、`lsbmask[8]`和`count[16]`将复杂的位操作转化为O(1)的查表，时间换空间或空间换时间的经典权衡
3. **宏抽象**: `setop`宏将四个集合运算的结构相似性抽象为参数化代码，优雅地消除了重复
4. **位运算恒等式**: `s ⊆ t ⇔ s & ~t == 0`，`s - t = s & ~t`等集合论的位运算映射
5. **现代硬件适配**: 硬件popcount、SIMD指令集、`restrict`关键字等现代C特性可以大幅提升性能

位向量是所有C程序员工具箱中的基础数据抽象。对CII位向量实现的理解能帮助你洞悉标准库（`std::bitset`）、数据库位图索引、布隆过滤器、网络协议标志位等众多系统背后的共同原理。
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch12_环形缓冲区_Ring.html">第12章 环形缓冲区 Ring</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch14_格式化_Fmt.html">第14章 格式化 Fmt</a></div>
</div>
