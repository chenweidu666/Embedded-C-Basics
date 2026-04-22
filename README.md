# NPU 开发必备 C 语言基础 - 章节导览
<link rel="stylesheet" href="../assets/print-b5.css">
**专注于嵌入式开发和 NPU 寄存器编程所需的 C 语言核心知识。不覆盖完整的 C 语言教程，只提取实战中最关键的部分。**

---

## 📚 章节列表
| Chapter | Document | Content | Time |
|---------|----------|---------|------|
| **Chapter 1** | 01-data-types-and-memory-layout.md | 基础类型、struct/union、内存对齐与 Padding、位域 | 2-3 hours |
| **Chapter 2** | 02-pointers-and-memory.md | 指针进阶、函数指针、void* 泛型、malloc 陷阱 | 2-3 hours |
| **Chapter 3** | 03-bitwise-and-register-ops.md | 位运算、寄存器读写宏、volatile、内存屏障 | 3-4 hours |
| **Chapter 4** | 04-preprocessor-and-build.md | 预处理器技巧、条件编译、内联函数、编译警告 | 2-3 hours |
| **Chapter 5** | 05-strings-and-memory-ops.md | 字符串本质、strcpy 陷阱、缓冲区溢出、二进制序列化 | 2-3 hours |
| **Chapter 6** | 06-embedded-data-structures.md | 单向链表、侵入式双向链表、Ring Buffer、内存池 | 3-4 hours |
| **Chapter 7** | 07-sort-and-search.md | 冒泡/qsort、二分搜索、Top-K 提取、查表法 | 2-3 hours |
| **Chapter 8** | 08-file-io-and-serialization.md | 文件 I/O、二进制序列化、轻量配置解析、模型加载 | 2-3 hours |
| **Chapter 9** | 09-concurrency-and-multitasking.md | 线程同步、生产者-消费者、状态机、协程 | 3-4 hours |
| **Chapter 10** | 10-engineering-practices.md | 编码规范、错误处理、日志系统、单元测试、性能优化 | 3-4 hours |

**总计**: 10 章，24-31 小时完成

---

## 🔗 进阶学习
- **硬件理论**: 移步 `npu/basics/`
- **实操与工具**: 移步 `npu/debug/`
- **开发指南**: 移步 `npu/develop/`
- **Linux 嵌入式开发**: 移步 [Embedded-Linux-Handbook](https://github.com/chenweidu666/Embedded-Linux-Handbook) (交叉编译 → 内核 → 驱动)