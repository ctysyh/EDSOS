# TSL 对 π-calculus 与 Actor 模型的结构化模拟方案

> 待完善优化

---

## 一、基础设定

### 源语言语法（简化）

#### π-calculus（核心子集）

```text
P, Q ::= 0                     // 空进程
       | a⟨b⟩.P                // 输出：在通道 a 上发送名字 b
       | a(x).P                // 输入：在通道 a 上接收名字，绑定为 x
       | (νa)P                 // 私有通道
       | P | Q                 // 并行
       | !P                    // 复制
```
- 名字 $a, b$ 属于可传递的通道标识符集合 $\mathcal{N}_{\text{name}}$。

#### Actor 模型（核心子集）

```text
A ::= spawn(Body)              // 创建新 Actor，返回其地址 addr
    | send(addr, msg)          // 异步发送消息
    | receive { msg → Body }   // 串行处理下一条消息
    | self                      // 当前 Actor 地址
```

---

## 二、TSL 编码基础设施

### 1. **通道表示：祖先 publ 字段**
每个通道 $a$ 在 TS 中对应一个**祖先节点中的 `publ` 字段对**：
```tsl
publ {
    Name a_payload;   // 存放传递的名字（即另一字段的绑定引用）
    bool a_ready;     // 标志位（仅用于调试；实际同步靠 Event）
}
```
> 实际运行中，`a_ready` 可省略，因同步由 `wait a_event` 保证。

### 2. **名字表示：绑定引用（BindingRef）**
- 名字 $b$ 不是值，而是 **对某 publ 字段的绑定承诺**；
- 在 TSL 中，通过 `ance` 字段声明，如 `ance { Name x; }`；
- 其物理源由祖先链或 `Lift` 决定。

### 3. **事件命名规范**
- 每个通道 $a$ 配套一个 EDSOS Event：`event_a_ready`；
- 无内存实体，仅作调度屏障。

---

## 三、π-calculus → TSL 的结构化翻译

我们定义翻译函数 $\llbracket \cdot \rrbracket : \text{π-process} \to \text{TSL-node-type}$，参数为**当前作用域中已知的通道绑定环境 $\Gamma$**（$\Gamma(a) = (n_a, f_a)$ 表示通道 $a$ 对应节点 $n_a$ 的字段 $f_a$）。

### 翻译规则

#### (1) 空进程

$$
\llbracket 0 \rrbracket_\Gamma \triangleq
\texttt{node Nil \{ code \{ instruct \{ finish this 0; \} \} \}}
$$

#### (2) 输出 $a⟨b⟩.P$

假设 $\Gamma(a) = (n_a, \texttt{payload})$，且名字 $b$ 已解析为绑定 $(n_b, f_b)$。
$$
\llbracket a⟨b⟩.P \rrbracket_\Gamma \triangleq
\texttt{node Send\_a \{}
\quad \texttt{data \{}
\quad\quad \texttt{ance \{ Name out\_payload; \}}  // 绑定到 n_a.payload
\quad\quad \texttt{// 若 b 是本地字段，直接 cpy；否则需 ance 声明}
\quad \texttt{\}}
\quad \texttt{code \{}
\quad\quad \texttt{instruct \{}
\quad\quad\quad \texttt{cpy out\_payload } \llbracket b \rrbracket_{\text{name}}\texttt{;}
\quad\quad\quad \texttt{signal event\_a\_ready all;}
\quad\quad\quad \texttt{exe (() ()) } \llbracket P \rrbracket_\Gamma\texttt{;}
\quad\quad \texttt{\}}
\quad \texttt{\}}
\texttt{\}}
$$

> 注：$\llbracket b \rrbracket_{\text{name}}$ 返回字段名（如 `my_data`），该字段必须已在本节点或祖先中声明。

#### (3) 输入 $a(x).P$

$$
\llbracket a(x).P \rrbracket_\Gamma \triangleq
\texttt{node Recv\_a \{}
\quad \texttt{data \{}
\quad\quad \texttt{ance \{ Name in\_payload; \}}  // 绑定到 n_a.payload
\quad\quad \texttt{priv \{ Name x; \}}         // 接收缓冲区
\quad \texttt{\}}
\quad \texttt{code \{}
\quad\quad \texttt{instruct \{}
\quad\quad\quad \texttt{wait event\_a\_ready;}   // ← 无实体事件
\quad\quad\quad \texttt{cpy x in\_payload;}     // 解析绑定链，零拷贝
\quad\quad\quad \texttt{exe ((x) ()) } \llbracket P \rrbracket_{\Gamma[x \mapsto (this, x)]}\texttt{;}
\quad\quad \texttt{\}}
\quad \texttt{\}}
\texttt{\}}
$$

#### (4) 私有通道 $(\nu a)P$

在**当前节点内创建局部 publ 字段**，并扩展 $\Gamma$：

```tsl
// 在父节点中
publ { Name a_payload; }

// 然后翻译 P，其中 Γ' = Γ ∪ {a ↦ (this, a_payload)}
push worker (⟦P⟧_{Γ'} (...));
```

> 这利用了 TS 的**作用域封闭性**：子节点若未被绑定，无法访问 `a_payload`。

#### (5) 并行 $P \mid Q$

```tsl
push p_node (⟦P⟧_Γ ());
push q_node (⟦Q⟧_Γ ());
// 不立即 pop，允许并发执行
```

- 两个子节点共享同一祖先链 $\mathcal{A}$；
- 调度器可真并行执行。

#### (6) 复制 $!P$

```tsl
cycl stop_flag {
    push replica (⟦P⟧_Γ ());
    // 可选择是否 pop（若 pop 则串行复制，否则并发）
}
```

---

## 四、Actor → TSL 的结构化翻译

Actor 地址 = **节点 ID + 其公开接口字段绑定契约**。

### 翻译规则

#### (1) `spawn(Body)`

```tsl
// 创建新 Actor 节点
push new_actor (ActorBody () 
    // 将必要资源绑定给它，如日志、配置等
);
// 返回“地址”：即 new_actor 的节点 ID（运行时值）
// 调用者可通过 ance 字段后续绑定其 publ 接口
```

#### (2) `send(addr, msg)`

假设 `addr` 对应节点 $n$，其消息缓冲区为 `n.publ.inbox`，配套事件为 `event_inbox_ready`。

```tsl
// 发送方需已通过 ance 绑定该 inbox
cpy peer_inbox msg;           // 写入共享字段
signal event_inbox_ready all; // 触发接收方
```

#### (3) `receive { msg → Body }`

```tsl
node MyActor {
    data {
        publ { bool keep_to_run; }
        priv { Message current_msg; }
    }
    code {
        instruct {
            cycl keep_to_run 
        }
        publ {
            fn main () => () {
                inst_scri {
                    wait event_inbox_ready;   // ← 无实体事件
                    cpy current_msg inbox;    // inbox 通过 ance 绑定到某 publ 字段
                    exe ((current_msg) ()) body;
                    // 循环回到 receive
                }
            }
            fn body (Message) => () {
                // 实际处理逻辑
            }
        }
    }
}
```

#### (4) `self`

- 在 TSL 中，当前节点可通过 `this` 引用自身；
- 其“地址”即节点 ID，在 `push` 时由调度器分配；
- 若需传递给他人，需对方通过 `ance` 绑定其 publ 字段。

---

## 五、形式化语义对应（模拟关系 $\mathcal{R}$）

我们定义 $\mathcal{R} \subseteq (\text{π/Actor-State}) \times (\text{TS-Heap})$：

### 对 π-calculus

$(\Pi, \mathcal{N}) \ \mathcal{R}\ \mathcal{H}$ 当且仅当：
1. **通道一致性**：对每个通道 $a$，存在唯一祖先节点 $n_a$，使得  
   $n_a.\texttt{publ.a\_payload} = v$ 当且仅当 $\Pi$ 中最近一次 $a⟨b⟩$ 发送了对应名字；
2. **进程对应**：$\Pi$ 中每个活跃进程对应 $\mathcal{H}$ 中一个叶子节点，其 $\mathcal{A}$ 包含所有自由通道的源节点；
3. **事件状态**：若 $\Pi$ 中有进程阻塞于 $a(x)$，则对应 TS 节点状态为 `blocked`，且注册于 `event_a_ready`。

### 对 Actor

$(\mathcal{A}, M) \ \mathcal{R}\ \mathcal{H}$ 当且仅当：
1. **Actor 对应**：每个 Actor 对应一个 TS 节点，其 `priv` 存储私有状态；
2. **消息队列**：消息内容存储于某个 CTRN 或祖先 `publ` 字段；
3. **mailbox 同步**：若 Actor 阻塞于 `receive`，则其 TS 节点处于 `wait event_inbox_ready` 状态。

---

## 六、关键性质证明（概要）

### 定理（结构保真性）

> 若源程序（π 或 Actor）执行一步，则存在对应的 TSL 节点操作序列（`cpy`, `signal`, `wait`, `push` 等），使得模拟关系 $\mathcal{R}$ 保持。

**证明要点**：
- **输出/发送** → `cpy` + `signal`，修改 publ 字段并触发事件；
- **输入/接收** → `wait` + `cpy`，由调度器协调顺序；
- **创建** → `push`，新节点继承必要祖先链；
- **并行** → 多个兄弟节点，调度器并发执行；
- **无状态事件** → 不引入额外堆对象，符合 EDSOS 语义。

### 定理（隔离性）

> 无共同祖先且未显式绑定同一 CTRN 的节点，无法互相访问数据。

**证明**：由 $\text{Resolve}(n, f)$ 的定义（仅沿 $\mathcal{A}$ 和显式绑定链查找），平级节点无共享路径。

---

## 七、总结：TS 作为并发模型的统一结构基底

| 特性 | π-calculus | Actor | TS 实现方式 |
|------|------------|-------|-------------|
| **通信载体** | 通道名字 | Actor 地址 | **祖先 publ 字段 + ance 绑定** |
| **同步机制** | 阻塞输入 | 阻塞 receive | **EDSOS Event（无实体）** |
| **动态拓扑** | 名字传递 | 地址传递 | **绑定引用传递（能力安全）** |
| **并发单元** | 匿名进程 | 封装 Actor | **TS 节点（结构身份）** |
| **资源共享** | 无 | 无 | **显式结构共享（零拷贝）** |

**最终结论**：
TS 模型通过 **祖先链 $\mathcal{A}$ 提供静态作用域**、**EDSOS Event 提供无状态同步**、**字段声明策略控制可见性**，实现了对 π-calculus 与 Actor 模型的**结构保真、能力安全、零开销**嵌入。这不仅是模拟，更是**将并发语义内化为程序结构本身**——这正是 TS “计算即结构演化”理念的最高体现。