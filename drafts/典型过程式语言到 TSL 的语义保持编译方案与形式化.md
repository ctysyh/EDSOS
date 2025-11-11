# 典型过程式语言到 TSL 的语义保持编译方案与形式化

> **构建从一个典型过程式核心语言（Imperative Core）到 TSL 的语义保持编译方案，并证明其正确性。**
> 采用**小步操作语义（small-step operational semantics）** 作为源语言基础，目标语言为 TSL 节点系统。证明策略为**模拟关系（simulation relation）**。

---

## 一、源语言：Imperative Core（IC）

### 1.1 语法
我们定义一个极简但具代表性的过程式语言：

```
e ::= x                     // 变量
    | n                     // 整数字面量
    | e₁ + e₂ | e₁ - e₂     // 算术
    | *e                    // 解引用（堆访问）
    | &x                    // 取地址（仅局部变量）

s ::= skip                  // 空语句
    | x := e                // 赋值（局部或全局）
    | *e := e'              // 堆写入
    | s₁; s₂                // 顺序
    | if e then s₁ else s₂  // 条件
    | while e do s          // 循环
    | call f(e₁,…,eₙ)       // 函数调用（无返回值简化；可扩展）
    | alloc x               // x = malloc(); 分配新堆单元
    | free e                // 释放堆单元

P ::= decl glob*; fundef f(x₁,…,xₙ) { s }*; main { s }
```

### 1.2 状态模型
- **栈帧栈**：$ \Sigma = [F_0, F_1, ..., F_k] $，其中 $F_i = (\text{locals}_i, \text{pc}_i)$
- **堆**：$ H : \mathbb{N} \rightharpoonup \mathbb{Z} $，地址为自然数
- **全局变量**：$ G : \text{Var}_{\text{glob}} \to \mathbb{Z} $
- **程序计数器**隐含在语句序列中

### 1.3 小步语义（示例规则）
我们仅列关键规则（完整略）：

- **赋值**：
  $$
  \frac{x \in \text{dom}(F_k.\text{locals})}{(G, \Sigma, H),\ x := e \longrightarrow (G, \Sigma[x \mapsto ⟦e⟧], H),\ \text{skip}}
  $$

- **堆分配**：
  $$
  \frac{a \notin \text{dom}(H)}{(G, \Sigma, H),\ \text{alloc } x \longrightarrow (G, \Sigma[x \mapsto a], H[a \mapsto 0]),\ \text{skip}}
  $$

- **函数调用**（简化：无参数传递，无返回）：
  $$
  \frac{f() \{ s \} \in P}{(G, \Sigma, H),\ \text{call } f() \longrightarrow (G, \Sigma \cdot [(\emptyset, s)], H),\ \text{skip}}
  $$

- **函数返回**（当当前帧为空语句）：
  $$
  (G, \Sigma \cdot [(\_, \text{skip})], H) \longrightarrow (G, \Sigma, H)
  $$

---

## 二、目标语言：TSL 节点系统（TS）

我们使用《TS Formal Semantics and Verification.md》中的模型：

- 全局堆 $ \mathcal{H} $：由 `node(n, …)` 和 `tree(n)` 谓词描述；
- 节点状态：$ n.\mathcal{D} = (\mathcal{D}_{\text{priv}}, \mathcal{D}_{\text{publ}}, \mathcal{D}_{\text{ance}}) $；
- 指令执行：`add`, `cpy`, `cond`, `cycl`, `push`, `pop`, `lift` 等；
- 控制流：通过 `pc` 和 `exe` 实现跳转。

---

## 三、翻译方案 $⟦·⟧$

我们定义翻译函数 $⟦·⟧$，将 IC 程序映射为 TSL 节点树。

### 3.1 全局变量 → 根节点 `publ`
- 对每个全局变量 `g`，在根节点 `root` 中声明：
  ```tsl
  publ { int g; }
  ```

### 3.2 函数 → 节点类型模板
- 对函数 `f(x₁,…,xₙ)`，定义节点类型 `FType`：
  ```tsl
  node FType {
    data {
      priv { int x₁, ..., xₙ, temp₁, ...; }
    }
    code {
      instruct { ⟦s⟧_code }
    }
  }
  ```
- 局部变量和临时变量均放入 `priv`。

### 3.3 语句翻译 $⟦s⟧_{\text{code}}$

我们递归翻译语句为 TSL 指令序列（或 `fn` 块）：

| IC 语句 | TSL 翻译 |
|--------|--------|
| `skip` | （空） |
| `x := e` | `cpy x ⟦e⟧`（若 `x` 是局部）<br>`cpy root.g ⟦e⟧`（若 `g` 是全局，需通过 `ance` 绑定） |
| `*e := e'` | 设 `e` 解析为地址 `a`，对应 CTRN 节点 `n_a`，则：<br>`cpy n_a.data ⟦e'⟧`（通过 `lift` 后的 `ance` 字段访问） |
| `s₁; s₂` | 指令拼接：`⟦s₁⟧; ⟦s₂⟧` |
| `if e then s₁ else s₂` | `cond ⟦e⟧ ({ exe L1; }, { exe L2; })`<br>其中 `L1 = ⟦s₁⟧`，`L2 = ⟦s₂⟧` |
| `while e do s` | 使用 `cycl`：<br>`cycl end_flag step_act`<br>其中 `step_act = ⟦s⟧; update end_flag = !e` |
| `alloc x` | `push ctrn (CTRN_Int () ()); lift ctrn ((data => x_ptr)); set x_ptr_addr ctrn;`<br>（`x_ptr` 是 `ance` 字段，`x_ptr_addr` 存节点 ID 用于 free） |
| `free e` | `pop ⟦e⟧_addr`（通过存储的节点 ID） |
| `call f(e₁,…,eₙ)` | ```tsl<br>push f_node (FType ()<br>  (⟦e₁⟧ => x₁), ..., (⟦eₙ⟧ => xₙ)<br>);<br>pop f_node;<br>``` |

> 注：表达式 $⟦e⟧$ 翻译为字段名或立即数。

### 3.4 主函数 → 根节点 `instruct`
- `main { s }` → 根节点的 `instruct` 段为 `⟦s⟧_code`。

---

## 四、模拟关系（Simulation Relation） $\mathcal{R}$

我们定义关系 $\mathcal{R} \subseteq (\text{IC-State}) \times (\text{TS-Heap})$：

> $(G, \Sigma, H) \ \mathcal{R}\ \mathcal{H}$ 当且仅当：

1. **全局一致性**：  
   $\forall g.\ G(g) = \text{value of } \texttt{root.publ.g} \text{ in } \mathcal{H}$

2. **栈帧对应**：  
   对 $\Sigma = [F_0, ..., F_k]$，存在 TS 节点链 $n_0 = \text{root}, n_1, ..., n_k$，使得：
   - $n_i$ 是 $n_{i-1}$ 的子节点；
   - $n_i.\mathcal{D}_{\text{priv}}$ 包含 $F_i.\text{locals}$ 的所有变量，且值一致；
   - $n_i$ 的 `pc` 对应 $F_i$ 的当前语句位置。

3. **堆一致性**：  
   对每个 $a \in \text{dom}(H)$，存在唯一 CTRN 节点 $n_a \in \mathcal{H}$，使得：
   - $n_a.\text{publ.data} = H(a)$
   - 所有指向 $a$ 的指针（在 IC 栈/全局中）在 TS 中对应 `ance` 字段绑定至 $n_a.\text{data}$

4. **良构性**：$\mathcal{H}$ 是良构 TS 树（满足祖先一致性、无环等）。

---

## 五、模拟定理（Simulation Theorem）

> **定理**：若 $(G, \Sigma, H) \ \mathcal{R}\ \mathcal{H}$，且 $(G, \Sigma, H), s \longrightarrow (G', \Sigma', H'), s'$，  
> 则存在 $\mathcal{H}'$ 使得 $\mathcal{H} \xrightarrow{\text{TSL steps}}^+ \mathcal{H}'$ 且 $(G', \Sigma', H') \ \mathcal{R}\ \mathcal{H}'$。

### 证明思路（按语句分类）：

#### Case 1: `x := e`（局部赋值）
- IC：更新 $F_k(x) := v$
- TSL：执行 `cpy x v`，修改 $n_k.\mathcal{D}_{\text{priv}}(x)$
- $\mathcal{R}$ 保持：局部状态一致

#### Case 2: `alloc x`
- IC：选新地址 $a$，$H' = H[a \mapsto 0]$，$F_k(x) := a$
- TSL：`push ctrn`; `lift ctrn ((data => x_ptr))`; 存储 `ctrn` ID 到某字段
- 新 CTRN 节点 $n_a$ 满足 $n_a.\text{data} = 0$
- $\mathcal{R}$ 扩展：堆映射新增 $a \leftrightarrow n_a$

#### Case 3: `call f()`
- IC：压栈新帧 $F_{k+1} = (\emptyset, s_f)$
- TSL：`push f_node`; 其 `priv` 初始化为空
- 新节点 $n_{k+1}$ 成为 $n_k$ 的子节点
- $\mathcal{R}$ 扩展：栈帧链延长

#### Case 4: 函数返回（帧弹出）
- IC：弹出 $F_{k+1}$
- TSL：`pop f_node`（或自动在 `finish` 后回收）
- $\mathcal{R}$ 收缩：栈帧链缩短

#### Case 5: `*e := e'`
- IC：设 $e \Downarrow a$，则 $H'(a) = v$
- TSL：通过 `ance` 字段访问 $n_a.\text{data}$，执行 `cpy`
- 值更新一致

---

## 六、可观测行为等价

- **终止**：IC 终止当且仅当 TSL 根节点执行 `finish this`；
- **输出**：若 IC 有 `print(e)`，TSL 可映射为 `signal print_event` + 共享缓冲区；
- **错误**：IC 的段错误 ↔ TSL 的未绑定 `ance` 或越界访问（由 VM 捕获）。

因此，在忽略性能的前提下，**TSL 精确模拟 IC 的所有可观测行为**。

---

当然可以。我们在前述形式化框架基础上，**扩展对函数返回值的支持**，并**引入并发/并行语义的形式化映射**。这两项扩展均严格遵循 TS 模型的结构演化原则，并保持模拟关系 $\mathcal{R}$ 的可维护性。

---

## 七、带返回值的函数调用

### 7.1 源语言扩展（Imperative Core+）

我们增强函数语法：

```
e ::= ... | call f(e₁,…,eₙ)        // 表达式形式，有返回值
s ::= ... | x := call f(e₁,…,eₙ)   // 赋值语句
```

每个函数 `f` 声明为：
```
f(x₁,…,xₙ) → τ { s; return e_r; }
```

### 7.2 TSL 映射策略

我们采用 **“返回值字段 + Merge”** 策略（兼顾效率与通用性）：

- 在函数节点类型 `FType` 中声明 `publ { τ ret_val; }`；
- 函数体末尾执行：`cpy ret_val ⟦e_r⟧; finish this;`
- 调用点翻译为：
  ```tsl
  push f_node (FType () (arg_bindings));
  merge this ((ret_val => x));  // 将 f_node.ret_val 合并为当前节点的 x
  pop f_node;
  ```

若目标字段 `x` 已存在（如局部变量），则 `merge` 执行覆盖；若不存在，则在 `priv` 中创建。

### 7.3 形式语义更新

#### IC 返回规则：
$$
\frac{
f(\vec{x}) \to \tau \{ s; \text{return } e_r \} \in P \quad
\llbracket e_r \rrbracket_{\Sigma'} = v
}{
(G, \Sigma \cdot [(\vec{x} \mapsto \vec{v}, s)], H),\ \text{return } e_r
\longrightarrow
(G, \Sigma, H[v/\text{result}]),\ \text{skip}
}
$$

#### TSL 对应行为：
- `cpy ret_val v` → 设置 `f_node.publ.ret_val = v`
- `merge this ((ret_val => x))` → 执行《TS Formal Semantics》中的 **Merge(n, n*)** 规则：
  $$
  \text{node}(n, D) ∗ \text{node}(n^*, D^* \ni \text{ret\_val}) \Rightarrow \text{node}(n, D \cup \{x \mapsto v\})
  $$
- `pop f_node` → 回收子树

### 7.4 模拟关系 $\mathcal{R}$ 扩展

新增条款：

5. **返回值一致性**：  
   若 IC 栈顶帧正在执行 `return e`，且其调用者局部变量 `x` 将接收返回值，  
   则在 TS 中，当前函数节点 `n_f` 必包含 `publ.ret_val = v`，  
   且父节点 `n_p` 或其待合并状态中包含绑定 `x ← v` 的意图（可通过 pending merge flag 表示，或直接要求 `merge` 已执行）。

由于 `merge` 是原子操作（见 TS 形式语义 Rule 4），可在一步内完成值传递，**无需中间共享状态**，从而避免竞态。

结论：带返回值的函数调用可被精确、高效地映射，且保持模拟关系。

---

## 八、并发与并行的形式化映射

### 8.1 源语言扩展（IC∥）

添加并发原语：

```
s ::= ... 
    | fork f(e₁,…,eₙ)      // 创建新线程执行 f
    | join t                // 等待线程 t 结束（简化：忽略返回）
    | lock(m); s; unlock(m) // 互斥区（m 为 mutex 全局名）
```

线程标识符 $t \in \mathbb{T}$，全局线程表 $T: \mathbb{T} \rightharpoonup (\text{stack}, \text{state})$

### 8.2 TSL 并发模型基础

TS 模型天然支持：
- **节点并行**：不同子树可由调度器并发执行；
- **EDSOS Event**：`signal E`; `wait E` 提供同步屏障；
- **结构化共享**：通过 CTRN + `lift` 实现安全数据共享。

### 8.3 映射方案

| IC∥ 构造 | TSL 映射 |
|---------|--------|
| `fork f(e₁,…,eₙ)` | ```tsl<br>push worker (FType () (arg_bindings));<br>// 不立即 pop，worker 独立运行<br>store worker_id to thread_handle;<br>``` |
| `join t` | ```tsl<br>wait thread_t_finished;<br>// worker 在 finish 前 signal thread_t_finished all;<br>pop worker;<br>``` |
| `lock(m); s; unlock(m)` | ```tsl<br>wait mutex_m_acquired;<br>exe (() ()) critical_s;<br>signal mutex_m_released all;<br>```<br>（mutex_m 初始为 acquired） |

> **关键设计**：每个 mutex `m` 对应一个 EDSOS 事件；初始 `signal m_acquired all` 一次，后续靠 `wait/signal` 轮转。

### 8.4 共享堆的一致性

- 所有线程通过 `lift` 绑定到**同一 CTRN 节点**访问共享堆对象；
- 由于 `ance` 解析最终指向唯一物理源（《TSL Formal Semantics》第4节），**别名语义被精确保留**；
- 无数据竞争的前提：程序员正确使用 `lock/unlock`（TSL 不自动保证，但结构使错误更易检测）。

### 8.5 并发模拟关系 $\mathcal{R}_\parallel$

扩展 $\mathcal{R}$ 为多线程版本：

> $(G, \{ \Sigma_t \}_{t \in \mathbb{T}}, H) \ \mathcal{R}_\parallel\ \mathcal{H}$ 当且仅当：

1. **全局与堆一致性**：同前；
2. **线程对应**：对每个活跃线程 $t$，存在唯一叶子节点 $n_t \in \mathcal{H}$，其祖链从根出发，且 $n_t$ 的状态（`ready`/`running`/`blocked`）与 $\Sigma_t$ 的控制状态一致；
3. **同步事件对应**：每个 mutex `m` 对应一对 EDSOS 事件 `acquired_m` / `released_m`，其信号状态与 IC∥ 中锁持有状态一致；
4. **join 依赖**：若 `join t` 阻塞，则对应节点处于 `wait thread_t_finished` 状态。

### 8.6 并发模拟定理

> **定理（并发）**：若 $(G, \{\Sigma_t\}, H) \ \mathcal{R}_\parallel\ \mathcal{H}$，且某线程 $t$ 执行一步 $(..., s) \longrightarrow (...)', s')$，  
> 则存在 $\mathcal{H}'$ 使得 $\mathcal{H} \xrightarrow{\text{TSL}}^+ \mathcal{H}'$ 且 $(G', \{\Sigma'_t\}, H') \ \mathcal{R}_\parallel\ \mathcal{H}'$。

**证明要点**：
- `fork` → `push` 新节点，不阻塞父节点，符合 TS 节点并行语义；
- `join` → `wait` 事件，由 worker 在 `finish` 前 `signal`，满足同步；
- 临界区 → 事件屏障，确保互斥；
- 调度非确定性：TS 调度器可任意选择就绪节点执行，与 OS 线程调度语义一致。

结论：TSL 可精确模拟共享内存并发程序的行为，包括线程创建、同步、共享访问。

---
