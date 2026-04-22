# 第 5 章 - 字符串与内存操作
<link rel="stylesheet" href="../assets/print-b5.css">

## 📝 本章总结
本章深入剖析 C 语言字符串的本质（NULL 终止符的内存布局）、`<string.h>` 核心函数的底层实现与性能差异、缓冲区溢出原理，以及嵌入式场景下的安全字符串处理与二进制数据序列化。

---

## 📖 本章内容
1. 字符串的本质：`char[]` vs `char*` 与内存布局
2. `<string.h>` 核心函数深度解析
3. 缓冲区溢出 (Buffer Overflow) 原理与防范
4. 嵌入式安全字符串处理 (`snprintf` vs `strlcpy`)
5. NPU 场景：配置参数的序列化与反序列化
6. 排错：越界、段错误、`strlen` 死循环

---

## 1. 字符串的本质：`char[]` vs `char*`

在 C 语言中，字符串本质上是一个以 `\0` (NULL) 结尾的 `char` 数组。理解它的内存布局是避免越界的关键。

### 1.1 字符串字面量 vs 字符数组

```c
// 方式 A：字符串字面量（存放在只读数据段 .rodata）
const char *msg = "Hello NPU"; 
// msg 是一个指针，指向 .rodata 中的地址
// *msg = 'X'; // ❌ 编译通过，运行时报错 (Segmentation Fault)

// 方式 B：字符数组（存放在栈上）
char buf[] = "Hello NPU";
// buf 是一个数组，内容被拷贝到栈空间
// buf[0] = 'X'; // ✅ 安全，修改栈上的副本
```

**内存布局示意：**
```mermaid
graph TD
    subgraph .rodata (只读段)
        A["H e l l o   N P U \\0"]
    end
    subgraph 栈空间 (Stack)
        B["H e l l o   N P U \\0"]
    end
    
    msg[指针] --> A
    buf[数组] -. 拷贝 .-> B
```

### 1.2 `sizeof` vs `strlen`

这是嵌入式开发中最容易混淆的两个概念：

| 函数/宏 | 行为 | 计算时机 | 示例结果 (`"Hello"`) |
|---------|------|----------|---------------------|
| `sizeof(str)` | 计算变量占用的总字节数 | 编译期 | 指针系统：`8` (64位) / 数组：`6` |
| `strlen(str)` | 计算到 `\0` 之前的字符数 | 运行期 | `5` |

**经典陷阱：**
```c
char str[10] = "Hello";
printf("sizeof=%zu, strlen=%zu\n", sizeof(str), strlen(str));
// 输出: sizeof=10, strlen=5
// sizeof 返回的是数组声明的大小 (10)，strlen 返回的是有效内容长度 (5)

void print_size(char arr[]) {
    printf("%zu\n", sizeof(arr)); // 输出 8！数组退化为指针
}
print_size(str);
```

---

## 2. `<string.h>` 核心函数深度解析

嵌入式开发中，90% 的内存崩溃都源于对 `<string.h>` 函数的误用。

### 2.1 `memcpy` vs `memmove`：重叠内存的陷阱

```c
char data[] = "ABCDEFGH";

// ✅ memcpy：源和目的不重叠时使用，性能最优
memcpy(data + 2, data, 3); // ❌ 未定义行为！重叠区域结果不确定

// ✅ memmove：安全处理重叠区域
memmove(data + 2, data, 3); // 结果: "ABABCDEF" (正确)
```

**底层实现逻辑：**
```mermaid
graph LR
    Check{内存重叠？}
    Check -->|否| Fast[memcpy: 逐块拷贝，SIMD优化]
    Check -->|是| CheckDir{目的 > 源？}
    CheckDir -->|是 (向后覆盖)| Backward[memmove: 从后向前拷贝]
    CheckDir -->|否 (向前覆盖)| Forward[memmove: 从前向后拷贝]
```

### 2.2 `strcpy` / `strncpy` / `strlcpy`：安全性的演进

```c
char dst[5];

// ❌ strcpy：不检查长度，极易溢出
strcpy(dst, "HelloWorld"); // 💥 栈溢出，程序崩溃

// ⚠️ strncpy：不保证 NULL 终止！
strncpy(dst, "HelloWorld", 5); // dst 变成 "Hello" (没有 \0！)
printf("%s\n", dst); // 可能打印出随机内存内容

// ✅ strlcpy (BSD/Linux)：始终添加 \0，返回原始长度
size_t len = strlcpy(dst, "HelloWorld", sizeof(dst));
// dst = "Hell\0", len = 10 (告诉你原始字符串有多长)
```

**嵌入式安全实践：**
```c
#define SAFE_COPY(dst, src) do { \
    strlcpy(dst, src, sizeof(dst)); \
    log_debug("Copied %zu/%zu bytes", strlen(dst), sizeof(dst)-1); \
} while(0)
```

---

## 3. 缓冲区溢出 (Buffer Overflow) 原理与防范

### 3.1 溢出是如何发生的？

```c
void parse_command(const char *input) {
    char cmd_buf[16];
    strcpy(cmd_buf, input); // 攻击者传入 100 字节的字符串
    // 覆盖返回地址 → 执行恶意代码 💥
}
```

**内存布局示意：**
```mermaid
graph TB
    subgraph 栈增长方向 (高地址 → 低地址)
        Ret[返回地址 (Return Address)]
        SavedBP[保存的基址指针]
        Buf1[cmd_buf[15]]
        Buf2[...]
        Buf3[cmd_buf[0]]
    end
    
    Input["输入 100 字节"] -. 溢出 .-> Buf3
    Input -. 覆盖 .-> SavedBP
    Input -. 劫持 .-> Ret
```

### 3.2 编译器防护机制

现代编译器提供了多层防护，但嵌入式环境可能默认未开启：

| 防护机制 | 编译标志 | 作用 | 性能开销 |
|----------|----------|------|----------|
| Stack Canary | `-fstack-protector` | 检测栈溢出 | ~1-3% |
| ASLR | 系统级 | 随机化内存布局 | 无 |
| NX Bit | 硬件/系统 | 栈不可执行 | 无 |
| FORTIFY_SOURCE | `-D_FORTIFY_SOURCE=2` | 替换不安全的 libc 函数 | ~0% |

**建议在嵌入式 Makefile 中至少启用：**
```makefile
CFLAGS += -fstack-protector -D_FORTIFY_SOURCE=2
```

---

## 4. 嵌入式安全字符串处理

### 4.1 `snprintf` 的正确用法

```c
char log_buf[128];

// ✅ 安全格式化
int written = snprintf(log_buf, sizeof(log_buf), 
                       "NPU Task %d: Status=%s", 
                       task_id, status_str);

// 检查是否截断
if (written >= sizeof(log_buf)) {
    log_warn("Log message truncated! (needed %d bytes)", written);
}
```

### 4.2 轻量级整数转字符串 (避免 printf 体积膨胀)

在资源受限的 MCU 中，`printf` 可能占用 10KB+ 的 Flash。手写转换更轻量：

```c
// 无符号整数转字符串 (小端序写入)
static inline int u32_to_str(uint32_t val, char *buf, int size) {
    char tmp[12];
    int i = 0;
    
    do {
        tmp[i++] = (val % 10) + '0';
        val /= 10;
    } while (val > 0 && i < sizeof(tmp));
    
    int len = i;
    if (len >= size) len = size - 1;
    
    // 反转字符串
    for (int j = 0; j < len; j++) {
        buf[j] = tmp[len - 1 - j];
    }
    buf[len] = '\0';
    return len;
}
```

---

## 5. NPU 场景：配置参数的序列化与反序列化

NPU 驱动通常需要接收用户态传来的配置参数。直接使用字符串解析效率低且不安全，推荐 **二进制序列化**。

### 5.1 定义通信协议

```c
// ⚠️ 必须使用 packed，防止编译器插入 padding
typedef struct __attribute__((packed)) {
    uint32_t magic;      // 0xDEADBEEF (协议头)
    uint16_t version;    // 协议版本
    uint16_t cmd_id;     // 命令 ID
    uint32_t task_id;    // 任务 ID
    uint32_t data_len;   // 后续数据长度
    uint8_t  payload[0]; // 柔性数组 (Flexible Array Member)
} NpuPacket_t;
```

### 5.2 序列化 (用户态 → 内核态)

```c
void build_packet(NpuPacket_t *pkt, uint32_t task_id, const void *data, size_t len) {
    pkt->magic    = 0xDEADBEEF;
    pkt->version  = 1;
    pkt->cmd_id   = 0x01;
    pkt->task_id  = task_id;
    pkt->data_len = len;
    
    // 拷贝 payload
    if (len > 0 && data) {
        memcpy(pkt->payload, data, len);
    }
}
```

### 5.3 反序列化 (内核态解析)

```c
int parse_packet(const uint8_t *raw, size_t raw_len) {
    if (raw_len < sizeof(NpuPacket_t)) {
        return -EINVAL; // 包太短
    }
    
    const NpuPacket_t *pkt = (const NpuPacket_t *)raw;
    
    // 1. 校验 Magic Number
    if (pkt->magic != 0xDEADBEEF) return -EPROTO;
    
    // 2. 校验数据长度 (防止越界读取)
    if (raw_len < sizeof(NpuPacket_t) + pkt->data_len) {
        return -EMSGSIZE;
    }
    
    // 3. 安全访问 payload
    process_npu_task(pkt->task_id, pkt->payload, pkt->data_len);
    return 0;
}
```

---

## 6. 排错：常见字符串/内存陷阱

### 6.1 `strlen` 死循环
**现象**: 程序卡死，CPU 100%。
**原因**: 传入的指针指向的内存中没有 `\0` 终止符。
```c
char buf[4] = {'A', 'B', 'C', 'D'}; // 没有 \0！
strlen(buf); // 💥 一直向后读，直到遇到随机 \0 或段错误
```
**修复**: 始终确保字符串以 `\0` 结尾，或使用 `memchr` 限定搜索范围。

### 6.2 `strtok` 的线程安全问题
**现象**: 多线程解析字符串时数据错乱。
**原因**: `strtok` 使用静态全局变量保存状态，不可重入。
```c
// ❌ 线程不安全
char *tok = strtok(input, ",");

// ✅ 使用 strtok_r (Linux) 或 strtok_s (Windows)
char *saveptr;
char *tok = strtok_r(input, ",", &saveptr);
```

### 6.3 格式化字符串漏洞
**现象**: 程序打印乱码或崩溃。
**原因**: `printf(user_input)` 中，用户输入包含 `%s` `%x` 等格式符。
```c
// ❌ 危险！
printf(user_string); 

// ✅ 始终使用固定格式字符串
printf("%s", user_string);
```

---

## 🔧 实操练习

1. **实现安全的 `strlcpy`**: 如果你的系统没有 `strlcpy`，请自行实现它，并编写测试用例覆盖：正常拷贝、截断、空字符串、源为空指针。
2. **二进制协议解析器**: 定义一个包含 3 个字段的 `packed` 结构体，将其序列化为字节流，再通过指针偏移逐个字段反序列化打印，验证字节序是否符合预期。

---

**最后更新**: 2026-04-22
**维护者**: 苏亚雷斯 (Suarez)