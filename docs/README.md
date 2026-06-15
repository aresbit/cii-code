# C Interfaces and Implementations — 中文翻译与代码详解

> 原书: _C Interfaces and Implementations: Techniques for Creating Reusable Software_
> 作者: David R. Hanson, Addison-Wesley, 1997
> 翻译日期: 2026-06-15
> 翻译方法: paper-agent 并行翻译 + frank 手工编写 + linter 自动扩充

## 目录

### 第一部分: 基础

| 章 | 标题 | 文件 | 大小 |
|----|------|------|------|
| 1 | 引言 | [ch01_引言_Introduction.md](ch01_引言_Introduction.md) | 26KB |
| 2 | 接口与实现 | [ch02_接口与实现_Interfaces.md](ch02_接口与实现_Interfaces.md) | 36KB |
| 3 | 原子 (Atoms) | [ch03_原子_Atoms.md](ch03_原子_Atoms.md) | 33KB |
| 4 | 异常与断言 (Exceptions) | [ch04_异常与断言_Exceptions.md](ch04_异常与断言_Exceptions.md) | 51KB |
| 5 | 内存管理 I (Memory) | [ch05_内存管理I_Memory.md](ch05_内存管理I_Memory.md) | 8KB |
| 6 | 内存管理 II (Arena) | [ch06_内存管理II_Arena.md](ch06_内存管理II_Arena.md) | 40KB |

### 第二部分: 容器与数据结构

| 章 | 标题 | 文件 | 大小 |
|----|------|------|------|
| 7 | 链表 (List) | [ch07_链表_List.md](ch07_链表_List.md) | 10KB |
| 8 | 表 (Table) | [ch08_表_Table.md](ch08_表_Table.md) | 41KB |
| 9 | 集合 (Set) | [ch09_集合_Set.md](ch09_集合_Set.md) | 8KB |
| 10 | 动态数组 (Array) | [ch10_动态数组_Array.md](ch10_动态数组_Array.md) | 26KB |
| 11 | 序列 (Seq) | [ch11_序列_Seq.md](ch11_序列_Seq.md) | 41KB |
| 12 | 环形缓冲区 (Ring) | [ch12_环形缓冲区_Ring.md](ch12_环形缓冲区_Ring.md) | 39KB |
| 13 | 位向量 (Bit) | [ch13_位向量_Bit.md](ch13_位向量_Bit.md) | 28KB |

### 第三部分: 字符串与格式化

| 章 | 标题 | 文件 | 大小 |
|----|------|------|------|
| 14 | 格式化 (Fmt) | [ch14_格式化_Fmt.md](ch14_格式化_Fmt.md) | 43KB |
| 15 | 低级字符串 (Str) | [ch15_低级字符串_Str.md](ch15_低级字符串_Str.md) | 42KB |
| 16 | 高级字符串 (Text) | [ch16_高级字符串_Text.md](ch16_高级字符串_Text.md) | 34KB |

### 第四部分: 多精度算术

| 章 | 标题 | 文件 | 大小 |
|----|------|------|------|
| 17 | 扩展精度 (XP) | [ch17_扩展精度算术_XP.md](ch17_扩展精度算术_XP.md) | 45KB |
| 18 | 任意精度 (MP) | [ch18_任意精度算术_MP.md](ch18_任意精度算术_MP.md) | 32KB |
| 19 | 多精度计算器 (Calc) | [ch19_多精度计算器_Calc.md](ch19_多精度计算器_Calc.md) | 33KB |

### 第五部分: 并发

| 章 | 标题 | 文件 | 大小 |
|----|------|------|------|
| 20 | 线程 (Thread) | [ch20_线程_Thread.md](ch20_线程_Thread.md) | 10KB |
| 21 | 通道 (Channel) | [ch21_通道_Channel.md](ch21_通道_Channel.md) | 39KB |

## 每章结构

每章遵循统一格式:

1. **设计哲学与接口概述** — 模块的设计意图, API一览
2. **数据结构深度解析** — struct定义, 内存布局, ASCII图解
3. **原始CII代码逐行中文注释** — 关键函数的完整注释
4. **Modern-C 改进方案** — C11/C17 特性对比
5. **关键算法与性能分析** — 时间复杂度, 空间复杂度
6. **完整使用示例** — 可运行的代码示例

## 统计

| 指标 | 数值 |
|------|------|
| 章节总数 | 21 |
| 总文件大小 | ~690KB |
| 最大章节 | Ch04 (51KB) |
| 最小章节 | Ch05 (8KB) |
| 源码引用 | code/*.h, code/*.c (50+ 文件) |

## 相关文件

- `../task_plan.md` — 翻译计划
- `../findings.md` — 翻译过程中的发现
- `../progress.md` — 进度追踪
- `../code/` — CII 原始源代码
- `../cii.pdf` — 原书PDF (扫描版)
