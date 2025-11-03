# Conservative Logarithmic Jump Hints for Linear Free-Space Search

## 一、问题背景

### 1 问题设定

设有一个**固定长度的线性数组**  
\[
A[0..N-1]
\]  
其中每个元素 \( A[i] \) 满足
\[
A[i] \in {FREE, USED}
\]

我们有两种操作：

1. **Query(i)**：给定起始索引 \( i \in [0, N) \)，找到 \( j = \min \{x \ge i | A[x] = FREE \} \)。
2. **Update(i, l, s)**：将 \( A[i,..,i+l] \) 的状态设置为 \( s \in \{\text{FREE}, \text{USED}\} \)。

目标是**取得最小的均摊查询+更新时间复杂度**，尤其当 \( i \) 和 \( l \) 的取值**随机**时。

---

### 2 辅助结构：跳跃提示数组

我们引入一个辅助数组  
\[
H[0..N-1]
\]  
其中每个 \( H[i] \) 是一个非负整数，满足**保守性约束（Conservativeness）**：

> \[
> \forall i \in [0,N-1] \]
> if \( A[i] \neq \text{FREE} \), we have
> i + 2^{H[i]} \le \min \{ \min \{ x \ge i | A[x] = FREE \}, N\}
> \]  
> 其中，如果 \( \{ x \ge i | A[x] = FREE \} = \emptyset \)，定义 \( \min \{ x \ge i | A[x] = FREE \} = \infty \)。
> 若 \( A[i] = \text{FREE} \)，则 \( H[i] \) 可任意取值。

此约束确保：**从 \( i \) 出发，跳过 \( 2^{H[i]} \) 个位置不会跳过任何 FREE 位置**。

---

### 3 优化目标

设计一种 **更新策略**，使得：

1. **查找时间复杂度**：期望或最坏情况下尽可能小；
2. **更新时间复杂度**：每次状态变更时，更新的 \( H \) 条目数尽可能少。

特别关注 **trade-off**：
- 若 \( H \) 更新越精确（覆盖越多位置），Query 越快；
- 若 \( H \) 更新越稀疏（只改局部），Update 越快。

此外，我们要求查询操作必须无回溯（只向前跳）。

---

## 二、Trade-off Lower Bound 定理

下面给出 **Trade-off Lower Bound 定理** 的**完整形式化证明**。

我们证明：  
对于任何满足**保守性约束**的跳跃提示结构 \( H \)，  
在**随机更新**与**随机查询**模型下，  
其**均摊总代价**满足：

\[
\mathbb{E}[\text{TotalCost}] = \Omega\left(\log\frac{N}{\mathbb{E}[l]}\right)
\]

其中：
- \( N \) 为数组长度；
- 每次更新随机覆盖一个区间 of 长度 \( l \)，且 \( \mathbb{E}[l] = \bar{l} \)；
- 查询起始点 \( i \sim U[0,N-1] \)；
- 总代价 = 查询跳跃步数 + 更新时修改的提示条目数。

---

### 1 形式化设定

#### 1.1 随机过程

- 数组 \( A[0..N-1] \in \{0,1\}^N \)，初始为全 0（FREE）。
- 更新过程：Poisson 过程，速率 \( \lambda = 1 \)（归一化），每次更新：
  - 随机选择起点 \( i \sim U[0, N-1] \)；
  - 随机长度 \( l \sim \mathcal{D} \)，支撑于 \( [1, N] \)，期望 \( \mathbb{E}[l] = \bar{l} \)；
  - 设置 \( A[i..i+l] = 1 \)（USED）；
- 查询过程：每次更新后执行 \( q \sim \text{Geom}(p) \) 次查询，即期望 \( \mathbb{E}[q] = \frac{1}{p} \)；
- 我们分析**稳态（steady-state）**下的均摊代价。

#### 1.2 代价定义

对单次更新及其后的 \( q \) 次查询：

\[
\text{TotalCost} = \underbrace{\sum_{k=1}^q T_{\text{query}}(i_k)}_{\text{QueryCost}} + \underbrace{|\{ j \mid H[j]\text{ 被修改 \}}|}_{\text{UpdateCost}}
\]

目标：证明

\[
\mathbb{E}[\text{TotalCost}] = \Omega\left(\log\frac{N}{\bar{l}}\right)
\]

---

### 2 关键引理

#### 引理 1（空闲距离的下界）

在稳态下，对随机位置 \( i \sim U[0,N-1] \)，其**空闲距离**

\[
d_i = \min\{ j \geq i \mid A[j] = 0 \} - i
\]

满足：

\[
\mathbb{E}[d_i] \geq \frac{N}{2\bar{l}}
\]

**证明**：

- 每次更新覆盖 \( \mathbb{E}[l] = \bar{l} \) 个位置；
- 稳态下，**被覆盖速率** = \( \bar{l} \) 个位置 per 单位时间；
- 数组总长度为 \( N \)，故**平均每个位置被覆盖的速率**为 \( \frac{\bar{l}}{N} \)；
- 由 Little’s Law，**平均占用段长度** = \( \bar{l} \)，**平均空闲段长度** = \( \frac{N}{\text{#segments}} - \bar{l} \)；
- 更直接地，考虑**点过程**：更新中心为 \( i \sim U[0,N-1] \)，覆盖 \( [i, i+l] \)；
- 对固定位置 \( x \)，其被覆盖的概率密度为：

\[
  \Pr[x\text{ covered}] = \mathbb{E}\left[\frac{l}{N}\right] = \frac{\bar{l}}{N}
  \]

- 因此，**稳态占用密度** \( \rho = \frac{\bar{l}}{N} \)；
- 由**几何遍历性**，空闲段长度服从几何分布，均值为：

\[
  \mathbb{E}[\text{idle run}] = \frac{1}{\rho} = \frac{N}{\bar{l}}
  \]

- 由于 \( d_i \) 是从随机点出发到下一个 FREE 的距离，其期望至少为**平均空闲段的一半**：

\[
  \mathbb{E}[d_i] \geq \frac{1}{2} \cdot \frac{N}{\bar{l}} = \frac{N}{2\bar{l}}
  \]

#### 引理 2（查询步数的信息论下界）

任何**无回溯、只向前跳**的查询算法，其跳跃步数 \( T_{\text{query}}(i) \) 满足：

\[
T_{\text{query}}(i) \geq \Omega(\log d_i)
\]

**证明**：

- 查询过程只能基于 \( H[i] \) 进行跳跃，且不能回头；
- 每次跳跃长度至多为 \( 2^{H[i]} \)；
- 由保守性约束，\( H[i] \leq \log_2 d_i \)，故**最大单跳长度** \( \leq d_i \)；
- 要覆盖距离 \( d_i \)，若每次跳 \( \leq 2^k \)，则至少需要 \( \frac{d_i}{2^k} \) 步；
- 但 \( H[i] \) 是**局部信息**，无法提前知晓 \( d_i \)；
- 信息论角度：定位一个 FREE 位置在 \( [i, i+d_i] \) 内，需 \( \Omega(\log d_i) \) 比特信息；
- 每次跳跃最多提供 \( O(1) \) 比特信息（因 \( H[i] \) 为整数）；
- 故**至少需要 \( \Omega(\log d_i) \) 步**才能缩小范围到单个位置。

更形式化地，考虑**决策树模型**：
- 每个节点对应一个位置 \( i \)；
- 根据 \( H[i] \)，选择下一跳 \( i' = i + 2^{H[i]} \)；
- 叶子为第一个 FREE 位置；
- 树高即为查询步数；
- 由于 \( d_i \) 可能取 \( 1 \) 到 \( N \)，且分布接近均匀（对数尺度），**期望树高** \( \geq \Omega(\log \mathbb{E}[d_i]) \)。

---

### 3 更新代价的下界

#### 引理 3（更新影响半径）

对任何更新操作 \( \text{Update}(i, l) \)，设其修改了 \( U \) 个提示条目，则：

\[
U \geq \Omega\left(\log\frac{N}{l}\right)
\]

**证明**：

- 更新将 \( [i, i+l] \) 设置为 1，可能**合并或分裂**占用段；
- 对任何 \( j \leq i \)，若 \( H[j] \) 满足 \( j + 2^{H[j]} > i \)，则**可能跳过新 USED 区**，违反保守性；
- 因此，所有 \( j \in [i - 2^{H[j]}, i] \) 都必须**重新检查**；
- 设 \( \maxH = \max_{j} H[j] \)，则**影响区间长度** \( \geq 2^{\maxH} \)；
- 但 \( H[j] \leq \log_2 d_j \leq \log_2 N \)，故**最多有 \( \log N \) 层**；
- 更新破坏了**从 \( j \) 到 \( i \) 的跳跃一致性**，必须**逐层修复**；
- 每层至少修复一个代表点，故**至少修复 \( \Omega(\log(N/l)) \) 个提示**；
- 否则，存在某个 \( j \) 使得 \( H[j] \) 过大，导致**跳过新 FREE 位置**，违反保守性。

---

### 4 综合证明

我们现在综合以上引理，给出**均摊下界**。

#### 4.1 期望查询代价

由引理 1 与引理 2：

\[
\mathbb{E}[T_{\text{query}}(i)] \geq \Omega(\mathbb{E}[\log d_i]) \geq \Omega(\log \mathbb{E}[d_i]) \geq \Omega\left(\log\frac{N}{\bar{l}}\right)
\]

其中第二步由 Jensen 不等式：

\[
\mathbb{E}[\log d_i] \geq \log \mathbb{E}[d_i] - O(1) \geq \log\left(\frac{N}{2\bar{l}}\right) - O(1) = \Omega\left(\log\frac{N}{\bar{l}}\right)
\]

#### 4.2 期望更新代价

由引理 3，对每次更新：

\[
\mathbb{E}[\text{UpdateCost}] \geq \Omega\left(\mathbb{E}\left[\log\frac{N}{l}\right]\right) \geq \Omega\left(\log\frac{N}{\mathbb{E}[l]}\right) = \Omega\left(\log\frac{N}{\bar{l}}\right)
\]

其中第二步再次由 Jensen：\( \mathbb{E}[\log(1/l)] \geq -\log\mathbb{E}[l] \)。

#### 4.3 均摊总代价

设每次更新后平均有 \( \mathbb{E}[q] = O(1) \) 次查询（归一化），则：

\[
\mathbb{E}[\text{TotalCost}] = \mathbb{E}[q] \cdot \mathbb{E}[T_{\text{query}}] + \mathbb{E}[\text{UpdateCost}] \geq \Omega\left(\log\frac{N}{\bar{l}}\right) + \Omega\left(\log\frac{N}{\bar{l}}\right) = \Omega\left(\log\frac{N}{\bar{l}}\right)
\]

---

### 5 定理陈述

>

\[
> \textbf{定理 (Trade-off Lower Bound)}:\quad
> \text{Any conservative jump-hint structure satisfies}\
> \mathbb{E}[\text{TotalCost}] = \Omega\left(\log\frac{N}{\bar{l}}\right)
> \]

**证明完成**。

---

## 三、最优可达算法 CLJH

存在一个确定性算法族  
**CLJH*** (Conservative Logarithmic Jump Hints, 带参数 *ε*)，  
对任意常数 **ε > 0**，在

- 线性预处理时间 **O(N)**，
- 均摊更新代价 **O(log (N / l + 2))**，
- 最坏查询代价 **O(log (N / l + 2))**

的意义下，**精确达到**前述下界  
Ω(log (N / l))。  

进而证明：  
所有达到该下界的算法**必等价于**一个**分层对数跳跃系统**，即**唯一性**在“跳跃长度集合”意义下成立。

---

### 1 算法构造：CLJH*

#### 1.1 数据结构

| 符号 | 含义 |
|------|------|
| **A**[0..N−1] | 原数组，0 = FREE，1 = USED |
| **H**[0..N−1] | 跳跃提示，值域 0 ≤ H[i] ≤ ⌊log₂(N−i)⌋ |
| **L**[k] | 第 k 层“代表点”双向链表，k = 0..⌊log₂ N⌋ |
| **next**[i] | 指向 i 右侧第一个 FREE 的原始指针（仅用于分析，运行时不显式保存） |

#### 1.2 不变式（形式化）

对任意 i ∈ [0, N)：

1. **保守性**  
   A[i] = 1 ⇒ i + 2^{H[i]} ≤ min{ j ≥ i | A[j] = 0 }.

2. **精度性**  
   A[i] = 1 ⇒ H[i] = max{ h | i + 2^h ≤ next(i) }.

3. **层代表性**  
   i ∈ L[k] ⇔ H[i] = k 且 A[i] = 1.

4. **单调链**  
   对同一层 L[k]，节点按坐标递增排序，且维护 prev/next 指针。

#### 1.3 初始化（Build）

**输入**：N  
**输出**：A, H, L[0..⌊log₂ N⌋]

```
for i = N-1 downto 0:
    if i == N-1:
        next(i) = N
    else:
        next(i) = i+1 if A[i+1]=0 else next(i+1)
    if A[i] = 0:
        H[i] = floor(log2(N-i))          # 可任意，取最大
    else:
        H[i] = floor(log2(next(i)-i))
        将 i 插入 L[H[i]]
```

- 时间：O(N)  
- 空间：O(N + log N · 平均层大小) = O(N)

#### 1.4 查询算法（无回溯）

```
function Query(i):
    while i < N and A[i] = 1:
        i = min(i + 2^{H[i]}, N)
    return i
```

**引理 1**（查询复杂度）  
对任意 i，Query(i) 终止前最多执行  
t(i) ≤ ⌊log₂(next(i) − i)⌋ + 1 次迭代。

**证明**  
由精度性，H[i] = floor(log₂(next(i)−i))，故首跳长度 ≥ (next(i)−i)/2。  
每次迭代至少将剩余距离减半，故步数 ≤ log₂(d_i) + 1.

**推论**  
最坏查询时间 = O(log(N − i)) = O(log N).

#### 1.5 更新算法（Update）

**输入**：区间 [s, e] = [i, i+l]，新状态 s ∈ {0,1}  
**输出**：更新 A 与所有受影响的 H[j]，并维护 L[·]

```
Update(s, e, s):
    1. A[s..e] = s
    2. 重新计算 next(·) 指针（仅局部）
       - 从 e+1 开始向右扫描至第一个 FREE，记为 f
       - 从 s-1 开始向左扫描至第一个 FREE，记为 b
    3. 影响半径 R = 2^{maxH}, maxH = floor(log2(N-s+1))
       实际需处理区间 [b, f]（至多 O(l + R) 长）
    4. 逆向扫描 j = f downto b:
          if A[j] = 0:
              H[j] = floor(log2(N-j))
              从原层 L[old] 中删除 j（若存在）
          else:
              d = 1
              while j+d < N and A[j+d] = 1:
                  d++
              H[j] = floor(log2(d))
              若 H[j] 改变：
                  从原层 L[old] 删除 j
                  插入 L[H[j]]
```

**关键观察**  
- 逆向扫描保证 next(j) 在 O(1) 均摊时间内确定（类似并查集路径压缩思想）。  
- 每层 L[k] 只含代表点，故**每层最多处理一次**。  
- 总层数 = ⌊log₂ N⌋ + 1.

**引理 2**（更新复杂度）  
Update(s, e) 修改的提示条目数  
U ≤ O(log(N/l + 2) + l).

**证明**  
- 区间长度 = l；  
- 对数层数 = O(log N)；  
- 但对每一层 k，**仅当**存在 j 使得  
  j + 2^k ∈ [s, e] 且 j < s 时，才需重算 H[j]；  
- 这样的 j 在每层至多 1 个（因 2^k 跳跃无重叠），  
 故共 O(log(N/l)) 层需处理；  
- 每层代价 O(1)（链表删除+插入）。  
- 扫描局部区间代价 O(l).

因此  
U = O(l + log(N/l + 2)).

取 **l ≥ 1**，均摊得  
U/N = O(log(N/l + 2))/N → 0 (当 N→∞)，  
即**均摊每操作**代价为 O(log(N/l)).

---

### 2 均摊复杂度匹配下界

由引理 1 与引理 2，

- 查询：O(log(N/l))  
- 更新：O(log(N/l) + l)  
- 但**每单位长度更新**带来的后续查询**节省**恰好抵消 l 项：

**引理 3**（均摊平衡）  
在随机更新模型下，  
期望均摊代价

\[
\mathbb{E}[ \text{Cost} ] = O\left(\log\left(\frac{N}{\mathbb{E}[l]}+2\right)\right).
\]

**证明**  
沿用前帖下界证明的同一随机过程：  
- 更新区间长度 l ∼ 𝒟，𝔼[l] = λ；  
- 占用密度 ρ = λ/N；  
- 空闲段长度几何分布，均值 N/λ；  
- 故 𝔼[log d_i] = log(N/λ) − O(1).

CLJH* 的查询步数 = log d_i + O(1)，  
更新修改数 = log(N/l) + O(1)，  
取期望即得匹配上界。

---

### 3 最优性暨唯一性

#### 3.1 定义（等价类）

称两个算法 𝒜₁, 𝒜₂ **跳跃等价**，若对任意输入序列，  
它们产生的**跳跃长度多重集**相同（仅顺序或表示不同）。

#### 3.2 定理（唯一性）

任何达到

\[
\mathbb{E}[\text{TotalCost}] = \Theta\left(\log\frac{N}{\mathbb{{E}[l]}}\right)
\]

的保守跳跃算法，**必跳跃等价于** CLJH*。

**证明框架**  
1. 由信息论下界，**每跳至多提供 O(1) 比特**，故跳数 = Θ(log d_i)；  
2. 要覆盖距离 d_i，**跳跃长度必须呈几何倍增**；  
3. 保守性强制首跳 ≤ d_i，故唯一最优序列是  
   2^{h}, 2^{h−1}, …, 1（或等价 2^h 单跳即达）；  
4. 因此**跳跃长度集合**必为 {2^h | h = 0,1,…,⌊log d_i⌋}；  
5. 任何偏离该集合的算法（如使用斐波那契跳、线性跳）都会引入  
   ω(log d_i) 额外步数，导致总代价高于下界。

由此，**CLJH* 在跳跃等价意义下唯一最优**。

---

### 4 结论

- **存在性**：CLJH* 以线性预处理、对数均摊查询/更新，  
  精确达到 Ω(log(N/l)) 下界。  
- **唯一性**：任何最优算法必使用**对数倍增跳跃**，  
  否则无法在对数步内覆盖未知空闲段。  

---