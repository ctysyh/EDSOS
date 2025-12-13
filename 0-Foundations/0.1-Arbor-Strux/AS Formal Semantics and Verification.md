# AS Formal Semantics and Verification

---

> v0.2.1

---

- [AS Formal Semantics and Verification](#as-formal-semantics-and-verification)
  - [1. Introduction](#1-introduction)
  - [2. Basic Domains](#2-basic-domains)
  - [3. Local Node State (Spatial Coordinates + Computational Context)](#3-local-node-state-spatial-coordinates--computational-context)
  - [4. Global Configuration (No Global Time)](#4-global-configuration-no-global-time)
  - [5. Well-Formedness Constraints (Structural Invariants)](#5-well-formedness-constraints-structural-invariants)
    - [(WF1) Ancestor Consistency](#wf1-ancestor-consistency)
    - [(WF2) Child Closure](#wf2-child-closure)
    - [(WF3) Unique Parent \& Acyclicity](#wf3-unique-parent--acyclicity)
    - [(WF4) Binding Graph DAG Property](#wf4-binding-graph-dag-property)
  - [6. Field Resolution and State Function](#6-field-resolution-and-state-function)
    - [Definition 6.1 (Resolve)](#definition-61-resolve)
    - [Definition 6.2 (State Function)](#definition-62-state-function)
  - [7. Events and Causal Evolution (Temporal Dimension)](#7-events-and-causal-evolution-temporal-dimension)
    - [7.1 Events](#71-events)
    - [7.2 Causal Partial Order ($\\prec$)](#72-causal-partial-order-prec)
    - [7.3 Execution Semantics](#73-execution-semantics)
    - [7.4 Transient Zombie Axiom and State Dynamics](#74-transient-zombie-axiom-and-state-dynamics)
      - [Classification of Execution States](#classification-of-execution-states)
  - [8. Relativistic Observation (Distributed Perspective)](#8-relativistic-observation-distributed-perspective)
  - [9. Key Properties](#9-key-properties)
    - [Theorem 1 (Well-Formedness Preservation)](#theorem-1-well-formedness-preservation)
    - [Theorem 2 (Termination of Resolution)](#theorem-2-termination-of-resolution)
    - [Theorem 3 (Unique Physical Source)](#theorem-3-unique-physical-source)
    - [Theorem 4 (Turing Completeness)](#theorem-4-turing-completeness)
  - [10. Geometric Interpretation](#10-geometric-interpretation)
  - [11. Read and Write Permission Light Cone](#11-read-and-write-permission-light-cone)
    - [11.1 Motivation and Intuition](#111-motivation-and-intuition)
    - [11.2 Extended Semantic Domains](#112-extended-semantic-domains)
    - [11.3 Permission Consistency Constraints](#113-permission-consistency-constraints)
    - [11.4 Geometric Interpretation: The Light Cone Model](#114-geometric-interpretation-the-light-cone-model)
      - [Definition 11.1 (Depth Embedding)](#definition-111-depth-embedding)
      - [Definition 11.2 (Field Embedding)](#definition-112-field-embedding)
      - [Definition 11.3 (Binding Ray)](#definition-113-binding-ray)
      - [Definition 11.4 (Permission Light Cone)](#definition-114-permission-light-cone)
    - [11.5 Dynamic Evolution and Causal Enforcement](#115-dynamic-evolution-and-causal-enforcement)
    - [11.6 Static Analysis via Cone Inspection](#116-static-analysis-via-cone-inspection)


---

## 1. Introduction

Arbor Strux (AS) is a distributed, concurrent computational model grounded in *local state machines* whose interactions are mediated through *structural composition* and *field binding*. Unlike traditional global-state transition systems, AS does not assume a synchronized global clock. Instead, it embraces an asynchronous, event-driven paradigm where time emerges as a *causal partial order* over local actions, and space is defined by a dynamically evolving tree topology.

This document presents a rigorous formal semantics for AS, reframing it as a **Causal Arbor Network**—a system in which:
- **Space** is encoded by the tree structure (via ancestor and child relations),
- **Time** is captured by a happens-before causal relation over events,
- **Parallelism** arises from causally independent local evolutions,
- **Observation** is relative to each node’s causal past.

The model guarantees structural well-formedness, termination of field resolution, and Turing completeness, while enabling zero-copy data sharing with explicit, analyzable dependencies.

---

## 2. Basic Domains

We fix the following syntactic and semantic domains:

- **Node identifiers**: $\mathcal{N}$ — a countable set of globally unique names.
- **Execution states**:  
  $\mathsf{ExecStatus} = \{ \mathtt{ready},\ \mathtt{running},\ \mathtt{blocked},\ \mathtt{zombie},\ \mathtt{error} \}$.
- **Instructions**: $\mathsf{Instr}$ — a set of atomic operations including  
  $\mathtt{push}$, $\mathtt{lift}$, $\mathtt{exec}$, $\mathtt{cond}$, $\mathtt{cycl}$, $\mathtt{finish}$, etc.
  > These atomic operations are usually exposed as opcodes such as `psh` and `lft` in specific programming language interfaces (such as Arxil).
- **Field names**: $\mathcal{F} = \mathcal{F}_{\text{priv}} \uplus \mathcal{F}_{\text{publ}} \uplus \mathcal{F}_{\text{ance}}$.
- **Values**: $\mathcal{V}$ — a domain of base values (integers, booleans, etc.).
- **Binding references**:  
  $\texttt{BindingRef} ::= \texttt{Unbound} \mid \texttt{Bound}(m, f')$,  
  where $m \in \mathcal{N}$ and $f' \in \mathcal{F}_{\text{publ}}$.

---

## 3. Local Node State (Spatial Coordinates + Computational Context)

Each node $n \in \mathcal{N}$ maintains a **local state** $\sigma(n)$, defined as a 6-tuple:

$$
\sigma(n) = \big(
  \mathcal{E}(n),\ 
  \mathcal{A}(n),\ 
  \mathcal{C}(n),\ 
  \mathcal{D}(n),\ 
  \mathcal{I}(n),\ 
  \mathtt{pc}(n)
\big)
$$

| Component | Type | Semantics |
|----------|------|-----------|
| $\mathcal{E}(n)$ | $\in \mathsf{ExecStatus}$ | Execution status (control label) |
| $\mathcal{A}(n)$ | $\in \mathcal{N}^*$ | **Spatial coordinate**: ancestor chain $[p_1, \dots, p_k]$, where $p_1$ is the direct parent and $p_k$ is the root |
| $\mathcal{C}(n)$ | $\subseteq \mathcal{N}$ | **Spatial coordinate**: set of direct children |
| $\mathcal{D}(n)$ | $(\mathcal{D}_{\text{priv}}, \mathcal{D}_{\text{publ}}, \mathcal{D}_{\text{ance}})$ | Field environment |
| $\mathcal{I}(n)$ | $\in \mathsf{Instr}^*$ | Instruction sequence (program body) |
| $\mathtt{pc}(n)$ | $\in \mathbb{N}^+$ | Program counter (1-based index into $\mathcal{I}(n)$) |

> **Note**: $\mathcal{A}(n)$ and $\mathcal{C}(n)$ define the node’s position in the **tree space**; they are purely spatial and carry no temporal meaning.

---

## 4. Global Configuration (No Global Time)

A **global configuration** is a partial function:
$$
\Sigma : \mathcal{N} \rightharpoonup \text{NodeState}
$$
with domain $\mathrm{dom}(\Sigma) \subseteq \mathcal{N}$ denoting the set of currently alive nodes.

> **Crucially**, $\Sigma$ does not represent “the state at time $t$”, but rather a **causally consistent snapshot**—a set of local states that could coexist in some execution history.

---

## 5. Well-Formedness Constraints (Structural Invariants)

A configuration $\Sigma$ is **well-formed** iff the following conditions hold:

### (WF1) Ancestor Consistency  
For all $n \in \mathrm{dom}(\Sigma)$, if $\mathcal{A}(n) = [p_1, \dots, p_k]$, then:
- $p_i \in \mathrm{dom}(\Sigma)$ for all $i$,
- $\mathcal{A}(p_1) = [p_2, \dots, p_k]$,
- If $\mathcal{A}(n) = []$, then $n$ is the unique root.

### (WF2) Child Closure  
For all $n \in \mathrm{dom}(\Sigma)$ and $c \in \mathcal{C}(n)$:
- $c \in \mathrm{dom}(\Sigma)$,
- $\mathcal{A}(c) = [n] + \mathcal{A}(n)$.

### (WF3) Unique Parent & Acyclicity  
- Every non-root node has exactly one parent,
- There is no cycle in the parent-child relation.

> (WF1)–(WF3) together imply that $(\mathrm{dom}(\Sigma), E)$ forms a rooted tree, where $E = \{ (p, c) \mid c \in \mathcal{C}(p) \}$.

### (WF4) Binding Graph DAG Property  
Define the binding edge set:
$$
\mathcal{E}^{\text{bind}} = \{ (n, m) \mid \exists f \in \mathcal{F}_{\text{ance}},\ \mathcal{D}_{\text{ance}}(n)(f) = \texttt{Bound}(m, f') \}
$$
Then the graph $(\mathrm{dom}(\Sigma), \mathcal{E}^{\text{bind}})$ must be a directed acyclic graph (DAG).

---

## 6. Field Resolution and State Function

### Definition 6.1 (Resolve)  
For any $n \in \mathrm{dom}(\Sigma)$ and field name $f$:
$$
\texttt{Resolve}(n, f) =
\begin{cases}
(n, f) & \text{if } f \in \mathrm{dom}(\mathcal{D}_{\text{priv}}(n)) \cup \mathrm{dom}(\mathcal{D}_{\text{publ}}(n)) \\
\bot & \text{if } f \in \mathrm{dom}(\mathcal{D}_{\text{ance}}(n)) \land \mathcal{D}_{\text{ance}}(n)(f) = \texttt{Unbound} \\
\texttt{Resolve}(m, f') & \text{if } \mathcal{D}_{\text{ance}}(n)(f) = \texttt{Bound}(m, f')
\end{cases}
$$
By (WF4), this recursion terminates.

### Definition 6.2 (State Function)  
The resolved value of field $f$ at node $n$ is:
$$
S(n, f) =
\begin{cases}
\mathcal{D}_{\text{priv}}(n)(f) & \text{if } f \in \mathrm{dom}(\mathcal{D}_{\text{priv}}(n)) \\
\mathcal{D}_{\text{publ}}(n)(f) & \text{if } f \in \mathrm{dom}(\mathcal{D}_{\text{publ}}(n)) \\
\mathcal{D}_{\text{publ}}(m)(f') & \text{if } \texttt{Resolve}(n, f) = (m, f') \\
\texttt{Unbound} & \text{otherwise}
\end{cases}
$$

---

## 7. Events and Causal Evolution (Temporal Dimension)

### 7.1 Events  
An **event** is the application of an black box operation, e.g.:
- $\mathtt{exec}(n)$: execute current instruction of $n$,
- $\mathtt{push}(n, c)$: $n$ creates child $c$,
- $\mathtt{lift}(n, c)$: restructure subtree rooted at $c$ around $n$.

Each event modifies finitely many local states.

### 7.2 Causal Partial Order ($\prec$)  
The **happens-before** relation $\prec$ is the smallest partial order satisfying:
1. **Program order**: If $e_1, e_2$ act on the same node $n$, and $e_1$ precedes $e_2$ in $\mathcal{I}(n)$, then $e_1 \prec e_2$.
2. **Creation dependency**: If $e_1 = \mathtt{push}(n, c)$ and $e_2$ acts on $c$, then $e_1 \prec e_2$.
3. **Transitivity**: $\prec$ is closed under transitive composition.

### 7.3 Execution Semantics  
System evolution proceeds via event application:
- Start from a well-formed $\Sigma_0$.
- At each step, choose any $n$ with $\mathcal{E}(n) = \mathtt{running}$, apply its current instruction, yielding a new event $e$ and updated configuration $\Sigma'$.
- If multiple such nodes exist and their events are **causally independent** ($e_1 \not\prec e_2$ and $e_2 \not\prec e_1$), they may be executed in any order—this captures **true parallelism**.
- All operations preserve (WF1)–(WF4).

> There is **no global clock**; time is entirely defined by $\prec$.

### 7.4 Transient Zombie Axiom and State Dynamics

We introduce a liveness-oriented axiom that characterizes the temporal behavior of non-active execution states in well-formed configurations. This axiom captures an essential property of *well-behaved* Arbor Strux programs: temporary suspension does not imply indefinite stalling.

> **Axiom 7.4.1 (Transient Zombie Axiom).**  
> Let $\Sigma$ be a well-formed global configuration, and let $n \in \mathrm{dom}(\Sigma)$ such that  
> $\mathcal{E}(n) \in \{ \mathtt{ready},\ \mathtt{blocked} \}$.  
> Then there exists a finite causal sequence of events $e_1, \dots, e_k$ applicable to $\Sigma$—each preserving well-formedness—such that the resulting configuration $\Sigma'$ satisfies  
> $\mathcal{E}'(n) \in \{ \mathtt{running},\ \mathtt{zombie},\ \mathtt{error} \}$.

Intuitively, nodes in $\mathtt{ready}$ or $\mathtt{blocked}$ states are *temporarily inert*: they do not generate events themselves, but their state is guaranteed to evolve within a finite causal future, either by resuming computation ($\mathtt{running}$) or by being finalized ($\mathtt{zombie}$ or $\mathtt{error}$). We refer to such states as **transient zombie states**, emphasizing their provisional nature despite superficial quiescence.

This axiom  constitutes a *program-level liveness condition*. Configurations violating Axiom 7.4.1 correspond to pathological executions, which, while syntactically admissible under (WF1)–(WF4), exhibit undesirable non-termination at the node level.

#### Classification of Execution States

Under Axiom 7.4.1, the set $\mathsf{ExecStatus}$ admits a natural dynamical partition:

- **Active state**: $\mathtt{running}$ — the only state from which local events originate. Its termination is *program-determined*; the model imposes no intrinsic bound on its duration.
- **Transient (metastable) states**: $\mathtt{ready},\ \mathtt{blocked}$ — causally inert but guaranteed to exit within finite steps under the axiom.
- **Terminal (absorbing) states**: $\mathtt{zombie},\ \mathtt{error}$ — once entered, no further events involving $n$ are possible; these states are closed under all event applications.

Notably, the exit condition for $\mathtt{running}$ is *not* enforced by the model’s transition rules; it depends entirely on the instruction sequence $\mathcal{I}(n)$ (e.g., presence of $\mathtt{finish}$ or loop termination). In contrast, the eventual exit of transient states is *externally guaranteed* by Axiom 7.4.1, contingent on the global causal context (e.g., resolution of ancestor bindings, subtree completion, or structural reconfiguration via $\mathtt{lift}$).

This distinction underscores a fundamental asymmetry in AS dynamics: **activity is local and autonomous, while suspension is global and contextual**.

---

## 8. Relativistic Observation (Distributed Perspective)

- Each node $n$ observes only the effects of events in its **causal past**.
- When $n$ evaluates $S(n, f)$, the result reflects the last write to the physical source $(m, f_{\text{publ}})$ that is causally prior to $n$’s current event.
- Concurrent modifications by causally unrelated nodes are **not visible** until a causal link is established (e.g., via $\mathtt{lift}$ or rebinding).
- This enables **zero-copy sharing** with **explicit, static-analyzable dependency paths**.

---

## 9. Key Properties

### Theorem 1 (Well-Formedness Preservation)  
All atomic operations preserve constraints (WF1)–(WF4).

### Theorem 2 (Termination of Resolution)  
For any well-formed $\Sigma$ and node $n$, $\texttt{Resolve}(n, f)$ terminates in finitely many steps.

### Theorem 3 (Unique Physical Source)  
If $\texttt{Resolve}(n, f) = (m, f_{\text{publ}})$, then $f_{\text{publ}} \in \mathrm{dom}(\mathcal{D}_{\text{publ}}(m))$, and this pair is unique.

### Theorem 4 (Turing Completeness)  
AS can simulate any Turing machine by:
- Using $\mathcal{D}_{\text{publ}}$ fields as registers,
- Using $\mathtt{cond}$ for conditional branching,
- Using $\mathtt{cycl}$ for loops,
- Using $\mathtt{exec}$ for arbitrary jumps.

---

## 10. Geometric Interpretation

- **Space** = Tree topology, intrinsically defined by $\mathcal{A}(n)$ and $\mathcal{C}(n)$.
- **Time** = Causal partial order $\prec$ over events.
- **Parallelism** = Causally independent local evolutions.
- **Observation** = Projection onto a node’s causal past.
- **Global consistency** = Emergent property of causally closed configurations, not a synchronized global state.

> **Arbor Strux is not a state machine evolving in global time,  
> but a causal network of local machines,  
> whose spatial structure is a tree,  
> and whose temporal order is partial and observer-dependent.**

---

翻译成中文：

## 11. Read and Write Permission Light Cone

### 11.1 Motivation and Intuition

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

### 11.2 Extended Semantic Domains

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

### 11.3 Permission Consistency Constraints

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

### 11.4 Geometric Interpretation: The Light Cone Model

We now formalize the geometric intuition.

#### Definition 11.1 (Depth Embedding)
For each node $n$, define its depth:
$$
\delta(n) = 
\begin{cases}
0 & \text{if } \mathcal{A}(n) = [] \\
1 + \delta(p_1) & \text{if } \mathcal{A}(n) = [p_1, \dots]
\end{cases}
$$
This gives the $x$-coordinate: $x_n := \delta(n)$.

#### Definition 11.2 (Field Embedding)
Fix an injective encoding $\phi : \mathcal{F} \to \mathbb{R}$. For field $f$, set $y_f := \phi(f)$.

#### Definition 11.3 (Binding Ray)
A binding chain from $d$ to source $(s, f)$ induces a discrete ray:
$$
\mathcal{R}(d \leadsto s,f) = \big\{ (x_n, y_f) \mid n \in \text{path}(d \to s) \big\}
$$
where $\text{path}(d \to s)$ is the unique ancestor chain from $d$ up to $s$.

#### Definition 11.4 (Permission Light Cone)
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

### 11.5 Dynamic Evolution and Causal Enforcement

When a node $d$ attempts to execute an instruction that accesses a bound field $f$:

1. The runtime computes $\texttt{Resolve}(d, f) = (s, f_s)$;
2. It checks whether the requested privilege $p$ is compatible with $\mathcal{D}_{\text{publ}}(s)(f_s)$ (WF5);
3. It queries $\mathcal{A}\!\mathcal{C}\!\mathcal{C}(s)$:
   - If $p = \mathtt{READ}$ and no writer is active → grant access;
   - If $p = \mathtt{WRITE}$ and no other access is active → grant exclusive access;
   - Otherwise → block the node ($\mathcal{E}(d) := \mathtt{blocked}$) until conflicting accesses complete.

Upon completion, the access is removed from $\mathcal{A}\!\mathcal{C}\!\mathcal{C}(s)$, and waiting nodes are re-evaluated.

This mechanism ensures that **the light cone remains optically coherent**: no two conflicting beams illuminate the apex simultaneously.

### 11.6 Static Analysis via Cone Inspection

The light cone model enables powerful compile-time checks:

- **Race Detection**: Scan $\mathcal{L}(s,f)$ for multiple $\mathtt{WRITE}$ rays whose $x$-intervals overlap and lack a happens-before edge. Flag as potential race.
- **Over-Serialization**: Detect cones where all rays are $\mathtt{WRITE}$ despite operating on disjoint $y$-regions (different fields)—suggest privilege refinement.
- **Dead Binding**: Identify rays that terminate at unlit apices (fields never written)—optimize away.

Visualization tools can render $\mathcal{L}(s,f)$ as interactive 3D cones, with color-coded rays (green = READ, red = WRITE), allowing developers to “see” concurrency structure.

---

*End of Formal Semantics Document.*