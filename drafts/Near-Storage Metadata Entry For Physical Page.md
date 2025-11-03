# Near-Storage Metadata Entry For Physical Page

> **在无空闲链表、无全局位图的前提下，仅凭每个物理页（PP）的局部元数据，实现高效、正确、低开销的空闲页发现与内存分配。**

> 满足：
> - **自治性**：每个 PP 自包含足够信息，支持任意起点搜索；
> - **保守性**：绝不跳过 FREE 页（正确性优先）；
> - **紧凑性**：元数据 ≤ 8 字节/PP，适配近存计算（DPU/SmartSSD）；
> - **高效性**：查询 \( O(\log N) \)，更新 \( O(\log D + D) \)；
> - **无指针**：避免动态内存、缓存未命中、并发复杂性；
> - **硬件友好**：对齐、无分支、可向量化。

## 一、元数据页的保存位置与特征

### 1.1 邻近数据页的元数据布局

我们将所有数据页均分成若干等份，在每份开头的页面中直接保存元数据，按照元数据页内的偏移来直接索引它所管理的数据页号，节省了指针所占据的空间。具体来说，**一个连续的 2 GB 空间称为一个段落**，在这个段落的头 4 MB 是两页拼成的元数据页；元数据页内部是 (2^19 - 2^9 + 1) 条 NSME，每条 NSME 占据 8 字节，剩余空间即可灵活使用；元数据页之后还有 2 GB - 4 MB 空间，按 4 KB 为一页均分，它们就各自对应到了元数据页中的一条 NSME。

### 1.2 元数据页的硬件适应和扩展

由于存储硬件的规格中，通常使用 1000 进 1 而非 1024 进 1，且 2 × 2 MB 元数据页 + 2 GB 普通内存页也并不整，可以在元数据页中保留部分空间，使之达到整 2 GB 内恰好使用两页拼成的元数据页，剩余空间留作元数据页之间的某些跨段索引，以及更多的段落元信息（比如详细的硬件标识、某些用途的段落锁等）。

---

## 二、NSME 结构布局（8 字节 = 64 位）

```c
struct NSME {
    uint64_t state              : 6 ;  // 页状态
    uint64_t order              : 2 ;  // 巨页阶数（0=4K, 1=2M, 2=1G）
    uint64_t next_diftype_log2  : 6 ;  // 跳跃提示
    uint64_t jump_hint_prev     : 19;  // 分层链表前驱指针
    uint64_t jump_hint_next     : 19;  // 分层链表后继指针
    uint64_t ded                : 1 ;  // 奇偶校验码
    uint64_t zone_token         : 6 ;  // NUMA/区域标签
    uint64_t reserved           : 5 ;
};
```

| 字段 | 位宽 | 取值范围 | 说明 |
|------|------|--------|------|
| `state` | 6 | 0–63 | `FREE=0`, `LOCKED=1`, `CORRUPTED=2`, `RELIABLE=3`, `RESERVED=4`,… |
| `order` | 2 | 0–3 | 支持 4K/2M/1G 巨页 |
| **`next_free_log2`** | **6** | **0–63** | **核心：\( h = \lfloor \log_2(\text{next\_free} - i) \rfloor \)** |
| `jump_hint_prev` | 19 | 2^19 | 辅助结构，用于加速`next_free_log2`的更新 |
| `jump_hint_next` | 19 | 2^19 | 辅助结构，用于加速`next_free_log2`的更新 |
| `ded` | 1 | 0/1 | 奇偶校验码，供硬件使用 |
| `zone_hint` | 6 | 0–63 | 分区提示，用于优化访问模式 |
| `reserved` | 5 | — | 未来扩展（如安全标签、压缩状态、ECC 标识） |

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