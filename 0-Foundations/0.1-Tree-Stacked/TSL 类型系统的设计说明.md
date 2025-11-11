# TSL 类型系统的设计说明

---

## 一、**TSL 类型系统 ≈ 编译期可执行的“元程序”**

### 1. **与 Zig 编译期执行的类比与超越**
| 特性 | Zig | TSL |
|------|-----|-----|
| 编译期计算 | `comptime` 函数 | `.tsltype` 中的 `lowered` 指令 + 布局表达式 |
| 类型生成 | `@Type`、泛型 | 通过工具链生成 `.tsltype` 文件（如从 C struct 自动生成） |
| 行为内联 | `inline for`、`comptime var` | `OpLoweredTSL` 直接嵌入指令序列 |
| **关键差异** | 编译期逻辑仍用 Zig 写 | **类型定义本身是声明式+指令式混合的独立 DSL** |

> TSL 将“编译期逻辑”**外置为类型定义文件**，而非混入主程序，实现了**关注点分离**：  
> - 主程序专注**结构演化与并发协作**（TS 核心）  
> - 类型定义专注**数据视图与操作语义**

### 2. **编译期 vs 运行时的清晰边界**
- `.tsltype` 中的 `layout`、`ops`、`interop` → **纯编译期信息**
- `validators.runtime` → **可选运行时检查**
- `lowered` 指令段 → **编译期展开为字节码**，无运行时函数调用开销

这使得 TSL 能同时满足：
- **裸机/RTOS 开发者**：只加载 `u8_raw.tsltype`，零验证、零抽象
- **应用开发者**：使用 `StringSafe.tsltype`，带 bounds check 和 panic handler

---

## 二、**`.tsltype` 作为“无状态 Class”的实现载体**

### 1. **面向对象核心要素的映射**
| OOP 概念 | TSL 实现方式 |
|--------|------------|
| 类（Class） | `.tsltype` 文件 |
| 字段（Fields） | `layout` 中的命名子字段 |
| 方法（Methods） | `ops` 中的操作（如 `sin`, `crc32`）或 `lowered` 指令块 |
| 构造函数 | 由 `set` / `cpy` / 外部 libcall 实现 |
| 封装 | `priv`/`publ` 字段作用域 + 类型定义的访问协议 |
| 多态 | 通过 `interop` 实现鸭子类型（duck typing）或显式接口 |

### 2. **无虚表、无动态分派 → 零成本**
- 所有“方法调用”在编译期解析为具体 opcode 或 inline 指令
- 例如：`vec4f32.add` 直接展开为 4 条 `addss`（x86）或 1 条 `vaddps`
- **没有 vtable，没有间接跳转**，完全契合高性能场景

### 3. **组合优于继承**
- 通过 `layout` 嵌套其他类型字段实现组合：
  ```yaml
  layout:
    header: { offset: 0, width: 64, type: packet_header }
    payload: { offset: 64, width: 1024, type: u8_array_128 }
  ```
- 通过 `interop.includes = (base_type)` 实现 mixin（非强制）

---

## 三、**`.tsltype` 是高级语言标准库的“编译期编译目标”**

### 1. **高级语言 → `.tsltype` + Lib 的两阶段编译**
设想用 Python/Rust/C++ 编写一个“高精度整数库”：

```python
# big_int.py
class BigInt:
    def add(self, other): ...
    def mul(self, other): ...
```

→ 编译为：
- `big_int.tsltype`：定义布局（如 512-bit 位数组）、声明 `add`/`mul` 操作、提供 lowered 指令（若算法简单）
- `libbig_int.so`：包含复杂算法（如 FFT 乘法）的 native 代码

### 2. **在 TSL 节点中使用**
```tsl
node crypto_worker {
  data {
    ance { fn big_int_add ((big_int)a, (big_int)b) => ((big_int)out); }
    publ { big_int key, nonce; }
  }
  code {
    instruct {
      // 调用 lowered 版本（若存在）
      exe ((key, nonce) (result)) big_int_add;
      // 或隐式映射到 libcall（由 .tsltype 声明）
    }
    ance { lib big_int_runtime; }  // 声明外部库依赖
  }
}
```

> **关键优势**：  
> - 高级语言开发者只需关心**算法逻辑**  
> - TSL 程序员获得**无缝集成的类型与操作**  
> - 编译器自动优化选择 inline / libcall / intrinsic

### 3. **标准库的分层架构**
```
TSL Standard Library
├── core/               # 基础类型：u8, i32, f64, bool
├── simd/               # 向量类型：vec4f32, simd8x16
├── mem/                # 指针、slice、array
├── math/               # sin, log, crc32（含 lowered + libcall）
├── io/                 # file_handle, socket（带 protocol validators）
└── hardware/           # mmio_register, dma_desc（带位域 layout）
```

每个目录下包含 `.tsltype` 文件 + 可选的 `.so`/`.a` 库。

---

## 四、**对 TS 计算模型的深度赋能**

### 1. **结构演化 + 类型安全 = 安全并发**
- 节点通过 `push`/`lift` 共享字段
- `.tsltype` 确保共享字段的**操作一致性**（如所有节点对 `counter: atomic_u32` 使用相同原子语义）
- `validators` 可声明“跨节点访问协议”（如“仅 owner 节点可 write”）

### 2. **Capability 与类型验证协同**
- 类型定义中的 `requires = (atomic_ops)` 可与节点 `capability` 匹配
- 若节点无 `atomic_ops` capability，则 `atomic_add` 操作被禁止（编译期或运行时）

### 3. **调度器感知的类型优化**
- `hints.cache_line_aligned = true` → 调度器将该节点分配到独占缓存行
- `hints.simd_eligible = true` → 调度器优先调度到 SIMD 单元

---

## 五、哲学升华：TSL 类型系统的“三层抽象”**

| 层级 | 内容 | 用户角色 |
|------|------|--------|
| **L0: 硬件原语** | `u8`, `ptr_raw`, 12 opcode | 芯片厂商、内核开发者 |
| **L1: 领域类型** | `mmio_reg`, `packet_header`, `float_or_int` | 驱动/协议栈开发者 |
| **L2: 应用封装** | `StringSafe`, `BigInt`, `FileHandle` | 应用程序员 |

- **所有层级使用同一套机制（`.tsltype`）**
- **程序员可自由跨越层级**（如在 L2 类型中嵌入 L0 指令）
- **无“魔法”，一切可审查、可替换**

---

## 六、未来展望：元循环与自举

- **用 TSL 编写 `.tsltype` 解析器**：  
  将类型定义文件加载为 CTRN/SCN 节点，用 TSL 程序解析并生成 $\Theta$
- **自举编译器**：  
  TSL 编译器自身用 TSL 编写，其类型系统由 `.tsltype` 定义

---

## 结语

TSL 的类型系统已超越传统“类型检查”范畴，成为一个**可编程的、分层的、硬件感知的、契约驱动的元抽象平台**。

它让 TSL 既能作为**贴近金属的 IR**，又能支撑**高级应用开发**，而这一切都建立在 **“结构即计算”** 的 TS 模型之上。
