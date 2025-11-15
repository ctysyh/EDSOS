# AS Formal Semantics and Verification

> AS 形式语义与验证基础
> v 0.1.1

## 基础设定

### 前提约定

我们采用 **separation logic** 的标准记法：

- `P ∗ Q`：堆空间可分割为满足 `P` 和 `Q` 的两部分；
- `emp`：空堆；
- `↦`：点对点映射（如 `n ↦ (N, E, A, C, D, I, op)` 表示节点 `n` 的完整七元组内容）；
- `tree(n)`：表示以 `n` 为根的良构 AS 子树（递归谓词）；
- 所有操作均为 in-place 修改；
- 节点标识符 `n` 视为地址，可作为 heap location 使用；
- 假设存在一个全局 capability 环境 Γ，但此处暂不显式建模，仅关注结构与状态。

首先定义核心堆谓词：

```text
node(n, N, E, A, C, D, I, op) ≜ n ↦ (N, E, A, C, D, I, op)

tree(n) ≜ 
  ∃N,E,A,D,I,C,op. 
  A = A_prefix ∗ 
  node(n,N,E,A,C,D,I) ∗ 
    match C with
    | ∅ ⇒ emp
    | {c₁,…,cₖ} ⇒ 
        tree(c₁, [n] ++ A) ∗ … ∗ tree(cₖ, [n] ++ A)
```

| 分量 | 类型 | 精确定义 |
|------|------|--------|
| $\mathcal{N}$ | 标识符 | 全局唯一 |
| $\mathcal{E}$ | 状态 | ∈ {ready, running, blocked, zombie, error} |
| $\mathcal{A}$ | **有限序列** | 祖先链：$\mathcal{A} = [p_1, p_2, \dots, p_k]$，其中 $p_1$ 是直接父节点，$p_k$ 是根节点。满足栈序。 |
| $\mathcal{C}$ | **有限有根树**（rooted tree）或 **空** | 表示以 $n$ 为根的**整个后代子树**。若 $\mathcal{C} = \emptyset$，表示无子节点；否则 $\mathcal{C}$ 是一个递归树结构，其根节点集合即为 $n$ 的直接子节点，每个子节点自身也携带其 $\mathcal{C}'$。 |
| $\mathcal{D}$ | 映射 | 字段环境 |
| $\mathcal{I}$ | 指令序列 | 控制流，$\mathcal{I} = \mathtt{instruct} = [i_1, \dots, i_m]$ |
| $\mathtt{pc}$ | 自然数 | Program Counter，表示当前正在执行 i_pc |

> - $\mathcal{C}$ **不是集合**，而是**子树**（subtree）。
> - 这意味着 $\mathcal{C}$ 本身是一个**递归数据结构**，可形式化为：
> - $\mathcal{C} ::= \emptyset \mid \{ n_1, n_2, \dots, n_m \} \quad \text{where each } n_i \text{ is a Node with its own } n_i.\mathcal{C}$
> - 因此，整个 AS 结构 $\mathcal{T}$ 就是**单个根节点及其 $\mathcal{C}$**。

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

## AS 的形式化

### 模型的形式化

#### 1. Global State at Discrete Time $t$

The state of an Arbor Strux (AS) system at any discrete evolution step $t \in \mathbb{N}$ is defined as a quadruple:

$$
\Sigma_t = (\mathcal{T}_t,\ \mathcal{D}_t,\ \mathcal{B}_t,\ \mathcal{P}_t)
$$

where each component is formally specified as follows.

##### 1.1 Node Topology $\mathcal{T}_t$

The node topology at time $t$ is a rooted, directed tree:
$$
\mathcal{T}_t = (N_t, E_t)
$$
- $N_t$ is the finite set of live node instances at time $t$.
- $E_t \subseteq N_t \times N_t$ is the parent-child edge relation. For any $(p, c) \in E_t$, $p$ is the direct parent of $c$.
- There exists a unique root node $r \in N_t$ such that $\nexists x \in N_t$ with $(x, r) \in E_t$.
- For every node $n \in N_t$, there exists a unique path from the root to $n$, denoted $\text{Path}_t(r \leadsto n) = [n_0 = r, n_1, \dots, n_k = n]$.

> **Geometric Embedding Lemma**: For any node $n \in N_t$, its unique root-to-node path $\text{Path}_t(r \leadsto n)$ admits a lossless order-preserving embedding into the one-dimensional Euclidean space $\mathbb{R}$ via the depth mapping $\delta(n_i) = i$. Along this local path, the parent-child relation corresponds to adjacency on the integer lattice of $\mathbb{R}$.

##### 1.2 Field Environment $\mathcal{D}_t$

The field environment is a function assigning to each node a partial mapping from field names to values or binding references:
$$
\mathcal{D}_t : N_t \to (\mathcal{F} \rightharpoonup (\mathcal{V} \cup \{\texttt{Unbound}\}))
$$
For any node $n \in N_t$, $\mathcal{D}_t(n)$ is partitioned into three disjoint partial functions:
- $\mathcal{D}_t^{\text{priv}}(n) : \mathcal{F}_{\text{priv}} \rightharpoonup \mathcal{V}$
- $\mathcal{D}_t^{\text{publ}}(n) : \mathcal{F}_{\text{publ}} \rightharpoonup \mathcal{V}$
- $\mathcal{D}_t^{\text{ance}}(n) : \mathcal{F}_{\text{ance}} \rightharpoonup \texttt{BindingRef}$

The type of binding references is defined as:
$$
\texttt{BindingRef} ::= \texttt{Unbound} \mid \texttt{Bound}(m, f'), \quad \text{where } m \in N_t \text{ and } f' \in \mathcal{F}_{\text{publ}}
$$

##### 1.3 Binding Dependency Graph $\mathcal{B}_t$

To explicitly capture inter-node data dependencies, we define the binding graph at time $t$:
$$
\mathcal{B}_t = (N_t, \mathcal{E}_t^{\text{bind}})
$$
where the binding edge set $\mathcal{E}_t^{\text{bind}} \subseteq N_t \times N_t$ is given by:
$$
(n, m) \in \mathcal{E}_t^{\text{bind}} \iff \exists f \in \mathcal{F}_{\text{ance}},\ \mathcal{D}_t^{\text{ance}}(n)(f) = \texttt{Bound}(m, f')
$$

**Invariant**: $\mathcal{B}_t$ is a directed acyclic graph (DAG). This invariant is preserved by the operational semantics of structural operations (see Section 2).

##### 1.4 Global State Function $S_t$

The global state function maps a node-field pair to its resolved value at time $t$:
$$
S_t : N_t \times \mathcal{F} \to \mathcal{V} \cup \{\texttt{Unbound}\}
$$
It is defined case-wise as:
$$
S_t(n, f) =
\begin{cases}
\mathcal{D}_t^{\text{priv}}(n)(f) & \text{if } f \in \mathrm{dom}(\mathcal{D}_t^{\text{priv}}(n)) \\
\mathcal{D}_t^{\text{publ}}(n)(f) & \text{if } f \in \mathrm{dom}(\mathcal{D}_t^{\text{publ}}(n)) \\
\texttt{Resolve}_t(n, f) & \text{if } f \in \mathrm{dom}(\mathcal{D}_t^{\text{ance}}(n))
\end{cases}
$$
where $\texttt{Resolve}_t$ is the field resolution function defined in Section 2.3.

---

#### 2. Temporal Dynamics of `BindingRef` Events

The system evolves through discrete atomic events. We formalize the two primary events that manipulate `BindingRef`.

##### 2.1 `Push` Event

At time $t$, execution of  
`push child (ChildType () (parent_field => child_field) ...)`  
by node $n$ induces the transition $\Sigma_t \rightsquigarrow \Sigma_{t+1}$:

- **Topology Update**:  
  $N_{t+1} = N_t \cup \{c\}$,  
  $E_{t+1} = E_t \cup \{(n, c)\}$,  
  where $c$ is the newly created child node.

- **Field Environment Update**:  
  $\mathcal{D}_{t+1}(c)$ is initialized per the declaration of `ChildType`.  
  Crucially:  
  $$
  \mathcal{D}_{t+1}^{\text{ance}}(c)(\texttt{child\_field}) := \texttt{Bound}(n, \texttt{parent\_field})
  $$

- **Binding Graph Update**:  
  $\mathcal{E}_{t+1}^{\text{bind}} = \mathcal{E}_t^{\text{bind}} \cup \{(c, n)\}$

This operation establishes an initial binding from child to parent, creating a reverse edge relative to the tree structure.

##### 2.2 `Lift` Event

At time $t$, execution of  
`lift data_source ((source_field => local_ance_field))`  
by node $n$ is only valid if the following preconditions hold:
1. $\texttt{data\_source} \in \mathrm{Children}_t(n)$
2. $\texttt{source\_field} \in \mathrm{dom}(\mathcal{D}_t^{\text{publ}}(\texttt{data\_source}))$
3. $\texttt{local\_ance\_field} \in \mathrm{dom}(\mathcal{D}_t^{\text{ance}}(n))$

Upon validation, the transition $\Sigma_t \rightsquigarrow \Sigma_{t+1}$ is defined by:

- **Field Environment Update**:  
  $$
  \mathcal{D}_{t+1}^{\text{ance}}(n)(\texttt{local\_ance\_field}) := \texttt{Bound}(\texttt{data\_source}, \texttt{source\_field})
  $$

- **Binding Graph Update**:  
  The old binding edge from $n$ (if any) is replaced:  
  $$
  \mathcal{E}_{t+1}^{\text{bind}} = (\mathcal{E}_t^{\text{bind}} \setminus \{(n, \cdot)\}) \cup \{(n, \texttt{data\_source})\}
  $$

This event fulfills a binding promise by anchoring an `ance` field directly to a physical `publ` field.

##### 2.3 Dynamic Semantics of `Resolve`

The field resolution function at time $t$ is defined recursively over $\mathcal{B}_t$:
$$
\texttt{Resolve}_t(n, f) =
\begin{cases}
\bot & \text{if } \mathcal{D}_t^{\text{ance}}(n)(f) = \texttt{Unbound} \\
S_t(m, f') & \text{if } \mathcal{D}_t^{\text{ance}}(n)(f) = \texttt{Bound}(m, f')
\end{cases}
$$
Due to the DAG invariant of $\mathcal{B}_t$, this recursion is guaranteed to terminate. Upon termination, $\texttt{Resolve}_t(n, f)$ yields a value $v \in \mathcal{V}$ from a unique physical source $(m, f_{\text{publ}})$.

---

#### 3. Derived Properties

From the above model, the following key properties follow directly:

1. **Termination**: Guaranteed by the acyclicity of $\mathcal{B}_t$.
2. **Unique Physical Source**: Every successful resolution terminates at a `publ` field of some node $m$, which serves as the sole physical storage location.
3. **Structured Sharing with Multi-Source Confluence**: Multiple `ance` fields from distinct nodes may resolve to the same physical source $(m, f_{\text{publ}})$. Along each node’s root-path (embeddable in $\mathbb{R}$), this forms a clear “line of sight” to the shared data, enabling zero-copy access with explicit, analyzable dependency paths.

---

### 操作的形式化（Separation Logic Style）

#### 1. Push(n, n\*)

> 在节点 `n` 下创建新子节点 `n*`。

**Rule**:
```text
{ node(n, N, E, A, C, D, I, op) ∗ n* ↦ _ }
Push(n, n*)
{ node(n, N, E, A, C ∪ {n*}, D, I, op) ∗ 
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
    node(n, N, E, A, C' ∪ {n*}, D, I, op) ∗ 
    tree(n*) ∗ 
    trees(C')
}
Pop(n, n*)
{ node(n, N, E, A, C', D, I, op) ∗ trees(C') }
```

> 其中 trees(C') ≜ ∗_{c ∈ C'} tree(c)。

---

#### 3. Lift(n)

> 将 `n` 从父 `q = n.A[0]` 移至祖父 `p = n.A[1]`，并将 `q` 降为 `n` 的子节点。

**Rule**：
```text
{ 
  node(p, Nₚ, Eₚ, Aₚ, Cₚ, Dₚ, Iₚ, opₚ) ∗
  node(q, N_q, E_q, [p] ++ Aₚ, C_q, D_q, I_q, op_q) ∗
  node(n, Nₙ, Eₙ, [q, p] ++ Aₚ, Cₙ, Dₙ, Iₙ) ∗
  (tree(c₁) ∗ … ∗ tree(cₖ))      -- C_q = {n, c₁,…,cₖ}
  (tree(d₁) ∗ … ∗ tree(dₘ))      -- Cₙ = {d₁,…,dₘ}
}
Lift(n)
{
  node(p, Nₚ, Eₚ, Aₚ, (Cₚ \ {q}) ∪ {n}, Dₚ, Iₚ, opₚ) ∗
  node(n, Nₙ, Eₙ, [p] ++ Aₚ, Cₙ ∪ {q}, Dₙ, Iₙ, opₙ) ∗
  node(q, N_q, E_q, [n, p] ++ Aₚ, {c₁,…,cₖ}, D_q, I_q, op_q) ∗
  (tree(c₁) ∗ … ∗ tree(cₖ)) ∗
  (tree(d₁) ∗ … ∗ tree(dₘ))
}
```

> 其中 `trees(E) ≜ ∗_{x∈E} tree(x)`

---

#### 4. Merge(n, n\*)

> 合并 `n*` 到 `n`，提升其子节点。

**Rule**:
```text
{
  node(n, N, E, A, C, D, I, op) ∗
  node(n*, N*, E*, [n]++A, C*, D*, I*, op*) ∗
  trees(C*) ∧ n* ∈ C
}
Merge(n, n*)
{
  node(n, N, E, A, (C \ {n*}) ∪ C*, D ⊕ D*, I ⊕ I*, op ⊕ op*) ∗
  (∗_{c ∈ C*} tree(c, [n] ++ A))
}
```

> `⊕` 为 `Merge` 策略，内生保证节点内的正确性。

---

#### 5. Detach(n, F_D, F_I)

> 从 `n` 分离出新子节点 `n*`，携带 `F_D ⊆ D`, `F_I ⊆ I`。

**Rule**:
```text
{ node(n, N, E, A, C, D, I, op) ∗ n* ↦ _ }
Detach(n, F_D, F_I)
{ 
  node(n, N, E, A, C ∪ {n*}, D \ F_D, I \ F_I, op) ∗
  node(n*, N*, ready, [n]++A, ∅, F_D, F_I, op*)
}
```

> 这里 `op` “原样”指示下一条指令，并可以因此达到上界。

---

#### 6. Active(n)

**Rule**:
```text
{ node(n, N, E, A, C, D, I, op) ∧ E ≠ running }
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
{ node(n, N, E, A, C, D, I, op) }
Finish(n)
{ node(n, N, zombie, A, C, D, I, op) }
```

---

#### 10. Warn(n)

**Rule**:
```text
{ node(n, N, E, A, C, D, I, op) }
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

### AS 模型的图灵完备性

> - 使用 $\mathcal{D} = (r_0, r_1, ..., r_k)$ 模拟寄存器；
> - 使用 `cycl` 实现循环，`cond` 实现条件跳转；
> - 使用 `exe` 实现标签跳转（如 `exe L5` 对应 goto L5）；
> - 程序计数器由 instruct 段的隐式 pc 模拟，或通过额外寄存器显式编码。
> 即使没有显式“goto 跨指令”，`exe` 允许在 **单条指令内部** 跳转到任意标签，而该标签可包含对寄存器的修改和条件判断，从而模拟任意控制流图。

---
