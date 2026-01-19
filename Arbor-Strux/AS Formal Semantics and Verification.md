<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: CC-BY-4.0
-->

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
  $\mathtt{push}$, $\mathtt{pivot}$, $\mathtt{exec}$, $\mathtt{cond}$, $\mathtt{cycl}$, $\mathtt{finish}$, etc.
  > These atomic operations are usually exposed as opcodes such as `psh` and `pvt` in specific programming language interfaces (such as Arxil).
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
- $\mathtt{pivot}(n, c)$: restructure subtree rooted at $c$ around $n$.

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

Notably, the exit condition for $\mathtt{running}$ is *not* enforced by the model’s transition rules; it depends entirely on the instruction sequence $\mathcal{I}(n)$ (e.g., presence of $\mathtt{finish}$ or loop termination). In contrast, the eventual exit of transient states is *externally guaranteed* by Axiom 7.4.1, contingent on the global causal context (e.g., resolution of ancestor bindings, subtree completion, or structural reconfiguration via $\mathtt{pivot}$).

This distinction underscores a fundamental asymmetry in AS dynamics: **activity is local and autonomous, while suspension is global and contextual**.

---

## 8. Relativistic Observation (Distributed Perspective)

- Each node $n$ observes only the effects of events in its **causal past**.
- When $n$ evaluates $S(n, f)$, the result reflects the last write to the physical source $(m, f_{\text{publ}})$ that is causally prior to $n$’s current event.
- Concurrent modifications by causally unrelated nodes are **not visible** until a causal link is established (e.g., via $\mathtt{pivot}$ or rebinding).
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

*End of Formal Semantics Document.*