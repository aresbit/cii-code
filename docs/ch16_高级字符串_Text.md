# 第16章 高级字符串 (Text)

> 译自 David R. Hanson, *C Interfaces and Implementations*, Chapter 16: High-Level Strings.
> 翻译风格遵循原著的文学化编程 (Literate Programming) 传统——代码与注释交替出现，英文段落式论述包裹着被精确标记的代码块。

## 16.0 引言：为什么需要 Text？

第15章的 **Str** 接口提供了一套基于位置的 C 字符串操作函数。Str 的核心语义是 **copy**：每个产生新字符串的操作（如 `Str_cat`、`Str_dup`、`Str_reverse`）都会在堆上分配新的 `char*`。这带来了两个显著的代价：

1. **求长度是 O(n)**：C 字符串以 `\0` 结尾，要获取长度必须遍历整个字符串。
2. **无谓的内存分配**：在不修改字符串的只读场景中，每次拼接或截取都复制一整块内存是浪费的。

**Text** 接口以另一种字符串表示形式彻底改变了这一局面：

- **长度是 O(1)**：长度信息与字符数据捆绑在一起。
- **分配仅在必要时发生**：利用文本拼接 (splicing) 机制，许多操作可以零拷贝完成。
- **字符串不可变 (immutable)**：Text_T 描述的字符串不能被原地修改。
- **支持嵌入的空字符**：不像 C 字符串，Text_T 可以安全地包含 `'\0'`。

> **核心贡献陈述**: Text 通过将「长度+指针」打包为值语义的描述符，并利用 arena 分配器的拼接优化，在 C 语言的约束下实现了一种接近函数式语言的不可变字符串抽象。

---

## 16.1 接口设计

### 16.1.1 Text_T 结构体：描述符即值

Text 的核心是两字段的描述符 (descriptor)，**按值传递**：

```c
// text.h 第7-10行 — 导出的类型定义
typedef struct T {
    int len;           // 字符串长度（字符数）
    const char *str;   // 指向首字符的指针（不以 '\0' 结尾）
} T;
```

关键设计决策：
- `len` 字段使得求长度从 O(n) 降为 O(1)。
- `const char *str` 声明了不可变性——客户端可以读字符，但不能修改。
- **按值传递**：函数接收和返回 `Text_T` 本身（非指针），因此 Text 不从堆中分配描述符。
- 字符串不以 `'\0'` 结尾，必须通过 `s.str[0..s.len-1]` 访问。

```c
// text.h 第12-17行 — 预定义的字符集常量
extern const T Text_cset;   // 全部 256 个 8-bit 字符
extern const T Text_ascii;  // 前 128 个 ASCII 字符
extern const T Text_ucase;  // "ABCDEFGHIJKLMNOPQRSTUVWXYZ" (26个字符)
extern const T Text_lcase;  // "abcdefghijklmnopqrstuvwxyz" (26个字符)
extern const T Text_digits; // "0123456789" (10个字符)
extern const T Text_null;   // 空字符串 (len=0)
```

### 16.1.2 位置约定

Text 沿用了 Str 的位置 (position) 约定——位置标识字符之间的间隙，从 1 开始的正位置从左计数，非正位置从右计数。以字符串 `"Interface"` 为例：

```
 1   2   3   4   5   6   7   8   9   10
 I   n   t   e   r   f   a   c   e
-9  -8  -7  -6  -5  -4  -3  -2  -1   0
```

- 位置 1 在第一个字符 `'I'` 之前，位置 10 在最后一个字符 `'e'` 之后。
- 位置 0 在最后一个字符之后（等价于 `s.len + 1`）。
- 正位置 i 对应的字符索引是 `i - 1`；非正位置 i 对应的字符索引是 `i + s.len`。

---

## 16.2 API 接口逐函数详解

### 16.2.1 构造与销毁

#### `Text_put` — 从 C 字符串构造 Text_T（复制语义）

```c
// text.h 第18行
extern T Text_put(const char *str);
```

```c
// text.c 第84-90行
T Text_put(const char *str) {
    T text;
    assert(str);                        // 检查: str 不可为 NULL
    text.len = strlen(str);             // §16.1: 长度按 C 约定计算
    text.str = memcpy(                   // 使用 memcpy 而非 strcpy —
        alloc(text.len), str, text.len); // 不追加 '\0'，因为 Text_T 不以 '\0' 结尾
    return text;
}
```

**注意**：`Text_put` 将字符串**复制**到 Text 的私有字符串空间（string space）中。字符串空间由 Text 的 arena 分配器管理。可引发 `Mem_Failed`。

#### `Text_box` — 装箱已有数据（零拷贝）

```c
// text.h 第20行
extern T Text_box(const char *str, int len);
```

```c
// text.c 第65-72行
T Text_box(const char *str, int len) {
    T text;
    assert(str);          // 检查: str 不可为 NULL
    assert(len >= 0);     // 检查: len 不可为负
    text.str = str;       // 直接使用传入的指针，不复制数据
    text.len = len;       // §16.1.1: 长度直接记录
    return text;
}
```

**典型用法**：将常量字符串或客户端自己管理的内存装箱为 Text_T。

```c
static char editmsg[] = "Last edited by: ";
Text_T msg = Text_box(editmsg, sizeof(editmsg) - 1);
// sizeof(editmsg)-1 省略了末尾的 '\0'，防止它被当作字符串内容
```

#### `Text_get` — 从 Text_T 导出 C 字符串

```c
// text.h 第19行
extern char *Text_get(char *str, int size, T s);
```

```c
// text.c 第91-100行
char *Text_get(char *str, int size, T s) {
    assert(s.len >= 0 && s.str);         // 检查: 描述符有效性
    if (str == NULL)
        str = ALLOC(s.len + 1);           // str 为 NULL 时自动分配 (Mem 分配器)
    else
        assert(size >= s.len + 1);        // str 不为 NULL 时，size 必须足够
    memcpy(str, s.str, s.len);            // 用 memcpy 而非 strncpy —
    str[s.len] = '\0';                    // 因为 s.str 可能包含 '\0'
    return str;                           // 手动追加 '\0'
}
```

**关键区分**：`Text_get` 使用 `Mem` 的通用分配器 (`ALLOC`)，而 `Text_put` 使用 `Text` 的私有 `alloc`。二者是不同的内存池。

#### `Text_sub` — 截取子串（零拷贝）

```c
// text.h 第21行
extern T Text_sub(T s, int i, int j);
```

```c
// text.c 第73-83行
T Text_sub(T s, int i, int j) {
    T text;
    assert(s.len >= 0 && s.str);
    i = idx(i, s.len);                   // 将位置转为索引 (§16.1.2)
    j = idx(j, s.len);
    if (i > j) { int t = i; i = j; j = t; }  // 确保 i <= j (位置可以任意顺序传入)
    assert(i >= 0 && j <= s.len);
    text.len = j - i;                    // 子串长度 = 索引差
    text.str = s.str + i;                // 零拷贝！仅偏移指针
    return text;
}
```

**核心观察**：`Text_sub` 不复制任何字符。它只是构造了一个新的描述符，指向原字符串的子区域。这是 Text 拼接 (splicing) 哲学的典型体现。

#### `Text_len` — 获取长度

Text 不提供独立的 `Text_len` 函数——直接读取 `s.len` 字段即可。接口将结构体暴露给客户端正是为此目的：

```c
// 直接字段访问 (替代 Text_len(s))
int n = s.len;   // O(1)，无需遍历
```

---

### 16.2.2 拼接操作 (Splicing Operations)

#### `Text_cat` — 字符串拼接（智能零拷贝）

```c
// text.h 第23行
extern T Text_cat(T s1, T s2);
```

```c
// text.c 第123-148行
T Text_cat(T s1, T s2) {
    assert(s1.len >= 0 && s1.str);
    assert(s2.len >= 0 && s2.str);
    // 优化 1: 空字符串短路
    if (s1.len == 0) return s2;          // s1 为空 → 直接返回 s2
    if (s2.len == 0) return s1;          // s2 为空 → 直接返回 s1

    // 优化 2: 拼接检测 — s1 和 s2 是否已在内存中相邻？
    if (s1.str + s1.len == s2.str) {     // s2 紧跟在 s1 之后
        s1.len += s2.len;                // 直接扩展 s1 的长度字段（修改本地副本）
        return s1;                       // 零拷贝！
    }

    // 优化 3: s1 位于字符串空间末尾？
    {
        T text;
        text.len = s1.len + s2.len;
        if (isatend(s1, s2.len)) {       // s1 在 arena 尾部，且有空余空间
            text.str = s1.str;
            memcpy(alloc(s2.len), s2.str, s2.len);  // 只复制 s2
        } else {
            char *p;
            text.str = p = alloc(s1.len + s2.len);   // 全量复制
            memcpy(p,          s1.str, s1.len);
            memcpy(p + s1.len, s2.str, s2.len);
        }
        return text;
    }
}
```

**拼接 (splicing) 的三层优化层次**：

| 层级 | 条件 | 行为 | 内存分配 |
|------|------|------|----------|
| L1: 空字符串 | s1 或 s2 为空 | 返回另一方 | 无 |
| L2: 物理相邻 | `s1.str + s1.len == s2.str` | 扩展 s1 的 len | 无 |
| L3: Arena 尾部 | s1 在字符串空间末尾 | 复制 s2, 利用 s1 | 仅 s2 |
| L4: 一般情况 | 以上都不满足 | 完整复制两者 | s1+s2 |

> **这就是 Text 与 Str 最根本的区别**：Str 总是执行 L4（全量复制），而 Text 在 L1-L3 场景下实现了零拷贝或部分拷贝。这就是 "splice vs copy 语义" 的核心。

#### `Text_dup` — 重复拼接

```c
// text.h 第24行
extern T Text_dup(T s, int n);
```

```c
// text.c 第101-122行
T Text_dup(T s, int n) {
    assert(s.len >= 0 && s.str);
    assert(n >= 0);
    if (n == 0 || s.len == 0)            // 优化 1: n=0 或空串
        return Text_null;
    if (n == 1)                           // 优化 2: n=1 直接返回自身
        return s;
    {
        T text;
        char *p;
        text.len = n * s.len;
        if (isatend(s, text.len - s.len)) {  // 优化 3: s 在 arena 尾部
            text.str = s.str;
            p = alloc(text.len - s.len);     // s 当作第 1 份，只需复制 n-1 份
            n--;
        } else
            text.str = p = alloc(text.len);   // 全量分配
        for ( ; n-- > 0; p += s.len)
            memcpy(p, s.str, s.len);           // 循环复制
        return text;
    }
}
```

#### `Text_reverse` — 反转字符串

```c
// text.h 第25行
extern T Text_reverse(T s);
```

```c
// text.c 第149-165行
T Text_reverse(T s) {
    assert(s.len >= 0 && s.str);
    if (s.len == 0)                      // 优化 1: 空串
        return Text_null;
    else if (s.len == 1)                 // 优化 2: 单字符
        return s;
    else {
        T text;
        char *p;
        int i = s.len;
        text.len = s.len;
        text.str = p = alloc(s.len);     // 分配新空间
        while (--i >= 0)
            *p++ = s.str[i];             // 逆序复制
        return text;
    }
}
```

#### `Text_map` — 字符映射

```c
// text.h 第26行
extern T Text_map(T s, const T *from, const T *to);
```

```c
// text.c 第166-194行
T Text_map(T s, const T *from, const T *to) {
    static char map[256];                // 静态查找表（非线程安全！）
    static int inited = 0;               // 初始化标志
    assert(s.len >= 0 && s.str);
    if (from && to) {                    // 首次调用: 构建映射表
        int k;
        for (k = 0; k < (int)sizeof map; k++)
            map[k] = k;                  // 默认: 恒等映射
        assert(from->len == to->len);    // from 和 to 必须等长
        for (k = 0; k < from->len; k++)
            map[(unsigned char)from->str[k]] = to->str[k];
        inited = 1;
    } else {
        assert(from == NULL && to == NULL);
        assert(inited);                  // 后续调用: from 和 to 均为 NULL 时复用上次映射
    }
    if (s.len == 0) return Text_null;
    else {
        T text;
        int i;
        char *p;
        text.len = s.len;
        text.str = p = alloc(s.len);
        for (i = 0; i < s.len; i++)
            *p++ = map[(unsigned char)s.str[i]];
        return text;
    }
}
```

**典型用法**：
- 大小写转换：`Text_map(s, &Text_ucase, &Text_lcase)` 将大写字母折叠为小写。
- 后续调用 `Text_map(s, NULL, NULL)` 复用相同映射。

**注意**：`Text_map` 使用静态查找表，因此**不可重入 (non-reentrant)**，不适合多线程环境。

#### `Text_cmp` — 字符串比较

```c
// text.h 第27行
extern int Text_cmp(T s1, T s2);
```

```c
// text.c 第195-208行
int Text_cmp(T s1, T s2) {
    assert(s1.len >= 0 && s1.str);
    assert(s2.len >= 0 && s2.str);
    if (s1.str == s2.str)               // 优化: 同一块内存 → 短者为小
        return s1.len - s2.len;
    else if (s1.len < s2.len) {          // s1 更短时先比较前 s1.len 字节
        int cond = memcmp(s1.str, s2.str, s1.len);
        return cond == 0 ? -1 : cond;    // 前缀相等 → s1 较小
    } else if (s1.len > s2.len) {        // s2 更短时先比较前 s2.len 字节
        int cond = memcmp(s1.str, s2.str, s2.len);
        return cond == 0 ? +1 : cond;    // 前缀相等 → s1 较大
    } else
        return memcmp(s1.str, s2.str, s1.len);  // 等长时直接比较
}
```

**使用 `memcmp` 而非 `strcmp`** 的原因：Text_T 可能包含嵌入的空字符。

---

### 16.2.3 位置转换与子串分析

#### `Text_pos` — 任意位置转正位置

```c
// text.h 第22行
extern int Text_pos(T s, int i);
```

```c
// text.c 第59-64行
int Text_pos(T s, int i) {
    assert(s.len >= 0 && s.str);
    i = idx(i, s.len);          // 正/负位置 → 索引
    assert(i >= 0 && i <= s.len);
    return i + 1;               // 索引 → 正位置
}
```

**内部宏 `idx`**：
```c
// text.c 第9行 — 位置转索引
#define idx(i, len) ((i) <= 0 ? (i) + (len) : (i) - 1)
// 正位置 i  → i - 1      (位置 1 对应索引 0)
// 非正位置 i → i + len    (位置 -1 对应索引 len-1)
```

#### `Text_chr` / `Text_rchr` — 单字符查找

```c
// text.h 第28-29行
extern int Text_chr (T s, int i, int j, int c);
extern int Text_rchr(T s, int i, int j, int c);
```

```c
// Text_chr: 在 s[i:j] 中从左查找字符 c (text.c 第229-239行)
int Text_chr(T s, int i, int j, int c) {
    // <转换 i 和 j 为 0..s.len 间的索引>
    i = idx(i, s.len); j = idx(j, s.len);
    if (i > j) { int t = i; i = j; j = t; }  // 确保 i <= j
    assert(i >= 0 && j <= s.len);
    for ( ; i < j; i++)
        if (s.str[i] == c)
            return i + 1;  // 索引转正位置: 返回字符左侧的位置
    return 0;               // 未找到
}

// Text_rchr: 在 s[i:j] 中从右查找字符 c (text.c 第240-250行)
int Text_rchr(T s, int i, int j, int c) {
    // <同上的位置转换>
    while (j > i)
        if (s.str[--j] == c)
            return j + 1;
    return 0;
}
```

**返回值语义**：返回的是该字符**左侧**的正位置。返回 0 表示未找到。

#### `Text_upto` / `Text_rupto` — 字符集查找

```c
// text.h 第30-31行
extern int Text_upto (T s, int i, int j, T set);
extern int Text_rupto(T s, int i, int j, T set);
```

```c
// Text_upto: 在 s[i:j] 中查找字符集 set 中任一字符的最左出现 (text.c 第251-262行)
int Text_upto(T s, int i, int j, T set) {
    assert(set.len >= 0 && set.str);
    // <位置转换>
    for ( ; i < j; i++)
        if (memchr(set.str, s.str[i], set.len))  // 用 memchr 而非 strchr —
            return i + 1;                          // 因为 set 和 s 都可能包含 '\0'
    return 0;
}

// Text_rupto: 在 s[i:j] 中查找字符集 set 中任一字符的最右出现 (text.c 第263-274行)
int Text_rupto(T s, int i, int j, T set) {
    assert(set.len >= 0 && set.str);
    // <位置转换>
    while (j > i)
        if (memchr(set.str, s.str[--j], set.len))
            return j + 1;
    return 0;
}
```

#### `Text_find` / `Text_rfind` — 子串查找

```c
// text.h 第35-36行
extern int Text_find (T s, int i, int j, T str);
extern int Text_rfind(T s, int i, int j, T str);
```

```c
// Text_find: 在 s[i:j] 中查找子串 str (text.c 第275-293行)
int Text_find(T s, int i, int j, T str) {
    assert(str.len >= 0 && str.str);
    // <位置转换>
    if (str.len == 0) return i + 1;            // 空串: 总是在位置 i 找到
    else if (str.len == 1) {                    // 单字符: 退化为 Text_chr
        for ( ; i < j; i++)
            if (s.str[i] == *str.str)
                return i + 1;
    } else                                      // 一般情况: 滑动窗口匹配
        for ( ; i + str.len <= j; i++)          // 保证不越界
            if (equal(s, i, str))               // 使用 memcmp 宏
                return i + 1;
    return 0;
}

// equal 宏 (text.c 第12-13行)
#define equal(s, i, t) \
    (memcmp(&(s).str[i], (t).str, (t).len) == 0)
```

#### `Text_match` / `Text_rmatch` — 前缀/后缀匹配

```c
// text.h 第37-38行
extern int Text_match (T s, int i, int j, T str);
extern int Text_rmatch(T s, int i, int j, T str);
```

```c
// Text_match: 检查 s[i:j] 是否以 str 开头 (text.c 第354-369行)
int Text_match(T s, int i, int j, T str) {
    assert(str.len >= 0 && str.str);
    // <位置转换>
    if (str.len == 0) return i + 1;
    else if (str.len == 1) {
        if (i < j && s.str[i] == *str.str)
            return i + 2;                       // 返回 str 之后的位置
    } else if (i + str.len <= j && equal(s, i, str))
        return i + str.len + 1;
    return 0;
}

// Text_rmatch: 检查 s[i:j] 是否以 str 结尾 (text.c 第370-386行)
int Text_rmatch(T s, int i, int j, T str) {
    assert(str.len >= 0 && str.str);
    // <位置转换>
    if (str.len == 0) return j + 1;
    else if (str.len == 1) {
        if (j > i && s.str[j-1] == *str.str)
            return j;                           // 返回 str 之前的位置
    } else if (j - str.len >= i && equal(s, j - str.len, str))
        return j - str.len + 1;
    return 0;
}
```

#### `Text_any` / `Text_many` / `Text_rmany` — 步进操作

```c
// text.h 第32-34行
extern int Text_any  (T s, int i, T set);        // 跳过 1 个匹配字符
extern int Text_many (T s, int i, int j, T set); // 向右跳过连续匹配
extern int Text_rmany(T s, int i, int j, T set); // 向左跳过连续匹配
```

```c
// Text_any (text.c 第313-321行): 检查 s[i] 是否在 set 中
int Text_any(T s, int i, T set) {
    i = idx(i, s.len);
    assert(i >= 0 && i <= s.len);
    if (i < s.len && memchr(set.str, s.str[i], set.len))
        return i + 2;          // i+1 = 该字符的位置, i+2 = 跳过该字符后的位置
    return 0;
}

// Text_many (text.c 第322-337行): 向右连续跳过 set 中的字符
int Text_many(T s, int i, int j, T set) {
    // <位置转换>
    if (i < j && memchr(set.str, s.str[i], set.len)) {
        do i++;
        while (i < j && memchr(set.str, s.str[i], set.len));
        return i + 1;          // 返回第一个不在 set 中的字符左侧的位置
    }
    return 0;
}

// Text_rmany (text.c 第338-353行): 向左连续跳过 set 中的字符
int Text_rmany(T s, int i, int j, T set) {
    // <位置转换>
    if (j > i && memchr(set.str, s.str[j-1], set.len)) {
        do --j;
        while (j >= i && memchr(set.str, s.str[j], set.len));
        return j + 2;          // 返回第一个不在 set 中的字符右侧的位置
    }
    return 0;
}
```

#### `Text_fmt` — 格式化输出

```c
// text.h 第39-41行
extern void Text_fmt(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision);
```

```c
// text.c 第387-396行
void Text_fmt(int code, va_list_box *box,
    int put(int c, void *cl), void *cl,
    unsigned char flags[], int width, int precision) {
    T *s;
    assert(box && flags);
    s = va_arg(box->ap, T*);              // 注意: 接受 Text_T* 而非 Text_T
    assert(s && s->len >= 0 && s->str);
    Fmt_puts(s->str, s->len, put, cl, flags,  // 委托给 Fmt_puts 处理
        width, precision);
}
```

**为什么传指针而非传值？** 在可变参数列表中，传递小结构体（两个 word）可能与 `double` 混淆，不具有可移植性。因此 `Text_fmt` 消耗的是 `Text_T*`。

---

### 16.2.4 内存管理

#### Text 的 Arena 分配器

Text 维护自己的字符串空间（string space），结构为链表式的 arena：

```c
// text.c 第42-46行
static struct chunk {
    struct chunk *link;   // 指向下一个 chunk
    char *avail;          // 当前 chunk 中第一个空闲字节
    char *limit;          // 当前 chunk 的末尾（one past the end）
} head = { NULL, NULL, NULL }, *current = &head;
```

```c
// text.c 第47-58行 — 私有分配函数
static char *alloc(int len) {
    assert(len >= 0);
    if (current->avail + len > current->limit) {   // 当前 chunk 空间不足
        current = current->link =                   // 追加新 chunk
            ALLOC(sizeof (*current) + 10*1024 + len); // 至少 10KB + 请求大小
        current->avail = (char *)(current + 1);     // 数据区域紧跟在 chunk 头之后
        current->limit = current->avail + 10*1024 + len;
        current->link = NULL;
    }
    current->avail += len;       // 指针前移
    return current->avail - len; // 返回分配空间的起始地址
}
```

**关键性质**：
- 字符紧邻存储，无对齐填充，无块头开销。
- `isatend` 宏利用了这一性质来检测字符串是否位于 arena 尾部：

```c
// text.c 第10-11行
#define isatend(s, n) ((s).str+(s).len == current->avail\
    && current->avail + (n) <= current->limit)
```

当 `s.str + s.len` 等于 `current->avail`（即下一个空闲字节的地址）时，`s` 就在尾部。这是 `Text_cat` 和 `Text_dup` 实现拼接优化的关键前提。

#### `Text_save` / `Text_restore` — 栈式回滚

```c
// text.h 第42-43行
extern Text_save_T Text_save(void);
extern void Text_restore(Text_save_T *save);
```

```c
// text.c 第14-17行 — 保存点结构
struct Text_save_T {
    struct chunk *current;  // 当时活动的 chunk
    char *avail;            // 当时空闲字节的位置
};

// text.c 第209-216行 — Text_save
Text_save_T Text_save(void) {
    Text_save_T save;
    NEW(save);
    save->current = current;
    save->avail = current->avail;
    alloc(1);                  // 创建"空洞" — 防止拼接跨越保存点
    return save;
}

// text.c 第217-228行 — Text_restore
void Text_restore(Text_save_T *save) {
    struct chunk *p, *q;
    assert(save && *save);
    current = (*save)->current;        // 恢复活动 chunk 指针
    current->avail = (*save)->avail;   // 恢复空闲位置
    FREE(*save);                       // 释放保存点结构
    for (p = current->link; p; p = q) { // 释放之后所有 chunk
        q = p->link;
        FREE(p);
    }
    current->link = NULL;
}
```

**关键设计细节**：`Text_save` 调用 `alloc(1)` 在字符串空间中插入一个 1 字节的"空洞"。这确保了 `isatend` 对于保存在空洞之前的任何字符串都会失败，从而保证不会有字符串跨越保存点边界——这保护了 `Text_restore` 的安全语义：回滚后，所有在保存点之后分配的 Text_T 描述符都将失效（变为悬挂指针）。

> **警告**：`Text_restore` 之后继续使用在回滚时间点之后创建的 Text_T 是一个**未检查的运行时错误**（unchecked runtime error）。这是一个危险但高效的折中。

---

## 16.3 Text 与 Str 的核心对比

| 维度 | Str (第15章) | Text (第16章) |
|------|-------------|---------------|
| **表示** | `char*` (以 `'\0'` 结尾) | `Text_T { int len; const char *str; }` |
| **求长度** | O(n): 遍历到 `'\0'` | O(1): 读取 `len` 字段 |
| **嵌入 `'\0'`** | 不支持 (被当作终止符) | 完全支持 |
| **不可变性** | 无保证 (可原地修改) | 强制不可变 (`const char *`) |
| **拼接语义** | Copy: 每次操作都分配新内存 | **Splice**: 尽可能零拷贝 |
| **位置系统** | 字符串参数 + `i, j` 位置 | 描述符 + `i, j` 位置 |
| **内存模型** | 调用方管理分配 | Text 管理私有 arena |
| **线程安全** | 大部分函数可重入 | `Text_map` 不可重入 (静态查找表) |
| **可变参数** | `Str_catv` 支持变长拼接 | 无 (需多次调用 `Text_cat`) |

### Splice vs Copy 的哲学差异

```
Str 哲学 (Copy):
  "每次操作创建一个独立的副本。你可以随意修改结果，不用担心影响其他引用。"

Text 哲学 (Splice):
  "让尽可能多的操作共享底层存储。描述符是轻量的视图，字符串是共享的数据。"
```

Str 的模型更简单、更安全（每个结果独立），适合字符串会被频繁修改的场景。Text 的模型更接近函数式编程中的"不可变持久数据结构"——子串和拼接操作通常是 O(1) 的指针运算——适合编译器的符号表、文本分析器、语法高亮器等只读密集型的应用。

---

## 16.4 文本段落级操作：换行符处理与对齐

虽然 Text 接口没有直接提供"段落级"操作函数（如自动换行、对齐），但是利用位置系统和字符集常量，可以简洁地实现这些操作。

### 换行符分割

```c
// 将文本按换行符分割
Text_T next_line(Text_T text, int *pos) {
    int start = *pos;
    if (start > text.len) return Text_null;      // 已到末尾
    int end = Text_chr(text, start, 0, '\n');    // 查找换行符
    if (end == 0) {                               // 最后一行(无换行符)
        *pos = text.len + 1;
        return Text_sub(text, start, 0);
    }
    *pos = end + 1;                               // 跳过换行符
    return Text_sub(text, start, end);            // 返回不含 '\n' 的行
}
```

### 空白符裁剪

```c
// 利用预定义的字符集
Text_T whitespace = Text_box(" \t\n\r", 4);

// left-trim: 跳过前导空白
int start = Text_upto(text, 1, 0, whitespace);
start = (start == 0) ? text.len + 1 : start;

// right-trim: 跳过尾部空白
int end = Text_rupto(text, 1, 0, whitespace);
end = (end == 0) ? 1 : end;

// 裁剪后的文本
Text_T trimmed = Text_sub(text, start, end);
```

### 文本对齐

```c
// 左对齐：在右侧填充空格至指定宽度
Text_T pad_right(Text_T line, int width) {
    if (line.len >= width) return line;
    int pad_len = width - line.len;
    char *spaces = ALLOC(pad_len);
    memset(spaces, ' ', pad_len);
    Text_T padding = Text_box(spaces, pad_len);
    return Text_cat(line, padding);
    // 注意: spaces 的内存由客户端管理。在 Text_restore 之前必须保持有效。
}

// 居中对齐
Text_T pad_center(Text_T line, int width) {
    if (line.len >= width) return line;
    int left_pad = (width - line.len) / 2;
    int right_pad = width - line.len - left_pad;
    // ... 生成左右 padding
    return Text_cat(Text_cat(left_pad_text, line), right_pad_text);
}
```

### 单词分词器

结合 `Text_upto` + `Text_many` 的经典模式（与 Str 章节中的 `ids.c` 同理）：

```c
// 从文本中提取所有单词 (字母、数字、下划线)
void tokenize(Text_T text) {
    // 构建标识符字符集
    char id_chars[] = "abcdefghijklmnopqrstuvwxyz"
                      "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
                      "0123456789_";
    Text_T id_set = Text_box(id_chars, sizeof(id_chars) - 1);

    int i = 1, j;
    while ((i = Text_upto(text, i, 0, id_set)) > 0) {
        j = Text_many(text, i, 0, id_set);
        Text_T token = Text_sub(text, i, j);   // 零拷贝提取
        // 处理 token ...
        i = j;
    }
}
```

---

## 16.5 现代 C 改进方向

Text 接口发表于 1997 年，其设计仍具有很强的前瞻性，但在当代 C 编程（C11/C17/C23）和 Unicode 普及的背景下，有若干值得讨论的改进方向。

### 16.5.1 Unicode 与 UTF-8 支持

Text 将字符串视为 8-bit 字节序列是有意为之——它支持任意字节值，包括 `'\0'`。这构成了在最底层支持 UTF-8 多字节编码的基础：

```c
// UTF-8 字符解码辅助函数 (Modern-C 风格)
typedef struct {
    Text_T text;
    int pos;         // 当前字节位置
} Text_Iter;

// 获取下一个 Unicode 码点 (code point)
int32_t text_next_codepoint(Text_Iter *iter) {
    if (iter->pos > iter->text.len) return -1;
    unsigned char c = (unsigned char)iter->text.str[iter->pos - 1];
    // UTF-8 解码逻辑 (参见 RFC 3629)
    // ...
    return codepoint;
}
```

**改进思路**：在 Text 上层构建一个 `Text_UTF8` 包装层，提供码点级 (codepoint-level) 的位置系统——按逻辑字符而非字节计数。原始 Text 的 `Text_chr` 等函数仍可用于字节级操作。

### 16.5.2 宽字符支持

`wchar_t` 和 `char16_t`/`char32_t` 是标准 C 对宽字符和 Unicode 的支持。Text 的模式可以参数化为泛型：

```c
// 概念性: Text 的宽字符变体 (现代 C11+)
typedef struct {
    int     len;          // 宽字符数量
    const wchar_t *str;   // 指向宽字符序列
} Text_Wide;
```

然而，原始的 Text 设计刻意不采用 `void*` 泛型化——它保持具体的 `char*` 以利用 `memcmp`/`memcpy`/`memchr` 等 C 标准库函数。宽字符版本需要重写这些底层操作。

### 16.5.3 线程安全

`Text_map` 的静态 `map[256]` 数组使其在多线程环境下不安全。改进方案：

```c
// 方案 A: 每次调用在栈上分配 map (简单但增加开销)
T Text_map_threadsafe(T s, const T *from, const T *to) {
    unsigned char map[256];
    // ... 每次构建 map, 无静态变量
}

// 方案 B: 接受 map 参数，由调用方管理生命周期
T Text_map_ex(T s, const T *from, const T *to, unsigned char map[256]);

// 方案 C: 使用 thread_local 存储 (C11)
_Thread_local static char map[256];
_Thread_local static int inited = 0;
```

### 16.5.4 边界检查与安全性

Text 使用 `assert` 进行运行时检查，这些检查在 `NDEBUG` 构建中会被移除。对于安全敏感的代码，可以考虑：

```c
// 使用 checked 返回类型而非 assert
typedef struct {
    T    value;
    bool ok;
} Text_Result;

Text_Result Text_sub_checked(T s, int i, int j) {
    Text_Result r = { .ok = false };
    if (s.len < 0 || s.str == NULL) return r;
    // ...
    r.ok = true;
    return r;
}
```

### 16.5.5 内存池策略

当前 Text 的 arena 是无限增长的——`Text_save`/`Text_restore` 提供了手动垃圾回收，但需要在正确的时机调用。对于长时间运行的程序，可以考虑：

- **引用计数**：在描述符中增加引用计数，`Text_cat` 和 `Text_sub` 增加计数，释放时递减。
- **标记-清除 GC**：如 Further Reading 中提到的 Icon 语言的 XPL 压缩算法——跟踪所有"注册"的 Text_T，将活跃字符串紧凑到 arena 头部。
- **Rope 数据结构**：将字符串表示为二叉树，拼接是 O(1)，子串是 O(log n)。在文本编辑器场景中这是更优的选择。

---

## 16.6 应用场景

### 16.6.1 富文本编辑器 / IDE

Text 的不可变共享语义天然适合文本编辑器中的以下操作：

- **Undo/Redo 栈**：每次编辑操作产生一个新的 Text_T，旧的 Text_T 保留在 undo 历史中。由于拼接语义，未修改的部分零拷贝共享。
- **语法高亮**：`Text_sub` 提取 token 进行语法分析，`Text_map` 应用颜色映射。
- **增量解析**：修改一行时，只需重新解析该行——其余部分仍然由相同的 Text_T 描述。

### 16.6.2 编译器构造

- **符号表**：标识符作为 Text_T 存储，`Text_cmp` 进行快速比较。
- **字符串驻留 (Interning)**：Text_T 可以结合 Atom 接口（第3章）实现字符串驻留——相同内容的字符串在内存中只保留一份。
- **源码位置追踪**：`Text_sub` 提取错误的源行进行诊断输出。

### 16.6.3 网络协议解析

- **HTTP 头解析**：`Text_upto(text, pos, 0, Text_box(":\r\n",3))` 定位冒号，`Text_sub` 提取字段名和值。
- **零拷贝包处理**：将网络缓冲区 `Text_box` 装箱，然后逐字段解析——不需要复制负载数据。

### 16.6.4 日志与文本分析

- **日志行过滤**：`Text_find` 在每行中搜索关键词。
- **CSV 解析**：`Text_upto` 跳过逗号，`Text_sub` 提取字段。
- **文本统计**：`Text_chr` 计数特定字符，`len` 字段提供 O(1) 长度。

---

## 16.7 总结

Text 是 Str 之上的高层抽象。它的核心创新在于：

1. **O(1) 求长度**：通过将长度与数据捆绑在一起。
2. **Splice 而非 Copy**：利用 arena 分配器的空间局部性，将尽可能多的字符串操作变为指针运算。
3. **不可变性**：简化了并发推理，使子串可以安全地共享底层存储。
4. **嵌入空字符**：使 Text 能够处理任意二进制数据，不局限于 C 字符串约定。
5. **栈式内存管理**：`Text_save`/`Text_restore` 提供了确定性的空间回收，适合解析器中的临时字符串生命期。

Text 在设计上的克制（仅 400 行 C 代码）和其实现的优雅（三层拼接优化、arena 分配器、位置系统）使其成为 C 语言接口设计的经典范例。

---

## 参考资料

- Griswold, R. E. (1972). *The SNOBOL4 Programming Language*. Prentice-Hall.
- Griswold, R. E. and M. T. Griswold (1990). *The Icon Programming Language*. Peer-to-Peer Communications.
- McKeeman, W. M., J. J. Horning, and D. B. Wortman (1970). *A Compiler Generator*. Prentice-Hall.
- Hanson, D. R. (1980). "A Portable Storage Management System for the Icon Programming Language." *Software---Practice and Experience*, 10:489--500.
- Hansen, W. J. (1992). "Subsequence References: First-Class Values for Substrings." *ACM Transactions on Programming Languages and Systems*, 14(4):471--489.
- Boehm, H.-J., R. Atkinson, and M. Plass (1995). "Ropes: an Alternative to Strings." *Software---Practice and Experience*, 25(12):1315--1330.

---

*本章翻译整合了原著英文版第16章全文（第269-295页）与源代码 `text.h` (45行) 和 `text.c` (397行)，并补充了现代C编程视角的扩展讨论。*
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch15_低级字符串_Str.html">第15章 低级字符串 Str</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch17_扩展精度算术_XP.html">第17章 扩展精度 XP</a></div>
</div>
