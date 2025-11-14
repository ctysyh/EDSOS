# TS Formal Semantics and Verification

> TS 形式语义与验证基础

## 基础设定

### 前提约定

我们采用 **separation logic** 的标准记法：

- `P ∗ Q`：堆空间可分割为满足 `P` 和 `Q` 的两部分；
- `emp`：空堆；
- `↦`：点对点映射（如 `n ↦ (N, S, A, C, D, I, op)` 表示节点 `n` 的完整七元组内容）；
- `tree(n)`：表示以 `n` 为根的良构 TS 子树（递归谓词）；
- 所有操作均为 in-place 修改；
- 节点标识符 `n` 视为地址，可作为 heap location 使用；
- 假设存在一个全局 capability 环境 Γ，但此处暂不显式建模，仅关注结构与状态。

首先定义核心堆谓词：

```text
node(n, N, S, A, C, D, I, op) ≜ n ↦ (N, S, A, C, D, I, op)

tree(n) ≜ 
  ∃N,S,A,D,I,C,op. 
  A = A_prefix ∗ 
  node(n,N,S,A,C,D,I) ∗ 
    match C with
    | ∅ ⇒ emp
    | {c₁,…,cₖ} ⇒ 
        tree(c₁, [n] ++ A) ∗ … ∗ tree(cₖ, [n] ++ A)
```

| 分量 | 类型 | 精确定义 |
|------|------|--------|
| $\mathcal{N}$ | 标识符 | 全局唯一 |
| $\mathcal{S}$ | 状态 | ∈ {ready, running, blocked, zombie, error} |
| $\mathcal{A}$ | **有限序列** | 祖先链：$\mathcal{A} = [p_1, p_2, \dots, p_k]$，其中 $p_1$ 是直接父节点，$p_k$ 是根节点。满足栈序。 |
| $\mathcal{C}$ | **有限有根树**（rooted tree）或 **空** | 表示以 $n$ 为根的**整个后代子树**。若 $\mathcal{C} = \emptyset$，表示无子节点；否则 $\mathcal{C}$ 是一个递归树结构，其根节点集合即为 $n$ 的直接子节点，每个子节点自身也携带其 $\mathcal{C}'$。 |
| $\mathcal{D}$ | 映射 | 字段环境 |
| $\mathcal{I}$ | 指令序列 | 控制流，$\mathcal{I} = \mathtt{instruct} = [i_1, \dots, i_m]$ |
| $\mathtt{pc}$ | 自然数 | Program Counter，表示当前正在执行 i_pc |

> - $\mathcal{C}$ **不是集合**，而是**子树**（subtree）。
> - 这意味着 $\mathcal{C}$ 本身是一个**递归数据结构**，可形式化为：
> - $\mathcal{C} ::= \emptyset \mid \{ n_1, n_2, \dots, n_m \} \quad \text{where each } n_i \text{ is a Node with its own } n_i.\mathcal{C}$
> - 因此，整个 TS 结构 $\mathcal{T}$ 就是**单个根节点及其 $\mathcal{C}$**。

---

### 良构性条件

1. **祖先一致性（Ancestor Consistency）**  
   $ \forall n, \forall c \in \mathcal{C}^n: \mathcal{A}^c = [n] \mathbin{+\!+} \mathcal{A}^n $

2. **父子互反性（Parent-Child Reciprocity）**  
   $ \forall n, \forall c \in \mathcal{C}^n: n \in \mathcal{A}^c[0] \land n = \text{parent}(c) $

3. **无环性（Acyclicity）**  
   $ \neg \exists n_0, ..., n_k. n_0 = n_k \land \forall i < k, n_{i+1} \in \mathcal{C}^{n_i} $

4. **唯一父性（Unique Parent）**  
   $ \forall n, |\{ p \mid n \in \mathcal{C}^p \}| \leq 1 $

---

## TS 模型的形式化

### 操作的形式化（Separation Logic Style）

#### 1. Push(n, n\*)

> 在节点 `n` 下创建新子节点 `n*`。

**Rule**:
```text
{ node(n, N, S, A, C, D, I, op) ∗ n* ↦ _ }
Push(n, n*)
{ node(n, N, S, A, C ∪ {n*}, D, I, op) ∗ 
  node(n*, N*, ready, [n] ++ A, ∅, D*, I*, op) }
```

> 其中 `N*`, `D*`, `I*` 为输入参数。

---

#### 2. Pop(n, n\*)

> 删除 `n* ∈ DirectChildren(n)` 及其整个子树。

**Rule**:
```text
{ 
  ∃C'. 
    node(n, N, S, A, C' ∪ {n*}, D, I, op) ∗ 
    tree(n*) ∗ 
    trees(C')
}
Pop(n, n*)
{ node(n, N, S, A, C', D, I, op) ∗ trees(C') }
```

> 其中 trees(C') ≜ ∗_{c ∈ C'} tree(c)。

---

#### 3. Lift(n)

> 将 `n` 从父 `q = n.A[0]` 移至祖父 `p = n.A[1]`，并将 `q` 降为 `n` 的子节点。

**Rule**：
```text
{ 
  node(p, Nₚ, Sₚ, Aₚ, Cₚ, Dₚ, Iₚ, opₚ) ∗
  node(q, N_q, S_q, [p] ++ Aₚ, C_q, D_q, I_q, op_q) ∗
  node(n, Nₙ, Sₙ, [q, p] ++ Aₚ, Cₙ, Dₙ, Iₙ) ∗
  (tree(c₁) ∗ … ∗ tree(cₖ))      -- C_q = {n, c₁,…,cₖ}
  (tree(d₁) ∗ … ∗ tree(dₘ))      -- Cₙ = {d₁,…,dₘ}
}
Lift(n)
{
  node(p, Nₚ, Sₚ, Aₚ, (Cₚ \ {q}) ∪ {n}, Dₚ, Iₚ, opₚ) ∗
  node(n, Nₙ, Sₙ, [p] ++ Aₚ, Cₙ ∪ {q}, Dₙ, Iₙ, opₙ) ∗
  node(q, N_q, S_q, [n, p] ++ Aₚ, {c₁,…,cₖ}, D_q, I_q, op_q) ∗
  (tree(c₁) ∗ … ∗ tree(cₖ)) ∗
  (tree(d₁) ∗ … ∗ tree(dₘ))
}
```

> 其中 `trees(S) ≜ ∗_{x∈S} tree(x)`

---

#### 4. Merge(n, n\*)

> 合并 `n*` 到 `n`，提升其子节点。

**Rule**:
```text
{
  node(n, N, S, A, C, D, I, op) ∗
  node(n*, N*, S*, [n]++A, C*, D*, I*, op*) ∗
  trees(C*) ∧ n* ∈ C
}
Merge(n, n*)
{
  node(n, N, S, A, (C \ {n*}) ∪ C*, D ⊕ D*, I ⊕ I*, op ⊕ op*) ∗
  (∗_{c ∈ C*} tree(c, [n] ++ A))
}
```

> `⊕` 为 `Merge` 策略，内生保证节点内的正确性。

---

#### 5. Detach(n, F_D, F_I)

> 从 `n` 分离出新子节点 `n*`，携带 `F_D ⊆ D`, `F_I ⊆ I`。

**Rule**:
```text
{ node(n, N, S, A, C, D, I, op) ∗ n* ↦ _ }
Detach(n, F_D, F_I)
{ 
  node(n, N, S, A, C ∪ {n*}, D \ F_D, I \ F_I, op) ∗
  node(n*, N*, ready, [n]++A, ∅, F_D, F_I, op*)
}
```

> 这里 `op` “原样”指示下一条指令，并可以因此达到上界。

---

#### 6. Active(n)

**Rule**:
```text
{ node(n, N, S, A, C, D, I, op) ∧ S ≠ running }
Active(n)
{ node(n, N, ready, A, C, D, I, op) }
```

---

#### 7. Wait(n)

**Rule**:
```text
{ node(n, N, running, A, C, D, I, op) }
Wait(n)
{ node(n, N, blocked, A, C, D, I, op) }
```

---

#### 8. Yield(n)

**Rule**:
```text
{ node(n, N, running, A, C, D, I, op) }
Yield(n)
{ node(n, N, ready, A, C, D, I, op) }
```

---

#### 9. Finish(n)

**Rule**:
```text
{ node(n, N, S, A, C, D, I, op) }
Finish(n)
{ node(n, N, zombie, A, C, D, I, op) }
```

---

#### 10. Warn(n)

**Rule**:
```text
{ node(n, N, S, A, C, D, I, op) }
Warn(n)
{ node(n, N, error, A, C, D, I, op) }
```

---

### 执行的形式化

#### 11. `exec label env`

> 跳转至同节点内标签 `label` 对应的代码块（视为子过程）

$$
\frac{
\text{code}(label) \Downarrow_{\text{inner}} (\Delta\mathcal{D}, \Delta H)
}{
\langle \texttt{exec label}, n, H \rangle \Downarrow \langle n[\mathcal{D} \mapsto \mathcal{D} \oplus \Delta\mathcal{D}], H \oplus \Delta H \rangle
}
$$

---

#### 12. `cond f (t_act, f_act)`

> 若字段 $f$ 为真，则执行 $t\_act$（如 `exe T`），否则执行 $f\_act$。

$$
\frac{
f \in \mathcal{D}^n \lor (\exists a \in \mathcal{A}^n.\, f \in \mathcal{D}^a) \\
v = \text{read}(f) \\
\langle act, n, H \rangle \Downarrow \langle n', H' \rangle \quad \text{where } act = 
    \begin{cases}
    t\_act & \text{if } v \neq 0 \\
    f\_act & \text{otherwise}
    \end{cases}
}{
\langle \texttt{cond } f\ (t\_act, f\_act), n, H \rangle \Downarrow \langle n', H' \rangle
}
$$

---

#### 13. `cycl end_flag step_act`

> 重复执行 `step_act` 直到 `end_flag` 为真

$$
\frac{
v = \text{read}(end\_flag) \\
v \neq 0
}{
\langle \texttt{cycl } end\_flag\ step\_act, n, H \rangle \Downarrow \langle n, H \rangle
}
$$
$$
\frac{
v = \text{read}(end\_flag) = 0 \\
\langle step\_act, n, H \rangle \Downarrow \langle n_1, H_1 \rangle \\
\langle \texttt{cycl } end\_flag\ step\_act, n_1, H_1 \rangle \Downarrow \langle n', H' \rangle
}{
\langle \texttt{cycl } end\_flag\ step\_act, n, H \rangle \Downarrow \langle n', H' \rangle
}
$$

---

### 整个 instruct 段的大步语义

定义：
$$
\texttt{run\_instruct}(n, H) \Downarrow (n', H')
$$

规则：
- 若 $\mathcal{I} = []$，则 $\langle n, H \rangle \Downarrow \langle n[\mathsf{pc} \mapsto m+1], H \rangle$；
- 若 $\mathcal{I} = i :: rest$，
  $$
  \frac{
    \langle i, n, H \rangle \Downarrow \langle n_1, H_1 \rangle \\
    \texttt{run\_rest}(rest, n_1[\mathsf{pc} \mapsto 2], H_1) \Downarrow \langle n', H' \rangle
  }{
    \texttt{run\_instruct}([i]++rest, n, H) \Downarrow \langle n', H' \rangle
  }
  $$

> 保证了**线性顺序执行**，每条指令是原子黑盒。

---

## 完备性证明

### TS 模型的图灵完备性

> - 使用 $\mathcal{D} = (r_0, r_1, ..., r_k)$ 模拟寄存器；
> - 使用 `cycl` 实现循环，`cond` 实现条件跳转；
> - 使用 `exe` 实现标签跳转（如 `exe L5` 对应 goto L5）；
> - 程序计数器由 instruct 段的隐式 pc 模拟，或通过额外寄存器显式编码。
> 即使没有显式“goto 跨指令”，`exe` 允许在 **单条指令内部** 跳转到任意标签，而该标签可包含对寄存器的修改和条件判断，从而模拟任意控制流图。

---
