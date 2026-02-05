<!--
SPDX-FileCopyrightText: © 2025-2026 Bib Guake
SPDX-License-Identifier: Apache-2.0
-->

# Arxil Type Specification

---

> v0.1.2-drafting

---

## 1. 引言

### 1.1 目的与范围

本文档定义 **Arxil 类型规范**（Arxil Type Specification），即 `.arxtype` 文件的语法、语义及其在 Arxil 编译模型中的角色。`.arxtype` 是 Arxil 语言与底层执行环境之间的**编译时契约接口**，用于声明类型的内存布局、支持的操作集、互操作规则、安全验证策略及目标平台实现映射。

本规范适用于：
- Arxil 编译器开发者
- 标准库与系统库作者
- 工具链（如 linter、IDE）构建者

> **注意**：`.arxtype` 不是 Arxil 源语言（`.axl`）的一部分，而是其外部类型系统接口。Arxil 程序通过类型名引用 `.arxtype` 定义，但不内联定义类型。

### 1.2 设计哲学

`.arxtype` 的设计遵循 Arxil 语言的核心原则：

- **语法自由，语义封闭**（Syntax Freedom, Semantic Closure）  
  允许高层语言灵活表达，但所有语义必须在编译期封闭、可验证。
  
- **零运行时开销**（Zero Runtime Cost）  
  所有类型信息在编译后擦除，生成代码不含类型标签或动态分发。

- **操作即契约**（Operation as Contract）  
  每个操作码（opcode）的行为由其操作数类型的 `.arxtype` 定义，而非语言内置。

- **后端无关，策略驱动**（Backend-Agnostic, Strategy-Driven）  
  支持将同一操作映射到原生指令、LLVM IR、WASM 等任意目标表示。

---

## 2. `.arxtype` 在 Arxil 编译模型中的角色

### 2.1 编译流程上下文

Arxil 指令的编译过程如下：

```
[ Arxil Source (.axl) ]
        ↓
add (a) ((b) (c))   ← a, b, c 声明为类型 "int32_t"
        ↓
[ Arxil Frontend ]
        ↓
查询 int32_t.arxtype 中 "add" 操作的实现策略
        ↓
[ Arxil Backend ]
        ↓
根据目标平台（x86_64 / wasm32 / llvm_ir）选择最优实现
        ↓
[ Target Code: Native ASM / WASM / LLVM IR / ... ]
```

- Arxil 的三级一体愿景：
  - 原生级：在 EDSOS (自研分布式 OS) 中运行，AOT 编译为机器码 + 数字签名
  - 兼容级：在 Windows / Linux 等经典 OS 中运行，AOT 编译为机器码 + ArxilRT (轻量运行时)
  - 托管级：在 ArxilVM 中运行，JIT 直接编译运行 Arxil Instruct
- 无论在哪一层，程序的核心逻辑和数据结构都源自同一份 Arxil 源码，而 `.arxtype` 是贯穿始终的“元数据契约”。

### 2.2 核心职责

`.arxtype` 文件承担以下职责：

| 职责 | 说明 |
|------|------|
| **类型标识** | 命名、大小、ABI 对齐 |
| **内存布局** | 字段偏移、宽度、子类型 |
| **操作声明** | 支持哪些操作码 |
| **实现策略** | 每个操作在不同目标上的实现路径 |
| **安全契约** | 编译时与运行时检查规则 |
| **互操作协议** | 与其他类型的转换、绑定、指针能力 |
| **文档与提示** | 供开发者和工具链使用 |

---

## 3. `.arxtype` 语法规范草稿（EBNF）

> **version 0.3-alpha**

```ebnf
(* ====== Arxil .arxtype Extensible EBNF Framework v0.3-alpha ====== *)

type_file       = [ spec_decl ], type_decl, { extra_top_level } ;
spec_decl       = "spec_version", "=", string_literal, ";" ;

type_decl       = "type", IDENT, "{",
                    { type_block },
                  "}" ;

type_block      = core_block
                | layout_block
                | ops_block
                | interop_block
                | validators_block
                | hints_block
                | extension_block   (* ← 关键：允许未来扩展 *)
                ;

(* --- Core: 必需且稳定的最小集 --- *)
core_block      = "size", "=", size_expr, ";",
                  [ "name", "=", string_literal, ";" ] ;

size_expr       = UINT, ("bits" | "bytes") ;

(* --- Layout: 字段级内存描述 --- *)
layout_block    = "layout", "=", "{",
                    { field_decl, ";" },
                  "}" ;

field_decl      = type_ref, IDENT, "->", offset_expr ;
type_ref        = IDENT ;               (* 如 f32, i64 *)
offset_expr     = UINT ;                (* 未来可扩展为 expr *)

(* --- Ops: 操作重载集合 --- *)
ops_block       = "ops", IDENT, "{",
                    { operand_pattern },
                  "}" ;

operand_pattern = "operands",
                  "targ", "{", field_list, "}",
                  "goal", "{", field_list, "}",
                  "{",
                    { implementation },
                    [ validators_block ],
                  "}" ;

field_list      = "(", [ field_use, { ",", field_use } ], ")" ;
field_use       = "(", type_ref, ")", IDENT [ "." IDENT ]* ;
                  (* 支持 x.real, a[i].imag 等，当前仅支持 .field *)

implementation  = instmm_impl
                | asm_impl
                | unknown_impl          (* ← 容忍未知实现类型 *)
                ;

instmm_impl     = "@instmm", "{",
                    [ regi_decl ],
                    "code", "{", { instmm_stmt, ";" }, "}",
                  "}" ;

asm_impl        = target_selector, "@asm", "{",
                    "code", "=", string_literal, ";",
                  "}" ;

(* --- Target Selector: 多后端路由 --- *)
target_selector = "@", IDENT, [ "+", feature_list ] ;
feature_list    = IDENT, { "+", IDENT } ;

(* --- Interop, Validators, Hints: 当前保留结构，内容可空 --- *)
interop_block   = "interop", "{", { any_key_value }, "}" ;
validators_block= "validators", "{", { any_key_value }, "}" ;
hints_block     = "hints", "{", { any_key_value }, "}" ;

(* --- Extension & Future-Proofing --- *)
extension_block = "@" IDENT, "{", { any_token_seq }, "}" ;
any_key_value   = IDENT, "=", any_value, ";" ;
any_value       = string_literal | UINT | "(" list ")" | "{" block "}" ;
any_token_seq   = .* ;  (* 解析器可跳过未知块 *)

(* --- Lexical --- *)
IDENT           = ? letter (letter | digit | "_")* ? ;
UINT            = ? unsigned integer ? ;
string_literal  = '"' .* '"' ;
```

---

## 4. 语义规则与最佳实践

### 4.1 操作码策略选择

- 编译器匹配当前 `target` 中第一个字段所属类型，优先选择其 Implementation 中的对应平台的 `asm`，否则选择 `instmm`：
- 各个 Implementation 的选择策略，尤其是一些可以有非常多细微变异的操作，例如加法的是否添加运行时溢出校验等，这是需要一个精确的定义，来告诉编译器何时选择哪一个 Implementation，并且需要以合理的方式，让程序员在需要时可以手动掌控各个细微变异版本之间的选择，这里的实现可以是采取不同的变异版本命名，同时在基本版本中提供一些按策略重定向逻辑；
- 若多个实现匹配同一 `target` 且未声明选择策略，行为未定义。

### 4.2 内建操作码命名约定

- 以简短清晰为基本原则；
- 特别是 Arxil 并不保留 "." 的使用，这不是一个语言保留词法，因此对于复杂类型的、或多种有层级关系的操作码，可以手动设置这种 namespace-like 风格的命名。

### 4.3 用于 FFI

- 可以在 `interop` 部分中，定义各种关联类型之间的转换，尤其是如何从关联类型转换成为当前类型的规则；
- 此外，这也可以用于面向 C 的类型对应和类型层级的调用约定。

### 4.4 安全检查

- `validators` 是在类型声明中手动书写的，应该具有统一的接口，编译器按照类似于回调的方式直接执行，并获取返回值；
- 类型声明中应该相应地声明各种错误情况下的提示信息文本，这里的设计应该向 Rust 学习，尽可能精确、全面。

---

## 5. 与外部生态的集成

### 5.1 从 LLVM 提取 `.arxtype`

- 学习重点：**硬件级优化知识库**。
- 解析 Target `.td` 文件（如 `X86InstrInfo.td`）获取指令选择规则。
- 学习参考它的 Intrinsic 定义。


### 5.2 从 .NET Runtime 提取 `.arxtype`

- 学习重点: **清晰的元数据模型**(ECMA-335)、成熟的 AOT/JIT 协同（RyuJIT, NativeAOT）、优秀的跨平台库（如 System.Math）。
- 从中提取**完整的类型 ABI 描述**，包括BCL 类型布局（`System.Numerics.Complex`）、P/Invoke 互操作协议、AOT/JIT 协同 fallback 机制。
- **AOT/JIT 协同模式**: .NET 的 NativeAOT 和 RyuJIT 共享同一套 IL 和元数据。这正是 Arxil 三级模型的理想状态：原生级 AOT 和托管级 JIT 共享同一套 .arxtype。我们可以学习其如何在编译时做出最优的 AOT 决策，同时保留足够的信息供 JIT 使用。
- 优势: MIT 许可证极其宽松，社区活跃，文档完善，发展路线图清晰，技术风险远低于从零构建或深度依赖一个更复杂的项目。

### 5.3 从 Rust 学习编译器提示

- 学习重点：**精确的编译器提示语言**。
- 特别是它的**能力要求**，可以集成到每一个实现的私有 `validators_block` 中，并提供具体的提示信息。

---

## 6. 实验性示例

> **服从的格式规范版本：v0.3-alpha**

```arxtype
type complex_f32 {
    size = 64 bits;
    layout = {
        f32 real -> 0;   // 明确声明起始字节
        f32 imag -> 32;
    };

    ops add {
        operands targ {(complex_f32)a} goal {((complex_f32)x) ((complex_f32)y)} {
            @instmm {
                regi {f32 xmm0}
                code {
                    (f32)add (xmm0) ((x.real) (y.real));
                    (f32)cpy (a.real) (xmm0);
                    (f32)add (xmm0) ((x.imag) (y.imag));
                    (f32)cpy (a.imag) (xmm0);
                    // 此处仅为示例，用于展示寄存器的使用，实际上这里不需要寄存器也可以实现
                    // 特别地，这里会以类似于函数调用的逻辑展开，哪怕这里以f32寄存器声明xmm0，在内联调用f32.add时仍然会将其视为一个缓存中的变量，使得f32.add中仍然额外需要一个新寄存器来真正实现加法的寄存
                    // 此外，这也是约定只能使用标准类型集中定义的类型及其操作码的原因，因为标准类型集中一定会提供匹配各个目标平台的汇编实现
                    // 对于这种间接运算的运算符，如果没有额外的优化（比如SIMD），不然是可以只写出instmm，同样能够最终转换为汇编
                }
            }
            @x64 @asm {
                ... // 这里如果写一个SIMD的实现，那么就是有意义的；如果直接等于把f32.add的逻辑复制粘贴过来，那么是没有任何意义的
            }
            validators { // 验证器可以写在 per-operands situation 里，也可以写在外面和所有 operands situations 并列，表示通用的验证逻辑
                ...
                // 内部可以添加检查到每种错误时的人类可读提示文本，供 IDE 读取和打印
            }
        }
        operands targ {(complex_f32)a} goal {((complex_f32)a) ((complex_f32)x)} { // 这声明了对于 in-place 操作的优化展开
            @instmm {
                code {
                    (f32)add (a.real) ((a.real) (x.real));
                    (f32)add (a.imag) ((a.imag) (x.imag));
                    // 这里展示了不使用寄存器的方法，前一种情况也是可以这么写的
                }
            }
            ...
        }
        operands targ {(complex_f32)a} goal {((f32)x1) ((f32)x2) ((f32)x3) ((f32)x4)} { // 这就是关联类型的互操作性，因为不同的具体类型会导向不同的内部处理
            ...
        }
    }

    ops sub {
        ...
    }
    // 像这样，一一列举出所有的操作码及其展开

    interop {
        pointer_info {
            is_pointer   = false;
            pointer_type = "complex_f32_ptr";
        }
    }
}
```

---

## 7. 未来扩展方向

- **泛型支持**：通过关联到多个不同的具体类型的特殊类型表达，还可以通过不同的组合定义不同的泛型。
- **表达式求值**：`BitExpr` 支持 `8*N` 等算术。
- **依赖管理**：`.arxtype` 可导入其他类型。
- **形式化验证注解**：集成分离逻辑断言。

---

## Appendix

> 语言规范中有关类型系统的规定的章节摘抄

### 1 The Role of `.arxtype`

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

### 2 Linking and Resolution

When the compiler encounters a type name such as `int32_t` or `hardware.mmio_reg`, it searches for a corresponding `.arxtype` file. Programmers can also declare the file path of the type in `meta_data`.

The resolution process yields a **type descriptor** containing all metadata defined in the `.arxtype` file. This descriptor is then used throughout compilation for:
- **Layout computation**: Determining the byte/bit size and alignment of fields.
- **Opcode validation**: Ensuring that an instruction like `add %x %y %z` is only emitted if the `.arxtype` for the underlying type includes an `add` operation in its `ops` block.
- **Interoperability checking**: Validating cross-type assignments or bindings (e.g., whether a `(u32)` can be implicitly converted to `(i64)` per the `interop` rules).

Crucially, **no type information is retained at runtime**. The generated ArxilVM bytecode or native code contains only raw memory accesses and operation dispatches—no type tags, vtables, or dynamic checks. All safety and correctness must be ensured at compile time through the `.arxtype` contract.

> **Design Principle**: *Types are compile-time contracts, not runtime entities.* This aligns with Arxil’s goal of providing bare-metal performance while enabling high-level reasoning through external verification tools and disciplined composition.

### 3 About Ordinary Opcode in `.arxtype`

Opcodes like `add`, `mul`, `and`, and `cpy` are Ordinary Opcodes. Their legality and precise runtime behavior are **entirely determined by the types of their operands in the scope of the data type of the `OperTarg`**. The compiler consults the `.arxtype` definitions to verify that the requested operation is supported (i.e., listed in the type's `ops` set) and to generate the correct low-level code.

Every custom Ordinary Opcode **must** explicitly define the following in its `.arxtype` declaration:

1. **Name**
   - Must be a valid Arxil identifier (see [§2.2](#3-about-ordinary-opcode-in-arxtype)).  
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

### 4 Interruptible Execution Constraint

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

For in-place operations (e.g., `add (a) (a b)`), the initial value of `a` must be preserved until it is fully consumed. During that interval, `a` does not count as available scratch space. The above formula uses a conservative worst-case estimate.

Operations marked `@atomic` are exempt from this constraint, as they execute uninterruptibly.

Violations of this constraint render the opcode ineligible for use in interruptible contexts and will cause compilation failure.