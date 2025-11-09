# TS 与 Node 的形式化叙述

> Tree-Stacked 是树性和栈序的统一：
> - 从执行视角（自底向上）：控制流是**栈式的**——函数调用压栈，返回弹栈。
> - 从结构视角（自顶向下）：并行任务是**树状的**——父任务派生子任务，形成分叉。

---

## 一、节点七元组定义

一个 **Node** 是七元组：
$$
n = (\mathcal{N}, \mathcal{S}, \mathcal{A}, \mathcal{C}, \mathcal{D}, \mathcal{I}, \mathtt{pc})
$$
其中分量含义如下：

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

## 二、TS 结构的精确定义（基于子树）

一个 **Tree-Stacked 结构** $\mathcal{T}$ 是一个**单节点** $r$（根），其 $r.\mathcal{A} = []$，且其 $r.\mathcal{C}$ 是一个**良构子树**，满足：

### 良构性条件（Well-formedness, WF）：
对任意节点 \(n \in \mathcal{T}\)（包括根及其所有后代）：
1. **祖先一致性**：若 $n.\mathcal{A} = [p_1, p_2, \dots, p_k]$，则：
   - $p_1$ 是 $n$ 的直接父节点；
   - $p_{i+1}$ 是 $p_i$ 的直接父节点；
   - $p_k$ 是根节点（$p_k.\mathcal{A} = []$）。
2. **父子互反性（基于子树）**：
   $$
   m \in \text{DirectChildren}(n) \iff n = m[0].\mathcal{A}
   $$
   其中 $\text{DirectChildren}(n)$ 是 $n.\mathcal{C}$ 的根节点集合（即第一层子节点）。
3. **无环性**：不存在节点 $n$ 使得 $n \in \text{Descendants}(n)$。
4. **唯一父性**：每个非根节点有且仅有一个直接父节点（由 $\mathcal{A}[0]$ 唯一确定）。

---

## 三、基本操作的形式化

### 1. Push(n, n*)

在节点 $n$ 下创建新子节点 $n^*$，作为其直接子节点。

$$
\begin{aligned}
\operatorname{Push}(n, n^*) = 
&\text{ let } \mathcal{A}^* := [n] \frown n.\mathcal{A} \\
&\text{ let } n^*_{\text{new}} := (\mathcal{N}^*, \texttt{ready}, \mathcal{A}^*, \emptyset, \mathcal{D}^*, \mathcal{I}^*) \\
&\quad n.\mathcal{C} := n.\mathcal{C} \cup \{n^*_{\text{new}}\} \quad \text{(as new root of subtree)}
\end{aligned}
$$

> $n.\mathcal{C}'$ 是原 $n.\mathcal{C}$ 与新节点组成的**新子树**（直接子节点集合扩展）。

---

### 2. Pop(n, n*)

删除 $n^* \in \text{DirectChildren}(n)$ 及其**整个子树**（即丢弃 $\mathcal{C}^*$）。

$$
\begin{aligned}
\operatorname{Pop}(n, n^*) = 
&\text{ require } n^* \in \text{DirectChildren}(n) \\
&\text{ // discard entire subtree rooted at } n^* \\
&\quad n.\mathcal{C} := n.\mathcal{C} \setminus \{n^*\}
\end{aligned}
$$

---

### 3. Lift(n)

将节点 \(n\) 从其当前父节点移除，并挂载到其祖父节点 $n.\mathcal{A}[1]$ 下。

$$
\begin{aligned}
\operatorname{Lift}(n) = 
&\text{ let } q := n.\mathcal{A}[0] \quad \text{(current parent)} \\
&\text{ let } p := n.\mathcal{A}[1] \quad \text{(current grandparent)} \\
&\text{ // Step 1: remove } n \text{ from } q.\mathcal{C} \\
&\quad q.\mathcal{C} := q.\mathcal{C} \setminus \{n\} \\
&\text{ // Step 2: remove } q \text{ from } p.\mathcal{C} \\
&\quad p.\mathcal{C} := p.\mathcal{C} \setminus \{q\} \\
&\text{ // Step 3: update } n.\mathcal{C} \\
&\quad n.\mathcal{C} := n.\mathcal{C} \cup \{q\} \quad \text{(q as new root of subtree)} \\
&\text{ // Step 4: update } p.\mathcal{C} \\
&\quad p.\mathcal{C} := p.\mathcal{C} \cup \{n\} \quad \text{(n as new root of subtree)} \\
&\text{ // Step 4: assert } n \text{ to } q.\mathcal{A} \\
&\quad q.\mathcal{A} := [n] \frown q.\mathcal{A} \\
&\text{ // Step 5: remove } q \text{ from } n.\mathcal{A} \\
&\quad n.\mathcal{A} := n.\mathcal{A} \setminus \{q\}
\end{aligned}
$$

> 将 $q.\mathcal{C} \setminus \{n\}$ 下降为 $n$ 的新子树。

---

## 四、扩展操作的形式化

### 4. Merge(n, n*)

将直接子节点 $n^*$ 的**内容合并入 $n$**，并将其子树提升为 $n$ 的直接子节点。

$$
\begin{aligned}
\operatorname{Merge}(n, n^*) = 
&\text{ require } n^* \in \text{DirectChildren}(n) \\
&\text{ // 1. Merge data and instructions} \\
&\quad n.\mathcal{D} := n.\mathcal{D} \oplus \mathcal{D}^* \quad (\oplus = \text{merge policy}) \\
&\quad n.\mathcal{I} := n.\mathcal{I} \oplus \mathcal{I}^* \quad (\oplus = \text{merge policy}) \\
&\text{ // 2. Promote } n^*.\mathcal{C} \text{ to be part of } n.\mathcal{C} \\
&\quad n.\mathcal{C} := (n.\mathcal{C} \setminus \{n^*\}) \cup \text{DirectChildren}(n^*) \\
&\text{ // 3. Update ancestors of promoted children} \\
&\quad \forall c \in \text{DirectChildren}(n^*): \\
&\quad\quad c.\mathcal{A} := c.\mathcal{A} \setminus \{n^*\}
\end{aligned}
$$

> 子树结构被提升一层，但整体树性保持。

---

### 5. Divide(n, F_D, F_I)

从 \(n\) 中分离出内容，创建新子节点 \(n^*\)，其子树初始为空。

$$
\begin{aligned}
\operatorname{Divide}(n, F_D, F_I) = 
&\text{ create } n^* = (\mathcal{N}^*, \texttt{ready}, [n] \frown n.\mathcal{A}, \emptyset, F_D, F_I) \\
&n.\mathcal{D} := n.\mathcal{D} \setminus F_D \\
&n.\mathcal{I} := n.\mathcal{I} \setminus F_I \\
&n.\mathcal{C} := n.\mathcal{C} \cup \{n^*\} \quad \text{(n^* as new root of subtree)}
\end{aligned}
$$

> 新节点 $n^*$ 带有空子树（$n^*.\mathcal{C} = \emptyset$），可后续通过 Push 扩展。

## 五、状态性操作的形式化

### 6. Active(n)

$$
\begin{aligned}
\operatorname{Active}(n) = 
&\text{ require } n.\mathcal{S} \ne \mathtt{running} \\
&\quad n.\mathcal{S} = \mathtt{ready}
\end{aligned}
$$

### 7. Wait(n)

$$
\begin{aligned}
\operatorname{Wait}(n) = 
&\text{ require } n.\mathcal{S} = \mathtt{running} \\
&\quad n.\mathcal{S} = \mathtt{blocked}
\end{aligned}
$$

### 8. Yield(n)

$$
\begin{aligned}
\operatorname{Yield}(n) = 
&\text{ require } n.\mathcal{S} = \mathtt{running} \\
&\quad n.\mathcal{S} = \mathtt{ready}
\end{aligned}
$$

### 9. Finish(n)

$$
\begin{aligned}
\operatorname{Finish}(n) = 
&\quad n.\mathcal{S} = \mathtt{zombie}
\end{aligned}
$$

### 10. Warn(n)

$$
\begin{aligned}
\operatorname{Warn}(n) = 
&\quad n.\mathcal{S} = \mathtt{error}
\end{aligned}
$$
