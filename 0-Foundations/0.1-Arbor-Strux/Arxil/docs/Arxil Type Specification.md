# Arxil Type Specification

---

> v0.2.1 — Draft
> last change — 2025.11.16

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
  支持将同一操作映射到原生指令、库调用、LLVM IR、WASM 等任意目标表示。

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

### 2.2 核心职责

`.arxtype` 文件承担以下职责：

| 职责 | 说明 |
|------|------|
| **类型标识** | 命名、大小、ABI 对齐 |
| **内存布局** | 字段偏移、宽度、子类型 |
| **操作声明** | 支持哪些操作码（如 `add`, `sin`） |
| **实现策略** | 每个操作在不同目标上的实现路径（native/asm/libcall/lowered） |
| **安全契约** | 编译时与运行时检查规则 |
| **互操作协议** | 与其他类型的转换、绑定、指针能力 |
| **文档与提示** | 供开发者和工具链使用 |

---

## 3. `.arxtype` 语法规范草稿（EBNF）

### version 0.1

```ebnf
ArxilTypeFile = TypeDecl ;

TypeDecl = "type", IDENT, "{",
             TypeName,
             TypeSize,
             [TypeLayout],
             [TypeOps],
             [TypeInterop],
             [TypeValidators],
             [TypeDocs],
             [TypeHints],
           "}" ;

TypeName = "name", "=", STRING, ";" ;

TypeSize = "size", "=", INTEGER, ("bits" | "bytes"), ";" ;

TypeLayout = "layout", "{", { LayoutField }, "}" ;

LayoutField = IDENT, "{",
                "offset", "=", BitExpr, ";",
                "width",  "=", BitExpr, ";",
                "type",   "=", IDENT, ";",
                ["tag", "=", STRING, ";"],   (* Optional tag, e.g., "reserved" *)
              "}" ;

BitExpr = INTEGER ;  (* In the future, it can be extended to simple arithmetic expressions, e.g., "8*4" *)

TypeOps = "ops", "{", { OpDecl }, "}" ;

OpDecl = IDENT, "{",
           [OpNative],
           [OpAsmMap],
           [OpLibcall],
           [OpIntrinsic],
           [OpPointerArith],
           [OpUnary],
           [OpMayTrap],
           [OpLoweredArxil],
         "}" ;

OpNative      = "native", "=", BOOLEAN, ";" ;
OpUnary       = "unary", "=", BOOLEAN, ";" ;
OpMayTrap     = "may_trap", "=", BOOLEAN, ";" ;

OpPointerArith = "pointer_arithmetic", "=", BOOLEAN, ";",
                 ["scale_by", "=", INTEGER, ";"] ;

OpAsmMap = "asm", "{", { AsmTarget }, "}" ;

AsmTarget = IDENT, "=", STRING, ";" ;
(* e.g., x86_64 = "addss %xmm0, %xmm1"; *)

OpLibcall = "libcall", "=", STRING, ";" ;
(* e.g., libcall = "__f32_sin"; *)

OpIntrinsic = "intrinsic", "=", BOOLEAN, ";" ;

OpLoweredArxil = "lowered", "=", InstScriBlock, ";" ;

InstScriBlock = "'inst_scri'", "{", { InstLine }, "}" ;

InstLine = IDENT, { Operand }, ";" ;

Operand = IDENT | INTEGER | "(" IDENT ")" ;
(* IDENT here refers to the field name in the layout, or temporary register *)

TypeInterop = "interop", "{",
                [InteropPointerType],
                [InteropImplicitFrom],
                [InteropImplicitTo],
                [InteropBindingCompat],
                [InteropArrayOK],
              "}" ;

InteropPointerType     = "pointer_type", "=", IDENT, ";" ;
InteropImplicitFrom    = "implicit_from", "=", TypeList, ";" ;
InteropImplicitTo      = "implicit_to",   "=", TypeList, ";" ;
InteropBindingCompat   = "binding_compatible_with", "=", TypeList, ";" ;
InteropArrayOK         = "array_element_ok", "=", BOOLEAN, ";" ;

TypeList = "(", [IDENT, { ",", IDENT }], ")" ;

TypeValidators = "validators", "{",
                   [CompileTimeChecks],
                   [RuntimeChecks],
                 "}" ;

CompileTimeChecks = "compile_time", "=", CheckList, ";" ;

RuntimeChecks = "runtime", "{", { RuntimeCheckDecl }, "}" ;

CheckList = "(", [STRING, { ",", STRING }], ")" ;

RuntimeCheckDecl = IDENT, "{",
                     "condition", "=", ConditionExpr, ";",
                     "action", "=", ActionSpec, ";",
                     ["enabled_by_default", "=", BOOLEAN, ";"],
                     ["requires", "=", CapabilityList, ";"],
                   "}" ;

ConditionExpr = STRING ;  (* e.g., "address == 0" *)
ActionSpec = STRING ;     (* e.g., "trap PTR_NULL" *)
CapabilityList = "(", [IDENT, { ",", IDENT }], ")" ;

TypeDocs = "docs", "{",
             [DocUsage],
             [DocErrors],
           "}" ;

DocUsage = "usage", "=", STRING, ";" ;
DocErrors = "errors", "{", { ErrorCodeMapping }, "}" ;

ErrorCodeMapping = IDENT, "=", STRING, ";" ;
(* e.g., PTR_DANGLING = "Dangling pointer dereference" *)

TypeHints = "hints", "{", { HintDecl }, "}" ;

HintDecl = IDENT, "=", (BOOLEAN | INTEGER | STRING), ";" ;
(* e.g., cache_line_aligned = true; simd_eligible = false; *)
```

### version 0.2.0

```ebnf
(* Top Level *)
ArxilTypeFile = TypeDecl ;

TypeDecl = "type", IDENT, "{",
             TypeName,
             TypeSize,
             [TypeLayout],
             [TypeOps],
             [TypeInterop],
             [TypeValidators],
             [TypeDocs],
             [TypeHints],
           "}" ;

(* Basic Info *)
TypeName = "name", "=", STRING, ";" ;
TypeSize = "size", "=", INTEGER, ("bits" | "bytes"), ";" ;

(* Memory Layout *)
TypeLayout = "layout", "{", { LayoutField }, "}" ;
LayoutField = IDENT, "{",
                "offset", "=", BitExpr, ";",
                "width",  "=", BitExpr, ";",
                "type",   "=", IDENT, ";",
                ["tag", "=", STRING, ";"],
              "}" ;
BitExpr = INTEGER ;  (* Future: simple arithmetic *)

(* Operations and Strategies *)
TypeOps = "ops", "{", { OpDecl | OpAlias }, "}" ;

OpDecl = IDENT, "{",
           [OpUnary],
           [OpMayTrap],
           [OpPointerArith],
           { Implementation },
         "}" ;

OpAlias = "alias", IDENT, "as", IDENT, "{", Implementation, "}" ;

Implementation =
    ("native" | "asm" | "libcall" | "intrinsic" | "lowered"),
    "{",
      [TargetSelector],
      CodeOrRef,
      [FallbackRef],
    "}" ;

TargetSelector = "target", "=", TargetSpec, ";" ;
TargetSpec = IDENT, { "+", IDENT } ;  (* e.g., x86_64+sse4.2 *)

CodeOrRef =
    ("code", "=", STRING, ";") |
    ("block", "=", LangTaggedBlock, ";") ;

LangTaggedBlock = "'", LangTag, "'", "{", RawCode, "}" ;
LangTag = "inst_scri" | "llvm_ir" | "wasm_text" | "ptx" | IDENT ;
RawCode = (* raw string with brace balancing *) ;

FallbackRef = "fallback", "=", IDENT, ";" ;

(* Operation Flags *)
OpUnary       = "unary", "=", BOOLEAN, ";" ;
OpMayTrap     = "may_trap", "=", BOOLEAN, ";" ;
OpPointerArith = "pointer_arithmetic", "=", BOOLEAN, ";",
                 ["scale_by", "=", INTEGER, ";"] ;

(* Interoperability *)
TypeInterop = "interop", "{",
                [InteropPointerType],
                [InteropImplicitFrom],
                [InteropImplicitTo],
                [InteropBindingCompat],
                [InteropArrayOK],
              "}" ;

InteropPointerType     = "pointer_type", "=", IDENT, ";" ;
InteropImplicitFrom    = "implicit_from", "=", TypeList, ";" ;
InteropImplicitTo      = "implicit_to",   "=", TypeList, ";" ;
InteropBindingCompat   = "binding_compatible_with", "=", TypeList, ";" ;
InteropArrayOK         = "array_element_ok", "=", BOOLEAN, ";" ;
TypeList = "(", [IDENT, { ",", IDENT }], ")" ;

(* Validation *)
TypeValidators = "validators", "{",
                   [CompileTimeChecks],
                   [RuntimeChecks],
                 "}" ;

CompileTimeChecks = "compile_time", "=", CheckList, ";" ;
CheckList = "(", [STRING, { ",", STRING }], ")" ;

RuntimeChecks = "runtime", "{", { RuntimeCheckDecl }, "}" ;
RuntimeCheckDecl = IDENT, "{",
                     "condition", "=", ConditionExpr, ";",
                     "action", "=", ActionSpec, ";",
                     ["enabled_by_default", "=", BOOLEAN, ";"],
                     ["requires", "=", CapabilityList, ";"],
                   "}" ;

ConditionExpr = Expr ;
Expr = Term, { ("&&" | "||"), Term } ;
Term = Factor, { ("==" | "!=" | "<" | ">"), Factor } ;
Factor = IDENT | INTEGER | "(" Expr ")" ;

ActionSpec = STRING ;     (* e.g., "trap ADD_OVERFLOW" *)
CapabilityList = "(", [IDENT, { ",", IDENT }], ")" ;

(* Documentation & Hints *)
TypeDocs = "docs", "{",
             [DocUsage],
             [DocErrors],
           "}" ;
DocUsage = "usage", "=", STRING, ";" ;
DocErrors = "errors", "{", { ErrorCodeMapping }, "}" ;
ErrorCodeMapping = IDENT, "=", STRING, ";" ;

TypeHints = "hints", "{", { HintDecl }, "}" ;
HintDecl = IDENT, "=", (BOOLEAN | INTEGER | STRING), ";" ;
```

---

## 4. 语义规则与最佳实践

### 4.1 操作码策略选择

- 编译器应按以下优先级选择 `Implementation`：
  1. 匹配当前 `target` 的 `native` 或 `asm`
  2. 匹配的 `intrinsic`
  3. `libcall`
  4. `lowered`（通用回退）
- 若多个实现匹配同一 `target`，行为未定义（应避免）。

### 4.2 内建操作码命名约定

- 基本操作：`add`, `sub`, `mul`, `div`, `sin`, `sqrt`...
- 语义变体：`add_wrapping`, `add_checked`, `rotate_left`
- 平台特化：通过 `OpAlias` 声明，如 `alias add_x86_adx as add { ... }`

### 4.3 Intrinsic 映射规则

- 若 `intrinsic { code = "llvm.sin.f32"; }`，则映射到 LLVM intrinsic `@llvm.sin.f32`。
- 若仅 `intrinsic = true;`，则默认映射为 `@llvm.<opname>.<type_suffix>`。

### 4.4 安全检查

- `compile_time` 检查应在类型解析阶段执行。
- `runtime` 检查应在 lowering 阶段插入 trap 或日志代码。
- `enabled_by_default = false` 的检查需显式启用（如 `-C check-overflow`）。

---

## 5. 与外部生态的集成

### 5.1 从 LLVM 提取 `.arxtype`

可通过以下步骤自动化生成基础类型库：

1. **解析 `LangRef.rst`** 获取 LLVM intrinsics 列表。
2. **解析 Target `.td` 文件**（如 `X86InstrInfo.td`）获取指令选择规则。
3. **映射操作**：
   - DAG 节点 `add` → `.arxtype` 中的 `add` 操作
   - 指令 `ADD32rr` → `native { target = x86_64; }`
   - Intrinsic `llvm.sin.f32` → `intrinsic { code = "llvm.sin.f32"; }`
4. **生成 `.arxtype` 文件**（如 `i32.arxtype`, `f64.arxtype`）。

### 5.2 从 .NET Runtime 提取 `.arxtype`

- 学习重点: **清晰的元数据模型**(ECMA-335)、成熟的 AOT/JIT 协同（RyuJIT, NativeAOT）、优秀的跨平台库（如 System.Math）。

---

## 6. 实验性示例

> 服从的格式规范版本：v0.2.1

```arxtype
type complex_f32 {
    size = 64 bits;
    layout = {
        f32 real -> 0;   // 明确声明起始字节
        f32 imag -> 32;
    };

    is_pointer   = false;
    pointer_type = "complex_f32_ptr";

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
            @inst_modl {
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
}
```

---

## 7. 附录：未来扩展方向

- **泛型支持**：通过关联到多个不同的具体类型的特殊类型表达，还可以通过不同的组合定义不同的泛型。
- **表达式求值**：`BitExpr` 支持 `8*N` 等算术。
- **依赖管理**：`.arxtype` 可导入其他类型。
- **形式化验证注解**：集成分离逻辑断言。

---

*End of Type Specification Document.*