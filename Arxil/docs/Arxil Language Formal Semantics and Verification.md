<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: Apache-2.0
-->

# Arxil Language Formal Semantics and Verification

---

> v0.2.1-drafting

---

- [Arxil Language Formal Semantics and Verification](#arxil-language-formal-semantics-and-verification)
  - [0 Introduction](#0-introduction)
    - [0.1 Role and Scope](#01-role-and-scope)
    - [0.2 Relationship to AS Model](#02-relationship-to-as-model)
    - [0.3 Document Structure Overview](#03-document-structure-overview)
  - [1 Basic Domains and Notation](#1-basic-domains-and-notation)
    - [1.1 Inherited Domains from AS](#11-inherited-domains-from-as)
    - [1.2 Arxil-Specific Domains](#12-arxil-specific-domains)
  - [2 Program State Model](#2-program-state-model)
    - [2.1 Global State](#21-global-state)
    - [2.2 Local Node State](#22-local-node-state)
    - [2.3 Field Environment](#23-field-environment)
    - [2.4 Read and Write Permission Light Cone](#24-read-and-write-permission-light-cone)
      - [2.4.1 Motivation and Intuition](#241-motivation-and-intuition)
      - [2.4.2 Extended Semantic Domains](#242-extended-semantic-domains)
      - [2.4.3 Permission Consistency Constraints](#243-permission-consistency-constraints)
      - [2.4.4 Geometric Interpretation: The Light Cone Model](#244-geometric-interpretation-the-light-cone-model)
        - [Definition 2.1 (Depth Embedding)](#definition-21-depth-embedding)
        - [Definition 2.2 (Field Embedding)](#definition-22-field-embedding)
        - [Definition 2.3 (Binding Ray)](#definition-23-binding-ray)
        - [Definition 2.4 (Permission Light Cone)](#definition-24-permission-light-cone)
      - [2.4.5 Dynamic Evolution and Causal Enforcement](#245-dynamic-evolution-and-causal-enforcement)
      - [2.4.6 Static Analysis via Cone Inspection](#246-static-analysis-via-cone-inspection)
  - [3. Static Semantics (Type System)](#3-static-semantics-type-system)
    - [3.1 Type Environment ($\\Theta$)](#31-type-environment-theta)
    - [3.2 Well-Formedness Rules](#32-well-formedness-rules)
  - [4. Dynamic Semantics (Operational Semantics)](#4-dynamic-semantics-operational-semantics)
    - [4.1 Configuration](#41-configuration)
    - [4.2 Small-Step Transition Relation ($\\longrightarrow$)](#42-small-step-transition-relation-longrightarrow)
      - [4.2.1 Ordinary Opcodes](#421-ordinary-opcodes)
      - [4.2.2 Generic Control Opcodes](#422-generic-control-opcodes)
      - [4.2.3 Structural Opcodes](#423-structural-opcodes)
  - [5. Field Resolution and Pointer Semantics](#5-field-resolution-and-pointer-semantics)
    - [5.1 The $\\text{Resolve}$ Function](#51-the-textresolve-function)
    - [5.2 The $\\text{Deref}$ Function](#52-the-textderef-function)
    - [5.3 The `gerf` Opcode](#53-the-gerf-opcode)
  - [6. Key Properties and Proofs](#6-key-properties-and-proofs)
    - [6.1 Termination of $\\text{Resolve}$](#61-termination-of-textresolve)
    - [6.2 Unique Physical Source](#62-unique-physical-source)
    - [6.3 Type Safety\*\* (Sketch)](#63-type-safety-sketch)
    - [6.4 Structural Integrity](#64-structural-integrity)
  - [7. Conclusion](#7-conclusion)
  - [Appendix: Mapping Arxil Syntax to Formal Constructs](#appendix-mapping-arxil-syntax-to-formal-constructs)


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

- `Opcode`: Arxil 完整操作码集合 (`psh`, `pvt`, `exec`, `add`, `wait`, etc.).
- `CausalLabel`: 因果事件标签集合（在 `meta_data` 中自定义），供程序声明自己的执行事件，要求地或触发地。形式化为 `Label : String | UUID`，但语义上视为原子符号。

---

## 2 Program State Model

### 2.1 Global State

合规 AS 树，形式化定义为节点标识符到其本地状态的部分映射: $\Sigma : \mathcal{N} \rightharpoonup \sigma(n)$. $\Sigma$ 必须满足 ASFS 中的 (WF1)-(WF4) 良构约束。

### 2.2 Local Node State

> Directly adopts the definition from ASFS Section 3.

- $\sigma(n) = (\mathcal{E}(n)、\mathcal{A}(n)、\mathcal{C}(n)、\mathcal{D}(n)、\mathcal{I}(n)、\mathtt{pc}(n))$
- 结合各组件在 Arxil 执行过程中的作用简要说明 (e.g., $\mathcal{E}(n)$ is modified by `wait`/`fnsh`; $\mathcal{I}(n)$ and $\mathtt{pc}(n)$ drive the instruction loop).
- $\mathtt{pc}(n)$ 的取值范围是 $\mathcal{I}(n)$ 的索引，即 $\mathtt{pc}(n) \in \{ 0, \dots, | \mathcal{I}(n) | \}$，且当 $\mathtt{pc}(n) = \mathcal{I}(n)$ 时，节点进入 `zombie` 状态.

### 2.3 Field Environment

在 AS 形式化语义定义基础上扩展，新增类型映射 $\Gamma$.
- $\mathcal{D}(n) = (\mathcal{D}\_\text{priv}^n、\mathcal{D}\_\text{publ}^n、\mathcal{D}\_\text{ance}^n)$
- $\Gamma(n) : (\mathcal{F}\_\text{priv} \cup \mathcal{F}\_\text{publ} \cup \mathcal{F}\_\text{ance}) \longrightarrow \text{TypeID}$ (节点 $n$ 的静态类型环境).

### 2.4 Read and Write Permission Light Cone

#### 2.4.1 Motivation and Intuition

In the causal arbor network of Arbor Strux, field binding chains define paths of data dependency from descendant nodes to their physical sources in ancestors. While this structure enables zero-copy sharing and analyzable reference paths, it provides no built-in mechanism to prevent race conditions when multiple descendants concurrently write to the same public field.

To reconcile structural freedom with data-race safety, we introduce **field-level read/write permissions** and model their semantics through a geometric metaphor: the **permission light cone**.

> **Intuition**:  
> Imagine embedding the AS tree into a 2D Euclidean plane:
> - The **x-axis** represents depth in the tree (root at $x=0$, children at increasing $x$).
> - The **y-axis** linearly orders field names within each node’s public namespace.
>
> A binding chain from a descendant $n$ to its resolved source $(m, f)$ becomes a polyline in the $(x,y)$-plane, ending at $(x_m, y_f)$. All such chains resolving to the same $(m,f)$ form a **cone** emanating backward from the source.
>
> Now assign to each node along a chain a **filter**—a permission lens:
> - `READ` → transparent filter (light passes through, no modification);
> - `WRITE` → reflective filter (light bounces back, modifying the source);
> - Mixed or inconsistent filters cause optical interference—i.e., data races.
>
> The entire structure becomes a **3D permission light cone**, where a third axis $z$ indexes distinct binding chains converging on the same source. This cone not only visualizes dependencies but also encodes concurrency constraints.

This section formalizes this intuition while preserving its explanatory power.

#### 2.4.2 Extended Semantic Domains

We extend the basic domains of Section 2 as follows:

- **Privilege types**:
  $$
  \mathsf{Priv} ::= \mathtt{READ} \mid \mathtt{WRITE}
  $$

- **Field declaration with privilege**:
  Each public field in $\mathcal{D}_{\text{publ}}(n)$ is now annotated:
  $$
  \mathcal{D}_{\text{publ}}(n) : \mathcal{F}_{\text{publ}} \rightharpoonup \mathcal{V} \times \mathsf{Priv}
  $$
  The privilege indicates the *default export mode* of the field.

- **Ancestor binding with declared intent**:
  Binding references are enriched with access intent:
  $$
  \texttt{BindingRef} ::= \texttt{Unbound} \mid \texttt{Bound}(m, f', p)
  $$
  where $p \in \mathsf{Priv}$ is the privilege *requested by the descendant*.

- **Node-local access registry**:
  Each node $n$ maintains a dynamic map of active accesses:
  $$
  \mathcal{A}\!\mathcal{C}\!\mathcal{C}(n) \subseteq \mathcal{N} \times \mathcal{F}_{\text{publ}} \times \mathsf{Priv}
  $$
  recording which child (or self) currently holds which privilege on which field.

#### 2.4.3 Permission Consistency Constraints

We impose new well-formedness conditions on configurations.

> **(WF5) Source Privilege Compatibility**  
> For any node $n$, field $f \in \mathrm{dom}(\mathcal{D}_{\text{publ}}(n))$, and any descendant $d$ such that  
> $\texttt{Resolve}(d, f') = (n, f)$ with declared privilege $p_d$,  
> it must hold that:
> - If $p_d = \mathtt{WRITE}$, then the source field’s declared privilege includes $\mathtt{WRITE}$;
> - If $p_d = \mathtt{READ}$, then the source field’s declared privilege includes $\mathtt{READ}$.

> **(WF6) Concurrent Access Exclusivity**  
> For any node $n$, field $f$, and any two distinct active accesses  
> $(c_1, f, p_1), (c_2, f, p_2) \in \mathcal{A}\!\mathcal{C}\!\mathcal{C}(n)$,  
> it must not be the case that $p_1 = \mathtt{WRITE}$ and $p_2 \in \{\mathtt{READ}, \mathtt{WRITE}\}$.

These rules enforce that **at most one writer may access a field at any time**, and readers are blocked during writes—mirroring mutual exclusion.

#### 2.4.4 Geometric Interpretation: The Light Cone Model

We now formalize the geometric intuition.

##### Definition 2.1 (Depth Embedding)
For each node $n$, define its depth:
$$
\delta(n) = 
\begin{cases}
0 & \text{if } \mathcal{A}(n) = [] \\
1 + \delta(p_1) & \text{if } \mathcal{A}(n) = [p_1, \dots]
\end{cases}
$$
This gives the $x$-coordinate: $x_n := \delta(n)$.

##### Definition 2.2 (Field Embedding)
Fix an injective encoding $\phi : \mathcal{F} \to \mathbb{R}$. For field $f$, set $y_f := \phi(f)$.

##### Definition 2.3 (Binding Ray)
A binding chain from $d$ to source $(s, f)$ induces a discrete ray:
$$
\mathcal{R}(d \leadsto s,f) = \big\{ (x_n, y_f) \mid n \in \text{path}(d \to s) \big\}
$$
where $\text{path}(d \to s)$ is the unique ancestor chain from $d$ up to $s$.

##### Definition 2.4 (Permission Light Cone)
For a fixed source $(s, f)$, let $\mathcal{C}(s,f)$ be the set of all descendants $d$ such that $\texttt{Resolve}(d, \cdot) = (s,f)$. Index them as $\{d_1, d_2, \dots, d_k\}$. The **permission light cone** is the set:
$$
\mathcal{L}(s,f) = \bigcup_{i=1}^k \Big( \mathcal{R}(d_i \leadsto s,f) \times \{i\} \Big) \subseteq \mathbb{R}^3
$$
with coordinates $(x, y, z)$. Each ray carries a **permission profile**:
$$
\Pi_i = \big[ p_{n}^{(i)} \big]_{n \in \text{path}(d_i \to s)}
$$
where $p_n^{(i)}$ is the privilege declared by $d_i$ at node $n$ (typically constant along the ray).

> **Optical Semantics**:
> - A ray with all $p = \mathtt{READ}$ is a **transmissive beam**—it observes but does not perturb.
> - A ray containing $p = \mathtt{WRITE}$ is a **reflective beam**—it modifies the apex $(x_s, y_f, \cdot)$, potentially affecting other rays.
> - Two reflective rays in the same cone with overlapping $x$-projections and no causal order constitute a **race interference pattern**.

#### 2.4.5 Dynamic Evolution and Causal Enforcement

When a node $d$ attempts to execute an instruction that accesses a bound field $f$:

1. The runtime computes $\texttt{Resolve}(d, f) = (s, f_s)$;
2. It checks whether the requested privilege $p$ is compatible with $\mathcal{D}_{\text{publ}}(s)(f_s)$ (WF5);
3. It queries $\mathcal{A}\!\mathcal{C}\!\mathcal{C}(s)$:
   - If $p = \mathtt{READ}$ and no writer is active → grant access;
   - If $p = \mathtt{WRITE}$ and no other access is active → grant exclusive access;
   - Otherwise → block the node ($\mathcal{E}(d) := \mathtt{blocked}$) until conflicting accesses complete.

Upon completion, the access is removed from $\mathcal{A}\!\mathcal{C}\!\mathcal{C}(s)$, and waiting nodes are re-evaluated.

This mechanism ensures that **the light cone remains optically coherent**: no two conflicting beams illuminate the apex simultaneously.

#### 2.4.6 Static Analysis via Cone Inspection

The light cone model enables powerful compile-time checks:

- **Race Detection**: Scan $\mathcal{L}(s,f)$ for multiple $\mathtt{WRITE}$ rays whose $x$-intervals overlap and lack a happens-before edge. Flag as potential race.
- **Over-Serialization**: Detect cones where all rays are $\mathtt{WRITE}$ despite operating on disjoint $y$-regions (different fields)—suggest privilege refinement.
- **Dead Binding**: Identify rays that terminate at unlit apices (fields never written)—optimize away.

Visualization tools can render $\mathcal{L}(s,f)$ as interactive 3D cones, with color-coded rays (green = READ, red = WRITE), allowing developers to “see” concurrency structure.

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
- **Structural Opcode Preconditions**: Checks that operands in `psh`/`pvt` refer to fields that exist in the current scope and have compatible types for binding.
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
- *Rule for `pvt external_node (...)`*: Updates $n.\mathcal{D}\_ \text{ance}(\text{local_ance_field}) := \mathtt{Bound}(\text{data_source}, \text{source_field})$ in $\Sigma$.

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