# 第 10 章 - C 语言工程实践与代码质量
<link rel="stylesheet" href="../assets/print-b5.css">

## 📝 本章总结
本章讲解嵌入式 C 语言开发的工程实践：编码规范、错误处理模式、断言与防御性编程、轻量级日志系统实现、单元测试入门、内存泄漏检测，以及性能优化技巧。目标是写出**可维护、可调试、高性能**的生产级代码。

---

## 📖 本章内容
1. 编码规范与命名约定 (Google C Style / Linux Kernel Style)
2. 错误处理模式 (返回值 vs errno vs 错误码枚举)
3. 断言 (`assert`) 与防御性编程
4. 日志系统实现 (分级日志、环形日志 buffer、日志轮转)
5. 单元测试入门 (手写测试框架)
6. 内存泄漏检测 (Valgrind / AddressSanitizer)
7. 性能优化技巧与编译器优化标志
8. 代码审查 Checklist (针对嵌入式 C 代码)

---

## 1. 编码规范与命名约定

好的命名让代码自我解释。嵌入式项目推荐遵循 **Linux Kernel 风格** 或 **Google C Style**。

### 1.1 命名约定速查

| 元素 | 风格 | 示例 |
|------|------|------|
| 宏定义 | `UPPER_SNAKE_CASE` | `#define NPU_MAX_CHANNELS 8` |
| 类型 (typedef/struct/enum) | `PascalCase` 或 `SnakeCase_t` | `typedef struct { ... } NpuConfig_t;` |
| 函数 | `snake_case` | `int npu_init(void);` |
| 局部变量 | `snake_case` | `int frame_count = 0;` |
| 全局变量 | `g_snake_case` | `int g_debug_level = 0;` |
| 常量 | `kPascalCase` 或 `UPPER_SNAKE_CASE` | `const int kBufferSize = 1024;` |

### 1.2 头文件组织规范

```c
/* npu_driver.h */
#ifndef NPU_DRIVER_H  // 1. 头文件保护
#define NPU_DRIVER_H

// 2. 标准库头文件
#include <stdint.h>
#include <stddef.h>

// 3. 项目内部头文件
#include "npu_regs.h"
#include "npu_config.h"

// 4. 类型定义
typedef struct {
    uint32_t base_addr;
    int dma_channel;
} NpuDevice_t;

// 5. 函数声明 (公共 API)
int npu_init(NpuDevice_t *dev);
void npu_release(NpuDevice_t *dev);

// 6. 内联函数 (仅限短小函数)
static inline uint32_t npu_get_version(void) {
    return 0x0102; // v1.2
}

#endif // NPU_DRIVER_H
```

---

## 2. 错误处理模式

C 语言没有异常机制，错误处理必须通过返回值和状态码显式传递。

### 2.1 错误码枚举 (推荐)

```c
typedef enum {
    NPU_OK = 0,
    NPU_ERR_INVALID_ARG = -1,
    NPU_ERR_TIMEOUT = -2,
    NPU_ERR_MEMORY = -3,
    NPU_ERR_HARDWARE = -4,
    NPU_ERR_BUSY = -5,
} NpuStatus_t;

NpuStatus_t npu_load_model(const char *path) {
    if (!path) return NPU_ERR_INVALID_ARG;
    
    int fd = open(path, O_RDONLY);
    if (fd < 0) return NPU_ERR_INVALID_ARG;
    
    // ... 加载逻辑 ...
    
    if (load_failed) return NPU_ERR_HARDWARE;
    
    return NPU_OK;
}

// 使用
NpuStatus_t status = npu_load_model("/data/model.bin");
if (status != NPU_OK) {
    log_error("Model load failed: %s", npu_status_str(status));
}
```

### 2.2 错误处理辅助函数

```c
const char *npu_status_str(NpuStatus_t status) {
    switch (status) {
        case NPU_OK: return "Success";
        case NPU_ERR_INVALID_ARG: return "Invalid argument";
        case NPU_ERR_TIMEOUT: return "Operation timed out";
        case NPU_ERR_MEMORY: return "Memory allocation failed";
        case NPU_ERR_HARDWARE: return "Hardware error";
        case NPU_ERR_BUSY: return "Device busy";
        default: return "Unknown error";
    }
}
```

---

## 3. 断言 (`assert`) 与防御性编程

### 3.1 `assert` 的正确使用

`assert` 用于捕获**绝不应该发生**的内部逻辑错误，而不是处理用户输入错误。

```c
#include <assert.h>

void process_frame(uint8_t *data, size_t len) {
    // ✅ 检查前置条件 (编程错误)
    assert(data != NULL);
    assert(len > 0 && len <= MAX_FRAME_SIZE);
    
    // ❌ 不要用 assert 检查用户输入！
    // assert(user_input != NULL); // 发布版 NDEBUG 会禁用 assert，导致崩溃
    
    // ✅ 用户输入必须用 if 检查
    if (!data || len == 0) {
        return NPU_ERR_INVALID_ARG;
    }
}
```

### 3.2 编译期断言 (`static_assert`)

C11 引入的 `_Static_assert` 可以在编译期检查类型大小、宏定义等。

```c
// 确保结构体大小符合硬件要求
_Static_assert(sizeof(NpuCommand_t) == 16, "NpuCommand_t must be 16 bytes");

// 确保 DMA 缓冲区对齐
_Static_assert((DMA_BUFFER_SIZE % 4096) == 0, "DMA buffer must be 4K aligned");
```

---

## 4. 日志系统实现 (分级日志、环形日志 buffer)

嵌入式系统通常没有完整的 syslog 服务，需要手写轻量级日志模块。

### 4.1 分级日志宏

```c
typedef enum {
    LOG_DEBUG = 0,
    LOG_INFO,
    LOG_WARN,
    LOG_ERROR,
    LOG_FATAL
} LogLevel_t;

extern LogLevel_t g_log_level;

#define LOG(level, fmt, ...) do { \
    if (level >= g_log_level) { \
        printf("[%s][%s:%d] " fmt "\n", \
               log_level_str(level), __FILE__, __LINE__, ##__VA_ARGS__); \
    } \
} while(0)

// 便捷宏
#define log_debug(fmt, ...) LOG(LOG_DEBUG, fmt, ##__VA_ARGS__)
#define log_info(fmt, ...)  LOG(LOG_INFO, fmt, ##__VA_ARGS__)
#define log_warn(fmt, ...)  LOG(LOG_WARN, fmt, ##__VA_ARGS__)
#define log_error(fmt, ...) LOG(LOG_ERROR, fmt, ##__VA_ARGS__)
```

### 4.2 环形日志缓冲区 (掉电保护)

```c
#define LOG_BUF_SIZE (32 * 1024) // 32KB 循环缓冲区

static char log_ring_buf[LOG_BUF_SIZE];
static volatile uint32_t log_write_pos = 0;

void log_to_ring(const char *msg) {
    size_t len = strlen(msg);
    for (size_t i = 0; i < len; i++) {
        log_ring_buf[log_write_pos % LOG_BUF_SIZE] = msg[i];
        log_write_pos++;
    }
}

// 系统崩溃时打印最近 1KB 日志
void dump_recent_logs(void) {
    uint32_t start = (log_write_pos > 1024) ? log_write_pos - 1024 : 0;
    for (uint32_t i = start; i < log_write_pos; i++) {
        putchar(log_ring_buf[i % LOG_BUF_SIZE]);
    }
}
```

---

## 5. 单元测试入门 (手写测试框架)

不需要引入庞大的测试框架，嵌入式项目可以用极简的断言宏实现单元测试。

### 5.1 手写测试宏

```c
#include <stdio.h>
#include <stdlib.h>

#define TEST_SUITE(name) static int test_##name(void)
#define TEST_RUN(name) do { \
    printf("Running %-30s ... ", #name); \
    if (test_##name() == 0) printf("PASSED\n"); \
    else { printf("FAILED\n"); failed++; } \
} while(0)

#define ASSERT_EQ(a, b) do { \
    if ((a) != (b)) { \
        printf("ASSERT FAIL: %s != %s at %s:%d\n", #a, #b, __FILE__, __LINE__); \
        return 1; \
    } \
} while(0)

#define ASSERT_TRUE(cond) do { \
    if (!(cond)) { \
        printf("ASSERT FAIL: %s is false at %s:%d\n", #cond, __FILE__, __LINE__); \
        return 1; \
    } \
} while(0)
```

### 5.2 测试用例示例

```c
TEST_SUITE(ring_buffer_push_pop) {
    RingBuffer_t rb;
    ring_init(&rb);
    
    uint8_t data;
    ASSERT_EQ(ring_push(&rb, 42), 0);
    ASSERT_EQ(ring_pop(&rb, &data), 0);
    ASSERT_EQ(data, 42);
    
    // 测试空缓冲区
    ASSERT_EQ(ring_pop(&rb, &data), -1);
    
    return 0;
}

int main(void) {
    int failed = 0;
    TEST_RUN(ring_buffer_push_pop);
    // 添加更多测试...
    
    if (failed) {
        printf("\n%d test(s) failed!\n", failed);
        return 1;
    }
    printf("\nAll tests passed!\n");
    return 0;
}
```

---

## 6. 内存泄漏检测 (Valgrind / AddressSanitizer)

### 6.1 AddressSanitizer (ASan) — 现代编译器内置

```bash
# 编译时启用 ASan
gcc -fsanitize=address -g -o test_npu test_npu.c

# 运行，泄漏会自动报告
./test_npu
```

**ASan 输出示例：**
```
==1234==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 1024 byte(s) in 1 object(s) allocated from:
    #0 0x7f... in malloc
    #1 0x401... in npu_load_model src/npu.c:45
SUMMARY: AddressSanitizer: 1024 byte(s) leaked in 1 allocation(s).
```

### 6.2 Valgrind (传统方案)

```bash
valgrind --leak-check=full --show-leak-kinds=all ./test_npu
```

---

## 7. 性能优化技巧与编译器优化标志

### 7.1 GCC 优化级别

| 标志 | 说明 | 适用场景 |
|------|------|----------|
| `-O0` | 无优化，调试默认 | 开发调试 |
| `-O1` | 基础优化 | 平衡调试与性能 |
| `-O2` | 标准优化 (推荐生产) | 大多数嵌入式项目 |
| `-O3` | 激进优化 (可能增大体积) | 性能关键路径 |
| `-Os` | 优化体积 (减小 Flash 占用) | Flash 受限的 MCU |

### 7.2 常见优化技巧

```c
// 1. 循环展开 (手动或编译器自动 -funroll-loops)
// 原始
for (int i = 0; i < 1000; i++) sum += arr[i];
// 展开
for (int i = 0; i < 1000; i += 4) {
    sum += arr[i] + arr[i+1] + arr[i+2] + arr[i+3];
}

// 2. 局部性优化 (结构体按访问频率排列)
struct __attribute__((packed)) HotData {
    int frequently_accessed; // 放前面
    char rarely_accessed[100]; // 放后面
};

// 3. 避免频繁函数调用 (使用 inline)
static inline void clear_bit(volatile uint32_t *reg, int bit) {
    *reg &= ~(1U << bit);
}
```

---

## 8. 代码审查 Checklist (针对嵌入式 C 代码)

在提交代码前，对照此清单自查：

| 类别 | 检查项 |
|------|--------|
| **内存** | 所有 `malloc` 是否有对应的 `free`？是否有越界访问？ |
| **并发** | 共享变量是否用 Mutex/原子操作保护？是否有死锁风险？ |
| **错误处理** | 所有返回值是否被检查？错误路径是否正确清理资源？ |
| **类型安全** | 是否使用了 `stdint.h` 固定宽度类型？是否有符号/无符号混用？ |
| **可移植性** | 是否依赖特定平台字节序？结构体是否使用 `packed`？ |
| **性能** | 循环中是否有不必要的函数调用？是否可以使用查表法？ |
| **日志** | 关键路径是否有日志？日志级别是否合理？ |
| **注释** | 复杂逻辑是否有注释？注释是否解释了"为什么"而不是"做什么"？ |

---

## 🔧 实操练习

1. **实现完整的日志模块**: 集成第 4 节的分级日志和环形缓冲区，支持输出到串口和文件，并实现日志轮转（最多保留 3 个历史文件）。
2. **为 Ring Buffer 编写单元测试**: 使用第 5 节的测试框架，覆盖正常推入/弹出、满/空状态、边界条件。
3. **ASan 泄漏检测实战**: 故意在代码中制造 3 种不同类型的内存泄漏，使用 AddressSanitizer 检测并修复它们。

---

**最后更新**: 2026-04-22
**维护者**: 苏亚雷斯 (Suarez)