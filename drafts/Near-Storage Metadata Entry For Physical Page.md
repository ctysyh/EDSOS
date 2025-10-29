# Near-Storage Metadata Entry For Physical Page

> **在无空闲链表、无全局位图的前提下，仅凭每个物理页（PP）的局部元数据，实现高效、正确、低开销的空闲页发现与内存分配。**

> 满足：
> - **自治性**：每个 PP 自包含足够信息，支持任意起点搜索；
> - **保守性**：绝不跳过 FREE 页（正确性优先）；
> - **紧凑性**：元数据 ≤ 4 字节/PP，适配近存计算（DPU/SmartSSD）；
> - **高效性**：查询 \( O(\log N) \)，更新 \( O(\log D + D) \)；
> - **无指针**：避免动态内存、缓存未命中、并发复杂性；
> - **硬件友好**：对齐、无分支、可向量化。

## 一、元数据页的保存位置与特征

### 1.1 邻近数据页的元数据布局

我们将所有数据页均分成若干等份，在每份开头的页面中直接保存元数据，按照元数据页内的偏移来直接索引它所管理的数据页号，节省了指针所占据的空间。

### 1.2 元数据页的大小和内容

元数据页通常采用 2 MB 大页，**紧密连续排布**多个 NSME。每个 NSME 对应一个 4 KB 物理页帧。对于 4 字节 NSME，一个元数据页可包含 2^19 项，覆盖 2 GB 内存。

### 1.3 元数据页的硬件适应和扩展

由于存储硬件的规格中，通常使用 1000 进 1 而非 1024 进 1，且 2 MB 元数据页 + 2 GB 普通内存页也并不整，可以在元数据页中保留部分空间，使之达到整 GB 内（称为一个段落）恰好使用 1 个元数据页，剩余空间留作元数据页之间的某些跨段索引，以及更多的段落元信息（比如详细的硬件标识、某些用途的段落锁等）。

---

## 二、NSME 结构布局（4 字节 = 32 位）

```c
struct NSME {
    uint32_t state          : 4;  // 页状态
    uint32_t order          : 2;  // 巨页阶数（0=4K, 1=2M, 2=1G）
    uint32_t type           : 2;  // 页类型
    uint32_t next_free_log2 : 6;  // 跳跃提示
    uint32_t zone_hint      : 4;  // NUMA/区域提示
    uint32_t ecc_buffer     : 8;  // ECC 错误计数（老化预测）
    uint32_t reserved       : 6;  // 保留（未来扩展，如加密标签）
};
```

| 字段 | 位宽 | 取值范围 | 说明 |
|------|------|--------|------|
| `state` | 4 | 0–15 | `FREE=0`, `LOCKED=1`, `CORRUPTED=2`, `RELIABLE=3`, … |
| `order` | 2 | 0–3 | 支持 4K/2M/1G 巨页（3=保留） |
| `type` | 2 | 0–3 | `COMMON=0`, `METADATA=1`, `RESERVED=3` |
| **`next_free_log2`** | **6** | **0–63** | **核心：\( h = \lfloor \log_2(\text{next\_free} - i) \rfloor \)** |
| `zone_hint` | 4 | 0–15 | 物理区域 ID（用于 locality-aware 分配） |
| `ecc_buffer` | 8 | 0–255 | 保留用于 DPU 或其他硬件可实现自动 ECC |
| `reserved` | 6 | — | 未来扩展（如安全标签、压缩状态） |

---

## 三、核心机制：`next_free_log2`

### 3.1 语义定义
对 PP 索引 \( i \)（以 4KB 为单位）：
- 若 \( A[i] = \text{FREE} \)，则 \( \texttt{next\_free\_log2}[i] \) 可任取（不需要修改，因为看到FREE直接可用）；
- 若 \( A[i] \neq \text{FREE} \)，则必须满足 **保守性约束**：
  \[
  i + 2^{\texttt{next\_free\_log2}[i]} \leq \min \{ j \geq i \mid A[j] = \text{FREE} \}
  \]

### 3.2 查询算法（无回溯）
```c
uint64_t find_next_free(uint64_t i) {
    while (i < N && nsme[i].state != FREE) {
        i += (1ULL << nsme[i].next_free_log2);
    }
    return i;
}
```
- **正确性**：由保守性保证；
- **效率**：期望步数 \( O(\log d) \)，\( d = \text{next\_free}(i) - i \)。

### 3.3 更新策略（Lazy + 批量）
- **仅更新被分配/释放的区间** \([s, e]\)；
- **不更新前驱**（Lazy）：因保守性仍成立，最坏仅增加 1 跳；
- **高效赋值**：利用 \( h[i] = \lfloor \log_2(f - i) \rfloor \) 的单调性，分段批量写。

```c
void update_next_free_log2(uint32_t s, uint32_t e, uint32_t f) {
    for (uint32_t i = s; i <= e; ) {
        uint32_t h = 31 - __builtin_clz(f - i);
        uint32_t end = min(e + 1, f - (1U << h));
        // 循环展开：一次写 4 个
        for (; i + 3 < end; i += 4) {
            nsme[i].next_free_log2 = h;
            nsme[i+1].next_free_log2 = h;
            nsme[i+2].next_free_log2 = h;
            nsme[i+3].next_free_log2 = h;
        }
        while (i < end) nsme[i++].next_free_log2 = h;
    }
}
```

---