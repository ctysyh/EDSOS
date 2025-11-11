# TSL Language Specification

> TSL 语言规范
> 其中节点结构新增加了 `imme` 和 `futu` 的区分，但对应部分内容暂未更新

---

## 1. 节点定义语法与结构

TSL（Tree-Stacked Language）以 **节点（node）** 为基本程序单元。每个节点是一个自包含的结构体，同时承担传统编程语言中“函数”、“栈帧”和“线程”的角色。节点定义采用声明式语法，分为三个逻辑区域：

```tsl
node example_node {
    meta_data { ... }
    data { ... }
    code { ... }
}
```

实际运行时的具体布局在节点创建时即固定，顺序如下（ABI 稳定）：

| 段序 | 名称             |
|------|------------------|
| 0    | `meta_data`      |
| 1    | `data_futu_ance` |
| 2    | `data_futu_publ` |
| 3    | `data_futu_priv` |
| 4    | `data_imme_ance` |
| 5    | `data_imme_publ` |
| 6    | `data_imme_priv` |
| 7    | `code_instruct`  |
| 8    | `code_ance`      |
| 9    | `code_publ`      |
| 10   | `code_priv`      |

### 1.1 `meta_data` 区：元信息声明

- **类型元信息（cptm）**：声明本节点使用的类型定义（如 `@T_i32`），用于编译期类型检查。支持语法糖复用 TS 根节点或外部配置文件，以简化书写并保证类型一致性。
- **运行时字段（rntm）**：如 `state`（ready/running/blocked/zombie/error）、`capability`（访问权限）、`refcount`（引用计数）等，由运行时系统管理。

### 1.2 `data` 区：结构化状态存储

节点的状态完全由其 `data` 段承载，分为三类字段，具有严格的作用域语义：
- `ance`：**祖先引用字段**，不持有实际存储，仅表示对其他节点中某字段的引用承诺；
- `publ`：**公开字段**，本节点拥有实际存储，可被子孙节点通过绑定机制访问；
- `priv`：**私有字段**，仅在本节点内部可见，子孙节点无法访问。

> 示例：
> ```tsl
> data {
>     ance { int x; bool flag; }
>     publ { int buffer; }
>     priv { int temp; }
> }
> ```

### 1.3 `code` 区：执行逻辑载体

- `instruct`：核心指令序列，按顺序执行，其执行由 $\mathtt{pc}$ 记录；
- 函数块：按 `fn`（命名指令块）组织，本质是带标签的指令子序列；
- 函数参数与返回值仅为接口契约注解，无运行时语义。

> 示例：
> ```tsl
> code {
>     instruct { add buffer 0x1; finish this 0; }
>     ance { lib hardware; fn __hardware_checksum (int) => (int); }
>     publ { fn add_int ((int)a, (int)b) => ((int)result) { result = a + b; } }
>     priv { fn submit_er () => () { inst_scri { warn this 0; } } }
> }
> ```

---

## 2. 字段模型与作用域规则

TSL 的字段解析基于 **显式声明 + 静态作用域**，杜绝隐式行为。

### 2.1 三类字段的语义

| 字段类型 | 存储位置 | 可见性 | 用途 |
|--------|--------|--------|------|
| `priv` | 本节点 `data_priv` 段 | 仅本节点 | 封装私有状态，确保信息隐藏 |
| `publ` | 本节点 `data_publ` 段 | 本节点 + 绑定到它的子孙 | 对外数据接口，共享源头 |
| `ance` | 无本地存储 | 仅本节点 | 声明对外部字段的依赖契约 |

### 2.2 字段查找优先级

当在节点内引用字段名 `f` 时，按以下顺序解析：
1. 若 `f ∈ dom(priv) ∪ dom(publ)` → 直接使用本节点字段；
2. 若 `f ∈ dom(ance)` → 沿绑定链递归解析至真实 `publ` 字段；
3. 否则 → 编译错误（无效字段）。

> **关键原则**：任何对非本地字段的访问，必须通过 `ance` 块显式声明。

---

## 3. 指令与函数语义

### 3.1 `fn`：命名指令块，非传统函数

```tsl
fn add_int ((int)a, (int)b) => ((int)result) {
    result = a + b;
}
```
- 参数 `(a, b)` 和返回值 `result` 是命名指令块内作用域的代名，需在调用时显式绑定数据段中的已有字段；
- 实际执行时不创建新栈帧，仅跳转至该指令块起始地址；
- 允许递归调用；
- 编译器将其视为带符号标签的连续指令序列。

### 3.2 `exe`：同节点内跳转

```tsl
exe ((x, y) (add_buffer)) add_int;
```
- 语义：跳转至 `add_int` 指令块入口；
- **零上下文切换**：不改变 capability 域、祖链、TLB 视图；
- 绑定后，指令块直接访问、修改相关字段。

> 与传统 `call` 的本质区别：`exe` 是 **结构内指令地址跳转**，而非栈帧切换。

### 3.3 循环与递归策略

- **循环**：通过 `cycl` + `exe` 实现同节点内重复执行，无额外开销；
- **尾递归**：优化为循环；
- **非尾递归**：必须使用 `push` 创建新节点，以保证祖链正确性和避免栈溢出。

---

## 4. 结构化共享机制

TSL 通过 **显式绑定** 实现安全、高效的跨节点数据共享，取代传统指针、全局变量或 shared_ptr。

### 4.1 两阶段绑定模型

- **阶段一：声明与预绑定（`push` 时）**  
  父节点创建子节点时，可将其自身的 `ance` 字段（即使未履行）绑定给子节点：
  ```tsl
  push child (ChildType () (parent_field => child_field) ());
  ```
  效果：`child.child_field` 与 `parent.parent_field` 形成绑定链。

- **阶段二：履行承诺（`lift` 时）**  
  当父节点获得真实数据源后，通过 `lift` 将其绑定到自身 `ance` 字段：
  ```tsl
  lift data_source ((source_field => local_ance_field));
  ```
  效果：
  - `local_ance_field` 解析为 `data_source.source_field`；
  - 所有绑定到该字段的子孙节点自动获得解析能力；
  - 整个绑定链指向同一物理存储，实现 **零拷贝共享**。

### 4.2 典型模式：CTRN（Cross Tree-Stacked Reference Node）

- CTRN 是仅含 `publ` 字段的纯数据节点（如 `example_ctrn`）；
- 作为共享缓冲区，可以被多个不同 TS 实例的节点各自绑定；
- 生命周期由引用计数管理，自动回收。

---

## 5. 类型系统设计理念（文字描述）

TSL 采用 **外置、契约式、零运行时开销** 的类型系统：

- **类型即元数据**：类型不是语言语法的一部分，而是编译器提示（compiler hints）；
- **自由优先于安全**：不内置运行时类型检查或竞态防护，依赖实际编码保障安全；
- **性能导向**：生成代码无类型 tag，无动态分派；
- **可扩展性**：支持任意用户定义类型（如 `u8`, `f16`, `simd4x32`），只需提供元信息（大小、操作集、互操作规则）；
- **Opcode 类型无关**：`add`、`cpy` 等指令本身无类型，合法性由上下文字段类型决定。

---

## 6. 内存与指针模型（文字描述）

### 6.1 节点虚拟地址空间 $V_n$

每个节点拥有一个私有的、逻辑连续的虚拟地址空间 $V_n = [0, L_n)$，布局严格按以下 **8 段顺序** 固定（ABI 级别稳定）：

| 段序 | 名称 | 内容 |
|------|------|------|
| 0 | `meta_data` | 运行时元字段（`cptm` 不保留） |
| 1 | `data_ance` | `ance` 字段（远程引用） |
| 2 | `data_publ` | `publ` 字段（本地存储） |
| 3 | `data_priv` | `priv` 字段（本地存储） |
| 4 | `code_instruct` | 核心指令字节码 |
| 5 | `code_ance` | `ance` 函数（远程引用/lib） |
| 6 | `code_publ` | `publ` 函数（本地存储） |
| 7 | `code_priv` | `priv` 函数（本地存储） |

> 所有字段在 $V_n$ 中拥有确定偏移，如同 C struct。

### 6.2 指针语义

- 指针值 = `(虚拟地址 v, 源节点 n_src)`；
- 支持指针算术：`p + k` 仅修改虚拟地址分量；
- 解引用时，先定位 `v` 所属字段，再根据字段类型决定是否触发跨节点解析；
- 越界访问由 TSLVM 的 guard pages 或 trap handler 捕获。

### 6.3 高级应用场景

- **跨字段数组**：将 `ance_a.x` 和 `ance_b.y` 视为连续 `i32` 数组；
- **动态代码组装**：通过 `code.ance` 中的 `fn` 的声明顺序构建虚拟指令流；
- **零拷贝共享**：多个子节点通过不同 $V_n$ 视图访问同一远程字段。

---

## 7. 典型编程示例解析

### 7.1 主节点：生成随机数并校验

```tsl
node example_main {
    data {
        publ { int x, y, add_buffer, checksum; bool is_no_extra_op; }
        priv { int add_buffer; } // 遮蔽父 publ，仅本节点使用
    }
    code {
        instruct {
            exe (((0x1), (0x10)) (x)) hardware::random_int;
            exe (((0x1), (0x10)) (y)) hardware::random_int;
            add add_buffer (x, y);
            exe ((add_buffer) (checksum)) hardware::hardware_checksum;
            // 创建子任务，绑定字段
            push add_op (example_check_add ()
                (add_buffer => add_buffer)
                (x => x) (y => y)
                (checksum => checksum)
                (is_no_extra_op => is_no_extra_op)
                (hardware::hardware_checksum => __hardware_checksum)
            );
            pop add_op 0; // 等待子任务结束
            cond (is_no_extra_op)
                ({ exe (() ()) submit_ac; })
                ({ exe (() ()) submit_er; });
        }
        // 私有函数块
        priv {
            fn submit_ac() => () { signal example_add_ac all; finish this 0; }
            fn submit_er() => () { err this 0; }
        }
        ance { lib hardware; } // 声明外部库依赖
    }
}
```

### 7.2 子节点：校验并可能扩展计算

```tsl
node example_check_add {
    data {
        ance { int add_buffer, x, y, checksum; bool is_no_extra_op; }
        publ { bool is_right; }
        priv { int checksum_buffer; }
    }
    code {
        instruct {
            // 创建缓冲区 CTRN
            push buffer_node (example_ctrn () () ());
            // 履行承诺：将 buffer_node.a 绑定到自己的 add_buffer
            lift buffer_node ((a => add_buffer));
            // 执行加法（直接操作 buffer_node.a）
            exe ((x, y) (add_buffer)) add_int;
            // 校验结果
            exe ((add_buffer, checksum, __hardware_checksum) (checksum_buffer, is_right)) check;
            cpy is_right is_no_extra_op;
            // 条件性扩展操作
            push extra_op (example_extra_add () (add_buffer => add_buffer) ());
            pop extra_op;
            // 重新校验
            exe ((add_buffer) (checksum_buffer)) __hardware_checksum;
            and is_right (checksum_buffer, checksum);
            and is_no_extra_op (is_right, is_no_extra_op);
            finish this 0;
        }
        // ...
    }
}
```

> **执行流程说明**：
> 1. `example_father` 创建 `son`，绑定未履行的 `example_value`；
> 2. 创建 `buffer_node`（CTR）；
> 3. `lift` 将 `buffer_node.a` 绑定到 `father.example_value`；
> 4. 绑定链自动建立：`son.example_element → father.example_value → buffer_node.a`；
> 5. `son` 中的指令现在可安全操作远程字段。

---
