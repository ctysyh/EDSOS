<!--
SPDX-FileCopyrightText: © 2025-2026 Bib Guake
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# System Overview

---

> v0.2.2

---

## 1 Introduction

Modern computing is increasingly defined by heterogeneity and distribution: multi-core CPUs, accelerators (GPUs, NPUs), disaggregated memory, and networked compute nodes are now the norm rather than the exception. Yet mainstream operating systems remain rooted in abstractions designed for single-machine, homogeneous environments. Processes, threads, and global virtual address spaces that obscure hardware reality behind layers of indirection.

This mismatch leads to fundamental limitations:
- **Poor composability**: Distributed applications must stitch together disparate APIs (POSIX, MPI, RPC) with inconsistent semantics.
- **Inefficient communication**: Cross-machine interaction is forced through message-passing or remote procedure calls, even when shared memory is physically possible.
- **Weak safety guarantees**: Concurrency control is largely left to application logic, resulting in pervasive race conditions and memory safety vulnerabilities.
- **Opaque resource management**: The OS hides hardware topology, making performance optimization and failure recovery reactive rather than structural.

**EDSOS (Explicitly Distributed Single Operating System) Framework** addresses these challenges by embracing distribution as a first-class concern and a core organizing principle instead of an afterthought to be abstracted away. It does so through a unified theoretical foundation: the **Arbor Strux (AS) Model**. Then a native language **Arxil** is designed as a faithful syntactic encoding of the AS model and acts as the structural execution model of **EDSOS Kernel**. Finally, as the comprehensive culmination of the whole framework, EDSOS Kernel is a general operating system mainly facing high performance server cluster environment. EDSOS Kernel also includes some contents of **Native-ISA**.

## 2 Theoretical basis: Arbor Strux (AS) Model

**Arbor Strux (AS) Model** is a formal computational framework that redefines concurrency and distribution through causal, tree-structured state machines.

Key aspects of the AS model include:
- **Causal Arbor Network**: Space is encoded as a dynamic tree topology; time emerges as a partial order over local events (no global clock).
- **Field Binding Chains**: Data sharing occurs via explicit paths from descendant nodes to public fields in ancestors, enabling zero-copy reference with analyzable dependency graphs. This derives some more properties and structures, includeing Permission Light Cone which is a detection model for reasoning about read/write permissions and race freedom, where concurrent accesses are safe if their “light cones” do not interfere causally.
- **Turing Completeness**: Despite its structural constraints, AS can simulate any Turing machine using conditional branching (`cond`), loops (`cycl`), and jumps (`exec`).

In engineering implementation, the AS model is linearly embedded into a one-dimensional Euclidean spacethe which is GVA/LVA space in EDSOS Kernel. This embedding is via the LENS (Linear Embedding of Node Structures with Skip Indexing) mechanism, with the original structural information of the AS model preserved via auxiliary data structures.

The AS model provides the mathematical foundation for EDSOS Framework’s safety, liveness, and expressiveness guarantees.

## 3 Language Interface: Arxil

Arxil is EDSOS Framework’s native intermediate language. It is designed as a **faithful syntactic encoding of the AS model**.

Key characteristics:
- Arxil source directly maps to AS operations (`psh`, `pvt`, `exec`, `cond`), making it suitable as input to the EDSOS loader.
- Arxil provides a textual representation of the Field Binding Chains, using the "=>" operator to describe how fields are bound from parent nodes into child nodes and supporting recursive binding to more distant descendants. This mechanism accurately realizes the concepts of the AS model while balancing implementation complexity.
- Functions (`fn`) are merely bounded, structured sequences of instructions without stack semantics. Parameters and return values serve as binding slots, with execution occurring in-place and without data copying. The `fn` somewhat analogous to C function-like macros.
- Mixed-language interoperability is supported via `'lang'` blocks (e.g., `'c'`, `'rust'`), subject to a lowering contract that ensures only declared bindings are accessed.
- Type information is externalized into `.arxtype` files, enabling rich compile-time semantics with zero runtime overhead.

Beyond this, significant efforts are being made to prove that Arxil constitutes one theoretically optimal solution for expressing parallel logic. Critically, this representation can be automatically derived by the compiler from arbitrary parallel program states following deterministic rules. Consequently, conservatively yet correctly translating existing code into Arxil suffices to enable automatic optimization to an optimal state within the framework.

Arxil serves as the bridge between high-level developer intent and the structural execution model of EDSOS Kernel much like C did for Unix, but grounded in formal semantics.

## 4 EDSOS Kernel

As its base, EDSOS Kernel implements the AS Model. EDSOS Kernel is built upon three interlocking design principles:

1. **Explicit Distribution**  
   Distribution is not hidden; it is made visible, programmable, and analyzable. Resources across machines are named in a single global namespace, and communication patterns reflect physical locality.

2. **Structural Execution**  
   Computation is organized as dynamically evolving tree structures (Arbor Strux), where content of its nodes completely covers execution, data and scope. Control flow (via structured jumps) and data flow (via field binding) are co-designed and unified within nodes, making each node can be independently executed in heterogeneous chips. This replaces the flat process/thread model with a hierarchical, self-describing structure.

3. **Loading-Time Centrality**  
   The loading phase is elevated to equal importance with compilation and runtime. Instead of exposing traditional syscall interfaces, EDSOS Kernel defines comprehensive stub functions that are automatically inlined into final machine instructions during the loading phase. This strategy addresses the syscall security issues commonly found in traditional operating systems, ensuring the encapsulation of EDSOS Kernel’s structure and functionality. The AS structure described by the loaded code also has its skeleton constructed during the loading phase.

EDSOS Kernel implements the AS model through a novel system architecture composed of the following layers:

1. Unified Virtual Address Space
   - All memory objects (nodes, buffers, devices) are named in a flat, 128-bit Global Virtual Address (GVA) space.
   - No central directory: name resolution is topology-adaptive and performed via distributed metadata (Near-Storage Metadata, NSM) and Path Cache inside dedicated DPU.
   - Local Virtual Address (LVA) provides a 64-bit view compatible with existing CPU MMUs; LVA and GVA share identical offsets for pointer portability.

2. Silicon Interposer
   - A minimal, privileged layer running on bare metal.
   - Handles kernel data structures management, interrupt delivery, and low-level context switching.
   - Does not appear as an AS node but provides all node instances an expecting environment.
   - Abstracts hardware dependencies, including DPU interfaces.
   - As an implementation of LENS, this layer allocates page table entries for each node instance mapped linearly into the one-dimensional virtual address space. It further implements cross-node field bindings—whenever possible—as shared physical mappings where distinct virtual addresses resolve to the same physical location.

3. Process Model Based on AS
   - The traditional “process” is replaced by an **Arbor Strux instance**—a tree of nodes.
   - Each node has its own instruction sequence, data fields, and execution state (`ready`, `running`, `blocked`, etc.).
   - Lifecycle operations (`psh`, `pop`, `pvt`, `mrg`) are atomic to user code and mediated by per-core schedulers.

4. Boot and Loading Flow
   - **Early Local View**: Each physical machine (PM) initializes independently, discovers local hardware, and prepares a transition image.
   - **Global Meta-AS Construction**: DPU-assisted election establishes a root and aggregates resources into a global hardware view.
   - **Capability-Guided Loading**: The loader instantiates AS trees, injects scheduling trampolines, applies capability policies, and pre-wires SPN (System Proxy Node) channels to services.

5. Hardware Dependency: DPU Acceleration
   - Distributed operations (GVA routing, cross-PM coherence) are accelerated by a dedicated Data Processing Unit (DPU).
   - In the absence of a dedicated DPU, EDSOS Kernel can still operate normally in single-machine mode and allows relevant services to be added to run on traditional cluster networks.

EDSOS Kernel delivers a unique combination of properties:

- **Scalable**: Plugin-friendly architecture supports future extensions (e.g., new consistency protocols, security modules).
- **Efficient**: Zero-copy sharing, asynchronous execution, and hardware-accelerated arbitration turn cross-machine communication into a lightweight thread-like operation.
- **Robust**: PM failures are recoverable; the AS structure maintains global state consistency through causal closure.
- **Expressive**: Simple applications use EDSOS Kernel “unaware” (via compatibility wrappers); performance-critical code gains fine-grained control over structure and placement.
- **Secure**: AS boundaries enforce strict isolation; root and system proxy nodes mediate all access to hardware and services, enabling easy insertion of security monitors or translation layers.
- **Compatible**: Legacy binaries can be encapsulated as single-node AS instances, ensuring “runnable” compatibility without compromising the native model.

---

## 5 Summary

This overview establishes EDSOS Framework as a **coherent rethinking of system software** from formal theory to hardware interface centered on structure, causality, and explicit distribution. The documents in each subdirectory of this repository elaborate on the details of each component.

---

*End of overview.*