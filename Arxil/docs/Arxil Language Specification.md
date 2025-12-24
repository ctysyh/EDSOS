<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: Apache-2.0
-->

# Arxil Language Specification

---

> v0.2.2

---

- [Arxil Language Specification](#arxil-language-specification)
  - [1. Introduction](#1-introduction)
    - [1.1 Role and Scope](#11-role-and-scope)
    - [1.2 Relationship to the AS Model](#12-relationship-to-the-as-model)
    - [1.3 Document Conventions](#13-document-conventions)
    - [1.4 Relationship to Foundational and Formal Documents](#14-relationship-to-foundational-and-formal-documents)
    - [1.5 Document Scope and Responsibilities](#15-document-scope-and-responsibilities)
  - [2. Lexical Structure](#2-lexical-structure)
    - [2.1 Character Set](#21-character-set)
    - [2.2 Identifiers](#22-identifiers)
    - [2.3 Keywords](#23-keywords)
    - [2.4 Literals](#24-literals)
      - [Integer Literals](#integer-literals)
      - [String Literals](#string-literals)
    - [2.5 Comments](#25-comments)
  - [3. Syntactic Structure](#3-syntactic-structure)
    - [3.1 Top-Level Structure](#31-top-level-structure)
    - [3.2 Node Declaration](#32-node-declaration)
    - [3.3 MetaData](#33-metadata)
      - [3.3.1 Type References](#331-type-references)
      - [3.3.2 `causal` Block](#332-causal-block)
    - [3.4 Data Section](#34-data-section)
    - [3.5 Code Section](#35-code-section)
      - [3.5.1 Instruction Block (`instruct`)](#351-instruction-block-instruct)
      - [3.5.2 Function Blocks (`fn`)](#352-function-blocks-fn)
      - [3.5.3 Language Blocks (`'lang'`)](#353-language-blocks-lang)
  - [4. Type System Interface](#4-type-system-interface)
    - [4.1 The Role of `.arxtype`](#41-the-role-of-arxtype)
    - [4.2 Type Annotations in Arxil](#42-type-annotations-in-arxil)
      - [Field Declarations (in `data` blocks)](#field-declarations-in-data-blocks)
      - [Function Parameter and Return Lists](#function-parameter-and-return-lists)
    - [4.3 Linking and Resolution](#43-linking-and-resolution)
    - [4.4 Pointer Semantics and Arithmetic](#44-pointer-semantics-and-arithmetic)
      - [4.4.1 Pointer as a Type Capability](#441-pointer-as-a-type-capability)
      - [4.4.2 Pointer Arithmetic Rules](#442-pointer-arithmetic-rules)
      - [4.4.3 Pointer Dereferencing Model](#443-pointer-dereferencing-model)
  - [5. Design Rationale and Usage Patterns *(Informative)*](#5-design-rationale-and-usage-patterns-informative)
    - [5.1 Philosophy Recap: Syntax Freedom, Semantic Closure](#51-philosophy-recap-syntax-freedom-semantic-closure)
    - [5.2 The `fn` as a Logical Contract, Not a Function Call](#52-the-fn-as-a-logical-contract-not-a-function-call)
    - [5.3 Structured Sharing with `ance`: The CARN Pattern](#53-structured-sharing-with-ance-the-carn-pattern)
    - [5.4 Integrating Legacy Code via `'lang'` Blocks and `.arxtype` Wrappers](#54-integrating-legacy-code-via-lang-blocks-and-arxtype-wrappers)
  - [Appendix A. About Arxil opcodes](#appendix-a-about-arxil-opcodes)
    - [A.1 Ordinary Opcodes](#a1-ordinary-opcodes)
      - [A.1.1 Mandatory Specification Requirements](#a11-mandatory-specification-requirements)
      - [A.1.2 Recommended Best Practices](#a12-recommended-best-practices)
      - [A.1.3 Interruptible Execution Constraint](#a13-interruptible-execution-constraint)
    - [A.2 Generic Opcodes](#a2-generic-opcodes)
      - [A.2.1 `exec` — Execute a Logical Contract](#a21-exec--execute-a-logical-contract)
      - [A.2.2 `cond` — Multi-way Conditional Dispatch](#a22-cond--multi-way-conditional-dispatch)
      - [A.2.3 `cycl` — Controlled Iteration Loop](#a23-cycl--controlled-iteration-loop)
      - [A.2.4 `emit` — Emit a Named Causal Commitment](#a24-emit--emit-a-named-causal-commitment)
      - [A.2.5 `obsv` — Declare a Causal Readiness Constraint](#a25-obsv--declare-a-causal-readiness-constraint)
      - [A.2.6 `wait` — Block Until Signaled](#a26-wait--block-until-signaled)
      - [A.2.7 `sgnl` — Signal a Blocked Node](#a27-sgnl--signal-a-blocked-node)
      - [A.2.8 `yiel` — Voluntary Yield](#a28-yiel--voluntary-yield)
      - [A.2.9 `fnsh` — Terminate Node](#a29-fnsh--terminate-node)
      - [A.2.10 `warn` — Emit Diagnostic Warning](#a210-warn--emit-diagnostic-warning)
      - [A.2.11 `gerf` — Get Reference](#a211-gerf--get-reference)
      - [A.2.12 `derf` — Dereference](#a212-derf--dereference)
    - [A.3 Structural Opcodes](#a3-structural-opcodes)
      - [A.3.1 `psh` — Push a New Child Node](#a31-psh--push-a-new-child-node)
      - [A.3.2 `pop` — Remove a Child Node](#a32-pop--remove-a-child-node)
      - [A.3.3 `lft` — Lift an External Node into the Ancestry Chain](#a33-lft--lift-an-external-node-into-the-ancestry-chain)
      - [A.3.4 `mrg` — Merge a Child Node into the Current Node](#a34-mrg--merge-a-child-node-into-the-current-node)
      - [A.3.5 `dtc` — Detach a New Node by Slicing Off Resources](#a35-dtc--detach-a-new-node-by-slicing-off-resources)


---

## 1. Introduction

### 1.1 Role and Scope

This document, the *Arxil Language Specification*, defines the official textual syntax for expressing programs that conform to the Arbor Strux (AS) computational model. It serves as the primary interface between human developers (and tooling) and the AS runtime system.

The scope of this specification is strictly limited to the **lexical and syntactic structure** of the Arxil language, along with the **design intent** behind its constructs. It describes *how* to write a valid Arxil program but does not redefine the underlying computational semantics, memory model, or formal guarantees of the AS system. Those foundational aspects are rigorously defined in separate documents: *AS Computational Model* and *AS Formal Semantics and Verification*.

In essence, this specification answers the question: “What is the correct way to write source code that instructs a AS-compliant system to build, evolve, and query a structured computation tree?”

### 1.2 Relationship to the AS Model

Arxil is not a conventional programming language that describes a sequence of imperative actions on a flat memory space. Instead, it is a **declarative blueprint** for constructing and manipulating instances of the AS model. Each major syntactic element in Arxil directly corresponds to a core concept in the AS model:

| Arxil Syntax Element          | AS Model Concept                                  |
| --------------------------- | ------------------------------------------------- |
| `node` declaration          | A AS Node — the fundamental unit of structure and execution. |
| `data { ance / publ / priv }` | The node’s field environment (`𝒟`), which holds its state with explicit visibility and sharing semantics. |
| `code { instruct }` and `fn` | The node’s instruction sequence (`ℐ`) and control flow logic, executed within the node’s context. |
| Operations like `psh`, `pop`, `lft`, `exec` | Atomic structural operations that drive the evolution of the AS tree, as formally specified in the AS operational semantics. |

Understanding this mapping is crucial: writing Arxil is not about issuing commands to a CPU, but about **declaring how a dynamic, hierarchical structure should be composed and how it should transform itself over time**.

### 1.3 Document Conventions

The following typographical conventions are used throughout this specification:

- **Keywords** (e.g., `node`, `data`, `publ`) appear in monospace font and are part of the Arxil lexical grammar.
- *Non-normative notes*, such as design rationale or usage advice, are presented in italicized paragraphs and do not affect the formal meaning of the language.
- Normative syntax is defined using Extended Backus-Naur Form (EBNF), with terminal symbols enclosed in double quotes (e.g., `"node"`).
- References to other specifications (e.g., *AS Computational Model*) are provided where deeper semantic understanding is required.

All examples in this document are illustrative and may omit certain details (such as full type annotations) for clarity, but they adhere to the grammar defined herein.

### 1.4 Relationship to Foundational and Formal Documents

The Arxil Language Specification defines the syntactic surface of the language, but its meaning is deeply rooted in a set of foundational computational and formal models. This specification assumes the reader's familiarity with these underlying concepts and serves as a bridge between high-level source code and their precise, low-level semantics.

*   **AS Computational Model**: The fundamental execution paradigm of Arxil is the Arbor Strux (AS) computational model. Every `node` declaration, every structural operation (`psh`, `pop`, `lft`), and the very notion of state residing in fields are direct textual representations of entities and processes defined in the *AS Computational Model* document. This model establishes the core principles of "computation as structural evolution," node-based concurrency, and structured sharing.
*   **Formal Semantics**: The exact, step-by-step behavior of all Arxil operations—both structural (like `Lift(n)`) and data-manipulating (like field resolution)—is formally specified using Separation Logic and operational semantics in the *AS Formal Semantics and Verification* document. This document provides the mathematical proof of correctness for the AS model and, by extension, for well-formed Arxil programs.
*   **Field Resolution and Type System**: The mechanics of how field names are resolved at runtime (especially `ance` fields) and how types govern operations and memory layout are detailed in the *Arxil Field Resolution* and *Arxil Type System Architecture* documents, respectively. These specifications explain the two-phase pointer dereferencing process, the binding chain fulfillment model, and the contract-based nature of the `.arxtype` system.

This specification **does not redefine** these foundational concepts. Instead, it leverages them to prescribe the correct way to write Arxil source code that faithfully encodes them.

### 1.5 Document Scope and Responsibilities

To maintain clarity and focus, the responsibilities of this document are strictly delineated from those of its companion specifications:

*   **This Document **(Arxil Language Specification):
    *   Defines the official lexical and syntactic grammar of Arxil (`.axl` files).
    *   Specifies the structure of type interface files (`.arxtype`).
    *   Describes the static (compile-time) rules for program well-formedness, including scoping, name resolution, and basic type compatibility checks based on `.arxtype` contracts.
    *   Provides high-level descriptions of the intended runtime effects of language constructs, always with reference to the formal documents for precision.

*   **Excluded from This Document**:
    *   **Formal Operational Semantics**: The precise, mathematical definition of how each Arxil instruction transforms the program state is the domain of the *AS Formal Semantics and Verification* document.
    *   **Proofs of Correctness**: Arguments for the safety, liveness, or Turing-completeness of the AS model are contained within the formal verification literature.
    *   **Runtime Implementation Details**: The specifics of how a Arxil Virtual Machine (ArxilVM) or an EDSOS kernel schedules nodes, manages virtual address spaces, or handles traps are outside the scope of this language specification.

---

## 2. Lexical Structure

This chapter defines the lexical grammar of Arxil, specifying how source text is decomposed into a sequence of tokens that form the input to the syntactic parser. All lexical rules are derived from the reference EBNF specification.

### 2.1 Character Set

Arxil source files must be encoded in UTF-8. The language uses only ASCII characters for its core syntax (keywords, operators, delimiters). Non-ASCII Unicode characters may appear within string literals or comments but must not be used in identifiers.

### 2.2 Identifiers

Identifiers are used to name nodes, fields, functions, and types. They are defined by the following rule:

`IDENT       = { LETTER | DIGIT | "_" | "." } ;`

Identifiers are case-sensitive and may contain letters, digits, underscores (`_`), and dots (`.`). The dot character is primarily intended for hierarchical naming (e.g., `hardware.mmio_reg`).

### 2.3 Keywords

Arxil reserves the following keywords, which cannot be used as identifiers:

```
node, meta_data, data, code,
imme, futu,
ance, publ, priv,
instruct, inst, fn, inst_scri, reserve, resv,
type, size, layout, ops, interop, validators, docs, hints,
psh, pop, lft, mrg, dtc,
actv, sgnl, wait, yiel, fnsh, warn,
exec, cond, cycl, gerf, derf,
native, asm, libcall,
this,
true, false
```

Note: Some keywords (e.g., `type`, `size`) belong to the `.arxtype` language and appear in Arxil only via type annotations or embedded constructs.

### 2.4 Literals

#### Integer Literals
Integer literals consist of one or more decimal digits:

`INTEGER = DIGIT { DIGIT };`

Hexadecimal, octal, or binary literals are not part of the core Arxil lexical grammar; such representations must be handled by frontend plugins if needed.

#### String Literals
String literals are sequences of characters enclosed in double quotes. Escape sequences are supported:

`STRING = '"' { CHAR_NO_QUOTE_OR_ESCAPE | ESCAPE_SEQ } '"';`  
`ESCAPE_SEQ = "\" ("n" | "t" | "\"" | "\" | "0");`

Valid escape sequences include:
- `\n` (newline)
- `\t` (tab)
- `\"` (double quote)
- `\\` (backslash)
- `\0` (null character)

### 2.5 Comments

Arxil supports C++-style line comments:

```axl
// This is a comment
```

Comments begin with `//` and extend to the end of the line. Block comments (`/* ... */`) are **not** supported in the base Arxil lexical grammar.

---

> **Note**: The complete lexical definitions (including auxiliary rules like `LETTER` and `DIGIT`) are formally specified in the *Arxil EBNF Specification* document under the section “Lexical Elements”. This chapter serves as a human-readable summary for language users and tool implementers.

---

## 3. Syntactic Structure

This section defines the concrete syntax of the Arbor Strux Language (Arxil). The grammar is presented in Extended Backus–Naur Form (EBNF), as specified in the companion document *Arxil EBNF Specification*. All terminal symbols are enclosed in double quotes (`""`); non-terminals are capitalized; square brackets `[X]` denote optional components; and braces `{X}` denote zero or more repetitions.

### 3.1 Top-Level Structure

A Arxil program consists of a sequence of node declarations. There is no implicit global scope—every construct must reside within a declared node.

```ebnf
ArxilProgram = { NodeDecl } ;
```

> **Note (Non-Normative):**  
> A Arxil program describes a static template for constructing a tree of runtime nodes. Execution begins by instantiating one or more root nodes, typically defined in the program.

### 3.2 Node Declaration

Each node is introduced by the `node` keyword followed by an identifier and a body enclosed in braces. The body may contain three optional sections: `meta_data`, `data`, and `code`, in that order.

```ebnf
NodeDecl = "node", IDENT, "{",
             [MetaData],
             [DataSection],
             [CodeSection],
           "}" ;
```

The sections serve distinct purposes:
- `meta_data`: Declares compile-time metadata (e.g., type references) and reserved runtime fields.
- `data`: Defines the node’s structured state.
- `code`: Contains executable logic.

> **Note (Non-Normative):**  
> The syntactic ordering of sections reflects the stable ABI layout of a runtime node, ensuring deterministic memory organization.

### 3.3 MetaData

#### 3.3.1 Type References

<TODO>

#### 3.3.2 `causal` Block

The `causal` block in `meta_data` is optional that depending to whether two of Generic Opcodes `emit` or `obsv` is used inside the node or is expected to appear in descendant nodes.

```ebnf
CausalDecl = "causal", "{", { CausalList }, "}" ;

CausalList = ("emits" | "obses"), "{", CausalLabel, { ",", CausalLabel }, "}" ;

CausalLabel = STRING ;
```

### 3.4 Data Section

> The `data` section declares the persistent state of the node, partitioned into two temporal categories (`imme` for immediate, `futu` for future-resolved) and three visibility categories (`ance`, `publ`, `priv`). Only the visibility categories affect name resolution and sharing semantics.

The `data` section of a node declaration defines its persistent state, organized into fields with explicit visibility and sharing semantics. The structure of this section is as follows:

```ebnf
DataSection = "data", "{", { DataBlock }, "}" ;

DataBlock = ("imme" | "futu"), "{", { FieldBlock }, "}" ;

FieldBlock = ("ance" | "publ" | "priv"), "{", { FieldDecl }, "}" ;

FieldDecl = TypeSpec, [ArraySize], IDENT, ";" ;

ArraySize = "[", INTEGER, "]" ;
```

Each field declaration consists of:
- A **type specifier** (`TypeSpec`), which refers to an external `.arxtype` definition (see §4).
- An optional **array size** (currently restricted to compile-time constants).
- An **identifier** naming the field.

Fields are categorized along two orthogonal dimensions: **temporal scope** (`imme` for immediate values, `futu` for future values) and **visibility scope** (`priv`, `publ`, `ance`). Each field must be declared within exactly one of the two temporal categories and one of the three visibility categorie.

*   **`priv` Fields**: These fields constitute the node's private state. They are accessible only from within the node's own `code` section and are invisible to any ancestor or descendant nodes. They are ideal for encapsulating internal implementation details.

*   **`publ` Fields**: These fields represent data owned by the current node that can be explicitly shared with descendant nodes. A `publ` field serves as the **physical source** for data sharing. Its value resides in the memory layout of the declaring node.

*   **`ance` Fields**: This is a cornerstone of the AS model's safe sharing mechanism. An `ance` field **does not allocate any storage**. Instead, it declares a *binding promise*—a contract stating that, at runtime, this field will be linked to a `publ` field of another node (typically an ancestor or a designated data carrier).

    The resolution of an `ance` field reference is a deterministic process. At compile time, the compiler constructs a binding dependency graph to verify type consistency across the entire chain. At runtime, any access to an `ance` field triggers a recursive lookup that either:
    1.  Successfully resolves to a unique physical storage location `(m, f_publ)`, where `f_publ` is a `publ` field of some node `m`, enabling zero-copy access; or
    2.  Fails if the promise remains unfulfilled (i.e., no `lft` operation has anchored the chain to a `publ` field), resulting in a runtime error or a blocking state depending on the execution policy.

    This formal resolution procedure, including its termination and uniqueness guarantees, is defined in detail in the **Arxil Field Resolution** document under the function `Resolve(n, f)`.

All field names within a single node's `data` section **MUST** be unique, regardless of their category (`priv`/`publ`/`ance`). The type of each field, specified by `TypeSpec`, is resolved against the type environment established by the referenced `.arxtype` files (see Section 4).

### 3.5 Code Section

The `code` section contains all executable content of the node, organized into instruction blocks and named functions.

```ebnf
CodeSection = "code", "{",
                [InstructBlock],
                [AnceBlock],
                [PublBlock],
                [PrivBlock]
              "}" ;

InstructBlock = "instruct", "{", { InstDecl }, "}" ;
AnceBlock     = "ance",     "{", { FnDecl }, "}" ;
PublBlock     = "publ",     "{", { FnDecl }, "}" ;
PrivBlock     = "publ",     "{", { FnDecl }, "}" ;
```

#### 3.5.1 Instruction Block (`instruct`)

The `instruct` block holds the main linear sequence of operations executed when the node is scheduled. Each instruction consists of an opcode and two operand lists for source and destination. Their execution is controlled by `pc` (program counter) of the node.

```ebnf
InstructBlock = "instruct", "{", { OrdnInst | ResvInst }, "}" ;

OrdnInst = OpCode, OperTarg, OperGoal, ";" ;

OpCode = IDENT ;

OperTarg = "(", Operand, { Operand }, ")" ;
OperGoal = "(", Operand, { Operand }, ")" ;

Operand = "(" FieldRef ")";
FieldRef = IDENT | "derf" "(" IDENT ")";
```

Operands refer to field names declared in the node’s `data` section. Parentheses around an identifier in an operand (e.g., `(x)`) are **ALWAYS** required.

*   **Ordinary Opcodes**: Opcodes like `add`, `mul`, `and`, and `cpy` are Ordinary Opcodes. Their legality and precise runtime behavior are **entirely determined by the types of their operands in the scope of the data type of the `OperTarg`**. The compiler consults the `.arxtype` definitions to verify that the requested operation is supported (i.e., listed in the type's `ops` set) and to generate the correct low-level code. Arxil specifies 12 *recommended* Ordinary Opcodes, along with best practice guidelines for any custom Ordinary Opcodes, as detailed in Appendix A.1.

*   **Generic Opcodes**: Some operators are part of Arxil's reserved language operators, including `exec`, `cond`, `cycl` and so on. They have special meanings and formats, falling under the exceptional cases difined here, as detailed in Appendix A.2. Expression `derf` is also as detailed there.

*   **Structural Opcodes**: Certain opcodes directly correspond to atomic operations on the AS tree structure itself. Their static and dynamic semantics are tightly coupled to the AS Computational Model. A complete list of these opcodes is provided in Appendix A.3.

#### 3.5.2 Function Blocks (`fn`)

Named instruction subsequences are declared using the `fn` keyword. A function has an explicit parameter list and return list, which serve as a *binding contract*—not as runtime stack frames.

```ebnf
FnDecl = "fn", IDENT,
         "(", [ParamList], ")", "=>", "(", [RetList], ")",
         ( "{", FnBody, "}" | ";" ) ;

ParamList = Param, { ",", Param } ;
RetList   = Param, { ",", Param } ;

Param = "(", TypeSpec, ")", IDENT ;

FnBody = { LangBlock } ;
```

> **Note (Normative):**
> A function may be defined inline (with a body) or left abstract (terminated by `;`), typically for external library bindings.

The parameter and return lists of a `fn` serve as **binding slots**. When a `fn` is invoked via the `exec` instruction, the caller provides concrete field names from its own context to bind to these slots. Execution then jumps directly to the `fn`'s instruction sequence, operating on the caller's original data without any stack frame creation or data copying.

#### 3.5.3 Language Blocks (`'lang'`)

To support interoperability and developer ergonomics, function bodies may embed code from other languages within quoted language blocks.

```ebnf
LangBlock = [ "@lowered" ], "'", LangTag, "'", "{", LangCode, "}" ;

LangTag = "inst_scri" | "c" | "cpp" | "rust" | IDENT ;

LangCode = { ANY_CHAR_EXCEPT_CURLY_BRACE_NESTED } ;
```

The `LangTag` identifies the embedded language dialect. The contents of `LangCode` are opaque to the Arxil parser but must adhere to the following contract:

> **Lowering Contract (Normative):**
> The embedded code must be translatable by a registered frontend plugin into a sequence of Arxil instructions that reference **only** the symbols declared in the enclosing function’s parameter and return lists. Once lowered, the block should be marked with the `@lowered` annotation to indicate it is ready for execution by the ArxilVM or EDSOS runtime.

> **Note (Non-Normative):**
> The `'inst_scri'` dialect is reserved for direct Arxil instruction sequences and requires no lowering.

---

## 4. Type System Interface

The Arxil language does not embed a built-in type system. Instead, it interfaces with an external, declarative type description mechanism based on `.arxtype` files. This design enables zero-runtime-overhead execution while preserving rich compile-time semantics for layout, operation selection, and interoperability.

### 4.1 The Role of `.arxtype`

All types referenced in Arxil source code are defined externally in `.arxtype` files. A `.arxtype` file provides a complete, machine-readable specification of a type’s:
- Memory layout (size, field offsets),
- Supported operations (e.g., `add`, `mul`, `set`),
- Interoperability rules (e.g., implicit conversions, pointer compatibility),
- Validation constraints (compile-time and runtime),
- Documentation and optimization hints.

The Arxil compiler consumes these `.arxtype` definitions during compilation to:
- Compute field offsets within node data segments,
- Validate that applied operations are supported by the operand types,
- Generate correct lowering sequences (e.g., intrinsic calls, assembly mappings),
- Enforce structural safety guarantees derived from the AS computational model.

> **Note**: The syntax and semantics of `.arxtype` files are defined in a separate document, *The `.arxtype` Language Specification*. This section only describes how Arxil *uses* those definitions.

### 4.2 Type Annotations in Arxil

In Arxil source code, types appear exclusively as **annotations** on fields and function parameters/returns. The syntax for a type annotation is:

```
(TypeName) identifier
```

This form is used in two contexts:

#### Field Declarations (in `data` blocks)
```axl
data {
    ance {
        BufferHandle shared_buffer;
    }
    publ {
        u32 counter;
        Vec3f position;
        hardware.mmio_reg control_reg;
    }
}
```

Here, `u32`, `Vec3f`, and `hardware.mmio_reg` are identifiers resolved against available `.arxtype` definitions. They are used without parentheses.

#### Function Parameter and Return Lists
```axl
fn compute_hash ((BufferHandle) buf, (u64) len) => ((u64) hash) {
    // ...
}
```

These annotations serve as **interface contracts**. They do not introduce local storage; instead, they declare the expected type of the bound field at the call site. The Arxil compiler uses these annotations to:
- Verify type compatibility during `exec` or `'lang'` block lowering,
- Select appropriate opcodes or library calls (via the `.arxtype`’s `ops` section).

### 4.3 Linking and Resolution

When the compiler encounters a type name such as `int32_t` or `hardware.mmio_reg`, it searches for a corresponding `.arxtype` file. Programmers can also declare the file path of the type in `meta_data`.

The resolution process yields a **type descriptor** containing all metadata defined in the `.arxtype` file. This descriptor is then used throughout compilation for:
- **Layout computation**: Determining the byte/bit size and alignment of fields.
- **Opcode validation**: Ensuring that an instruction like `add %x %y %z` is only emitted if the `.arxtype` for the underlying type includes an `add` operation in its `ops` block.
- **Interoperability checking**: Validating cross-type assignments or bindings (e.g., whether a `(u32)` can be implicitly converted to `(i64)` per the `interop` rules).

Crucially, **no type information is retained at runtime**. The generated ArxilVM bytecode or native code contains only raw memory accesses and operation dispatches—no type tags, vtables, or dynamic checks. All safety and correctness must be ensured at compile time through the `.arxtype` contract.

> **Design Principle**: *Types are compile-time contracts, not runtime entities.* This aligns with Arxil’s goal of providing bare-metal performance while enabling high-level reasoning through external verification tools and disciplined composition.

### 4.4 Pointer Semantics and Arithmetic

In Arxil, pointers are not a primitive language construct but a **derived capability** granted to specific types through their `.arxtype` definition. The creation, manipulation, and dereferencing of pointers are governed entirely by the type system contract.

#### 4.4.1 Pointer as a Type Capability
A type `T` can support pointer operations if and only if its `.arxtype` file explicitly declares this capability. This is done in the `interop` section by defining a corresponding pointer type, for example:
```arxtype
// In i32.arxtype
interop {
    pointer_type = ptr_i32;
}
```
The existence of `ptr_i32` (itself a type with its own `.arxtype` definition) enables the use of the address-of opcode (`gerf`) on fields of type `i32` and allows opcodes like `set` and `get` to operate on values of type `ptr_i32`. Generic Opcode `gerf` is as detailed in Appendix A.2.11.

#### 4.4.2 Pointer Arithmetic Rules
Pointer arithmetic (e.g., `p + k`) is permitted if the pointer's underlying `.arxtype` defines the necessary operations (`add`, `sub`) in its `ops` block and includes the `pointer_arithmetic` property. The semantics of the arithmetic—specifically, whether the integer offset `k` is scaled by the size of the pointed-to type—are defined by this property.

For instance, the `.arxtype` for `ptr_i32` might contain:
```arxtype
// In ptr_i32.arxtype
ops {
    add {
        pointer_arithmetic = true; // Implies scaling by sizeof(i32)
        magnification_times = 4;   // Equal to sizeof(i32)
        // ... other implementation details (native, asm, etc.)
    }
}
```
This contract ensures that `p + 1` advances the pointer by 4 bytes (the size of an `i32`), providing array-like semantics.

#### 4.4.3 Pointer Dereferencing Model
Dereferencing a pointer follows a strict two-phase resolution process to maintain memory safety and structural integrity:
1.  **Local Field Location**: The virtual address embedded in the pointer value is resolved to a specific field declaration within its source node's virtual address space.
2.  **Remote Binding Resolution**: If the located field is an `ance` field, the standard binding resolution protocol is invoked to find its ultimate physical storage location in a `publ` field of another node.

This formal process guarantees that every valid pointer dereference ultimately accesses a well-defined, writable memory location. The complete formal definition of this process is provided in the **Arxil Language Formal Semantics and Verification** document under the section "Pointer Value Resolution: `Deref(p)`".

Ultimately, the dereference operation correctly returns the memory location of the final, physical field. In the Arxil source code, this dereference semantics is specifically carried by Generic Opcode `derf`, as detailed in Appendix A.2.12. Each dereferencing access to the pointer field must explicitly trigger this two-stage parsing process through the derf instruction.

---

## 5. Design Rationale and Usage Patterns *(Informative)*

This section explains the underlying motivations behind key syntactic and semantic choices in Arxil. It is non-normative—its purpose is to guide correct usage, foster intuition, and illustrate idiomatic patterns that leverage the full power of the Arbor Strux computational model.

### 5.1 Philosophy Recap: Syntax Freedom, Semantic Closure

Arxil embodies a dual-layer design principle:

> **“Syntax freedom, semantic closure.”**

At the surface level, Arxil permits expressive flexibility: users may embed familiar high-level languages (e.g., C, Rust) within `'lang'` blocks, define rich type interfaces via `.arxtype` files, and organize logic using named instruction blocks (`fn`). This lowers the barrier to adoption and enables reuse of existing codebases.

However, beneath this syntactic freedom lies a strictly closed semantic model. Every construct must ultimately compile down to a sequence of **pure Arxil instructions** that:
- Operate only on fields declared in the current node’s `data` section,
- Respect the structural boundaries of the AS tree,
- Never introduce hidden state or implicit control flow.

This duality ensures that while developers enjoy ergonomic syntax, the resulting program remains analyzable, verifiable, and faithful to the AS model’s guarantees of safety, determinism, and structured concurrency.

### 5.2 The `fn` as a Logical Contract, Not a Function Call

In traditional languages, a function call implies stack allocation, parameter copying, and return-value handling. In Arxil, a `fn` declaration is fundamentally different:

```axl
fn add_int ((i32)a, (i32)b) => ((i32)result) { ... }
```

Here, `a`, `b`, and `result` are **not local variables**—they are *logical binding slots*. When invoked via `exec`, the caller explicitly maps concrete fields from its own `data` section (or ancestors’) onto these slots:

```axl
exec ((x, y) (sum)) add_int;
```

This binds `x → a`, `y → b`, and `sum → result`. No new stack frame is created; execution simply jumps to the labeled instruction block within the same node. The `fn` body operates directly on the caller’s storage.

This design enables:
- **Zero-overhead abstraction**: No calling convention, no register saving.
- **Composability**: Any field compatible with the slot’s type can be bound.
- **Clarity**: All data dependencies are explicit at the call site.

> **Best Practice**: Use `fn` to encapsulate reusable logic that operates on well-defined input/output contracts. Avoid treating it like a traditional subroutine with side effects beyond its declared interface.

### 5.3 Structured Sharing with `ance`: The CARN Pattern

The `ance` field category is central to Arxil’s approach to safe, zero-copy data sharing. Unlike pointers or global variables, `ance` fields declare *intent* without assuming implementation—they are promises to be fulfilled later via structural operations.

A canonical pattern is the **Cross Arbor Strux Reference Node (CARN)**:

1. A dedicated node (e.g., `buffer_carn`) is created with only `publ` fields:
   ```axl
   node buffer_carn {
       data { publ { u8[4096] payload; } }
   }
   ```

2. Other nodes declare `ance` fields expecting such data:
   ```axl
   node consumer {
       data { ance { u8[4096] input_buffer; } }
   }
   ```

3. During execution, a parent node uses `psh` to pre-bind and `lft` to fulfill:
   ```axl
   psh c (consumer () (my_buffer => input_buffer));
   lft buf_node ((payload => my_buffer));
   ```

After `lft`, all descendants bound to `my_buffer` transparently access `buf_node.payload`—no copying, no pointer arithmetic, no dangling references.

This pattern scales naturally across trees, supports dynamic reconfiguration, and integrates seamlessly with reference counting for automatic lifetime management.

> **Anti-Pattern**: Declaring shared state in `priv` or relying on global naming conventions. Always use `publ` + `ance` + explicit binding for inter-node communication.

### 5.4 Integrating Legacy Code via `'lang'` Blocks and `.arxtype` Wrappers

Arxil does not require rewriting the world. Instead, it provides a structured pathway to integrate legacy components:

1. **Wrap external types** in `.arxtype` files, specifying size, layout, and available operations:
   ```arxtype
   type MyLegacyStruct {
       name = "MyLegacyStruct";
       size = 64 bytes;
       ops { sin { libcall = "__legacy_sin_f64"; } }
   }
   ```

2. **Embed legacy logic** in `'lang'` blocks inside `fn` definitions:
   ```axl
   fn compute ((f64)x) => ((f64)y) {
       'c' {
           y = legacy_math_library_sin(x);
       }
   }
   ```

3. The frontend compiler **lowers** this block into a sequence of Arxil instructions that respect the `fn`’s contract (only reading `x`, only writing `y`).

This approach allows gradual migration: legacy code runs unchanged inside a “AS shell,” while new code benefits from structural safety and concurrency primitives. Over time, performance-critical or safety-sensitive parts can be rewritten natively in Arxil instruct.

> **Key Contract**: The content of any `'lang'` block must be lowered into instructions that reference **only** the symbols declared in the enclosing `fn`’s parameter and return lists. Violations break semantic closure and are rejected at compile time.

---

## Appendix A. About Arxil opcodes

### A.1 Ordinary Opcodes

Ordinary Opcodes are user-defined, type-bound data transformation primitives that operate exclusively on node fields and immediate literals. They represent pure, side-effect-free computations whose semantics are fully determined by the static types of their operands and the target platform’s capabilities. Unlike structural or control-flow instructions, Ordinary Opcodes do not alter the AS tree topology, scheduling state, or execution context.

Each Ordinary Opcode is declared within the `ops` block of a `.arxtype` file and invoked in Arxil source code via an `InstDecl` of the form:

```axl
OpCode OperTarg OperGoal;
```

where:
- **OperTarg** (operation target) is a parenthesized list of field identifiers that receive the result(s) of the computation (the *write set*),
- **OperGoal** (operation goal) is a parenthesized list of field identifiers and/or integer literals that provide input values (the *read set*).

This read-write separation originates from the small-step operational semantics of the AS model: an instruction computes a deterministic value from its *OperGoal* and writes the outcome to its *OperTarg*. This model enables precise alias analysis, memory safety verification, and structured concurrency reasoning.

#### A.1.1 Mandatory Specification Requirements

Every custom Ordinary Opcode **must** explicitly define the following in its `.arxtype` declaration:

1. **Name**
   - Must be a valid Arxil identifier (see §2.2).  
   - Recommended naming convention: `TypeName.opName` (e.g., `Matrix4x4.inv`). The dot (`.`) is part of the name string and carries no syntactic meaning; it serves only as a human-readable logically namespace delimiter.

2. **Operand Signature**
   - Must specify one or more *overloads*, each defining:
     - An ordered list of operands types, which may include:
       - Concrete type references (e.g., `(int32_t)`),
       - The special pseudo-type `integer` to match compile-time integer literals.

3. **Target Implementations**
   - For each supported backend, *expected to* provide either:
     - A native assembly template (with placeholders `@asm`), or
   - A default implementation expressed as a sequence of standard type set `instmm` statements.
   - If an operation is atomic on a given platform, the corresponding native entry **must** be prefixed with `@atomic`.

4. **Type Consistency Rule**
   - At compile time, the frontend resolves an opcode by:
     1. Identifying the static type $T$ of the first field in `OperTarg`,
     2. Loading the `.arxtype` file for $T$,
     3. Selecting the first overload whose `inputs` match the actual operand types (fields → exact type match; literals → match `integer`).
   - Mismatch results in a compilation error.

#### A.1.2 Recommended Best Practices

While not mandatory, the following greatly enhance correctness, portability, and developer experience:

- Provide a small-step operational semantics rule (informative).
- Document behavior for mixed-type or variable-arity scenarios via multiple overloads.
- Enumerate expected error conditions (e.g., division by zero) with suggested diagnostics.
- List undefined behaviors and specify whether runtime checks or hardware traps are relied upon.
- Include optional `lifecycle` hints indicating when input fields are read and when output fields may be safely overwritten (useful for advanced register allocation and interruption analysis).

#### A.1.3 Interruptible Execution Constraint

To ensure that non-atomic Ordinary Opcodes can be safely interrupted (e.g., by `yiel`, trap, or preemption) and later resumed without loss of correctness, their implementation must satisfy a strict bound on execution state size.

Let:
- $W$ be the target platform’s register width in bits (e.g., 64 for x86_64),
- $\mathcal{T}$ be the set of distinct field identifiers in `OperTarg`,
- $\text{size}(f)$ be the bit-width of field $f$ (derived from its `.arxtype` `size` attribute),
- $N_{\text{imm}}$ be the number of integer literals in `OperGoal`.

Then the total state required to save and restore the opcode’s execution context—comprising a micro-program counter, control flags, and all intermediate values—must not exceed:

$$
\left( \sum_{f \in \mathcal{T}} \text{size}(f) \right) + (N_{\text{gpr}} - N_{\text{imm}}) \times W \quad \text{bits}.
$$

**Rationale**:
- Fields in `OperTarg` serve as *scratch space*: their final value is defined by the opcode, so their storage may be reused for intermediate computation.
- Fields in `OperGoal` are read-only and contribute no scratch capacity.
- Each immediate literal requires one register slot for loading, hence the $N_{\text{imm}} \times W$ term.
- The general-purpose registers of the target platform provide the space of $N_{\text{gpr}} \times W$.

For in-place operations (e.g., `add (a) (a b)`), the initial value of `a` must be preserved until it is fully consumed. During that interval, `a` does not count as available scratch space. The above formula uses a conservative worst-case estimate that assumes all `OperTarg` fields are usable as scratch after their initial values (if any) are read.

Operations marked `@atomic` are exempt from this constraint, as they execute uninterruptibly.

Violations of this constraint render the opcode ineligible for use in interruptible contexts and will cause compilation failure.

---

### A.2 Generic Opcodes

Generic Opcodes are a set of reserved instructions in Arxil that govern control flow, subroutine invocation, and interactions with the runtime scheduler (ArxilVM or EDSOS). Unlike Ordinary Opcodes (which operate on field values) or Structural Opcodes (which manipulate tree topology), Generic Opcodes define *how execution proceeds* or *how a node communicates with the execution environment*.  

All Generic Opcodes appear as standalone instructions within an `instruct` block.

**Ten Generic Opcodes are exposed to Arxil programs**, listed below.

> **Note**: The opcodes `wait`, `sgnl`, `yiel`, `emit`, `obsv` and `warn` do not perform computation directly. Instead, they serve as *system calls* to the underlying runtime (ArxilVM/EDSOS), whose precise behavior may be platform-dependent but must conform to the abstract semantics defined here.

#### A.2.1 `exec` — Execute a Logical Contract

- Effect
  - Transfers control to a named `fn` declaration, binding the caller’s fields to the function’s formal parameters and return slots.

- Syntax
```ebnf
ExecDecl  = "exec", "(", "(", [ParamList], ")", "(", [RetList], ")", ")", FnName, ";" ;
ParamList = Field, { ",", Field } ;
RetList   = Field, { ",", Field } ;
Field     = Operand | "(" Operand ")" ;
FnName    = IDENT | "(" IDENT ")" ;
```

- Semantics
  - No stack frame is created; execution occurs *in place* using the actual fields provided.
  - The parameter list `(ParamList)` maps positionally to the `fn`’s input parameters; the return list `(RetList)` maps to its output fields.
  - If the target `fn` is abstract (declared with `;`), the runtime may lower it via `'lang'` blocks or native linkage.
  - The `FnName` must resolve to a visible function within the current node or an ancestor.

- Example
```axl
exec ((a, b)) ((sum)) Adder;
```

#### A.2.2 `cond` — Multi-way Conditional Dispatch

- Effect
  - Evaluates the integer value of field `f` at runtime and executes the `v`-th instruction block in the provided list, where `v = load(f)`.

- Syntax
```ebnf
CondDecl = "cond", "(", Flag, ")", "(", { Branch }, ")", ";" ;
Flag     = Operand ;
Branch   = "(", { OrdnInst | ExecDecl }, ")" ;
```

- Semantics
  - The `Flag` must refer to a field of Integer type in the current node’s data section, and its value should not bigger than the number of the following ways.

- Constraints
  - All `Branch` must be statically resolvable (no computed gotos).

- Example
```axl
cond (ready) ((exec ((a, b) (c, d)) process;) (exec ((a, e) (c, d)) retry;));
```

#### A.2.3 `cycl` — Controlled Iteration Loop

- Effect
  - Repeatedly executes a step action until a termination flag becomes true.

- Syntax
```ebnf
CyclDecl   = "cycl", "(", DoneFlag, ")", "(", StepAction, ")", ";" ;
DoneFlag   = Operand ;
(* Note: Flag should contain an BOOLEAN *)
StepAction = OrdnInst | ExecDecl ;
```

- Semantics
  - Before each iteration, the runtime checks `DoneFlag`. If `true`, the loop exits.
  - Otherwise, control transfers to `StepAction`, which must eventually cause `DoneFlag` to become `true`.
  - This construct enforces structured, verifiable loops aligned with the AS model’s deterministic execution.

- Example
```axl
cycl (loop_done) (exec ((prm, buf) (a, loop_done)) loop_body;);
```

---

#### A.2.4 `emit` — Emit a Named Causal Commitment

- Effect
  - Declares the completion of a logical phase by emitting a named label.

- Syntax
```ebnf
EmitDecl = "emit", "(", LabelList, ")", ";" ;
NodeList = Label, { ",", Label } ;
Label    = STRING ;
```

- Semantics
  - Corresponds to the same-name opcode in Observation-Triggered Causality (OTC).
  - Records that the current node has committed to the event identified by `"label"`, and notifies all descendant nodes with pending observations for this label.

- Constraints
  - All of the `Label` must be declared in `causal` block in `meta_data`.

- Example
```axl
emit ("Lib init is ready.");
```

---

#### A.2.5 `obsv` — Declare a Causal Readiness Constraint

- Effect
  - Declares that the node’s subsequent execution is contingent upon observing a specific label emitted by an ancestor.

- Syntax
```ebnf
ObsvDecl = "obsv", "(", LabelList, ")", ";" ;
NodeList = Label, { ",", Label } ;
Label    = STRING ;
```

- Semantics
  - Corresponds to the same-name opcode in Observation-Triggered Causality (OTC).
  - The node’s program counter will not advance beyond this point until some ancestor in its binding chain has emitted label that the node caring about.

- Constraints
  - All of the `Label` must be declared in `causal` block in `meta_data`.

- Example
```axl
obsv ("Lib init is ready.");
```

---

#### A.2.6 `wait` — Block Until Signaled

- Effect
  - Suspends the current node by transitioning it to the `blocked` state. Execution resumes only after a corresponding `sgnl`.

- Syntax
```ebnf
WaitDecl = "wait", "(", WaitSemaphore, { ",", WaitSemaphore }, ")", "(", [Options], ")", ";" ;
WaitSemaphore = IDENT ;
Options   = IDENT ;
```

- Semantics
  - This is a system call to the scheduler (ArxilVM/EDSOS).
  - The `WaitSemaphore` identifier *declare* an event name to wait somebody `sgnl` the same event name.
  - The optional `Options` identifier may specify synchronization policy as defined by the runtime.
  - While blocked, the node retains all field values, register values and program counter state.
  - The node remains ineligible for scheduling until signaled.

- Constraints
  - `Options` must be recognized by the target runtime; unrecognized identifiers result in undefined behavior (typically treated as a default wait).

- Example
```axl
wait (input_ready) ();
```

---

#### A.2.7 `sgnl` — Signal a Blocked Node

- Effect
  - Wakes up one or more nodes waiting on the specified event name, transitioning them from `blocked` to `ready`.

- Syntax
```ebnf
SgnlDecl = "sgnl", "(", WaitSemaphore, ")", "(", [Options], ")", ";" ;
WaitSemaphore = IDENT ;
Options  = IDENT ;
```

- Semantics
  - Dual to `wait`; forms a synchronization pair.
  - The `WaitSemaphore` identifier must match the one used in a corresponding wait.
  - If no node is waiting on that event name, `sgnl` will be in undefined behavior inside Arxil, but the runtime (ArxilVM or EDSOS) may give an precise definition.
  - The exact signaling policy (e.g., FIFO, broadcast) is runtime-defined.

- Example
```axl
sgnl (input_ready) ();
```

---

#### A.2.8 `yiel` — Voluntary Yield

- Effect
  - Temporarily pauses the current node and returns it to the `ready` state, allowing other nodes to run.

- Syntax
```ebnf
YielDecl = "yiel", "(", [Options], ")", ";" ;
Options  = IDENT ;
```

- Semantics
  - The node retains its execution context and will be rescheduled later.
  - `Options` may hint at yield priority or reason (e.g., `"io"`), but interpretation is runtime-specific.
  - Unlike `wait`, no external signal is required for resumption.

- Example
```axl
yiel (cooperative);
```

---

#### A.2.9 `fnsh` — Terminate Node

- Effect
  - Marks the current node as `finished`, ending its execution permanently.

- Syntax
```ebnf
FnshDecl = "fnsh", ";" ;
```

- Semantics
  - Corresponds to the `[Finish]` rule in the AS Formal Semantics.
  - The node is removed from the scheduler’s ready queue.
  - Children remain unaffected.
  - No instructions may follow `fnsh` in the same basic block.

- Example
```axl
fnsh;
```

---

#### A.2.10 `warn` — Emit Diagnostic Warning

- Effect
  - Places the specified nodes into the `error` state and logs a diagnostic warning.

- Syntax
```ebnf
WarnDecl = "warn", "(", NodeList, ")", ";" ;
NodeList = NodeRef, { ",", NodeRef } ;
NodeRef   = "this" | "all" | IDENT ;
```

- Semantics
  - `this`: refers to the current node.
  - `all`: refers to all nodes in the current task tree (AS).
  - `IDENT`: refers to a direct child node (i.e., pushed by this node); names are resolved in the local namespace only.
  - Affected nodes transition to `error` and are typically excluded from further scheduling.
  - Intended for unrecoverable or policy-violating conditions.

- Constraints
  - Only descendants (or self) can be targeted; cross-tree or ancestor references are disallowed.
  - The runtime may log additional context (e.g., exec trace, field dump).

- Example
```axl
warn (worker1, worker2, this);
```

---

#### A.2.11 `gerf` — Get Reference

- Effect
  - Get reference of a field as pointer value.

- Syntax
```ebnf
GerfDecl = "gerf", "(", pointer_field, ")", "(", goal_field, ")", ";" ;
pointer_field | goal_field = Operand ;
```

- Semantics
  - <TODO>

- Constraints
  - <TODO>

- Example
```axl
gerf (buffer_ptr) (buffer);
```

---

#### A.2.12 `derf` — Dereference

- Effect
  - Dereference a pointer field as a new virtual value field.

- Syntax
```ebnf
DerfExpr = "derf", "(", pointer_field, ")" ;
pointer_field = Operand ;
```

- Semantics
  - <TODO>

- Constraints
  - <TODO>

- Example
```axl
cpy (derf(buffer_ptr)) (goal_value_field);
```

---

### A.3 Structural Opcodes

Structural Opcodes are a class of instructions in the Arbor Strux Language (Arxil) that directly manipulate the hierarchical structure of the computation tree as defined by the AS Computational Model. These opcodes correspond to atomic structural transformations formally specified in the *AS Formal Semantics and Verification* document. Unlike data-manipulating or control-flow instructions, Structural Opcodes alter node relationships, ownership, and binding topologies. Their execution is governed by separation logic rules that ensure memory safety, binding consistency, and structural integrity.

Each Structural Opcode must appear within an `instruct` block and operates relative to the lexical and runtime context of the enclosing node. All identifiers referenced in operand lists must be resolvable within the current node’s scope at compile time.

The following five opcodes constitute the complete set of Structural Opcodes.

---

#### A.3.1 `psh` — Push a New Child Node

- Effect
  - Instantiates a new child node from a named template and attaches it to the current node. The new node inherits specified resources via explicit binding declarations, establishing zero-copy shared access to the parent’s `publ` or resolved `ance` fields and functions.

- Syntax
```ebnf
PshDecl = "psh", NodeName, "(", NodeTemplate,
          "(", [MetadataList], ")",
          "(", [DataList], ")",
          "(", [CodeList], ")",
        ")", ";" ;

NodeName       = IDENT ;
NodeTemplate   = IDENT ;

MetadataList   = BindingPairMeta, { ",", BindingPairMeta } ;
BindingPairMeta = "(", ThisMetaEntry, "=>", SonMetaEntry, ")" ;

DataList       = BindingPairData, { ",", BindingPairData } ;
BindingPairData = "(", ThisDataField, "=>", SonDataField, ")" ;

CodeList       = BindingPairCode, { ",", BindingPairCode } ;
BindingPairCode = "(", ThisCodeFn, "=>", SonCodeFn, ")" ;

ThisMetaEntry  = IDENT ;
SonMetaEntry   = IDENT ;
ThisDataField  = IDENT ;
SonDataField   = IDENT ;
ThisCodeFn     = IDENT ;
SonCodeFn      = IDENT ;
```

- Semantics
  - `NodeName` is a locally scoped alias for the newly created child, valid only within the current node’s instruction sequence. It does not persist as a runtime attribute of the child node.
  - `NodeTemplate` refers to a previously declared `node` template (e.g., `node Worker { ... }`).
  - Each binding pair `(A => B)` asserts that field/function `A` in the current node (which must be `publ` or `ance`) satisfies the binding contract of `B` in the child (which must be declared as `ance`).
  - `priv`-declared entities cannot appear on the left-hand side of any binding.
  - This instruction corresponds to the **Push(n, N\*, D\*, I\*)** rule in the AS formal semantics.

---

#### A.3.2 `pop` — Remove a Child Node

- Effect
  - Requests the removal of a specified child node. By default, the operation blocks until the child has entered the `zombie` state (via `fnsh`), ensuring graceful termination.

- Syntax
```ebnf
PopDecl = "pop", NodeName, ["(", Options, ")"], ";" ;

NodeName = IDENT ;
Options  = IDENT ;  // Reserved for future extensions (e.g., "force")
```

- Semantics
  - `NodeName` must refer to a direct child previously introduced via `psh` or accessible through structural context.
  - The default behavior enforces cooperative cleanup: if the child is not yet a zombie, the current node yields until it becomes one.
  - The optional `Options` parameter is reserved for future control over termination policy (e.g., non-blocking or forced deallocation), but is unused in the current version.
  - This instruction implements the **Pop(n)** structural rule, requiring the target node to be safely reclaimable.

---

#### A.3.3 `lft` — Lift an External Node into the Ancestry Chain

- Effect
  - Dynamically binds an existing external node as a logical ancestor of the current node by resolving its `ance` field promises against the external node’s `publ` exports.

- Syntax
```ebnf
LftDecl = "lft", NodeName,
          "(", [DataList], ")",
          "(", [CodeList], ")",
          ";" ;

DataList = BindingPairData, { ",", BindingPairData } ;
BindingPairData = "(", HisDataField, "=>", ThisDataField, ")" ;

CodeList = BindingPairCode, { ",", BindingPairCode } ;
BindingPairCode = "(", HisCodeFn, "=>", ThisCodeFn, ")" ;

HisDataField = IDENT ;
ThisDataField = IDENT ;
HisCodeFn = IDENT ;
ThisCodeFn = IDENT ;
```

- Semantics
  - `NodeName` refers to an already-instantiated node visible in the current lexical or runtime context (e.g., passed via inter-node coordination).
  - Each `(X => Y)` binds `X` (a `publ` entity in the external node) to `Y` (an `ance` declaration in the current node).
  - No new node is created; only binding resolution is performed.
  - This realizes the **Lift(n)** transformation, enabling safe, dynamic cross-subtree data sharing without copying.
  - Failure to resolve all `ance` promises results in a runtime error or blocked state, per the binding fulfillment protocol.

---

#### A.3.4 `mrg` — Merge a Child Node into the Current Node

- Effect
  - Absorbs the state and code of a specified child node into the current node’s virtual address space via structural append, then destroys the child.

- Syntax
```ebnf
MrgDecl = "mrg", NodeName, ["(", Options, ")"], ";" ;

NodeName = IDENT ;
Options  = STRING ;  // Layout directives (e.g., "{ insert_after: 'buf' }")
```

- Semantics
  - `NodeName` identifies the child node to be merged, which must be a direct descendant.
  - The merge performs a **structural concatenation** of the child’s `publ` data segment and function bindings into the parent’s layout.
  - The `Options` string provides hints for memory layout reorganization (e.g., insertion points), interpreted by the ArxilVM or lowering backend.
  - After merging, the child node itself will be *not exist*, and its resources become part of the current node.
  - This corresponds to the **Merge(n, n\*)** rule.

---

#### A.3.5 `dtc` — Detach a New Node by Slicing Off Resources

- Effect
  - Creates a new child node by transferring ownership of selected resources from the current node, effectively slicing off a portion of its state.

- Syntax
```ebnf
DtcDecl = "dtc", NodeName,
          "(", NodeTemplateForSon,
            "(", [MetadataList], ")",
            "(", [DataList], ")",
            "(", [CodeList], ")",
          ")",
          ["(", Options, ")"],
          ";" ;

NodeName           = IDENT ;
NodeTemplateForSon = IDENT ;

MetadataList = BindingPairMeta, { ",", BindingPairMeta } ;
BindingPairMeta = "(", ThisMetaEntry, "-->", SonMetaEntry, ")" ;

DataList = BindingPairData, { ",", BindingPairData } ;
BindingPairData = "(", ThisDataField, "-->", SonDataField, ")" ;

CodeList = BindingPairCode, { ",", BindingPairCode } ;
BindingPairCode = "(", ThisCodeFn, "-->", SonCodeFn, ")" ;

ThisMetaEntry  = IDENT ;
SonMetaEntry   = IDENT ;
ThisDataField  = IDENT ;
SonDataField   = IDENT ;
ThisCodeFn     = IDENT ;
SonCodeFn      = IDENT ;
Options        = STRING ;  // Reserved for detachment policies
```

- Semantics
  - `NodeName` is a local alias for the newly detached node.
  - `NodeTemplateForSon` specifies the template of the detached node.
  - The `-->` operator denotes **resource handoff**: the left-hand entity (`ThisX`) is transferred to satisfy the right-hand declaration (`SonX`) in the new node, and their attribute must be matched.
  - Unlike `psh` (which uses `=>` for sharing), `-->` implies transfer of exclusive ownership.
  - This implements the **Detach(n, N\*, D\*, I\*)** transformation, enabling dynamic task decomposition and state migration.
  - Post-detachment, the original fields in the parent will become invalid, disappear inside the parent's namespace.

---

*End of Spec Document.*