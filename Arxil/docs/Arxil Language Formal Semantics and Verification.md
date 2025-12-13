<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: Apache-2.0
-->

# Arxil Language Formal Semantics and Verification

---

> v0.1.1

---

## 1. 字段环境的形式化

我们将节点 $n$ 的字段环境 $\mathcal{D}^n$ 形式化为一个三元组：

$$
\mathcal{D}^n = (\mathcal{D}_{\text{priv}}^n,\ \mathcal{D}_{\text{publ}}^n,\ \mathcal{D}_{\text{ance}}^n)
$$

其中：
- $\mathcal{D}_{\text{priv}}^n : \text{Name} \rightharpoonup \text{Value}$ 是私有字段映射；
- $\mathcal{D}_{\text{publ}}^n : \text{Name} \rightharpoonup \text{Value}$ 是公开字段映射；
- $\mathcal{D}_{\text{ance}}^n : \text{Name} \rightharpoonup \text{BindingRef}$ 是 `ance` 字段到绑定引用的映射。

---

## 2. 字段解析函数 $\text{Resolve}(n, f)$

$\text{Resolve}$ 是一个偏函数，目标是将字段引用 $(n, f)$ 解析到其最终的物理存储位置 $(m, f_{\text{publ}})$，其中 $f_{\text{publ}} \in \text{dom}(\mathcal{D}_{\text{publ}}^m)$。

**定义**：给定良构 AS 树 $\mathcal{T}$ 和节点 $n \in \mathcal{T}$，对任意字段名 $f$：

- **[R1 - 本地字段]**  
  若 $f \in \text{dom}(\mathcal{D}_{\text{priv}}^n) \cup \text{dom}(\mathcal{D}_{\text{publ}}^n)$，则  
  $$
  \text{Resolve}(n, f) = (n, f)
  $$

- **[R2 - `ance` 字段解析]**  
  若 $f \in \text{dom}(\mathcal{D}_{\text{ance}}^n)$，令 $\text{ref} = \mathcal{D}_{\text{ance}}^n(f)$：
  - **[R2a - 未绑定]** 若 $\text{ref} = \texttt{Unbound}$，则 $\text{Resolve}(n, f) = \bot$（未定义，运行时错误或阻塞）；
  - **[R2b - 已绑定]** 若 $\text{ref} = \texttt{Bound}(m, f')$，则  
    $$
    \text{Resolve}(n, f) = \text{Resolve}(m, f')
    $$
    （递归调用）。

- **[R3 - 无隐式回溯]**  
  若以上均不满足，则 $\text{Resolve}(n, f) = \bot$。**不存在对祖先链的隐式遍历查找**。

---

## 3. 绑定操作的形式语义

为保证 $\text{Resolve}$ 行为符合预期，需精确定义 `psh` 与 `lft` 对 `BindingRef` 的操作。

### `psh` 操作的扩展语义

当执行  
```axl
push child (NodeTemplate () (parent_field => child_field) ...)
```  
在创建子节点 $c$ 后，必须执行：

$$
c.\mathcal{D}_{\text{ance}}(child\_field) := \texttt{Bound}(n, parent\_field)
$$

其中 $n$ 是执行 `psh` 的父节点。此操作建立初始绑定链，无论 $n.\mathcal{D}_{\text{ance}}(parent\_field)$ 当前是否已绑定。

---

### `lft` 操作的扩展语义

当执行  
```axl
lift external_node ((source_field => local_ance_field))
```  
必须满足：
- $\texttt{data\_source} \in \text{DirectChildren}(n)$
- $\texttt{source\_field} \in \text{dom}(data\_source.\mathcal{D}_{\text{publ}})$
- $\texttt{local\_ance\_field} \in \text{dom}(n.\mathcal{D}_{\text{ance}})$

操作成功后，执行：

$$
n.\mathcal{D}_{\text{ance}}(local\_ance\_field) := \texttt{Bound}(data\_source, source\_field)
$$

此赋值履行承诺，并将绑定链末端锚定于真实的 `publ` 字段。

---

## 4. 关键性质证明

基于上述定义，可证明以下核心性质：

### **终止性（Termination）**

> 对任何良构 AS 树 $\mathcal{T}$ 和节点 $n \in \mathcal{T}$，$\text{Resolve}(n, f)$ 要么在有限步内返回 $(m, f_{\text{publ}})$，要么返回 $\bot$。

**证明思路**：AS 树本身无环，且 `psh`/`lft` 建立的绑定关系严格遵循父子方向（从子到父，或从父到子的 `publ`），由 `BindingRef` 构成的依赖图亦无环，故递归调用必终止。

---

### **唯一物理源（Unique Physical Source）**

> 若 $\text{Resolve}(n, f) = (m, f_{\text{publ}})$，则 $f_{\text{publ}} \in \text{dom}(\mathcal{D}_{\text{publ}}^m)$，且这是该字段引用所对应的唯一物理存储位置。所有共享该 `ance` 字段的节点，最终均解析到同一 $(m, f_{\text{publ}})$ 对，实现零拷贝共享。

---

### **静态可推理性（Static Reasonability）**

> 编译器可在编译期检查所有 `ance` 字段是否在使用前被正确声明。虽无法 100% 静态证明所有 `ance` 字段都会被履行（取决于运行时 `lift` 调用），但可构建完整的绑定依赖图，并验证类型一致性（如绑定链上所有字段类型相同）。

---

## 5. 指针语义的形式化

### 指针值的正式定义

一个指针值 $p$ 是一个二元组：

$$
p ::= (v, n_{\text{src}})
$$

其中：
- $n_{\text{src}}$ 是有效的节点实例；
- $v \in V_{n_{\text{src}}}$ 是其虚拟地址空间中的地址。

---

### 指针值的解析：$\text{Deref}(p)$

我们的目标是定义一个函数 $\text{Deref}(p)$，它接受一个指针 $p$，并返回其最终指向的**物理存储单元**，即一个 $(m, f_{publ})$ 对。

整个过程分为两个阶段：**本地字段定位** 和 **远程绑定解析**。

#### 阶段一：本地字段定位 $\text{LocateField}(n, v)$

给定节点 $n$ 与地址 $v \in V_n$，定义：

$$
\text{LocateField}(n, v) =
\begin{cases}
\texttt{LocalPubl}(f) & \text{if } v \in [\text{SegLayout}^n(\texttt{data\_publ}).\text{start},\ \text{end}) \\
& \quad \text{and } \exists f,\ v = \text{start} + \text{FieldOffset}_{\text{publ}}^n(f) \\
\texttt{LocalPriv}(f) & \text{if } v \in [\text{SegLayout}^n(\texttt{data\_priv}).\text{start},\ \text{end}) \\
& \quad \text{and } \exists f,\ v = \text{start} + \text{FieldOffset}_{\text{priv}}^n(f) \\
\texttt{AnceRef}(f) & \text{if } v \in [\text{SegLayout}^n(\texttt{data\_ance}).\text{start},\ \text{end}) \\
& \quad \text{and } \exists f,\ v = \text{start} + \text{FieldOffset}_{\text{ance}}^n(f) \\
\bot & \text{otherwise (invalid address)}
\end{cases}
$$

#### 阶段二：完整解引用 $\text{Deref}(p)$

令 $p = (v, n_{\text{src}})$，计算 $loc = \text{LocateField}(n_{\text{src}}, v)$：

- **[D1]** 若 $loc = \texttt{LocalPubl}(f)$ 或 $\texttt{LocalPriv}(f)$，则  
  $$
  \text{Deref}(p) = (n_{\text{src}}, f)
  $$

- **[D2]** 若 $loc = \texttt{AnceRef}(f)$，则  
  $$
  \text{Deref}(p) = \text{Resolve}(n_{\text{src}}, f)
  $$

- **[D3]** 若 $loc = \bot$，则 $\text{Deref}(p) = \bot$

> **关键性质**：若 $\text{Deref}(p) \neq \bot$，其结果必为 $(m, f_{\text{publ}})$ 形式，其中 $f_{\text{publ}} \in \text{dom}(\mathcal{D}_{\text{publ}}^m)$。

---

## 6. 段布局与地址映射

### 节点段布局函数

对任意节点 $n$，定义其段布局函数：

$$
\text{SegLayout}^n : \text{SegmentID} \rightarrow (\text{start}, \text{end})
$$

返回各段在虚拟地址空间 $V_n$ 中的起止地址。

---

### 字段到偏移的映射

编译器根据类型环境 $\Theta$ 静态计算：

- $\text{FieldOffset}_{\text{publ}}^n : \text{dom}(\mathcal{D}_{\text{publ}}^n) \to \mathbb{N}$
- $\text{FieldOffset}_{\text{priv}}^n : \text{dom}(\mathcal{D}_{\text{priv}}^n) \to \mathbb{N}$
- $\text{FieldOffset}_{\text{ance}}^n : \text{dom}(\mathcal{D}_{\text{ance}}^n) \to \mathbb{N}$

---

## 7. 类型环境与操作合法性

### 编译期类型环境 $\Theta$

$$
\Theta : \text{TypeID} \rightarrow \{ \text{size} : \mathbb{N},\ \text{ops} : \text{OpSet},\ \text{interop} : \text{InteropRules} \}
$$

由前端或标准库提供，用于：
- 计算字段偏移；
- 检查 opcode 操作数类型是否支持该操作；
- 验证函数调用参数匹配。

---

### Opcode 合法性判定规则

所有 opcode（如 `add`, `load`, `cpy`）本身无类型。其合法性由上下文决定：

> `add %z %x %y` 是良类型的，当且仅当  
> $$
> \Gamma(x) = \Gamma(y) = \Gamma(z) = \tau \quad \text{且} \quad \texttt{add} \in \Theta(\tau).\text{ops}
> $$

其中 $\Gamma$ 为当前作用域的字段类型映射。
**小步语义（如 IEEE 754 加法）由 `$\Theta(\tau).\text{eval_op}$` 外置定义，不属于语言本体。**

---

### 运行时行为

- **无类型信息保留**：生成的 ArxilVM 字节码或 native code 不包含类型元数据；
- **错误处理**：类型错误在编译期报错；运行时越界/UB 由 EDSOS/ArxilVM 的边界机制捕获（非语言责任）。

---
