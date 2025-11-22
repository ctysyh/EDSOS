# dotnet 类型与操作的知识库及其生成工具链的需求规格说明书

> 版本：v0.1-MVT
> 目标：从 RyuJIT 源码中提取出“以类型为中心的操作闭包” (Type-Centric Operation Closure, TCOC)

## 关键需求描述

**以每一种具体类型为锚点（如 `Int64`），在其条目下完整列举所有“该类型参与的、可被 JIT 编译为机器码的操作”**，包括：
- 该类型作为**唯一操作数类型**的操作（如 `long + long`）  
- 该类型作为**结果类型**的操作（如 `int + long → long`）  
- 该类型作为**输入之一但被提升/转换后参与运算**的操作（如 `byte + long → long`）  
- 所有这些操作的 **IL 表达形式、语义规则、JIT 生成的平台相关机器码模板**

### MVP 阶段覆盖的类型范围

- 内置值类型（ECMA-335 primitive）
- BCL 中的重要 blittable struct，需满足
  - 无引用类型字段
  - 无托管对象

### 示例：类型 `T = Int64` 的操作集

| 场景 | 示例（C#） | 是否属于 `Int64` 的操作？ | 理由 |
|------|-----------|--------------------------|------|
| 1. 同类型二元运算 | `a + b` (`long + long`) | 是 | 核心操作 |
| 2. 混合类型运算（结果为 T） | `x + y` (`int + long`) | 是 | 结果是 `long`，T 是主导类型 |
| 3. 一元运算 | `+a`, `-a`, `~a` | 是 | 操作对象是 T |
| 4. 转换成为 T | `(long)5`, `(long)3.14f` | 是 | 生成 T 值 |
| 5. 从 T 转换成为其他类型 | `(float)myLong` | 否（属于 `float` 的操作） | 结果不是 T |
| 6. 比较（返回 bool） | `a == b`, `a > b` | 是 | 操作语义围绕 T 定义 |
| 7. 位移 | `a << 3`, `a >> n` | 是 | 位移量可为 `int`，但主体是 T |

> 判定原则：只要 **T 是操作的主要承载类型**，比如操作的结果类型为 T 或操作语义完全由 T 的值决定，就应归入 T 的操作集中。

## IL 序列与 JIT 行为的精确建模

### 1. 同类型 vs 混合类型：IL 不同，但 JIT 可能统一优化

- **同类型**（`long + long`）：
  ```il
  ldloc.0
  ldloc.1
  add
  ```
- **混合类型**（`int + long`）：
  ```il
  ldloc.0     // int
  conv.i8     // ↑ promote to long
  ldloc.1     // long
  add
  ```

虽然 IL 不同，但 **JIT 在优化后可能生成完全相同的机器码**

### 2. 记录“IL 路径”与“最终机器码”的映射

在知识库中，应保留 **多条 IL 路径 → 同一机器码模板** 的关系：

```json
{
  "type": "System.Int64",
  "operation": "add",
  "variants": [
    {
      "input_types": ["Int64", "Int64"],
      "il_pattern": ["ldarg.0", "ldarg.1", "add"],
      "platforms": {
        "x64": "add rax, rdx"
      }
    },
    {
      "input_types": ["Int32", "Int64"],
      "il_pattern": ["ldarg.0", "conv.i8", "ldarg.1", "add"],
      "platforms": {
        "x64": "add rax, rdx"   // same as above!
      }
    }
  ]
}
```

> 这样既保留了语言层语义，又反映了运行时优化事实。

### 3. 对 BCL struct，优先以其**公开静态方法/运算符**为操作单元，而非底层字段操作

例如对于 `Vector2`，其 `+` 操作在 C# 中是：

```csharp
public static Vector2 operator +(Vector2 left, Vector2 right)
{
    return new Vector2(left.X + right.X, left.Y + right.Y);
}
```

应该记录：
- 类型：`Vector2`
- 操作名：`op_Addition`
- 输入：`[Vector2, Vector2]`
- 输出：`Vector2`
- IL 原始：字段加载 + 标量加法 ×2
- JIT 优化后：**单条 SIMD 指令**

## TCOC EBNF Spec

```ebnf
(* ================================ *)
(* Type-Centric Operation Closure   *)
(* ================================ *)

tcoc-knowledge-base = { type-operation-entry } ;

type-operation-entry = 
    "type_name" ":" string ,
    "size_bits" ":" integer ,
    "is_blittable" ":" boolean ,
    "category" ":" ( "primitive" | "blittable_struct" ) ,
    [ "layout" ":" field-layout-list ] ,
    "operations" ":" operation-list ;

field-layout-list = "[" , field-layout , { "," , field-layout } , "]" ;
field-layout = 
    "{" ,
        "field" ":" string ,
        "offset_bytes" ":" integer ,
        "field_type" ":" string ,
    "}" ;

operation-list = "[" , operation , { "," , operation } , "]" ;

operation = 
    "{" ,
        "op_symbol" ":" string ,        (* e.g., "+", "Math.Abs" *)
        "op_kind" ":" op-kind ,
        "variants" ":" variant-list ,
    "}" ;

op-kind = 
    "binary_arithmetic"
  | "unary_arithmetic"
  | "bitwise"
  | "shift"
  | "comparison"
  | "conversion_to"
  | "intrinsic_unary"
  | "intrinsic_binary"
  | "operator_overload" ;

variant-list = "[" , variant , { "," , variant } , "]" ;

variant = 
    "{" ,
        "input_types" ":" type-name-list ,
        "il_sequence" ":" il-instruction-list ,
        "semantic" ":" string ,
        "platforms" ":" platform-map ,
    "}" ;

type-name-list = "[" , string , { "," , string } , "]" ;

il-instruction-list = "[" , il-instruction , { "," , il-instruction } , "]" ;

il-instruction = 
    "{" , "opcode" ":" string , [ "operand" ":" value ] , "}" ;

value = string | integer | "null" ;

platform-map = "{" , platform-entry , { "," , platform-entry } , "}" ;

platform-entry = 
    platform-name ":" platform-detail ;

platform-name = "x64" | "arm64" | "x86" | "wasm" | "loongarch64" ;

platform-detail = 
    "{" ,
        "machine_code_template" ":" string ,     (* e.g., "add <reg64>, <reg64>" *)
        [ "instruction_bytes_hex" ":" string ] , (* optional: e.g., "48 01 D0" *)
        [ "jit_intrinsic" ":" boolean ] ,
        [ "requires_feature" ":" string ] ,      (* e.g., "SSE2", "ASIMD" *)
        [ "jit_source_path" ":" string ] ,       (* e.g., "jit/codegenxarch.cpp" *)
        [ "jit_function" ":" string ] ,          (* e.g., "genCodeForAdd" *)
        [ "optimization_level" ":" ( "none" | "basic" | "simd" | "vectorized" ) ] ,
    "}" ;

(* ================================ *)
(* Lexical Elements                 *)
(* ================================ *)

string = '"' , { unicode-char - ( '"' | '\' ) | escape-seq } , '"' ;
integer = digit , { digit } ;
boolean = "true" | "false" ;
digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
unicode-char = ? any valid Unicode character ? ;
escape-seq = "\" , ( '"' | '\' | '/' | 'b' | 'f' | 'n' | 'r' | 't' | 'u' hex-digit hex-digit hex-digit hex-digit ) ;
hex-digit = digit | "A" | "B" | "C" | "D" | "E" | "F" | "a" | "b" | "c" | "d" | "e" | "f" ;
```

## RyuJIT 源码解析方案

1. 构建 RyuJIT 的 AST / Call Graph
     - 使用 **Roslyn + C++ parser（如 Clang LibTooling）** 解析 `jit/*.cpp`
     - 重点函数：
       - `impImport*`：IL 导入
       - `fgMorph*`：树形优化
       - `genCodeFor*`：代码生成
       - `lower*`：降级处理

2. 提取 Intrinsic 映射表
    RyuJIT 使用 `namedIntrinsic` 机制识别 BCL 方法：

    ```cpp
    // file: jit/hwintrinsiclist.h 或 jit/intrinsiclist.h
    HWINTRINSIC(Add, Vector128, Add, ...)
    ```

    → 可解析这些头文件，构建 **Method → Intrinsic ID → CodeGen 函数** 映射。

3. 平台条件编译处理
   - RyuJIT 源码中大量使用：
   ```cpp
   #ifdef TARGET_XARCH
       genCodeForAddXArch(...)
   #elif defined(TARGET_ARM64)
       genCodeForAddArm64(...)
   #endif
   ```

   - 需用预处理器模拟不同平台定义，分别解析。

4. 输出结构化 trace

## 动态 JIT Disasm 方案

考虑使用 `dotnet/jitutils` + Docker 矩阵的方案，或 Docker 容器本地部署 SharpLab 后端的方案。

共通：
1. 测试用例生成器：编写代码生成器，自动产出覆盖矩阵
2. 并行与缓存：并行提交和执行测试用例，对相同 `(Type, Op)` 缓存结果，避免重复编译
3. 汇编解析与模板提取