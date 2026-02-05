<!--
SPDX-FileCopyrightText: © 2025-2026 Bib Guake
SPDX-License-Identifier: CC-BY-4.0
-->

# Observation-Triggered Causality

*A Native Causal Synchronization Model for the Arbor Strux Computational Framework*

---

> v0.2.1

---

- [Observation-Triggered Causality](#observation-triggered-causality)
  - [Abstract](#abstract)
  - [1. Introduction](#1-introduction)
    - [1.1 Motivation: The Need for Structured Causal Coordination](#11-motivation-the-need-for-structured-causal-coordination)
    - [1.2 Core Insight: Time as Activated Causality](#12-core-insight-time-as-activated-causality)
  - [2. Theoretical Foundations](#2-theoretical-foundations)
    - [2.1 AS Spacetime Manifold Recap](#21-as-spacetime-manifold-recap)
    - [2.2 OTC as a Causal Extension Rule](#22-otc-as-a-causal-extension-rule)
    - [2.3 Event Coordinates in $\\mathcal{M}$](#23-event-coordinates-in-mathcalm)
  - [3. Relation to Classical Concurrency Theory](#3-relation-to-classical-concurrency-theory)
    - [3.1 Correspondence with Release/Acquire Semantics](#31-correspondence-with-releaseacquire-semantics)
    - [3.2 Principled Advantages of OTC](#32-principled-advantages-of-otc)
  - [4. Information-Theoretic Minimality](#4-information-theoretic-minimality)
    - [4.1 The Minimal Signal](#41-the-minimal-signal)
    - [4.2 Runtime Cost Analysis](#42-runtime-cost-analysis)
  - [5. Language Realization in Arxil](#5-language-realization-in-arxil)
    - [5.1 Syntax Extension](#51-syntax-extension)
    - [5.2 Semantic Specification](#52-semantic-specification)
    - [5.3 Placement in Arxil Opcode Taxonomy](#53-placement-in-arxil-opcode-taxonomy)
    - [5.4 Naming Rationale](#54-naming-rationale)
  - [6. Implementation Considerations](#6-implementation-considerations)
    - [6.1 Node State Extension](#61-node-state-extension)
    - [6.2 Scheduler Integration](#62-scheduler-integration)
    - [6.3 Static Verification](#63-static-verification)
  - [7. Conclusion](#7-conclusion)
  - [References](#references)


---

## Abstract

The Arbor Strux (AS) computational model redefines computation as the structural evolution of a dynamic tree of nodes, where time is not a global parameter but a partial order of causally related events. Within this framework, traditional synchronization primitives—rooted in shared memory and linear time—are ill-suited. This document introduces **Observation-Triggered Causality**(OTC), a minimal, native mechanism for expressing cross-node behavioral dependencies in AS. OTC enables a descendant node to declare that its execution is contingent upon having *observed* a specific event emitted by an ancestor. We formalize OTC as an extension to the AS causal semantics, demonstrate its deep correspondence with—and principled refinement of—release/acquire memory ordering from classical concurrent programming, and specify its realization in the Arxil language through the dual opcodes `emit` and `obsv`. Crucially, OTC incurs near-zero runtime overhead, preserves AS’s structural safety guarantees, and aligns with its philosophy: *structure is semantics, observation is time*.

---

## 1. Introduction

### 1.1 Motivation: The Need for Structured Causal Coordination

In the Arbor Strux model, each node executes independently within its local context, interacting with others solely through structured sharing (`publ`/`ance` bindings) and atomic structural operations (`psh`, `pvt`, etc.). While this eliminates data races and enables zero-copy communication, it leaves open a fundamental question:

> **How can a node express that its next action depends not on a *value*, but on the *completion of a specific behavior* in another node?**

Traditional solutions—such as polling a flag field or using external signaling mechanisms—violate AS principles:
- They expose internal state (fragile under optimization or restructuring),
- They incur continuous runtime cost (contradicting zero-overhead goals),
- They conflate *state observation* with *event observation*.

OTC addresses this gap by introducing a first-class notion of **named causal commitment**: a node may *emit* a label to announce the completion of a logical phase, and another may *observe* that label to condition its own execution. This transforms synchronization from an imperative “wait” into a declarative “I am ready only when I have observed X.”

### 1.2 Core Insight: Time as Activated Causality

OTC reframes time not as a backdrop against which events unfold, but as a **locally activated causal chain**, triggered by observation. In AS, a node does not “wait for time to pass”; rather, it **refuses to generate future events until a required causal precondition—expressed as an observed emission—is satisfied**. This aligns perfectly with AS’s relativistic observation model: what a node sees depends on its position in the tree and the causal past it has accumulated.

---

## 2. Theoretical Foundations

### 2.1 AS Spacetime Manifold Recap

We model the AS execution space as a discrete manifold:
\[
\mathcal{M} = (\mathcal{T}, \mathcal{F}, \prec)
\]
where:
- $\mathcal{T}$ is the tree of nodes with ancestor relation $\mathcal{A}(n)$,
- $\mathcal{F}$ is the field environment (values + binding chains),
- $\prec$ is the causal partial order over events.

An event $e$ is an atomic state transition (e.g., instruction execution). A node $n$ at event $e$ observes only the projection of $\mathcal{M}$ onto $\{ e' \mid e' \prec e \}$.

### 2.2 OTC as a Causal Extension Rule

We extend the AS operational semantics with a new inference rule:

> **(Rule OTC)**  
> Let $c$ be a node containing an `obsv(L)` instruction at program counter $pc$, and let $a \in \mathcal{A}(c)$.  
> If an event $e_a = \texttt{emit}(a, L)$ occurs, then a causal edge is established:
> \[
> e_a \prec e_c^{\text{next}}
> \]
> where $e_c^{\text{next}}$ is the first executable event in $c$ following `obsv(L)`.

This rule is well-formed: it respects tree topology (WF1–WF3), does not introduce cycles (edges flow ancestor → descendant), and does not interfere with field binding (WF4).

### 2.3 Event Coordinates in $\mathcal{M}$

An OTC dependency involves two events with distinct coordinates:
- **Emit event**: $(a, t_a, L)$ — spatial location $a$, temporal coordinate $t_a$, labeled $L$
- **Observe constraint**: $(c, \bot, L)$ — spatial location $c$, temporal coordinate deferred until $L$ is observed

The label $L$ serves as a **lightweight causal channel** embedded in the spacetime fabric of $\mathcal{M}$.

---

## 3. Relation to Classical Concurrency Theory

### 3.1 Correspondence with Release/Acquire Semantics

OTC is the structural analog of C++’s `memory_order_release` / `memory_order_acquire` pattern:

| Concept | C++ (Flat Threads) | AS/OTC (Structured Nodes) |
|--------|---------------------|----------------------------|
| Synchronization point | Atomic store with `release` | `emit(L)` |
| Observation point | Atomic load with `acquire` | `obsv(L)` |
| Shared signal | Global atomic variable (e.g., `flag`) | Label $L$ (pure metadata) |
| Visibility guarantee | All writes before `release` visible after `acquire` | All state stabilized before `emit` is safely observable after `obsv` |
| Scope | Global address space | Ancestor chain $\mathcal{A}(c)$ |

Both establish a *synchronizes-with* relationship that induces a happens-before edge.

### 3.2 Principled Advantages of OTC

Despite semantic similarity, OTC improves upon classical memory ordering in key ways:

1. **No Shared State Exposure**: Labels are symbolic; no need to allocate or manage shared flags.
2. **Zero Polling Overhead**: `obsv` is a scheduling constraint, not a spin loop.
3. **Structural Scoping**: Dependencies are confined to the ancestor chain, eliminating spurious wakeups.
4. **Dynamic Structure Safety**: Survives `pvt` operations because causal channels follow binding semantics.
5. **Declarative Intent**: Expresses *what must be observed*, not *how to wait*.

Thus, OTC is not merely an adaptation but a **refinement**—a higher-level abstraction that leverages AS’s structural discipline to eliminate entire classes of concurrency bugs.

---

## 4. Information-Theoretic Minimality

### 4.1 The Minimal Signal

We ask: *What is the smallest unit of information sufficient to convey “a specific behavior in a specific ancestor has completed”?*

Given the constraints of AS:
- Observability is limited to $\mathcal{A}(c)$,
- The ancestor chain is ordered and duplicate-free (by WF3),
- Node types provide static context,

we conclude that **a single string label $L$ is information-theoretically minimal**. It provides:
- **Behavioral disambiguation** within a node (e.g., `"init"` vs `"finalize"`),
- **Contextual resolution** across ancestors via type-aware lookup.

No global IDs, pointers, or program counters are needed.

### 4.2 Runtime Cost Analysis

| Component | Cost |
|---------|------|
| Label representation | Compile-time symbol → runtime integer ID (≤4 bytes) |
| Per-node state | Two small sets: `emitted: Set<Label>`, `pending_obsv: List<(Label, PC)>` |
| Matching overhead | On `emit`: traverse subtree pending lists (amortized O(1) in shallow trees) |
| Scheduling impact | No CPU burn; nodes remain unscheduled until causally ready |

This achieves **near-theoretical-minimum overhead** for cross-node coordination.

---

## 5. Language Realization in Arxil

### 5.1 Syntax Extension

We introduce two new generic opcodes in the Arxil language:

```arxil
// Emit a named causal commitment
emit ("label");

// Declare execution contingent on observing a label from an ancestor
obsv ("label");
```

These appear within `instruct` blocks like any other opcode.

### 5.2 Semantic Specification

- **`emit(L)`**:  
  Records that label $L$ has been emitted by the current node. Triggers a lightweight notification to all descendants with pending `obsv(L)`.

- **`obsv(L)`**:  
  Does not execute as a traditional instruction. Instead, it registers a causal readiness constraint: the node’s program counter advances only after some ancestor emits $L$. No state change (e.g., `blocked`) is introduced; the node simply remains unscheduled.

### 5.3 Placement in Arxil Opcode Taxonomy

`emit` and `obsv` are added to Appendix A.2 of the Arxil Language Specification as **Causal Control Opcodes**.

### 5.4 Naming Rationale

- **`emit`**: Emphasizes *active declaration* of completion, not passive signaling.
- **`obsv` **(observe): Highlights the *declarative, relativistic nature* of the dependency.  
  Avoids `await`, which implies imperative blocking—a concept alien to AS’s event-generation model.

The pair forms a **dual**: one *announces*, the other *conditions on announcement*.

---

## 6. Implementation Considerations

### 6.1 Node State Extension

Each node’s runtime state $\sigma(n)$ is extended with:
- $\mathcal{E}_{\text{emit}} \subseteq \mathcal{L}$: set of emitted labels,
- $\mathcal{P}_{\text{obsv}} \subseteq \mathcal{L} \times \mathbb{N}^+$: pending (label, PC) pairs.

### 6.2 Scheduler Integration

- On `emit(L)`: scheduler propagates $L$ down the subtree, resolving pending `obsv`.
- On `obsv(L)`: if $L$ is not yet observed in $\mathcal{A}(c)$, the node is marked *causally unready* and skipped during scheduling.

### 6.3 Static Verification

The compiler should:
- Warn if an `obsv(L)` has no statically reachable `emit(L)` in any ancestor path,
- Optimize label resolution using type and binding information.

---

## 7. Conclusion

Observation-Triggered Causality (OTC) is a native, minimal, and philosophically coherent solution to cross-node behavioral coordination in the Arbor Strux model. By grounding synchronization in the act of *observation* rather than *waiting*, and by leveraging the tree’s inherent structure for scoping and safety, OTC transcends the limitations of flat-memory concurrency primitives while preserving their essential semantics.

With just two opcodes—`emit` and `obsv`—and a single string label, OTC delivers:
- **Expressive power** equivalent to release/acquire ordering,
- **Runtime efficiency** approaching zero overhead,
- **Formal safety** guaranteed by AS’s well-formedness rules.

It embodies the AS ethos: computation is not about commanding time, but about declaring the causal conditions under which one is willing to act. In this light, OTC is not merely a feature—it is the natural language of time in a structured universe.

---

## References

1. *Arbor Strux Computational Model*, v2.1, 2025.  
2. *Arxil Language Specification*, v0.2.0, 2025.  
3. ISO/IEC 14882:2020, *Programming Languages — C++*.  
4. Adve, S. V., & Gharachorloo, K. (1996). *Shared Memory Consistency Models: A Tutorial*. IEEE Computer.

--- 

*End of Document*