# 第 4 章 - 预处理器与编译技巧

<link rel="stylesheet" href="../npu/assets/print-b5.css">

**掌握预处理器的高级用法、条件编译、内联函数和编译器警告。写出干净、可维护的嵌入式 C 代码。**

---

## 📖 本章内容

1. 宏定义的高级技巧
2. 条件编译
3. 内联函数 vs 宏
4. 编译器警告与静态检查
5. 多文件组织与头文件保护

---

## 1. 宏定义的高级技巧

### 1.1 字符串化与连接符

```c
// # 操作符：将参数转为字符串
#define STR(x)  #x
STR(hello)       // 展开为: "hello"

// ## 操作符：连接两个符号
#define CONCAT(a, b)  a##b
CONCAT(reg, _val)  // 展开为: reg_val

// 实用：自动生成寄存器名
#define NPU_REG(name)  (*(volatile uint32_t*)(NPU_BASE + NPU_##name##_OFF))
NPU_REG(CMD)  // 展开为: (*(volatile uint32_t*)(NPU_BASE + NPU_CMD_OFF))
```

### 1.2 do-while(0) 多行宏

```c
// ❌ 错误的多行宏
#define INIT_REG(addr, val) \
    *(volatile uint32_t*)(addr) = val; \
    log("initialized");

// 在 if 语句中会出错：
if (ready)
    INIT_REG(0x100, 0xFF);  // log() 无条件执行！

// ✅ 正确的写法
#define INIT_REG(addr, val) do { \
    *(volatile uint32_t*)(addr) = val; \
    log("initialized"); \
} while(0)
```

### 1.3 X-MACRO 技巧

当需要维护大量枚举和字符串映射时：

```c
// 定义列表
#define NPU_STATES(X) \
    X(IDLE,       0x00) \
    X(RUNNING,    0x01) \
    X(DONE,       0x02) \
    X(ERROR,      0xFF)

// 生成枚举
typedef enum {
    #define X(name, val) name = val,
    NPU_STATES(X)
    #undef X
} NpuState_t;

// 生成字符串数组
const char *state_names[] = {
    #define X(name, val) [name] = #name,
    NPU_STATES(X)
    #undef X
};

// 使用
printf("State: %s\n", state_names[current_state]);
```

---

## 2. 条件编译

### 2.1 基础语法

```c
#ifdef DEBUG
    printf("Debug: value = %d\n", x);
#endif

#if defined(HAS_DMA) && (DMA_CHANNELS > 4)
    // 只有有 DMA 且通道数 > 4 时才编译
#endif
```

### 2.2 平台适配

```c
#if defined(__arm__)
    #define MEMORY_BARRIER() __asm__ volatile("dsb" ::: "memory")
#elif defined(__x86_64__)
    #define MEMORY_BARRIER() __asm__ volatile("mfence" ::: "memory")
#elif defined(__riscv)
    #define MEMORY_BARRIER() __asm__ volatile("fence" ::: "memory")
#else
    #error "Unsupported architecture"
#endif
```

### 2.3 版本检查

```c
#if __STDC_VERSION__ >= 201112L
    // C11 特性可用
    #include <stdatomic.h>
    #define HAS_ATOMICS 1
#else
    #define HAS_ATOMICS 0
    #warning "C11 not available, atomics disabled"
#endif
```

---

## 3. 内联函数 vs 宏

### 3.1 优先用 inline 函数

| 特性 | 宏 | inline 函数 |
|------|-----|------------|
| 类型检查 | ❌ 无 | ✅ 有 |
| 调试支持 | ❌ 宏展开后难调试 | ✅ 正常调试 |
| 求值次数 | ⚠️ 参数可能被多次求值 | ✅ 只求值一次 |
| 性能 | 内联后等价 | 内联后等价 |

```c
// ❌ 宏的陷阱
#define MAX(a, b) ((a) > (b) ? (a) : (b))
int x = MAX(i++, j++);  // i 或 j 可能被 ++ 两次！

// ✅ inline 函数
static inline int max_int(int a, int b) {
    return (a > b) ? a : b;
}
int x = max_int(i++, j++);  // 安全
```

### 3.2 什么时候用宏？

| 场景 | 用宏 |
|------|------|
| 字符串化 `#` / 连接 `##` | ✅ 必须 |
| 编译期常量 | ✅ 必须 |
| 条件编译 | ✅ 必须 |
| 可变参数 (C99 前) | ✅ 必须 |
| 简单计算 | ❌ 用 inline |
| 寄存器映射 | ✅ 适合 (volatile 指针) |

---

## 4. 编译器警告与静态检查

### 4.1 推荐 GCC 警告标志

```bash
gcc -Wall -Wextra -Wpedantic \
    -Wstrict-prototypes -Wmissing-prototypes \
    -Wconversion -Wsign-compare \
    -Wshadow -Wuninitialized \
    -Werror  # 将警告视为错误 (生产环境推荐)
```

### 4.2 常见警告与修复

| 警告 | 原因 | 修复 |
|------|------|------|
| `unused variable` | 声明了但未使用 | 删除或加 `(void)var;` |
| `sign-compare` | 有符号 vs 无符号比较 | 统一类型或显式转换 |
| `implicit declaration` | 函数未声明 | 添加头文件或前置声明 |
| `format` | printf 格式不匹配 | 修正格式符 |
| `return type defaults to int` | C99 要求显式返回类型 | 写 `int main()` |

### 4.3 静态分析工具

```bash
# Cppcheck (安装: sudo apt install cppcheck)
cppcheck --enable=all --suppress=missingIncludeSystem *.c

# Clang Static Analyzer
scan-build gcc -c main.c
```

---

## 5. 多文件组织与头文件保护

### 5.1 头文件保护

```c
// npu_driver.h
#ifndef NPU_DRIVER_H   // 防止重复包含
#define NPU_DRIVER_H

// ... 声明 ...

#endif  // NPU_DRIVER_H
```

### 5.2 头文件应该放什么？

| 放在 `.h` | 放在 `.c` |
|-----------|----------|
| 类型定义 (struct, typedef) | 函数实现 |
| 函数声明 (原型) | 静态辅助函数 |
| 宏定义 | 静态全局变量 |
| 外部变量声明 (`extern`) | 内部常量 |

### 5.3 推荐目录结构

```
project/
├── include/
│   ├── npu_driver.h     # 公共 API
│   └── npu_regs.h       # 寄存器定义
├── src/
│   ├── npu_driver.c     # 驱动实现
│   ├── npu_dma.c        # DMA 模块
│   └── main.c           # 入口
└── Makefile
```

### 5.4 `extern` 的正确用法

```c
// config.h (声明)
extern uint32_t g_npu_base_addr;
extern int g_debug_level;

// config.c (定义)
uint32_t g_npu_base_addr = 0x40000000U;
int g_debug_level = 0;

// 其他文件
#include "config.h"
// 现在可以使用全局变量
```

**规则**：全局变量**只在一个 .c 文件中定义**，在 .h 中用 `extern` 声明。

---


---

## 📝 本章总结

本章讲解了预处理器指令（#define/#ifdef/#include）、宏函数、条件编译、编译流程和 Makefile 基础。

---

**最后更新**: 2026-04-21  
**维护者**: 苏亚雷斯 (Suarez)
