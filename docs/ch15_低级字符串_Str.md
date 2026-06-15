# 第15章 低级字符串 (Low-Level Strings)

> 原文: David R. Hanson, *C Interfaces and Implementations: Techniques for Creating Reusable Software*, Chapter 15.
> 源码: `code/str.h`, `code/str.c`, `code/text.h`, `code/text.c`

---

## 15.1 C语言字符串的困境

C不是一门以字符串处理见长的语言, 但它确实提供了操作字符数组的手段 —— 习惯上我们把这些字符数组称为**字符串**。在C语言惯例中, 一个包含N个字符的字符串由一个长度为N+1的字符数组表示, 最后一个字符是空字符(`'\0'`)。

C语言本身只提供了两个帮助处理字符串的特性: 指向字符的指针可以遍历字符数组; 字符串字面量可以初始化字符数组:

```c
char msg[] = "File not found";
// 等价于:
char msg[] = { 'F', 'i', 'l', 'e', ' ', 'n', 'o', 't',
               ' ', 'f', 'o', 'u', 'n', 'd', '\0' };
```

C标准库提供了一套操作以空字符结尾的字符串的函数, 定义在标准头文件 `<string.h>` 中, 包括拷贝、搜索、扫描、比较和转换字符串。`strcat` 是一个典型代表:

```c
char *strcat(char *dst, const char *src);
```

它将 `src` 追加到 `dst` 的末尾。`strcat` 暴露了 `<string.h>` 中定义的所有函数的**两个缺点**:

1. **调用者必须为结果分配空间** —— 例如 `dst` 必须由调用者分配。
2. **所有函数都是不安全的** —— 没有任何一个函数能检查结果字符串是否足够大。如果 `dst` 不够大, `strcat` 将向未分配的内存写入数据, 造成缓冲区溢出。

虽然 `strncat` 等函数接受额外的参数来限制拷贝的字符数, 但这只是缓解而非根治 —— 分配错误仍然可能发生。

### Hanson 的两层方案

David Hanson 在本书中提出了两个层次的字符串抽象来解决上述问题:

| 层次 | 接口 | 章节 | 核心设计 |
|------|------|------|----------|
| 低级 | **Str** | 第15章 | 基于**位置(position)**的子串操作; 自动分配结果空间; 保留C风格null-terminated约定 |
| 高级 | **Text** | 第16章 | 带**长度**的结构体(`Text_T`); 不可变字符串; 零拷贝子串; O(1)取长度 |

本章详细讲解 **Str** 接口, 并在此基础上引入 **Text** 接口的核心理念以展示C语言字符串抽象的完整演进路径。

---

## 15.2 Str接口: 基于位置的安全字符串操作

Str接口中的所有函数都通过一个指向以空字符结尾的字符数组的指针和**位置(position)** 来指定参数。位置标识字符之间的边界, 包括最后一个非空字符之后的位置。

### 15.2.1 位置约定

- **正位置** 从字符串左端开始计数: 位置 1 位于第一个字符之左。
- **非正位置** 从字符串右端开始计数: 位置 0 位于最后一个字符之右。

以字符串 `Interface` 为例:

```
 1   2   3   4   5   6   7   8   9   10
  I   n   t   e   r   f   a   c   e
-9  -8  -7  -6  -5  -4  -3  -2  -1   0
```

字符串 `s` 中两个位置 `i` 和 `j` 指定它们之间的子串, 记为 `s[i:j]`。位置可以以任意次序给出: `s[-4:0]` 和 `s[0:-4]` 都表示子串 `face`。子串可以为空: `s[3:3]` 和 `s[3:-7]` 都表示 `n` 和 `t` 之间的空子串。

### 15.2.2 为什么用位置而不是字符索引?

字符索引初看起来更自然, 但有诸多不便:

1. **顺序敏感**: 用索引指定子串时顺序至关重要。
2. **边界情况混乱**: 无法优雅地表示空子串或末尾子串。
3. **需要额外的负索引约定**: 虽然可以用负索引, 但比位置更笨拙。

位置的优势在于:
- 避开索引的上列边界困惑;
- 非正位置可以在不知道字符串长度的情况下访问字符串尾部;
- 正位置减一即得字符索引, 转换简单直观。

### 15.2.3 接口声明

```c
/* str.h -- Low-Level Strings Interface */
#ifndef STR_INCLUDED
#define STR_INCLUDED
#include <stdarg.h>
#include "fmt.h"

/* === 创建字符串的函数 (均分配空间) === */

extern char *Str_sub(const char *s, int i, int j);
extern char *Str_dup(const char *s, int i, int j, int n);
extern char *Str_cat(const char *s1, int i1, int j1,
                     const char *s2, int i2, int j2);
extern char *Str_catv(const char *s, ...);
extern char *Str_reverse(const char *s, int i, int j);
extern char *Str_map(const char *s, int i, int j,
                     const char *from, const char *to);

/* === 信息查询函数 (不分配空间) === */

extern int Str_pos(const char *s, int i);
extern int Str_len(const char *s, int i, int j);
extern int Str_cmp(const char *s1, int i1, int j1,
                   const char *s2, int i2, int j2);

/* === 搜索函数 === */

extern int Str_chr  (const char *s, int i, int j, int c);
extern int Str_rchr (const char *s, int i, int j, int c);
extern int Str_upto (const char *s, int i, int j, const char *set);
extern int Str_rupto(const char *s, int i, int j, const char *set);
extern int Str_find (const char *s, int i, int j, const char *str);
extern int Str_rfind(const char *s, int i, int j, const char *str);

/* === 跨越函数 (step over) === */

extern int Str_any   (const char *s, int i, const char *set);
extern int Str_many  (const char *s, int i, int j, const char *set);
extern int Str_rmany (const char *s, int i, int j, const char *set);
extern int Str_match (const char *s, int i, int j, const char *str);
extern int Str_rmatch(const char *s, int i, int j, const char *str);

/* === 格式化函数 === */

extern void Str_fmt(int code, va_list_box *box,
                    int put(int c, void *cl), void *cl,
                    unsigned char flags[], int width, int precision);

#endif
```

---

## 15.3 API逐函数中文注释

### 15.3.1 创建字符串的函数

以下所有函数为其结果**分配空间**, 均可能引发 `Mem_Failed` 异常。向本接口中的任何函数传递空字符串指针属于**受检运行时错误**(checked runtime error), 以下对 `Str_catv` 和 `Str_map` 的特别说明除外。

#### Str_sub — 提取子串

```c
extern char *Str_sub(const char *s, int i, int j);
```

返回 `s[i:j]`, 即字符串 `s` 中位置 `i` 和 `j` 之间的子串。位置可以任意次序给出。以下四个调用均返回 `"face"`:

```c
Str_sub("Interface",  6, 10)   // → "face"
Str_sub("Interface",  6,  0)   // → "face"
Str_sub("Interface", -4, 10)   // → "face"
Str_sub("Interface", -4,  0)   // → "face"
```

**约束**: `i` 和 `j` 必须指定 `s` 中的有效子串, 否则为受检运行时错误。

#### Str_dup — 复制n份

```c
extern char *Str_dup(const char *s, int i, int j, int n);
```

返回 `n` 份 `s[i:j]` 拷贝拼接而成的字符串。`n` 为负数属于受检运行时错误。

```c
Str_dup("Interface", 1, 0, 1)   // → "Interface"  (复制一整份)
Str_dup("ab", 1, 0, 3)          // → "ababab"     (三份拷贝拼接)
```

`Str_dup("Interface", 1, 0, 1)` 是一个常用的复制整串的惯用法: 用位置 `1` 和 `0` 涵盖全部字符串。

#### Str_cat — 连接两个子串

```c
extern char *Str_cat(const char *s1, int i1, int j1,
                     const char *s2, int i2, int j2);
```

返回 `s1[i1:j1]` 和 `s2[i2:j2]` 的拼接结果。

```c
Str_cat("Interface", 1, 6, "ion", 1, 0)  // → "Interfacion"
```

#### Str_catv — 变长参数拼接

```c
extern char *Str_catv(const char *s, ...);
```

接受零个或多个三元组 `(字符串指针, 位置i, 位置j)`, 以空指针终止参数列表。返回所有这些子串的拼接结果。

```c
Str_catv("Interface", -4, 0, " plant", 1, 0, NULL)
// → "face plant"
```

参数列表中每三个一组: 字符串指针 + 位置 i + 位置 j。`NULL` 终止参数列表。

#### Str_reverse — 反转子串

```c
extern char *Str_reverse(const char *s, int i, int j);
```

返回由 `s[i:j]` 中的字符按相反顺序排列而成的字符串。

```c
Str_reverse("Interface", 1, 0)  // → "ecafretnI"
```

#### Str_map — 字符映射

```c
extern char *Str_map(const char *s, int i, int j,
                     const char *from, const char *to);
```

返回一个字符串, 对其中的字符按 `from` 和 `to` 指定的映射进行转换。`s[i:j]` 中出现在 `from` 里的每个字符被映射到 `to` 中对应的字符; 未出现在 `from` 中的字符保持不变。

```c
Str_map(s, 1, 0, "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
                  "abcdefghijklmnopqrstuvwxyz")
// 返回 s 的一个副本, 其中大写字母被替换为对应的小写字母
```

Str_map 有三种调用模式:
- **普通模式**: `from` 和 `to` 均非空且 `s` 非空 —— 进行一次映射并返回新字符串。
- **记忆模式**: `from` 和 `to` 均非空且 `s` 为空 —— 只建立默认映射, 返回 `NULL`。后续调用可用 `from=NULL, to=NULL` 复用该映射。
- **复用模式**: `from` 和 `to` 均为 `NULL` 且 `s` 非空 —— 使用最近一次调用建立的映射。

**受检运行时错误**:
- `from` 和 `to` 中只有一个为空;
- `from` 和 `to` 非空但长度不同;
- `s`、`from`、`to` 三者均为空;
- 首次调用 Str_map 时 `from` 和 `to` 均为空。

---

### 15.3.2 信息查询函数

这些函数返回字符串或位置的信息, **不分配空间**。

#### Str_pos — 位置规范化

```c
extern int Str_pos(const char *s, int i);
```

返回与 `s[i:i]` 对应的正位置。正位置减一即得索引, 因此 `Str_pos` 常用于需要索引的场合:

```c
if (s 指向 "Interface")
    printf("%s\n", &s[Str_pos(s, -4) - 1]);  // 打印 "face"
```

`Str_pos(s, -4)` 返回 `6` (正位置), 减一得索引 `5`, 即字符 `f`。

#### Str_len — 子串长度

```c
extern int Str_len(const char *s, int i, int j);
```

返回 `s[i:j]` 中的字符数。

```c
Str_len("Interface", -4, 0)   // → 4  ("face")
Str_len("Interface", 1, 0)    // → 9  (整串)
```

#### Str_cmp — 子串比较

```c
extern int Str_cmp(const char *s1, int i1, int j1,
                   const char *s2, int i2, int j2);
```

按字典序比较 `s1[i1:j1]` 和 `s2[i2:j2]`。返回值:
- `< 0`: `s1[i1:j1]` 小于 `s2[i2:j2]`
- `= 0`: 两者相等
- `> 0`: `s1[i1:j1]` 大于 `s2[i2:j2]`

当较短串等于较长串的前缀时, 较短串被视为较小。

---

### 15.3.3 搜索函数

搜索成功时返回**正位置**, 反映搜索结果; 搜索失败时返回**0**。名称中包含 `_r` 的函数从右端搜索, 其余从左端搜索。

#### Str_chr / Str_rchr — 搜索字符

```c
extern int Str_chr (const char *s, int i, int j, int c);
extern int Str_rchr(const char *s, int i, int j, int c);
```

返回 `s[i:j]` 中最左端(或最右端)出现字符 `c` 之前的位置, 或 0 表示未找到。

```c
Str_chr("Interface", 1, 0, 'f')   // → 6  (f 的左位置)
Str_rchr("Interface", 1, 0, 'e')  // → 9  (最后一个 e 的左位置)
```

位置返回值可以无缝作为下一次操作参数:
```c
i = Str_chr(s, 1, 0, ':');   // 找到冒号
j = Str_chr(s, i+1, 0, ':'); // 找到下一个冒号
```

#### Str_upto / Str_rupto — 搜索集合中的任意字符

```c
extern int Str_upto (const char *s, int i, int j, const char *set);
extern int Str_rupto(const char *s, int i, int j, const char *set);
```

返回 `s[i:j]` 中出现在集合 `set` 中最左端(或最右端)字符之前的位置, 或 0。

```c
Str_upto("The rain in Spain", 1, 0, "aeiou")  // → 3  (找到 'e')
```

**约束**: 向 `Str_upto` 或 `Str_rupto` 传递空 `set` 属于受检运行时错误。

#### Str_find / Str_rfind — 搜索子串

```c
extern int Str_find (const char *s, int i, int j, const char *str);
extern int Str_rfind(const char *s, int i, int j, const char *str);
```

返回 `s[i:j]` 中最左端(或最右端)出现 `str` 之前的位置, 或 0。

```c
Str_find ("The rain in Spain", 1, 0, "in")   // → 7
Str_rfind("The rain in Spain", 1, 0, "in")   // → 16 (最右端的 in)
```

**约束**: 向 `Str_find` 或 `Str_rfind` 传递空 `str` 属于受检运行时错误。

**注意**: `Str_rchr`、`Str_rupto` 和 `Str_rfind` 虽然从右端搜索, 但返回的是所搜索字符或字符串**之左**的位置。

---

### 15.3.4 跨越函数 (Step Over)

这些函数"跨越"子串, 返回紧随匹配子串之后(或之前)的正位置。与搜索函数不同, 它们在匹配成功后继续向前(或向后)越过匹配内容。

#### Str_any — 跳过单个集合字符

```c
extern int Str_any(const char *s, int i, const char *set);
```

如果 `s[i:i+1]` 在 `set` 中, 返回该字符之后的位置; 否则返回 0。

```c
Str_any("abc123", 4, "0123456789")  // → 5  (跳过 '1')
Str_any("abc123", 1, "0123456789")  // → 0  ('a' 不在集合中)
```

#### Str_many — 跳过连续集合字符

```c
extern int Str_many(const char *s, int i, int j, const char *set);
```

如果 `s[i:j]` 以 `set` 中的一个或多个连续字符开头, 返回紧随这段连续字符之后的位置; 否则返回 0。

```c
Str_many("123abc", 1, 0, "0123456789")  // → 4  (跳过 "123")
```

#### Str_rmany — 从右端跳过连续集合字符

```c
extern int Str_rmany(const char *s, int i, int j, const char *set);
```

如果 `s[i:j]` 以 `set` 中的一个或多个连续字符结尾, 返回这段连续字符之前的位置。

```c
Str_rmany("abc123", 1, 0, "0123456789")  // → 4  ("123" 之前的 a 位置)
```

实用示例 —— 去除尾部空白:
```c
Str_sub(name, 1, Str_rmany(name, 1, 0, " \t"))
// 返回 name 去掉尾部空格和制表符后的副本
```

#### Str_match — 匹配前缀

```c
extern int Str_match(const char *s, int i, int j, const char *str);
```

如果 `s[i:j]` 以 `str` 开头, 返回紧随 `str` 之后的位置; 否则返回 0。

```c
Str_match("Interface", 1, 0, "Inter")  // → 6
```

#### Str_rmatch — 匹配后缀

```c
extern int Str_rmatch(const char *s, int i, int j, const char *str);
```

如果 `s[i:j]` 以 `str` 结尾, 返回 `str` 之前的位置; 否则返回 0。

```c
Str_rmatch("Interface", 1, 0, "face")  // → 6
```

#### 实际应用: basename 函数

`basename` 接受 UNIX 风格路径名, 返回不带目录和指定后缀的文件名:

```c
char *basename(char *path, int i, int j, const char *suffix) {
    i = Str_rchr(path, i, j, '/');              // 找到最右端斜杠
    j = Str_rmatch(path, i + 1, 0, suffix);     // 匹配后缀
    return Str_dup(path, i + 1, j, 1);          // 提取文件主体
}
```

| 输入 | 输出 |
|------|------|
| `basename("/usr/jenny/main.c", 1, 0, ".c")` | `main` |
| `basename("../src/main.c", 1, 0, "")` | `main.c` |
| `basename("main.c", 1, 0, "c")` | `main.` |
| `basename("examples/wfmain.c", 1, 0, "main.c")` | `wf` |

---

### 15.3.5 格式化函数

#### Str_fmt — Fmt接口的转换函数

```c
extern void Str_fmt(int code, va_list_box *box,
                    int put(int c, void *cl), void *cl,
                    unsigned char flags[], int width, int precision);
```

这是一个符合 Fmt 接口规范的转换函数。它从可变参数中消费三个参数 —— 字符串指针和两个位置 —— 并按 Fmt 的 `%s` 风格格式化子串。

```c
Fmt_register('S', Str_fmt);
Fmt_print("%10S\n", "Interface", -4, 0);   // 打印 "______face" (_表示空格)
```

---

## 15.4 示例: 打印标识符 (ids.c)

`ids.c` 程序打印输入中所有的 C 关键字和标识符, 展示了 Str 系列函数的典型用法, 特别是搜索函数与跨越函数的配合:

```c
#include <stdlib.h>
#include <stdio.h>
#include "fmt.h"
#include "str.h"

int main(int argc, char *argv[]) {
    char line[512];
    static char set[] = "0123456789_"
        "abcdefghijklmnopqrstuvwxyz"
        "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    Fmt_register('S', Str_fmt);
    while (fgets(line, sizeof line, stdin) != NULL) {
        int i = 1, j;
        while ((i = Str_upto(line, i, 0, &set[10])) > 0) {
            j = Str_many(line, i, 0, set);
            Fmt_print("%S\n", line, i, j);
            i = j;
        }
    }
    return EXIT_SUCCESS;
}
```

**执行流程分析**:

1. `Fmt_register('S', Str_fmt)` 将 `%S` 格式码与 `Str_fmt` 关联。
2. 外层 `while` 循环逐行读取输入。
3. 内层 `while` 循环扫描 `line[i:0]` 中的下一个标识符:
   - `Str_upto(line, i, 0, &set[10])` 搜索 `line[i:0]` 中下一个字母或下划线(跳过前10个数字字符)。 `&set[10]` 指向 `"abcdef..."`。
   - 若找到, `Str_many(line, i, 0, set)` 跨越连续的标识符字符(字母、数字、下划线)。
   - `Fmt_print("%S\n", line, i, j)` 使用 `Str_fmt` 打印子串。
   - `i = j` 使下一轮迭代从当前位置继续搜索。

以 `int main(int argc, char *argv[]) {` 为例:

```
位置:   1   5   10   14   20   26   ...
字符串:  int main (int  argc ,  char ...
         ^   ^   ^    ^     ^    ^
         i=1 i=5 i=10  i=14  i=20 i=26
```

**关键洞察**: 这个程序中没有任何内存分配 —— 位置约定让我们可以在不复制字符串的情况下遍历和提取子串。

---

## 15.5 Str实现: 核心机制

### 15.5.1 位置到索引的转换: `idx` 宏

```c
#define idx(i, len) ((i) <= 0 ? (i) + (len) : (i) - 1)
```

给定位置 `i` 和字符串长度 `len`, `idx(i, len)` 返回 `i` 右侧字符的索引:

- 正位置: `i <= 0` 不成立, 返回 `i - 1` (因为位置1对应索引0, 位置2对应索引1, ...)
- 非正位置: `i <= 0` 成立, 返回 `i + len` (例如位置 `-4` 在长度为9的字符串中 = `-4 + 9 = 5`, 对应字符 `f`)

### 15.5.2 统一的参数规范化: `convert` 宏

```c
#define convert(s, i, j) do { int len; \
    assert(s); len = strlen(s); \
    i = idx(i, len); j = idx(j, len); \
    if (i > j) { int t = i; i = j; j = t; } \
    assert(i >= 0 && j <= len); } while (0)
```

`convert` 宏封装了所有 Str 函数共用的参数规范化步骤:

1. **断言** `s` 非空 (`assert(s)`)
2. **获取长度**: `len = strlen(s)`
3. **转换为索引**: `i = idx(i, len)`, `j = idx(j, len)` —— 将位置转换为 0 到 `len` 范围内的索引
4. **排顺序**: 若 `i > j`, 交换两者, 保证 `i <= j`
5. **验证范围**: `assert(i >= 0 && j <= len)` —— 确保 `i` 和 `j` 指定 `s` 中的有效位置

转换后, `j - i` 即为指定子串的长度。

### 15.5.3 Str_sub 的实现

```c
char *Str_sub(const char *s, int i, int j) {
    char *str, *p;
    convert(s, i, j);                    // 规范化参数
    p = str = ALLOC(j - i + 1);          // 分配 j-i 个字符 + '\0'
    while (i < j)
        *p++ = s[i++];                   // 逐字符拷贝
    *p = '\0';                           // 添加空字符结尾
    return str;
}
```

`ALLOC(j - i + 1)` 中的 `+1` 为终止空字符预留空间。`convert` 后 `j` 的值指向子串之后的位置(可能是空字符), 因此 `j-i` 恰好是子串长度。

### 15.5.4 Str_dup 的实现

```c
char *Str_dup(const char *s, int i, int j, int n) {
    int k;
    char *str, *p;
    assert(n >= 0);                       // n必须非负
    convert(s, i, j);
    p = str = ALLOC(n * (j - i) + 1);     // 分配 n份长度 + '\0'
    if (j - i > 0)                        // 仅在子串非空时拷贝
        while (n-- > 0)
            for (k = i; k < j; k++)
                *p++ = s[k];              // 逐份逐字符拷贝
    *p = '\0';
    return str;
}
```

当 `n=1` 且 `j-i > 0` 时, 效果等同于 `Str_sub`。

### 15.5.5 Str_reverse 的实现

```c
char *Str_reverse(const char *s, int i, int j) {
    char *str, *p;
    convert(s, i, j);
    p = str = ALLOC(j - i + 1);
    while (j > i)
        *p++ = s[--j];    // 从右向左拷贝, 先递减再取值
    *p = '\0';
    return str;
}
```

逆向遍历: `j` 从子串末尾向 `i` 方向递减, 每次先将 `j` 减一再取值。

### 15.5.6 Str_cat 和 Str_catv 的实现

**Str_cat** —— 两个子串的拼接:

```c
char *Str_cat(const char *s1, int i1, int j1,
              const char *s2, int i2, int j2) {
    char *str, *p;
    convert(s1, i1, j1);
    convert(s2, i2, j2);
    p = str = ALLOC(j1 - i1 + j2 - i2 + 1);  // 总长度 + '\0'
    while (i1 < j1) *p++ = s1[i1++];         // 拷贝第一段
    while (i2 < j2) *p++ = s2[i2++];         // 拷贝第二段
    *p = '\0';
    return str;
}
```

虽然可以直接调用 `Str_catv`, 但 `Str_cat` 使用频率高, 值得拥有专门的高效实现。

**Str_catv** —— 变长参数的拼接。需要**两趟遍历**:

```c
char *Str_catv(const char *s, ...) {
    char *str, *p;
    const char *save = s;
    int i, j, len = 0;
    va_list ap;

    // 第一趟: 计算总长度
    va_start(ap, s);
    while (s) {
        i = va_arg(ap, int);
        j = va_arg(ap, int);
        convert(s, i, j);
        len += j - i;
        s = va_arg(ap, const char *);
    }
    va_end(ap);

    // 分配空间 + 第二趟: 拷贝
    p = str = ALLOC(len + 1);
    s = save;
    va_start(ap, s);
    while (s) {
        i = va_arg(ap, int);
        j = va_arg(ap, int);
        convert(s, i, j);
        while (i < j) *p++ = s[i++];
        s = va_arg(ap, const char *);
    }
    va_end(ap);
    *p = '\0';
    return str;
}
```

第一趟计算总长度以便一次性分配足够空间; 第二趟实际拷贝。两次遍历必须重新启动 `va_list`, 因此第一趟开始时保存 `s` 到 `save` 以便恢复。

### 15.5.7 Str_map 的实现

```c
char *Str_map(const char *s, int i, int j,
              const char *from, const char *to) {
    static char map[256] = { 0 };       // 静态映射表, 进程生命周期内持久

    if (from && to) {
        /* 构建映射表 */
        unsigned c;
        for (c = 0; c < sizeof map; c++)
            map[c] = c;                  // 默认: 每个字符映射到自己
        while (*from && *to)
            map[(unsigned char)*from++] = *to++;  // 设置特定映射
        assert(*from == 0 && *to == 0);  // 要求 from 和 to 长度相等
    } else {
        /* 复用之前的映射 */
        assert(from == NULL && to == NULL && s);
        assert(map['a']);                // 确保非首次调用
    }

    if (s) {
        /* 执行映射 */
        char *str, *p;
        convert(s, i, j);
        p = str = ALLOC(j - i + 1);
        while (i < j)
            *p++ = map[(unsigned char)s[i++]];   // 通过查表映射
        *p = '\0';
        return str;
    } else
        return NULL;
}
```

`(unsigned char)` 强制类型转换防止值超过 127 的字符被符号扩展为负索引。

### 15.5.8 Str_pos 和 Str_len 的实现

```c
int Str_pos(const char *s, int i) {
    int len;
    assert(s);
    len = strlen(s);
    i = idx(i, len);           // 转为索引 0..len
    assert(i >= 0 && i <= len);
    return i + 1;              // 转为正位置 1..len+1
}
```

`Str_pos` 将任意位置转换为正位置: 先转为索引, 验证范围, 再加一。

```c
int Str_len(const char *s, int i, int j) {
    convert(s, i, j);
    return j - i;              // 转换后 j-i 即为长度
}
```

### 15.5.9 Str_cmp 的实现

```c
int Str_cmp(const char *s1, int i1, int j1,
            const char *s2, int i2, int j2) {
    convert(s1, i1, j1);
    convert(s2, i2, j2);
    s1 += i1;                   // 指针移到第一个字符
    s2 += i2;

    if (j1 - i1 < j2 - i2) {             // s1 较短
        int cond = strncmp(s1, s2, j1 - i1);
        return cond == 0 ? -1 : cond;     // 相等前缀 → s1 < s2
    } else if (j1 - i1 > j2 - i2) {      // s2 较短
        int cond = strncmp(s1, s2, j2 - i2);
        return cond == 0 ? +1 : cond;     // 相等前缀 → s1 > s2
    } else
        return strncmp(s1, s2, j1 - i1);  // 等长, 直接比较
}
```

较短者决定比较长度。当 `strncmp` 返回 0 且长度不等时, 较短者作为较长者的前缀, 被视为较小。

**注意**: C 标准规定 `strncmp` 和 `memcmp` 必须将字符视为 **unsigned char** 进行比较。但有些实现错误地将字符当普通(可能带符号)的 `char` 处理, 导致值超过 127 时产生错误结果。本书的实现使用了正确的语义。

### 15.5.10 搜索函数的实现

**Str_chr**(从左搜索字符):

```c
int Str_chr(const char *s, int i, int j, int c) {
    convert(s, i, j);
    for ( ; i < j; i++)
        if (s[i] == c)
            return i + 1;       // 返回正位置(索引加一)
    return 0;                   // 未找到
}
```

**Str_rchr**(从右搜索字符):

```c
int Str_rchr(const char *s, int i, int j, int c) {
    convert(s, i, j);
    while (j > i)
        if (s[--j] == c)
            return j + 1;
    return 0;
}
```

**Str_find**(搜索子串)。将长度为 0 或 1 的搜索串作为特例处理以避免开销:

```c
int Str_find(const char *s, int i, int j, const char *str) {
    int len;
    convert(s, i, j);
    assert(str);
    len = strlen(str);
    if (len == 0)
        return i + 1;                        // 空串总在开头
    else if (len == 1) {
        for ( ; i < j; i++)
            if (s[i] == *str)
                return i + 1;
    } else
        for ( ; i + len <= j; i++)
            if (strncmp(&s[i], str, len) == 0)  // 比较 len 个字符
                return i + 1;
    return 0;
}
```

**关键细节**: `i + len <= j` 确保匹配不会越过子串边界。

**Str_rfind** 是镜像实现, 小心避免匹配越过子串开头:

```c
int Str_rfind(const char *s, int i, int j, const char *str) {
    int len;
    convert(s, i, j);
    assert(str);
    len = strlen(str);
    if (len == 0)
        return j + 1;
    else if (len == 1) {
        while (j > i)
            if (s[--j] == *str) return j + 1;
    } else
        for ( ; j - len >= i; j--)
            if (strncmp(&s[j-len], str, len) == 0)
                return j - len + 1;
    return 0;
}
```

### 15.5.11 跨越函数的实现

**Str_many** 跨越开头连续的集合字符:

```c
int Str_many(const char *s, int i, int j, const char *set) {
    assert(set);
    convert(s, i, j);
    if (i < j && strchr(set, s[i])) {   // 第一个字符在集合中?
        do i++;
        while (i < j && strchr(set, s[i]));  // 继续跨越
        return i + 1;                    // 返回紧随连续段之后的正位置
    }
    return 0;
}
```

**Str_rmany** 从右端跨越连续集合字符:

```c
int Str_rmany(const char *s, int i, int j, const char *set) {
    assert(set);
    convert(s, i, j);
    if (j > i && strchr(set, s[j-1])) {   // 最后一个字符在集合中?
        do --j;
        while (j >= i && strchr(set, s[j]));  // 继续后退
        return j + 2;                      // 返回连续段之前的正位置
    }
    return 0;
}
```

**`Str_rmany` 返回值分析**: 当 `do-while` 循环终止时, `j` 等于 `i - 1` 或是不在 `set` 中的字符的索引。在第一种情况下, 应返回 `i+1`(第一个字符之前的位置); 在第二种情况下, 应返回 `s[j]` 之右的位置。`j+2` 对这两种情况都正确:
- 若 `j = i - 1`, `j + 2 = i + 1`(该跨度之前的位置)
- 若 `j` 是不在 `set` 中的字符, `j + 2` 即为该字符之右的位置

**Str_match** 匹配前缀:

```c
int Str_match(const char *s, int i, int j, const char *str) {
    int len;
    convert(s, i, j);
    assert(str);
    len = strlen(str);
    if (len == 0) return i + 1;
    else if (len == 1) {
        if (i < j && s[i] == *str) return i + 2;
    } else if (i + len <= j && strncmp(&s[i], str, len) == 0)
        return i + len + 1;
    return 0;
}
```

**Str_rmatch** 匹配后缀:

```c
int Str_rmatch(const char *s, int i, int j, const char *str) {
    int len;
    convert(s, i, j);
    assert(str);
    len = strlen(str);
    if (len == 0) return j + 1;
    else if (len == 1) {
        if (j > i && s[j-1] == *str) return j;
    } else if (j - len >= i && strncmp(&s[j-len], str, len) == 0)
        return j - len + 1;
    return 0;
}
```

### 15.5.12 Str_fmt 的实现

```c
void Str_fmt(int code, va_list_box *box,
             int put(int c, void *cl), void *cl,
             unsigned char flags[], int width, int precision) {
    char *s;
    int i, j;
    assert(box && flags);
    s = va_arg(box->ap, char *);    // 消费三个参数: 字符串指针
    i = va_arg(box->ap, int);        //           位置 i
    j = va_arg(box->ap, int);        //           位置 j
    convert(s, i, j);
    Fmt_puts(s + i, j - i, put, cl, flags, width, precision);
}
```

`Str_fmt` 从 Fmt 的可变参数列表中消费三个参数, 将位置转换为索引后, 委托给 `Fmt_puts` 执行实际的格式化输出。

---

## 15.6 Text接口: 带长度的字符串结构

### 15.6.1 从Str到Text: 设计动机

Str接口解决了 `<string.h>` 的两个核心问题:
1. **安全性**: Str函数自行分配结果空间, 杜绝缓冲区溢出
2. **便利性**: 位置约定使子串操作统一而优雅

但它仍然有两个内在局限:
1. **长度计算**: `convert` 宏每次都要调用 `strlen()`, 时间复杂度 O(n)
2. **不必要的分配**: 很多场景下(如只读扫描), 字符串不必复制

Hanson在第16章中设计了 **Text** 接口来解决这些问题。在此我们将 Text 接口的核心理念与 Str 对照阐述, 形成完整的认知图景。

### 15.6.2 Text_T 结构体

```c
#define T Text_T
typedef struct T {
    int len;           // 字符串长度 (不含 '\0')
    const char *str;   // 指向字符数据的指针
} T;
```

Text_T 是一个**值类型**结构体, 仅含两个字段:
- `len`: 字符串长度, O(1) 可获取, 无需遍历
- `str`: 指向实际字符数据的指针, `const` 表明数据是**不可变(immutable)** 的

Text 的字符串**不以 `'\0'` 结尾**, 可以嵌入任意字节(包括 `'\0'`)。Text 字符串是不可变的 —— 不能在原地修改。

### 15.6.3 数据共享与零拷贝设计

Text_T 的核心创新在于**子串操作的零拷贝**:

```c
T Text_sub(T s, int i, int j) {
    T text;
    i = idx(i, s.len);
    j = idx(j, s.len);
    if (i > j) { int t = i; i = j; j = t; }
    text.len = j - i;          // 仅设置新的长度
    text.str = s.str + i;      // 指向同一块数据的偏移位置
    return text;               // 不分配新内存!
}
```

这个设计类似于现代语言中的 **string view** 概念: `Text_sub` 不复制数据, 只创建一个指向原始数据子区间的"视图"。这是一种经典的**零拷贝设计模式**。

### 15.6.4 arena分块内存分配

Text使用一种**arena(竞技场)分配器**来管理内存:

```c
static struct chunk {
    struct chunk *link;
    char *avail;
    char *limit;
} head = { NULL, NULL, NULL }, *current = &head;

static char *alloc(int len) {
    if (current->avail + len > current->limit) {
        // 当前chunk不够, 分配新chunk (10KB + len)
        current = current->link =
            ALLOC(sizeof(*current) + 10*1024 + len);
        current->avail = (char *)(current + 1);
        current->limit = current->avail + 10*1024 + len;
        current->link = NULL;
    }
    current->avail += len;
    return current->avail - len;
}
```

设计要点:
- 每次分配一个 **10KB + 请求大小** 的大块(chunk), 减少 `malloc` 调用次数
- `avail` 和 `limit` 跟踪当前 chunk 的可用空间
- 分配的字符串**不单独释放**; 通过 `Text_save`/`Text_restore` 批量回收

这是一种**批量化内存管理**策略, 特别适合编译器和文本处理等需要大量临时字符串的场景。

### 15.6.5 Text 连接优化

Text的 `Text_cat` 实现利用了几种优化:

```c
T Text_cat(T s1, T s2) {
    if (s1.len == 0) return s2;                // 优化1: 空串
    if (s2.len == 0) return s1;                // 优化2: 空串
    if (s1.str + s1.len == s2.str) {           // 优化3: 物理相邻
        s1.len += s2.len;
        return s1;                             // 零拷贝连接!
    }
    {
        T text;
        text.len = s1.len + s2.len;
        if (isatend(s1, s2.len)) {             // 优化4: s1在chunk末尾
            text.str = s1.str;
            memcpy(alloc(s2.len), s2.str, s2.len);  // 仅拷贝s2
        } else {
            char *p;
            text.str = p = alloc(s1.len + s2.len);
            memcpy(p,          s1.str, s1.len);
            memcpy(p + s1.len, s2.str, s2.len);
        }
        return text;
    }
}
```

四种情况:
1. `s1` 或 `s2` 为空 → 直接返回另一方(零拷贝)
2. `s2` 紧跟在 `s1` 的物理内存之后 → 扩展 `s1.len`(零拷贝)
3. `s1` 恰好位于 arena chunk 末尾 → 仅拷贝 `s2` 并附加到 chunk
4. 一般情况 → 分配新空间, 拷贝两方

`Text_dup` 也有优化: 当 `n == 1` 时直接返回原串:

```c
T Text_dup(T s, int n) {
    if (n == 0 || s.len == 0) return Text_null;
    if (n == 1) return s;           // 单份拷贝 → 零拷贝
    // ... 实际分配和拷贝 ...
}
```

---

## 15.7 与C标准库 `<string.h>` 的对比

### 15.7.1 <string.h> 的根本问题

| 函数 | 问题 |
|------|------|
| `strcpy(dst, src)` | 不检查 `dst` 大小; 若 `src` 长于 `dst` → 溢出 |
| `strcat(dst, src)` | 不检查 `dst` 剩余空间; 经典缓冲区溢出源头 |
| `strcmp(s1, s2)` | 必须扫描到 `\0`; 不能比较子串 |
| `strlen(s)` | O(n) 扫描; 无缓存机制 |
| `strtok(s, set)` | 修改原串(写入 `\0`); 不可重入(使用静态状态) |

`strncpy` 和 `strncat` 的所谓"安全"版本也存在陷阱:
- `strncpy(dst, src, n)` 不足够长时会**截断**但不添加 `\0`; 足够长时用 `\0` 填充剩余空间(性能浪费)
- `strncat(dst, src, n)` 的 `n` 不是目标缓冲区大小, 而是最多追加的字符数

### 15.7.2 Str接口的优势

| `<string.h>` 痛点 | Str 解决方案 |
|-------------------|-------------|
| 调用者分配结果 → 溢出风险 | **Str函数自行分配**, 保证空间足够 |
| 子串操作需手动指针运算 | **位置约定**, 统一子串表达 |
| 不能检查结果大小 | `convert` 中的 **assert 完整性检查** |
| 每次处理整串 | 可以通过 `i, j` 位置参数**限定操作范围** |
| 右端操作需要先计算长度 | **非正位置**无需知道长度即可访问尾部 |

### 15.7.3 Text接口的进一步优势

| 能力 | `<string.h>` | Str | Text |
|------|-------------|-----|------|
| O(1) 取长度 | 否 (O(n)) | 否 (O(n), `strlen`) | **是** (`Text_T.len`) |
| 嵌入 `\0` | 否 (以`\0`为终止符) | 否 | **是** |
| 零拷贝子串 | 否 | 否 (总是分配) | **是** (`Text_sub`) |
| 不可变性保证 | 否 | 否 | **是** (`const char *`) |
| 内存分配开销 | 调用者决定 | 每次操作都分配 | arena批量化, 按需分配 |

---

## 15.8 Modern-C改进

### 15.8.1 C11 Annex K: 边界检查接口

C11标准引入了 Annex K(又名 `__STDC_LIB_EXT1__`), 定义了一套带边界检查的"安全"字符串函数:

```c
// C11 Annex K 安全版本
errno_t strcpy_s(char *restrict s1, rsize_t s1max, const char *restrict s2);
errno_t strcat_s(char *restrict s1, rsize_t s1max, const char *restrict s2);
errno_t strncpy_s(char *restrict s1, rsize_t s1max, const char *restrict s2, rsize_t n);
errno_t strncat_s(char *restrict s1, rsize_t s1max, const char *restrict s2, rsize_t n);
int     strcmp_s(const char *restrict s1, rsize_t s1max, const char *restrict s2, int *indicator);
size_t  strnlen_s(const char *s, size_t maxsize);
```

**关键改进**:
- 每个目标缓冲区都要求传入 `s1max` —— 缓冲区的最大容量
- 溢出时调用**运行时约束处理函数**(默认行为可定制), 而不是静默破坏内存
- `rsize_t` 类型限制最大值为 `RSIZE_MAX`, 防止整数溢出

**Annex K 的争议**:
- 微软在 Visual Studio 中强制使用(将传统函数标记为deprecated)
- 但 Annex K 实际上只被微软完整实现; GCC 和 Clang 的实现不完整或有已知bug
- C11 将 Annex K 作为"可选"特性, 许多实现选择不支持
- 2019年, C标准委员会甚至考虑在 C2x 中废弃 Annex K
- 设计哲学争议: 运行时约束处理函数的设计是否合理? 调用者是否有机会纠正?

### 15.8.2 string_view 的概念

C11 Annex K 仍然保留了以 `\0` 为终止符的约定。更现代的方案来自 C++ 的 `std::string_view` 和 Rust 的 `&str`:

**核心思想**: 由一个 `(指针, 长度)` 对表示一个字符序列的**只读视图**, 视图本身不拥有数据。

这正是 Hanson 在1997年 Text 接口中已经实现了的设计:

```c
// Hanson (1997) Text_T
typedef struct T {
    int len;
    const char *str;
} T;
```

```cpp
// C++17 std::string_view (约2017)
class string_view {
    const char* data_;
    size_t size_;
};
```

```rust
// Rust &str (2010–2015期间设计)
// 内部表示为胖指针 (ptr, len)
// &str 是 &[u8] 的一个保证有效UTF-8的特殊形式
```

Hanson 比现代语言提前了近20年实现了这一设计, 展现了他在系统编程领域的深刻洞察力。

---

## 15.9 跨语言对照

### 15.9.1 数据模型对比

| 类型 | 语言 | 组成 | 所有权 | 可变性 | 终止符 |
|------|------|------|--------|--------|--------|
| `char *` | C | 指针 | 无 | 可 | `\0` |
| `Str` 位置约定 | C (本书) | 指针 + 位置 | 结果由函数分配 | 结果不可变 | `\0` |
| `Text_T` | C (本书) | `{len, const char*}` | 共享/arena | 不可变 | 无 |
| `std::string` | C++ | 指针 + size + capacity | 独占 | 可 | 无 |
| `std::string_view` | C++17 | 指针 + size | 无 (视图) | 不可变 | 无 |
| `&str` | Rust | 胖指针 (ptr + len) | 借用 (视图) | 不可变 | 无 |
| `String` | Rust | ptr + len + capacity | 独占 | 可 | 无 |

### 15.9.2 Text_T vs Rust &str

功能对应关系:

```
Text_sub    ≈  &s[i..j]           // 零拷贝子串视图
Text_len    ≈  s.len()            // O(1) 取长度
Text_cat    ≈  String::from(s1) + s2  // Text_cat有相邻优化
Text_box    ≈  &str 直接构造       // 从已有数据创建视图(不分配)
Text_put    ≈  String::from(s)     // 分配并拷贝
```

关键差异:
- Rust 通过**生命周期(lifetime)**和**借用检查器**在编译期保证 `&str` 不会在数据释放后被使用, 而 Text 依赖程序员的正确性保证
- Text 使用 arena 批量管理内存, Rust 使用所有权系统精确管理

### 15.9.3 Text_T vs C++ std::string_view

功能对应关系:

```
Text_sub        ≈  string_view::substr(pos, count)     // 零拷贝视图
Text_T{len,str} ≈  string_view(ptr, len)               // 构造语法
Text_pos        ≈  索引到位置的转换 (无直接对应, 方向相反)
Text_chr        ≈  string_view::find(c)
Text_find       ≈  string_view::find(s)
```

关键差异:
- `std::string_view` 有丰富的成员函数; Text_T 通过外部函数操作
- `std::string_view` 不在接口层面防止悬垂引用; Text通过arena管理生命周期
- Text 的位置约定(正/负位置双向定位)在设计上比 `string_view` 的纯索引 API 更灵活

### 15.9.4 Hanson 设计的先见之明

Hanson在1997年的Text接口中其实已经包含了现代字符串视图的所有关键概念:

| Hanson (1997) | 现代对应 |
|---------------|----------|
| 带长度的字符串表示 | Rust `&str`, C++ `string_view` |
| 不可变字符串语义 | Rust 默认不可变, C++ `const` |
| `Text_sub` 零拷贝 | Rust slicing, C++ `substr` |
| Arena 批量分配 | 反映了现代 arena allocator 思想 |
| `Text_cat` 相邻字符串优化 | 类似 Rust 某些库的 `Cow<str>` 优化 |

---

## 15.10 设计总结

Hanson 在本书中通过 **Str** 和 **Text** 两个接口完成了对C语言字符串处理的系统化改造:

### 第一层: Str —— 位置接口

- **解决的核心问题**: 安全性(自动分配空间) + 子串操作便利性(位置约定)
- **保留的C惯例**: 仍以 `\0` 结尾, 与现有C代码兼容
- **代价**: 每次操作调用 `strlen`(O(n)) 并分配新空间

### 第二层: Text —— 结构体接口

- **解决的核心问题**: O(1) 取长度 + 零拷贝子串 + 不可变性 + arena批量内存管理
- **代价**: 与 `<string.h>` 不兼容(不以 `\0` 结尾); 需要转换函数 `Text_get`/`Text_put`
- **预见性**: 本质上就是现代 `string_view` 的原型

### 三层架构总览

```
应用层:  位置约定 + Text_T 组合使用, 编写简洁安全的字符串处理代码
         ↑
第二层:  Text接口 (Text_T {len, const char*}) — 带长度, 零拷贝, 不可变
         ↑
第一层:  Str接口 (位置约定) — 子串安全操作, 自动分配
         ↑
底层:    C标准库 <string.h> — 原始操作, 不安全
```

这种分层设计体现了Hanson的"接口与实现分离"哲学: 每层解决特定问题, 上层依赖下层但不必受限于下层的设计缺陷。

### 关键教训

1. **位置优于索引**: 正/负双向定位消除了大量边界条件代码
2. **自动分配保证安全**: 谁生产结果谁负责分配空间, 这是避免缓冲区溢出最可靠的方法
3. **长度随数据携带**: `Text_T` 的 `{len, str}` 结构是字符串数据结构的基本形式, 现代语言无一例外采用了此设计
4. **零拷贝是可能的**: `Text_sub` 展示了即使在没有垃圾回收的C语言中, 通过 arena 和不可变性也可以实现零拷贝的字符串视图
5. **arena 管理临时对象**: 批量化内存管理在特定场景(编译器、解析器、格式化器)中性能远超逐对象 `malloc`/`free`

---

## 参考资料

- David R. Hanson, *C Interfaces and Implementations: Techniques for Creating Reusable Software*, Addison-Wesley, 1997. Chapter 15 (Low-Level Strings) and Chapter 16 (High-Level Strings).
- P. J. Plauger, *The Standard C Library*, Prentice Hall, 1992.
- Ralph E. Griswold and Madge T. Griswold, *The Icon Programming Language*, Peer-to-Peer Communications, 1990. — Str 接口的位置约定直接借鉴自 Icon 语言的字符串处理设施。
- ISO/IEC 9899:2011, Annex K: Bounds-checking interfaces.
- C++17 Standard, `std::string_view`.
- The Rust Programming Language, `&str` and `String` types.
---

<div style="display:flex; justify-content:space-between; padding:1em 0;">
<div><strong>← 上一章</strong><br><a href="ch14_格式化_Fmt.html">第14章 格式化 Fmt</a></div>
<div><strong><a href="index.html">📖 返回目录</a></strong></div>
<div style="text-align:right"><strong>下一章 →</strong><br><a href="ch16_高级字符串_Text.html">第16章 高级字符串 Text</a></div>
</div>
