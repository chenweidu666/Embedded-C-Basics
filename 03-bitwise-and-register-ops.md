# 第 3 章 - 位运算与寄存器编程

<link rel="stylesheet" href="../npu/assets/print-b5.css">

**嵌入式开发的核心能力。理解如何用位运算操作寄存器、读写硬件配置，以及 volatile 的关键作用。**

---

## 📖 本章内容

1. 位运算基础
2. 寄存器读写宏
3. volatile 关键字
4. 内存屏障 (Memory Barrier)
5. 位运算实战技巧

---

## 1. 位运算基础

### 1.1 六大位运算符

| 运算符 | 名称 | 示例 | 结果 |
|--------|------|------|------|
| `&` | 按位与 | `0b1100 & 0b1010` | `0b1000` |
| `\|` | 按位或 | `0b1100 \| 0b1010` | `0b1110` |
| `^` | 按位异或 | `0b1100 ^ 0b1010` | `0b0110` |
| `~` | 按位取反 | `~0b00001111` | `0b11110000` |
| `<<` | 左移 | `0b0001 << 3` | `0b1000` |
| `>>` | 右移 | `0b1000 >> 2` | `0b0010` |

### 1.2 核心操作模板

```c
uint32_t reg = 0;  // 模拟寄存器

// ✅ 置位 (Set bits)
reg |= (1 << 5);           // 将第 5 位置 1
reg |= (0x3 << 2);         // 将 bit[3:2] 设为 0b11

// ✅ 清零 (Clear bits)
reg &= ~(1 << 5);          // 将第 5 位清 0
reg &= ~(0xFF << 8);       // 将 bit[15:8] 全部清 0

// ✅ 读取特定位 (Extract field)
uint32_t val = (reg >> 4) & 0xF;  // 读取 bit[7:4] 的 4 位值

// ✅ 修改特定位 (Modify field)
reg = (reg & ~(0x7 << 10)) | (0x5 << 10);  // bit[12:10] = 0b101
```

### 1.3 常用宏定义

```c
// 通用位操作宏
#define BIT(n)        (1U << (n))
#define MASK(width)   ((1U << (width)) - 1)
#define GET_FIELD(reg, shift, width)  (((reg) >> (shift)) & MASK(width))
#define SET_FIELD(reg, shift, width, val) \
    ((reg) = ((reg) & ~(MASK(width) << (shift))) | (((val) & MASK(width)) << (shift)))
```

---

## 2. 寄存器读写宏

### 2.1 MMIO (Memory-Mapped I/O)

硬件寄存器映射到内存地址空间，通过指针访问：

```c
// NPU 寄存器基址 (伪地址，非真实硬件)
#define NPU_BASE_ADDR  0x40000000U

// 寄存器偏移
#define NPU_CMD_OFFSET   0x00
#define NPU_STATUS_OFFSET 0x04
#define NPU_DATA_OFFSET  0x10

// 通过宏访问
#define NPU_CMD    (*(volatile uint32_t*)(NPU_BASE_ADDR + NPU_CMD_OFFSET))
#define NPU_STATUS (*(volatile uint32_t*)(NPU_BASE_ADDR + NPU_STATUS_OFFSET))

// 使用
NPU_CMD = 0x01;                    // 写入命令
while (!(NPU_STATUS & BIT(0)));    // 等待就绪
```

### 2.2 结构化寄存器映射

```c
typedef volatile struct {
    uint32_t CMD;       // 0x00: 命令寄存器
    uint32_t STATUS;    // 0x04: 状态寄存器
    uint32_t RESERVED[2];
    uint32_t DATA;      // 0x10: 数据寄存器
} NpuRegs_t;

#define NPU ((NpuRegs_t*)NPU_BASE_ADDR)

// 使用更直观
NPU->CMD = 0x01;
while (!(NPU->STATUS & BIT(0)));
```

---

## 3. volatile 关键字

### 3.1 为什么需要 volatile？

编译器会**优化掉看似无用的内存访问**：

```c
// ❌ 没有 volatile
uint32_t *status = (uint32_t*)0x40000004;
while (*status == 0);  // 编译器优化：只读一次，变成死循环！

// ✅ 有 volatile
volatile uint32_t *status = (volatile uint32_t*)0x40000004;
while (*status == 0);  // 每次都从硬件地址重新读取
```

### 3.2 volatile 的规则

| 场景 | 需要 volatile? | 原因 |
|------|---------------|------|
| 硬件寄存器 | ✅ 必须 | 值可能被硬件修改 |
| 多线程共享变量 | ✅ 必须 (但不充分) | 防止编译器优化掉读写 |
| 信号处理函数中的标志 | ✅ 必须 | 信号可能异步修改 |
| 普通局部变量 | ❌ 不需要 | 编译器优化有益 |

### 3.3 volatile 的常见误区

```c
// ❌ volatile 不保证原子性
volatile int counter = 0;
counter++;  // 实际上是 读-改-写 三步，可能被中断打断

// ✅ 需要额外的同步机制 (原子操作或锁)
// 或者使用硬件提供的原子指令
```

**volatile 只告诉编译器"不要优化"，不保证线程安全。**

---

## 4. 内存屏障 (Memory Barrier)

### 4.1 为什么需要内存屏障？

现代 CPU 会**乱序执行**指令，编译器也会重排内存访问。对于硬件交互，我们必须保证**顺序**：

```c
// ❌ 没有屏障，顺序可能被重排
dma->addr = buffer_addr;
dma->len = 256;
dma->start = 1;  // 如果 start 先于 addr 写入，DMA 会读到错误地址

// ✅ 用屏障保证顺序
dma->addr = buffer_addr;
__sync_synchronize();  // 内存屏障
dma->len = 256;
dma->start = 1;
```

### 4.2 GCC 内存屏障

```c
// 编译器屏障 (防止编译器重排，但不阻止 CPU 重排)
__asm__ volatile("" ::: "memory");

// 完整屏障 (GCC 内置函数，同时阻止编译器和 CPU 重排)
__sync_synchronize();

// C11 标准内存序
#include <stdatomic.h>
atomic_thread_fence(memory_order_seq_cst);  // 最强顺序保证
```

### 4.3 实用建议

| 场景 | 推荐方案 |
|------|---------|
| 单核 + volatile | 通常足够 |
| 多核共享数据 | `stdatomic.h` + memory_order |
| 硬件寄存器序列 | volatile + 编译器屏障 |
| DMA 启动前 | 完整内存屏障 |

---

## 5. 位运算实战技巧

### 5.1 判断奇偶

```c
// 传统
if (n % 2 == 0) { ... }

// 位运算 (更快)
if ((n & 1) == 0) { ... }
```

### 5.2 交换两个数 (无临时变量)

```c
a ^= b;
b ^= a;
a ^= b;
```

### 5.3 向下对齐到 2 的幂

```c
// 将 x 向下对齐到 64 字节
x = x & ~63U;  // 等价于 x = (x / 64) * 64;

// 通用宏
#define ALIGN_DOWN(x, align) ((x) & ~((align) - 1))
#define ALIGN_UP(x, align)   (((x) + (align) - 1) & ~((align) - 1))
```

### 5.4 统计 1 的个数 (Population Count)

```c
// 编译器内置 (最优)
int count = __builtin_popcount(value);

// 手动实现 (Brian Kernighan 算法)
int count = 0;
while (value) {
    value &= (value - 1);  // 清除最低位的 1
    count++;
}
```

### 5.5 掩码提取寄存器字段 (实战示例)

```c
// NPU 状态寄存器布局 (伪定义):
// bit[31:16] 任务 ID
// bit[15:8]   错误码
// bit[7:4]    当前层号
// bit[3:0]    状态标志

#define GET_TASK_ID(status)   (((status) >> 16) & 0xFFFF)
#define GET_ERROR_CODE(status) (((status) >> 8) & 0xFF)
#define GET_LAYER_NUM(status)  (((status) >> 4) & 0xF)
#define GET_STATUS_FLAG(status) ((status) & 0xF)
```

---


---

## 📝 本章总结

本章涵盖了位运算（AND/OR/XOR/NOT/移位）、寄存器位操作（置位/清零/翻转/提取）、volatile 关键字和内存映射 I/O。

---

**最后更新**: 2026-04-21  
**维护者**: 苏亚雷斯 (Suarez)
