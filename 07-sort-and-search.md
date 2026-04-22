# 第 7 章 - 排序与搜索 (嵌入式实用版)
<link rel="stylesheet" href="../assets/print-b5.css">

## 📝 本章总结
本章摒弃了复杂的算法理论，专注于嵌入式场景下真正实用的排序与搜索技术：冒泡排序（小数据集最优）、快速排序（`qsort` 的自定义比较函数）、二分搜索（`bsearch`）、查表法（空间换时间），以及 NPU 推理结果 Top-K 提取。

---

## 📖 本章内容
1. 为什么嵌入式不需要复杂排序算法
2. 冒泡排序：数据量 < 50 时的最优解
3. 快速排序 (`qsort` 的使用与自定义比较函数)
4. 二分搜索 (`bsearch` + 手写实现)
5. NPU 场景：从大量推理结果中快速查找 Top-K
6. 查表法 (Lookup Table) — 用空间换时间的嵌入式智慧

---

## 1. 为什么嵌入式不需要复杂排序算法

在 PC 端开发中，我们习惯使用 `std::sort` 或 `Arrays.sort()`，底层是高度优化的混合算法。但在嵌入式/NPU 开发中，排序场景通常具有以下特点：

| 特征 | 说明 | 推荐方案 |
|------|------|----------|
| **数据量小** | 传感器数据、配置参数、推理结果通常 < 100 条 | 冒泡排序 / 插入排序 |
| **内存受限** | 无法承受递归带来的栈空间消耗 | 迭代实现 / 原地排序 |
| **实时性要求高** | 算法执行时间必须可预测（无最坏情况） | 避免 O(N²) 的极端退化 |
| **数据基本有序** | 新数据通常与旧数据位置接近 | 插入排序 (O(N) 最优) |

**核心原则**：**简单 > 复杂，可预测 > 平均最优**。

---

## 2. 冒泡排序：数据量 < 50 时的最优解

### 2.1 为什么冒泡排序在嵌入式中依然有用？

- **代码极简**：仅 5 行核心逻辑，易于理解和审计。
- **原地排序**：不需要额外内存分配。
- **稳定排序**：相等元素的相对顺序不变（对传感器时间戳排序很重要）。
- **小数据集性能最佳**：当 N < 20 时，冒泡排序的实际运行时间往往快于快速排序（无递归开销、无函数指针调用）。

### 2.2 优化版冒泡排序

```c
void bubble_sort(int *arr, int n) {
    for (int i = 0; i < n - 1; i++) {
        int swapped = 0; // 优化：如果一轮没交换，说明已有序
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = 1;
            }
        }
        if (!swapped) break; // 提前终止
    }
}
```

**性能分析：**
- 最好情况（已有序）：`O(N)`
- 最坏情况（逆序）：`O(N²)`
- 平均情况：`O(N²)`
- 空间复杂度：`O(1)`

---

## 3. 快速排序 (`qsort` 的使用与自定义比较函数)

当数据量较大（N > 100）且对性能有要求时，使用标准库的 `qsort`。

### 3.1 `qsort` 基础用法

```c
#include <stdlib.h>

int arr[] = {64, 34, 25, 12, 22, 11, 90};
int n = sizeof(arr) / sizeof(arr[0]);

// 比较函数：返回 <0, 0, >0
int compare_int(const void *a, const void *b) {
    int va = *(const int *)a;
    int vb = *(const int *)b;
    return (va > vb) - (va < vb); // 等价于 va - vb，但避免溢出
}

qsort(arr, n, sizeof(int), compare_int);
```

### 3.2 复杂结构体排序（NPU 推理结果按置信度降序）

```c
typedef struct {
    int class_id;
    float confidence;
    int x, y, w, h;
} Detection_t;

// 按 confidence 降序排序
int compare_detection(const void *a, const void *b) {
    const Detection_t *da = (const Detection_t *)a;
    const Detection_t *db = (const Detection_t *)b;
    
    if (da->confidence > db->confidence) return -1;
    if (da->confidence < db->confidence) return 1;
    return 0;
}

// 使用
Detection_t results[64];
int num_detections = 42;
qsort(results, num_detections, sizeof(Detection_t), compare_detection);
```

**注意**：`qsort` 是稳定排序吗？**不是**。如果需要稳定性，请使用归并排序或插入排序。

---

## 4. 二分搜索 (`bsearch` + 手写实现)

当数据**已排序**时，二分搜索可以将查找时间从 `O(N)` 降到 `O(log N)`。

### 4.1 `bsearch` 标准库用法

```c
// 必须在已排序的数组中查找
int key = 25;
int *found = (int *)bsearch(&key, arr, n, sizeof(int), compare_int);

if (found) {
    printf("Found %d at index %ld\n", *found, found - arr);
} else {
    printf("Not found\n");
}
```

### 4.2 手写二分搜索（更灵活）

```c
// 返回目标索引，未找到返回 -1
int binary_search(const int *arr, int n, int target) {
    int left = 0, right = n - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2; // 避免 (left+right) 溢出
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

**二分查找的边界条件：**
```mermaid
graph TD
    Start[开始查找] --> Check{left <= right?}
    Check -->|否| NotFound[返回 -1]
    Check -->|是| CalcMid[计算 mid]
    CalcMid --> Compare{arr[mid] == target?}
    Compare -->|是| Found[返回 mid]
    Compare -->|<| GoRight[left = mid + 1]
    Compare -->|>| GoLeft[right = mid - 1]
    GoRight --> Check
    GoLeft --> Check
```

---

## 5. NPU 场景：从大量推理结果中快速查找 Top-K

在目标检测任务中，NPU 可能输出几百个候选框，我们只需要置信度最高的 K 个（通常 K=5 或 10）。**不需要完整排序**，只需提取 Top-K。

### 5.1 最小堆法 (Min-Heap)

维护一个大小为 K 的最小堆，遍历所有结果，比堆顶大的就替换堆顶并重新调整。

```c
#define TOP_K 5

typedef struct {
    Detection_t heap[TOP_K];
    int size;
} TopKHeap_t;

// 向上调整 (插入后调用)
void heap_push(TopKHeap_t *h, Detection_t item) {
    if (h->size < TOP_K) {
        h->heap[h->size++] = item;
        // 上浮调整
        int i = h->size - 1;
        while (i > 0) {
            int parent = (i - 1) / 2;
            if (h->heap[i].confidence < h->heap[parent].confidence) break;
            Detection_t tmp = h->heap[i];
            h->heap[i] = h->heap[parent];
            h->heap[parent] = tmp;
            i = parent;
        }
    } else {
        // 堆满，替换堆顶并下沉
        if (item.confidence <= h->heap[0].confidence) return;
        h->heap[0] = item;
        heapify_down(h, 0);
    }
}

// 向下调整
void heapify_down(TopKHeap_t *h, int i) {
    while (1) {
        int smallest = i;
        int left = 2 * i + 1, right = 2 * i + 2;
        if (left < h->size && h->heap[left].confidence < h->heap[smallest].confidence)
            smallest = left;
        if (right < h->size && h->heap[right].confidence < h->heap[smallest].confidence)
            smallest = right;
        if (smallest == i) break;
        Detection_t tmp = h->heap[i];
        h->heap[i] = h->heap[smallest];
        h->heap[smallest] = tmp;
        i = smallest;
    }
}
```

### 5.2 简单遍历法 (K 很小时更优)

当 K ≤ 5 时，堆的维护开销反而不如直接遍历快：

```c
void get_top_k_simple(const Detection_t *results, int n, int k, Detection_t *top) {
    // 初始化 top 数组为极小值
    for (int i = 0; i < k; i++) top[i].confidence = -1.0f;
    
    for (int i = 0; i < n; i++) {
        // 找到当前最小的位置
        int min_idx = 0;
        for (int j = 1; j < k; j++) {
            if (top[j].confidence < top[min_idx].confidence) min_idx = j;
        }
        // 如果当前结果比最小的还大，替换
        if (results[i].confidence > top[min_idx].confidence) {
            top[min_idx] = results[i];
        }
    }
}
```

**性能对比：**
| N 大小 | K 大小 | 推荐方案 | 时间复杂度 |
|--------|--------|----------|------------|
| < 100 | 任意 | 完整排序 (`qsort`) | `O(N log N)` |
| 任意 | ≤ 5 | 简单遍历 | `O(N * K)` |
| > 1000 | ≤ 10 | 最小堆 | `O(N log K)` |

---

## 6. 查表法 (Lookup Table) — 用空间换时间的嵌入式智慧

在嵌入式系统中，**计算往往比内存访问更慢**。通过预计算结果并存储在表中，可以将 `O(1)` 的内存访问替代复杂的数学运算。

### 6.1 正弦/余弦查表

```c
// 预计算 0-360 度的正弦值 (Q15 定点数格式，避免浮点运算)
#define SINE_TABLE_SIZE 360
static const int16_t sine_table[SINE_TABLE_SIZE] = {
    0, 572, 1144, 1715, 2286, 2856, 3425, 3993, 4560, 5126,
    // ... 省略中间值 ...
    -5126, -4560, -3993, -3425, -2856, -2286, -1715, -1144, -572
};

// 查表获取 sin(θ) (θ 为 0-359 度)
static inline int16_t sin_lookup(int degrees) {
    if (degrees < 0) degrees = 360 + (degrees % 360);
    return sine_table[degrees % SINE_TABLE_SIZE];
}

// 对比：sin_lookup(45) 耗时 ~1 纳秒
//       sin(45 * PI / 180) 耗时 ~50 纳秒 (含浮点除法+三角函数)
```

### 6.2 Gamma 校正查表 (图像处理)

```c
// Gamma 2.2 校正查找表 (8-bit 图像)
static uint8_t gamma_table[256];

void init_gamma_table(void) {
    for (int i = 0; i < 256; i++) {
        gamma_table[i] = (uint8_t)(255.0f * powf(i / 255.0f, 1.0f / 2.2f));
    }
}

// 实时图像像素校正
uint8_t apply_gamma(uint8_t pixel) {
    return gamma_table[pixel]; // 直接查表，O(1)
}
```

### 6.3 量化查找表 (NPU 推理后处理)

NPU 通常输出 INT8 量化结果，需要反量化为 FLOAT：

```c
typedef struct {
    float scale;
    int32_t zero_point;
} QuantParam_t;

// 预计算反量化表 (256 个 INT8 值 → FLOAT)
static float dequant_table[256];

void init_dequant_table(QuantParam_t param) {
    for (int i = 0; i < 256; i++) {
        dequant_table[i] = (float)(i - param.zero_point) * param.scale;
    }
}

// 推理后快速反量化
float dequantize(int8_t qval) {
    return dequant_table[(uint8_t)qval]; // 无乘除，纯查表
}
```

---

## 🔧 实操练习

1. **实现 `qsort` 的比较函数**: 定义一个包含 `timestamp` 和 `sensor_value` 的结构体数组，按 `timestamp` 升序排序，如果 `timestamp` 相同则按 `sensor_value` 降序排序。
2. **手写二分查找变种**: 实现 `lower_bound` 函数，返回第一个大于等于目标值的元素索引（C++ `std::lower_bound` 的 C 语言实现）。
3. **查表法性能对比**: 编写程序对比 `sin()` 函数调用与查表法在 100 万次循环中的耗时差异。

---

## ❓ 常见排错 / 坑

| 现象 | 原因 | 解决方案 |
|------|------|----------|
| `qsort` 排序结果错误 | 比较函数返回值不符合 `<0, 0, >0` 约定 | 使用 `(a>b)-(a<b)` 模式，避免直接相减（溢出风险） |
| 二分查找死循环 | `mid` 计算错误或边界条件不匹配 | 始终使用 `left + (right-left)/2`，明确 `left<=right` 还是 `left<right` |
| 查表结果异常 | 索引越界或未初始化表 | 添加断言 `assert(idx < TABLE_SIZE)`，启动时初始化表 |
| Top-K 结果重复 | 未处理置信度完全相同的候选框 | 增加二级排序条件（如按 `class_id` 或 `box_area` 排序） |

---

**最后更新**: 2026-04-22
**维护者**: 苏亚雷斯 (Suarez)