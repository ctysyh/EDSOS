<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: Apache-2.0
-->

# Arxil Language Formal Semantics and Verification

---

> v0.2.1-drafting

---

## 0 Introduction

### 0.1 Role and Scope

本文档是为 《Arxil Language Specification》 提供精确数学语义的正式文档。目标读者包括编译器开发者、验证工具作者、语言设计者等。它不定义 AS 模型本身，而是将 Arxil 语法映射到 AS 模型的操作上。

### 0.2 Relationship to AS Model

本文档是《AS Formal Semantics and Verification》（以下简称 ASFS）的特化和实例化。所有核心概念（节点状态、执行状态、绑定引用等）均直接继承自 ASFS，同时，Arxil 的每一条合法程序在执行时产生的所有配置变迁，都必须是 ASFS 中允许的合法变迁。

### 0.3 Document Structure Overview
本文档从基础域开始，接着定义程序状态模型，再定义静态和动态语义，最后是关键性质证明。

---

## 1 Basic Domains and Notation

### 1.1 Inherited Domains from AS

- $\mathcal{N}$: 节点标识符
- $\text{ExecStatus} = {\mathtt{ready}, \mathtt{running}, \mathtt{blocked}, \mathtt{zombie}, \mathtt{error}}$.
- $\text{BindingRef} ::= \mathtt{Unbound} | \mathtt{Bound}(m, f')$，其中 $m \in \mathcal{N}$，$f' \in \mathcal{F}\_\text{publ}$. 详见 ASFS §3.2。
- $\mathcal{F} = \mathcal{F}\_\text{priv} ⊎ \mathcal{F}\_\text{publ} ⊎ \mathcal{F}\_\text{ance}$（注：`⊎` 代表不相交并集）.

### 1.2 Arxil-Specific Domains

- `Opcode`: Arxil 完整操作码集合 (`psh`, `lft`, `exec`, `add`, `wait`, etc.).
- `CausalLabel`: 因果事件标签集合（在 `meta_data` 中自定义），供程序声明自己的执行事件，要求地或触发地。形式化为 `Label : String | UUID`，但语义上视为原子符号。

---

## 2 Program State Model

### 2.1 Global State ($\Sigma$)

合规 AS 树，形式化定义为节点标识符到其本地状态的部分映射: $\Sigma : \mathcal{N} \rightharpoonup \sigma(n)$. $\Sigma$ 必须满足 ASFS 中的 (WF1)-(WF4) 良构约束。

### 2.2 Local Node State ($\sigma(n)$)

> Directly adopts the definition from ASFS Section 3.

- $\sigma(n) = (\mathcal{E}(n)、\mathcal{A}(n)、\mathcal{C}(n)、\mathcal{D}(n)、\mathcal{I}(n)、\mathtt{pc}(n))$
- 结合各组件在 Arxil 执行过程中的作用简要说明 (e.g., $\mathcal{E}(n)$ is modified by `wait`/`fnsh`; $\mathcal{I}(n)$ and $\mathtt{pc}(n)$ drive the instruction loop).
- $\mathtt{pc}(n)$ 的取值范围是 $\mathcal{I}(n)$ 的索引，即 $\mathtt{pc}(n) \in \{ 0, \dots, | \mathcal{I}(n) | \}$，且当 $\mathtt{pc}(n) = \mathcal{I}(n)$ 时，节点进入 `zombie` 状态.

### 2.3 Field Environment ($\mathcal{D}(n)$)

在 AS 形式化语义定义基础上扩展，新增类型映射 $\Gamma$.
- $\mathcal{D}(n) = (\mathcal{D}\_\text{priv}^n、\mathcal{D}\_\text{publ}^n、\mathcal{D}\_\text{ance}^n)$
- $\Gamma(n) : (\mathcal{F}\_\text{priv} \cup \mathcal{F}\_\text{publ} \cup \mathcal{F}\_\text{ance}) \longrightarrow \text{TypeID}$ (节点 $n$ 的静态类型环境).

---

## 3. Static Semantics (Type System)

### 3.1 Type Environment ($\Theta$)

Formalizes the `.arxtype` contract system.
- $\Theta : \text{TypeID} \longrightarrow  \{ \mathtt{size}: \mathbb{N}, \mathtt{layout}: \text{LayoutSpec}, \mathtt{ops}: \text{OpSet}, \mathtt{interop}: \text{InteropRules} \}$
- `InteropRules` 包含操作码签名和隐式转换规则（如有）。

### 3.2 Well-Formedness Rules

> 定义 Arxil 程序满足语法与静态有效性的判定条件

- **Field Declaration Check**: For each field declaration $f: T$, $T$ must be in $\text{dom}(\Theta)$.
- **Function Signature Check**: Parameters and return types of `fn` blocks must be well-formed under $\Theta$.
- **Opcode Typing Rule**: 数据操作类操作码（如`add (z) (x y);`）需满足互操作规则，并且 $\mathtt{add} \in \Theta(\tau).\mathtt{ops}$.
- **Structural Opcode Preconditions**: Checks that operands in `psh`/`lft` refer to fields that exist in the current scope and have compatible types for binding.
- 静态检查不验证绑定是否存在，只验证类型和字段名在作用域内合法。绑定有效性是动态的（可能阻塞或失败）。

---

## 4. Dynamic Semantics (Operational Semantics)

### 4.1 Configuration

运行时配置为二元组 $(\Sigma, n)$，其中 $\Sigma$ 为全局状态，$n$ 为当前调度节点标识符。$n$ 是由外部调度器选择的（如基于就绪队列），假设满足 $n \in \text{dom}(\Sigma)$ 且 $\mathcal{E}(n) = \mathtt{running}$，本形式化只关注单步执行。

### 4.2 Small-Step Transition Relation ($\longrightarrow$)

定义配置执行单步原子操作的演化规则

#### 4.2.1 Ordinary Opcodes

Rules for `cpy`, `add`, `sub`, etc. These rules read from/write to $\mathcal{D}(n)$ based on $\Gamma(n)$ and $\Theta$.

- *Example Rule*: $(\Sigma, n) \longrightarrow (\Sigma', n)$ where $\Sigma'(n).\mathcal{D}\_\text{publ}(z) := eval_op(\text{add}, \Sigma(n).\mathcal{D}(x), \Sigma(n).\mathcal{D}(y))$ given typing preconditions hold.

这些规则是在 `.arxtype` 中被对应的提供者定义的，不属于 Arxil 语言的内容，此处作为指引。副作用仅修改 $\mathcal{D}(n)$，不改变树结构或因果关系。

#### 4.2.2 Generic Control Opcodes

Rules for `exec`, `cond`, `cycl`, `wait`, `sgnl`, `yiel`, `fnsh`, `emit`, `obsv`.

- *Example Rule for `wait`*: If the condition is not met, $(\Sigma, n) \longrightarrow (\Sigma[n \mapsto \sigma'], n)$ where $\sigma'.\mathcal{E} = \mathtt{blocked}$ and $\sigma'.\mathtt{pc}$ is unchanged.
- *Example Rule for `fnsh`*: $(\Sigma, n) \longrightarrow (\Sigma[n \mapsto \sigma'], n)$ where $\sigma'.\mathcal{E} = \mathtt{zombie}$.

#### 4.2.3 Structural Opcodes

Rules that modify the global state $\Sigma$.

- *Rule for `psh child (...)`*: Creates a new node $c$, sets $c.\mathcal{A} = [n] ++ n.\mathcal{A}$, adds $c$ to $n.\mathcal{C}$, initializes $c.\mathcal{D} \_ \text{ance}$ according to the binding list, and updates $\Sigma$.
- *Rule for `lft external_node (...)`*: Updates $n.\mathcal{D}\_ \text{ance}(\text{local_ance_field}) := \mathtt{Bound}(\text{data_source}, \text{source_field})$ in $\Sigma$.

结构性操作是唯一能破坏良构性的操作，因此其前置条件必须极其严格。

---

## 5. Field Resolution and Pointer Semantics

### 5.1 The $\text{Resolve}$ Function

$\text{Resolve}$ 是一个偏函数，目标是将字段引用 $(n, f)$ 解析到其最终的物理存储位置 $(m, f_{\text{publ}})$，其中 $f_{\text{publ}} \in \text{dom}(\mathcal{D}_{\text{publ}}^m)$。

- **定义**：给定良构 AS 树 $\mathcal{T}$ 和节点 $n \in \mathcal{T}$，对任意字段名 $f$：
  - `[R1]` - 本地字段。若 $f \in \text{dom}(\mathcal{D}_{\text{priv}}^n) \cup \text{dom}(\mathcal{D}_{\text{publ}}^n)$，则  
    $$
    \text{Resolve}(n, f) = (n, f)
    $$
  - `[R2]` - `ance` 字段解析。若 $\text{ref} = \texttt{Unbound}$，则 $\text{Resolve}(n, f) = \bot$（未定义，运行时错误或阻塞）；若 $\text{ref} = \texttt{Bound}(m, f')$，则  
    $$
    \text{Resolve}(n, f) = \text{Resolve}(m, f')
    $$
    （递归调用）。
  - `[R3]` 若以上均不满足，则 $\text{Resolve}(n, f) = \bot$。**不存在对祖先链的隐式遍历查找**。
  - 由于 (WF4) 保证绑定图为 DAG，递归深度有限。

### 5.2 The $\text{Deref}$ Function

Formalizes the `derf` opcode.

- **Phase 1: $\text{LocateField}(n, v)$**: Maps a virtual address $v$ in node $n$'s space to a field descriptor ($\text{LocalPubl}(f)$, $\text{AnceRef}(f)$, etc.).
- **Phase 2: Full Dereference**: Uses $\text{Resolve}$ if the field is an `ance` reference.

### 5.3 The `gerf` Opcode

Defines how `gerf (ptr) (field);` constructs a pointer value $p = (v, n_{\text{src}})$ where $v$ is the virtual address of `field` in $n_{\text{src}}$'s address space.

---

## 6. Key Properties and Proofs

### 6.1 Termination of $\text{Resolve}$

证明对任何良构 AS 树 $\mathcal{T}$ 和节点 $n \in \mathcal{T}$，$\text{Resolve}(n, f)$ 要么在有限步内返回 $(m, f_{\text{publ}})$，要么返回 $\bot$。

### 6.2 Unique Physical Source

证明所有执行成功的 $\text{Resolve}$ or $\text{Deref}$ 操作，均对应唯一的 $(m, f_{\text{publ}})$ 对，为零拷贝语义提供支撑

### 6.3 Type Safety** (Sketch)

论证合规类型程序（满足第3章要求）在执行过程中（第4章）不会陷入类型错误状态，核心依赖静态组件（$\Theta$、$\Gamma$）与动态组件（$\mathcal{D}$）的分离设计，静态检查确保所有操作数类型合法，动态执行中，$\mathcal{D}(n)$ 中存储的值始终符合 $\Gamma(n)$ 的预期（因普通操作码由 $\Theta$ 定义，结构性操作码检查类型兼容），指针解引用通过 $\text{Resolve}$ + $\Gamma$ 保证目标类型正确。

### 6.4 Structural Integrity

证明所有结构性操作码均能维持 AS 形式化语义定义的 AS 树合规性

---

## 7. Conclusion

本形式化为 Arxil 提供了可机械验证的基础。
- 本文档构建了 Arxil 语言语法与底层 AS 计算模型之间完整、严谨的映射桥梁
- 本形式化理论文档为零拷贝、可分析依赖、结构安全等语言性质提供证明
- 本形式化理论文档为编译器正确性证明、静态分析工具、形式化验证框架等工作提供基础

---

## Appendix: Mapping Arxil Syntax to Formal Constructs

> 快速查询表，对应 Arxil 关键字与本文档中的形式化定义章节 (e.g., `psh` → Section 4.2.3, `ance` → Section 5.1).

---

*End of Formal Semantics Document.*