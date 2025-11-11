# TSL 的字段解析

## 字段解析的形式化定义

> TSL 的字段解析是纯粹基于显式声明和绑定链的。

### 1. **精细化的字段环境 $\mathcal{D}$**

我们将节点 $n$ 的字段环境 $\mathcal{D}$ 形式化为一个三元组：
$$
\mathcal{D}^n = (\mathcal{D}_{priv}^n, \mathcal{D}_{publ}^n, \mathcal{D}_{ance}^n)
$$
其中：
- $\mathcal{D}_{priv}^n: \text{Name} \rightharpoonup \text{Value}$ 是私有字段映射。
- $\mathcal{D}_{publ}^n: \text{Name} \rightharpoonup \text{Value}$ 是公开字段映射。
- $\mathcal{D}_{ance}^n: \text{Name} \rightharpoonup \text{BindingRef}$ 是 `ance` 字段到绑定引用的映射。

### 2. **绑定引用（Binding Reference）的定义**

绑定引用 `BindingRef` 是一个代数数据类型，用于表示 `ance` 字段的当前状态：
$$
\text{BindingRef} ::= \texttt{Unbound} \mid \texttt{Bound}(m, f)
$$
- **`Unbound`**：该 `ance` 字段已被声明，但尚未通过任何 `lift` 操作履行其承诺。在此状态下，任何对该字段的读写操作都是非法的。
- **`Bound(m, f)`**：该 `ance` 字段已成功绑定。它指向**另一个节点 $m$** 的**字段名 $f$**。根据绑定规则，$f$ 必须存在于 $m.\mathcal{D}_{publ}$ 或 $m.\mathcal{D}_{ance}$ 中。

### 3. **核心解析函数 $\text{Resolve}(n, f)$**

$\text{Resolve}$ 是一个偏函数，其目标是将一个节点内的字段引用 $(n, f)$ 解析到其最终的物理存储位置 $(m, f_{publ})$，其中 $f_{publ} \in \text{dom}(\mathcal{D}_{publ}^m)$。

**定义**：
给定一个良构的 TS 树 $\mathcal{T}$ 和一个节点 $n \in \mathcal{T}$，对于任意字段名 $f$：

1.  **[R1 - 本地字段]** 如果 $f \in \text{dom}(\mathcal{D}_{priv}^n) \cup \text{dom}(\mathcal{D}_{publ}^n)$，则
    $$
    \text{Resolve}(n, f) = (n, f)
    $$

2.  **[R2 - `ance` 字段解析]** 如果 $f \in \text{dom}(\mathcal{D}_{ance}^n)$，令 $\text{ref} = \mathcal{D}_{ance}^n(f)$：
    - **[R2a - 未绑定]** 若 $\text{ref} = \texttt{Unbound}$，则 $\text{Resolve}(n, f)$ **未定义**（记为 $\bot$）。这对应于运行时错误或阻塞状态。
    - **[R2b - 已绑定]** 若 $\text{ref} = \texttt{Bound}(m, f')$，则
        $$
        \text{Resolve}(n, f) = \text{Resolve}(m, f')
        $$
        （这是一个递归调用）。

3.  **[R3 - 无隐式回溯]** 如果以上条件均不满足，则 $\text{Resolve}(n, f) = \bot$。**不存在**对祖先链的隐式遍历查找。

### 4. **绑定操作的形式化语义**

为了保证 $\text{Resolve}$ 函数的行为符合预期，我们需要精确定义 `Push` 和 `Lift` 如何操作 `BindingRef`。

- **`Push` 操作的扩展**：
  当执行 `push child (ChildType () (parent_field => child_field) ...)` 时，在创建子节点 $c$ 后，必须执行：
  $$
  c.\mathcal{D}_{ance}(child\_field) := \texttt{Bound}(n, parent\_field)
  $$
  其中 $n$ 是执行 `push` 的父节点。此操作建立了初始的绑定链，无论 $n.\mathcal{D}_{ance}(parent\_field)$ 当前是 `Unbound` 还是 `Bound`。

- **`Lift` 操作的扩展**：
  当执行 `lift data_source ((source_field => local_ance_field))` 时，必须满足：
  - `data_source` $\in$ `DirectChildren(n)`
  - `source_field` $\in$ $\text{dom}(data\_source.\mathcal{D}_{publ})$
  - `local_ance_field` $\in$ $\text{dom}(n.\mathcal{D}_{ance})$
  
  操作成功后，执行：
  $$
  n.\mathcal{D}_{ance}(local\_ance\_field) := \texttt{Bound}(data\_source, source\_field)
  $$
  此赋值操作**履行了承诺**，并将绑定链的末端锚定在一个真实的 `publ` 字段上。

### 5. **关键性质证明（形式化工作的核心产出）**

基于以上定义，我们可以证明以下关键性质，它们是 TSL 安全性和可靠性的基石：

- **终止性（Termination）**：对于任何良构的 TS 树 $\mathcal{T}$ 和任何节点 $n \in \mathcal{T}$，$\text{Resolve}(n, f)$ 要么在有限步内返回一个结果 $(m, f_{publ})$，要么返回 $\bot$。**证明思路**：由于 TS 树本身无环，且 `Lift`/`Push` 建立的绑定关系严格遵循父子方向（从子到父，或从父到子的 `publ`），由 `BindingRef` 构成的依赖图也必然是无环的，因此递归调用必然终止。

- **唯一物理源（Unique Physical Source）**：如果 $\text{Resolve}(n, f) = (m, f_{publ})$，那么 $f_{publ} \in \text{dom}(\mathcal{D}_{publ}^m)$，并且这是该字段引用所对应的**唯一**物理存储位置。所有共享该 `ance` 字段的节点，最终都会解析到这个相同的 $(m, f_{publ})$ 对，从而实现零拷贝共享。

- **静态可推理性（Static Reasonability）**：编译器可以在编译期检查所有 `ance` 字段是否在使用前被正确声明。虽然无法100%静态证明所有 `ance` 字段都会被履行（这取决于运行时的 `lift` 调用），但它可以构建完整的绑定依赖图，并验证类型一致性（例如，绑定链上的所有字段必须具有相同类型）。

---

## TSL 的指针解析与指针运算

### 一、 前置定义回顾与扩展

为保证自洽性，我们首先明确并扩展几个关键概念。

#### 1.1 节点段布局函数

对于任意节点实例 $n$，我们定义其段布局函数：
$$
\text{SegLayout}^n: \text{SegmentID} \rightarrow (\text{start}, \text{end})
$$
该函数返回每个段（如 `data_ance`, `data_publ` 等）在虚拟地址空间 $V_n$ 中的起始和结束地址。此布局在节点创建时即固定。

#### 1.2 字段到偏移的映射

对于节点 $n$ 的每个字段段，我们有确定的字段到偏移映射：
- $\text{FieldOffset}_{publ}^n: \text{dom}(\mathcal{D}_{publ}^n) \rightarrow \mathbb{N}$
- $\text{FieldOffset}_{priv}^n: \text{dom}(\mathcal{D}_{priv}^n) \rightarrow \mathbb{N}$
- $\text{FieldOffset}_{ance}^n: \text{dom}(\mathcal{D}_{ance}^n) \rightarrow \mathbb{N}$

这些映射由编译器根据类型环境 $\Theta$ 静态计算得出。

#### 1.3 指针值的正式定义

一个指针值 $p$ 是一个二元组：
$$
p ::= (v, n_{\text{src}})
$$
其中：
- $n_{\text{src}}$ 是一个有效的节点实例。
- $v \in V_{n_{\text{src}}}$ 是一个虚拟地址。

---

### 二、 指针值的解析：$\text{Deref}(p)$

我们的目标是定义一个函数 $\text{Deref}(p)$，它接受一个指针 $p$，并返回其最终指向的**物理存储单元**，即一个 $(m, f_{publ})$ 对。

整个过程分为两个阶段：**本地字段定位** 和 **远程绑定解析**。

#### 2.1 阶段一：本地字段定位 $\text{LocateField}(n, v)$

给定一个节点 $n$ 和其虚拟地址空间内的地址 $v$，定位 $v$ 所属的具体字段。

**定义**：
$$
\text{LocateField}(n, v) =
\begin{cases}
\texttt{LocalPubl}(f) & \text{if } v \in [\text{SegLayout}^n(\texttt{data\_publ}).\text{start}, \text{SegLayout}^n(\texttt{data\_publ}).\text{end}) \\
& \quad \text{and } \exists f, v = \text{SegLayout}^n(\texttt{data\_publ}).\text{start} + \text{FieldOffset}_{publ}^n(f) \\
\texttt{LocalPriv}(f) & \text{if } v \in [\text{SegLayout}^n(\texttt{data\_priv}).\text{start}, \text{SegLayout}^n(\texttt{data\_priv}).\text{end}) \\
& \quad \text{and } \exists f, v = \text{SegLayout}^n(\texttt{data\_priv}).\text{start} + \text{FieldOffset}_{priv}^n(f) \\
\texttt{AnceRef}(f) & \text{if } v \in [\text{SegLayout}^n(\texttt{data\_ance}).\text{start}, \text{SegLayout}^n(\texttt{data\_ance}).\text{end}) \\
& \quad \text{and } \exists f, v = \text{SegLayout}^n(\texttt{data\_ance}).\text{start} + \text{FieldOffset}_{ance}^n(f) \\
\bot & \text{otherwise (invalid address)}
\end{cases}
$$

#### 2.2 阶段二：完整解引用 $\text{Deref}(p)$

现在，我们可以定义完整的解引用函数。

**定义**：
令 $p = (v, n_{\text{src}})$。
1.  计算 $loc = \text{LocateField}(n_{\text{src}}, v)$。
2.  根据 $loc$ 的结果分情况处理：
    - **[D1 - 本地公开/私有字段]**：
        - 若 $loc = \texttt{LocalPubl}(f)$ 或 $loc = \texttt{LocalPriv}(f)$，则
          $$
          \text{Deref}(p) = (n_{\text{src}}, f)
          $$
    - **[D2 - `ance` 字段引用]**：
        - 若 $loc = \texttt{AnceRef}(f)$，则调用之前定义的字段解析函数：
          $$
          \text{Deref}(p) = \text{Resolve}(n_{\text{src}}, f)
          $$
          （回忆：$\text{Resolve}$ 要么返回 $(m, f_{publ})$，要么返回 $\bot$）
    - **[D3 - 无效地址]**：
        - 若 $loc = \bot$，则 $\text{Deref}(p) = \bot$。

> **关键性质**：如果 $\text{Deref}(p) \neq \bot$，其结果必然是 $(m, f_{publ})$ 的形式，其中 $f_{publ} \in \text{dom}(\mathcal{D}_{publ}^m)$。这保证了所有有效指针最终都指向一个明确的、可写的物理存储位置。

---

### 三、 指针运算的形式化

指针运算（主要是加法）非常直接，因为它只操作指针的虚拟地址部分，不改变其源节点上下文。

#### 3.1 指针加法 $\text{PtrAdd}(p, k)$

**定义**：
令 $p = (v, n_{\text{src}})$ 为一个指针，$k \in \mathbb{Z}$ 为一个整数偏移量。
$$
\text{PtrAdd}(p, k) = (v + k, n_{\text{src}})
$$
结果是一个新的指针值 $p'$。

#### 3.2 运算后的解析

对运算结果进行解析，遵循相同的 $\text{Deref}$ 规则：
$$
\text{Deref}(\text{PtrAdd}(p, k)) = \text{Deref}((v + k, n_{\text{src}}))
$$
这个新指针可能：
- 指向同一个字段内的不同字节（如果 $k$ 很小）。
- 指向相邻的另一个字段。
- 指向段外的无效地址（此时 $\text{Deref} = \bot$）。
- **关键点**：即使 $p$ 原本指向一个 `ance` 字段，$p+k$ 也仍然是相对于 $n_{\text{src}}$ 的地址。只有在调用 $\text{Deref}(p+k)$ 时，才会触发对新地址 $v+k$ 的 `LocateField` 和可能的 `Resolve`。

> **示例**：假设 `n_src.data_ance` 段有两个 `i32` 字段 `a` 和 `b`，它们在 $V_{n_{\text{src}}}$ 中连续排列。指针 `p_a = (&a, n_src)`。那么 `p_b = PtrAdd(p_a, 4)`。当执行 `Deref(p_b)` 时，`LocateField` 会发现 `v+4` 对应的是字段 `b`，然后调用 `Resolve(n_src, b)`。

---

### 四、 形式化总结

通过以上定义，我们完成了对 TSL 指针语义的严格形式化：

- **指针值**：被精确定义为 `(虚拟地址, 源节点)` 对。
- **指针解析 (`Deref`)**：是一个两阶段过程：
  1.  **结构化定位** (`LocateField`)：利用编译期确定的段布局和字段偏移，将虚拟地址映射回源节点内声明的具体字段名。
  2.  **语义解析**：如果是本地字段，直接返回；如果是 `ance` 字段，则委托给已形式化的 `Resolve` 函数，完成跨节点的绑定链解析。
- **指针运算 (`PtrAdd`)**：仅修改虚拟地址分量，产生一个新的、同样带有源节点上下文的指针。其有效性完全由后续的 `Deref` 调用来决定。

---

## TSL 字段解析设计说明

> 本节内容为字段解析的文字叙述。

TSL 采用基于**节点命名空间**与**显式绑定规则**的字段解析机制。

一个节点即一个完整、封闭的命名空间。其中的字段分为三类：`ance`、`publ` 和 `priv`，各自具有明确的可见性语义与生命周期约束：

- **`publ` 字段**：属于当前节点自身的变量，**允许所有子孙节点通过祖先链回溯访问**（前提是子孙节点未遮蔽该名称）。`publ` 字段是跨节点共享数据的主要接口，通常用于 CTRN（Cross Tree-Stacked Reference Node）等纯数据载体。

- **`priv` 字段**：属于当前节点自身的变量，**仅在本节点内部可见**，子孙节点无法访问。用于封装节点私有状态，确保信息隐藏。

- **`ance` 字段**：**不持有实际存储**，而是表示对**某个祖先（或潜在祖先）节点中某字段的引用承诺**。其语义如下：
  - 必须在节点定义中显式声明（如 `ance { int x; }`），作为对外部依赖的接口契约；
  - 实际绑定在节点创建时（`push`）由父节点注入，格式为 `(ancestor_field => local_ance_name)`；
  - 绑定目标可以是父节点自身的 `publ` 字段，也可以是父节点尚未履行的 `ance` 字段（形成绑定链）；
  - 当某祖先节点后续通过 `lift` 操作将真实数据源（如 CTRN 的 `publ` 字段）注入到该绑定链的上游时，整个链自动解析至同一物理存储；
  - 字段查找时，`ance` 字段通过**递归解引用绑定链**，最终定位到形如 `real_source_node.publ.field_name` 的实际数据源。

字段解析遵循以下优先级规则（在当前节点内）：
1. 若字段名存在于 `priv` 或 `publ` 中，直接返回本地值；
2. 若存在于 `ance` 中，则沿绑定链递归解析，直至抵达真实 `publ` 字段
3. 否则，这是一个无效字段。

> **关键原则**：任何对非本地字段的访问，**必须通过 `ance` 块显式声明**。运行时祖先链的变化不会自动扩展当前节点的可见字段集——这确保了静态作用域的稳定性与程序的可推理性。

---

### `ance` 字段的延迟绑定与结构化共享

#### 基本语法与语义

```tsl
data {
    ance {
        type field_name;
        // 可声明多个字段，类型可为 int, bool, fn 等
    }
}
```

- **作用**：声明本节点需要从祖先链中接收的字段；
- **可见性**：仅在本节点内部可见（如同 `priv`），但实际存储位于其他节点；
- **生命周期**：绑定关系随节点树结构动态建立/销毁；
- **默认状态**：创建时未绑定（unfulfilled），直接读取将阻塞或报错（取决于运行时策略）。

---

#### 绑定机制：两阶段模型

##### 阶段一：**声明与预绑定（at `push`）**

父节点在创建子节点时，可将其自身的 `ance` 字段（即使尚未履行）绑定给子节点：

```tsl
push child (ChildType () (parent_field => child_field) ())
```

- 要求：`parent_field` 必须是当前节点已声明的 `ance` 字段；
- 效果：子节点的 `child_field` 与父节点的 `parent_field` 形成 **绑定链**；
- 此时若 `parent_field` 未履行，子节点同样处于“等待”状态。

##### 阶段二：**履行承诺（at `lift`）**

当某节点执行 `lift`，将其子节点的字段绑定到自身 `ance` 字段时，承诺被履行：

```tsl
lift data_source ((source_field => local_ance_field))
```

- 效果：
  1. 本地 `local_ance_field` 解析为 `data_source.source_field`；
  2. **自动递归传播**：所有绑定到 `local_ance_field` 的子孙节点，同步获得解析能力；
- 结果：整个绑定链指向同一物理存储，实现零拷贝共享。

---

#### 完整示例：延迟绑定与自动传导

```tsl
node example_father {
    data {
        ance {
            int example_value;  // 承诺：将由某个祖先提供
        }
    }
    code {
        instruct {
            // 1. 先创建子节点，绑定尚未履行的承诺
            push son (example_son () (example_value => example_element) ());
            
            // 2. 创建真实数据源
            push buffer_node (example_buffer () () ());
            
            // 3. 履行承诺：将 buffer_node.a 绑定到自己的 example_value
            lift buffer_node ((a => example_value));
            set example_value 0x1;
            // → 自动传导：son.example_element 现在可访问 buffer_node.a
        }
    }
}

node example_son {
    data {
        ance {
            int example_element;  // 依赖父节点提供的 example_value
        }
        priv {
            int extra_value;
        }
    }
    code {
        instruct {
            set extra_value 0x1;
            // 此指令将等待 example_element 被绑定后执行
            add example_element extra_value;
        }
    }
}

node example_buffer {
    data {
        publ {
            int a;  // 真实数据源
        }
    }
}
```

##### 执行流程说明：
1. `example_father` 创建 `son`，将其未履行的 `example_value` 绑定给 `son.example_element`；
2. 创建 `buffer_node`，其 `a` 字段初始为未定义（或默认值）；
3. `lift` 将 `buffer_node.a` 绑定到 `father.example_value`；
4. 绑定链自动建立：  
   `son.example_element → father.example_value → buffer_node.a`；
5. `son` 中的 `add` 指令现在可安全执行，直接操作 `buffer_node.a`。

---

#### 安全与验证保证

- **静态检查**：编译器确保所有 `ance` 字段在使用前已被声明；
- **动态检查**（可选）：
  - 若读取未履行的 `ance` 字段，节点进入 `blocked` 状态，等待绑定完成；
  - 或立即报错（适用于确定性系统）；
- **无循环绑定**：TS 形式化模型禁止绑定图中出现环；
- **作用域隔离**：未声明 `ance` 字段的节点，即使位于相同祖先链，也无法访问该字段。
