<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# System Overview

---

> v0.2.1

---

## 1 Motivation

Modern computing is increasingly defined by heterogeneity and distribution: multi-core CPUs, accelerators (GPUs, NPUs), disaggregated memory, and networked compute nodes are now the norm rather than the exception. Yet mainstream operating systems remain rooted in abstractions designed for single-machine, homogeneous environments—processes, threads, and global virtual address spaces that obscure hardware reality behind layers of indirection.

This mismatch leads to fundamental limitations:
- **Poor composability**: Distributed applications must stitch together disparate APIs (POSIX, MPI, RPC) with inconsistent semantics.
- **Inefficient communication**: Cross-machine interaction is forced through message-passing or remote procedure calls, even when shared memory is physically possible.
- **Weak safety guarantees**: Concurrency control is largely left to application logic, resulting in pervasive race conditions and memory safety vulnerabilities.
- **Opaque resource management**: The OS hides hardware topology, making performance optimization and failure recovery reactive rather than structural.

EDSOS (Explicitly Distributed Single Operating System) addresses these challenges by embracing distribution as a first-class concern—not as an afterthought to be abstracted away, but as a core organizing principle. It does so through a unified theoretical foundation: the **Arbor Strux (AS) Model**.

## 2 Core Principles

EDSOS is built upon five interlocking design principles:

1. **Explicit Distribution**  
   Distribution is not hidden; it is made visible, programmable, and analyzable. Resources across machines are named in a single global namespace, and communication patterns reflect physical locality.

2. **Structural Execution**  
   Computation is organized as dynamically evolving tree structures (Arbor Strux), where content of the nodes cover execution, data, and scope. This replaces the flat process/thread model with a hierarchical, self-describing structure.

3. **Loading-Time Centrality**  
   The loading phase is elevated to equal importance with compilation and runtime. Instead of exposing traditional syscall interfaces, EDSOS defines comprehensive stub functions that are automatically inlined into final machine instructions during the loading phase. This strategy addresses the syscall security issues commonly found in traditional operating systems, ensuring the encapsulation of EDSOS’s structure and functionality. The AS structure described by the loaded code also has its skeleton constructed during the loading phase.

4. **Unified Control and Data Flow**  
   Control flow (via structured jumps) and data flow (via field binding) are co-designed within the AS model. Control logic corresponds to variable fields, and variable fields similarly correspond to control logic; both are unified within nodes.

## 3 Theoretical basis: Arbor Strux (AS) Model

At its core, EDSOS implements the **Arbor Strux (AS) Model**—a formal computational framework that redefines concurrency and distribution through causal, tree-structured state machines.

Key aspects of the AS model include:
- **Causal Arbor Network**: Space is encoded as a dynamic tree topology; time emerges as a partial order over local events (no global clock).
- **Field Binding Chains**: Data sharing occurs via explicit paths from descendant nodes to public fields in ancestors, enabling zero-copy reference with analyzable dependency graphs.
- **Permission Light Cone**: A geometric model for reasoning about read/write permissions and race freedom, where concurrent accesses are safe if their “light cones” do not interfere causally.
- **Turing Completeness**: Despite its structural constraints, AS can simulate any Turing machine using conditional branching (`cond`), loops (`cycl`), and jumps (`exec`).

The AS model provides the mathematical foundation for EDSOS’s safety, liveness, and expressiveness guarantees.

## 4 Language Interface: Arxil

Arxil is EDSOS’s native intermediate language—designed as a **faithful syntactic encoding of the AS model**.

Key characteristics:
- Arxil source directly maps to AS operations (`psh`, `lft`, `exec`, `cond`), making it suitable as input to the EDSOS loader.
- Functions (`fn`) are logical contracts: parameters and returns act as binding slots, not stack frames. Execution proceeds in-place with no data copying.
- Mixed-language interoperability is supported via `'lang'` blocks (e.g., `'c'`, `'rust'`), subject to a lowering contract that ensures only declared bindings are accessed.
- Type information is externalized into `.arxtype` files, enabling rich compile-time semantics with zero runtime overhead.

Arxil serves as the bridge between high-level developer intent and the structural execution model of EDSOS—much like C did for Unix, but grounded in formal semantics.

## 5 EDSOS Architecture

EDSOS realizes the AS model through a novel system architecture composed of the following layers:

### Unified 128-bit Global Virtual Address Space (GVA)
- All memory objects (nodes, buffers, devices) are named in a flat, 128-bit GVA space.
- No central directory: name resolution is topology-adaptive and performed via distributed metadata (Near-Storage Metadata, NSM) and Path Cache inside dedicated DPU.
- Local Virtual Address (LVA) provides a 64-bit view compatible with existing CPU MMUs; LVA and GVA share identical offsets for pointer portability.

### Silicon Interposer
- A minimal, privileged layer running on bare metal.
- Handles TLB management, interrupt delivery, and low-level context switching.
- Does not appear as an AS node; invoked only during loading or exceptional conditions.
- Abstracts hardware dependencies, including DPU interfaces.

### Process Model Based on AS
- The traditional “process” is replaced by an **Arbor Strux instance**—a tree of nodes.
- Each node has its own instruction sequence, data fields, and execution state (`ready`, `running`, `blocked`, etc.).
- Lifecycle operations (`push`, `pop`, `lift`, `merge`) are atomic and mediated by per-core schedulers.

### Boot and Loading Flow
1. **Early Local View**: Each physical machine (PM) initializes independently, discovers local hardware, and prepares a transition image.
2. **Global Meta-AS Construction**: DPU-assisted election establishes a root and aggregates resources into a global hardware view.
3. **Capability-Guided Loading**: The loader instantiates AS trees, injects scheduling trampolines, applies capability policies, and pre-wires SPN (System Proxy Node) channels to services (e.g., `/fs`, `/net`).

### Hardware Dependency: DPU Acceleration
- Distributed operations (GVA routing, cross-PM coherence) are accelerated by a dedicated Data Processing Unit (DPU).
- In the absence of a dedicated DPU, EDSOS can still operate normally in single-machine mode and allows relevant services to be added to run on traditional cluster networks.

## 6 Benefits

EDSOS delivers a unique combination of properties:

- **Scalable**: Plugin-friendly architecture supports future extensions (e.g., new consistency protocols, security modules).
- **Efficient**: Zero-copy sharing, asynchronous execution, and hardware-accelerated arbitration turn cross-machine communication into a lightweight thread-like operation.
- **Robust**: PM failures are recoverable; the AS structure maintains global state consistency through causal closure.
- **Expressive**: Simple applications use EDSOS “unaware” (via compatibility wrappers); performance-critical code gains fine-grained control over structure and placement—all while preserving binary compatibility.
- **Secure**: AS boundaries enforce strict isolation; root and system proxy nodes mediate all access to hardware and services, enabling easy insertion of security monitors or translation layers.
- **Compatible**: Legacy binaries can be encapsulated as single-node AS instances, ensuring “runnable” compatibility without compromising the native model.

---

This overview establishes EDSOS not merely as a new kernel, but as a **coherent rethinking of system software**—from formal theory to hardware interface—centered on structure, causality, and explicit distribution. The following chapters elaborate on each component in detail.

---

*End of Document.*