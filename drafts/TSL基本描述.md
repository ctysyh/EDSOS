# 典型示例

```Tsl
node example_main {
    meta_data {
        state
        capability
    }
    data {
        public {
            int add_buffer;
            int x;
            int y;
            int checksum;
            bool is_no_extra_op;
        }
        private {
            int add_buffer;
        }
    }
    code {
        instruct {
            exe (((0x1), (0x10)) (x)) hardware::random_int;
            exe (((0x1), (0x10)) (y)) hardware::random_int;
            add add_buffer ((x) (y));
            exe ((add_buffer) (checksum)) hardware::hardware_checksum;
            push add_op (example_check_add () ((add_buffer => add_buffer) (x => x) (y => y) (checksum => checksum) (is_no_extra_op => is_no_extra_op)) (hardware::hardware_checksum => __hardware_checksum));
            pop add_op 0;
            cond (is_no_extra_op) (({exe (() ()) submit_ac}) ({exe (() ()) submit_er}));
        }
        priv {
            fn submit_ac () => () {
                inst_scri {
                    signal example_add_ac all;
                    finish this 0;
                }
            }
            fn submit_er () => () {
                inst_scri {
                    err this 0
                }
            }
        }
        ance {
            lib hardware;
        }
    }
}

node example_check_add {
    meta_data {
        state
        capability
    }
    data {
        ance {
            int add_buffer;
            int x;
            int y;
            int checksum;
            bool is_no_extra_op;
        }
        public {
            bool is_right;
        }
        private {
            int checksum_buffer;
        }
    }
    code {
        instruct {
            push buffer_node (example_ctrn () () ());
            lift buffer_node ((a => add_buffer));
            exe ((x, y) (add_buffer)) add_int;
            exe ((add_buffer, checksum, __hardware_checksum) (checksum_buffer, is_right)) check;
            cpy is_right is_no_extra_op;
            push extra_op (example_extra_add () (add_buffer => add_buffer) ());
            pop extra_op;
            exe ((add_buffer) (checksum_buffer)) __hardware_checksum;
            and is_right ((checksum_buffer) (checksum));
            and is_no_extra_op ((is_right) (is_no_extra_op));
            finish this 0;
        }
        ance {
            fn __hardware_checksum (int) => (int);
        }
        publ {
            fn add_int ((int)a, (int)b) => ((int)result) {
                result = a + b;
            }
        }
        priv {
            fn check ((int)num, (int)rightsum, (fn)__hardware_checksum) => ((int)computesum, (bool)is_right) {
                computesum = __hardware_checksum(num);
                is_right = (computesum == rightsum);
            }
        }
    }
}

node example_ctrn {
    meta_data {
        state
        capability
        refcount
    }
    data {
        publ {
            int a;
        }
    }
    // 作为一个基本的 CTRN, 没有指令段, 直接省略
}

node example_extra_add {
    meta_data {
        state
        capability
    }
    data {
        ance {
            int add_buffer;
        }
        priv {
            int extra_value;
        }
    }
    code {
        instruct {
            exe ((0x1, 0x10) (extra_value)) hardware::random_int;
            add add_buffer extra_value;
            finish this 0;
        }
        ance {
            lib hardware;
        }
    }
}
```

---


# 系统性梳理总结

以下是以上文 TSL 示例为纲，对 **TS（Tree-Stacked）计算模型** 的设计哲学、语言机制、系统语义与工程实现路径的全面总结。

---

## 一、核心理念：计算即结构演化

TS 模型的根本主张是：

> **程序不是一段在固定地址空间中跳转的指令序列，而是一棵在运行时动态生长、收缩、协作的结构树。**

这带来三大范式转变：

| 传统模型 | TS 模型 |
|--------|--------|
| 状态 = 内存地址 + 寄存器 | 状态 = 节点字段（结构化命名空间） |
| 控制流 = PC 跳转 | 控制流 = 树的局部执行 + 全局结构变更 |
| 并发 = 线程/进程切换 | 并发 = 节点并行（调度器管理子树） |

**TSL（Tree-Stacked Language）** 是这一理念的文本表达：它不描述“做什么”，而是描述“如何组织自己以安全、高效地做”。

---

## 二、节点（Node）：统一栈帧与线程的结构原子

### 2.1 节点的双重身份

一个 `node` 可同时代表：
- **轻量级**：单个函数调用的栈帧（短生命周期、无调度）；
- **重量级**：独立线程或任务（长生命周期、可并行调度）。

这种统一性源于节点的三个核心属性：

| 属性 | 作用 |
|------|------|
| **结构嵌套** | 子节点天然继承父节点 capability 与数据可见性 |
| **封闭命名空间** | 节点内字段无需前缀，编译器自动解析作用域链 |
| **显式生命周期** | `push`/`pop`/`finish` 显式控制存在性，替代隐式栈分配或 GC |

### 2.2 节点操作语义

| 操作 | 语义 | 开销 | 并行性 |
|------|------|------|--------|
| `push N (...)` | 在当前节点下创建子节点 N | 中（需调度器注册） | 子节点可真并行 |
| `pop N` | 销毁子节点（refcount=0 时实际回收） | 低 | — |
| `exe (...) fn_name` | 跳转至同节点内的命名指令块 | 极低（≈ jmp） | 串行（同节点内） |
| `finish this` | 标记当前节点为 zombie，等待 pop | 无 | — |

> **关键原则**：  
> - **节点内串行，节点间并行**；  
> - **调度开销仅发生在节点边界**。

---

## 三、指令与函数：语义标签 vs 控制流跳转

### 3.1 `fn`：命名指令块，非传统函数

```tsl
fn add_int ((int)a, (int)b) => ((int)result) {
    result = a + b;
}
```

- **参数与返回值仅为接口契约**（human-facing 注解），无运行时语义；
- 实际执行时，`a`, `b`, `result` 必须已在当前节点作用域中存在；
- 编译为一段连续 instruct，入口地址绑定至符号 `add_int`。

### 3.2 `exe`：同节点跳转，零上下文切换

```tsl
exe ((x, y) (add_buffer)) add_int
```

- 语义：跳转至 `add_int` 块起始地址；
- **不改变 TLB 视图、capability 域、祖链**；
- 所有字段访问仍基于当前节点命名空间解析。

> **与传统 call 的本质区别**：  
> `exe` 是 **结构内指令地址跳转**，`call` 是 **栈帧切换**。

### 3.3 循环与递归的处理

- **循环**：重复 `exe` 同一 `fn` 块 + 条件跳转，**无需新节点**；
- **尾递归**：优化为循环；
- **非尾递归**：必须 `push` 新节点（保证祖链正确性，避免栈溢出）。

---

## 四、结构化共享：lift 安全零拷贝

```tsl
push buf (example_buffer_node)
lift buf ((a => ance_buffer))
```

- 语义：将 `buf.a` 绑定为当前节点 `ance_buffer` 的别名；
- 根据祖链解析访问。

> **这是对 shared_ptr / mmap / global var 的结构性超越**：  
> 共享 = 显式结构关系 + capability 保护。

---

## 五、与传统二进制（ELF/EXE）的兼容策略

TS 不追求取代现有生态，而是提供 **渐进式结构化路径**：

### 5.1 三层兼容模型

| 层级 | 方法 | 适用场景 |
|------|------|----------|
| **沙箱加载** | 将 ELF 进程加载为大型节点 | Legacy 应用无缝运行 |
| **IR 转换** | LLVM/WASM → TSL IR | 源码可用，希望获得 TS 性能和安全性 |
| **混合执行** | TS 节点 + 传统进程共存 | 系统逐步迁移 |

### 5.2 二进制到节点的映射规则

| 传统概念 | TS 映射 |
|--------|--------|
| 函数入口 | 节点 或 同节点 `fn` 块 |
| call/ret | `push`/`pop`（跨节点） 或 `exe`（同节点） |
| 栈帧 | 节点 data 段 |
| 全局变量 | 在根节点中声明并逐层传导给所有子节点 |
| pthread_create | `push` 新节点，或 `push` TS 子树的代理根节点并动态生长 |

> **目标**：让 legacy 代码“穿上 TS 外衣”，逐步演进为原生结构化程序。

---

## 六、编译器与运行时协同设计

### 6.1 编译器智能：节点粒度决策

编译器需在 **安全性、开销、并行性** 间权衡：

| 策略 | 触发条件 | 行为 |
|------|--------|------|
| **节点化** | 函数含 capability 需求、需并行、递归 | 生成新节点，`push` 调用 |
| **内联化** | leaf 函数、小代码、高频调用 | 生成同节点 `fn`，`exe` 调用 |
| **循环优化** | for/while 结构 | 编译为同节点循环块 |

### 6.2 运行时（EDSOS 调度器）职责

- 管理节点树的全局视图；
- 在 `push` 时注册可调度单元；
- 在节点 finish 后回收资源（考虑 refcount）；
- 保证跨节点内存访问的 capability 安全。

---


# 前沿思考

TSL 不仅是一种语言，更是一套 **从硬件到语义、从内存布局到并发模型、从安全边界到执行流编排** 的完整计算范式重构。

---

## 一、TSL vs Rust：从“所有权规则”到“结构化内存本体”

### 1.1 Rust 的局限：静态检查 ≠ 结构安全

Rust 的所有权系统虽强大，但本质仍是**对传统栈/堆模型的修补**：
- 它通过借用检查器（borrow checker）防止悬空指针和数据竞争；
- 但它**无法改变内存布局的隐式性**：变量仍分布在栈帧或堆块中，生命周期由 RAII 控制；
- 高频函数调用仍导致**栈帧频繁压入弹出**；
- 复杂数据结构（如图、树）仍需 `Rc<RefCell<T>>` 等运行时开销机制。

### 1.2 TSL 的突破：内存"长相"由 data 段显式定义

在 TSL 中：

```tsl
node example {
    data {
        publ { int x; bool flag; }
        priv { int temp; }
    }
}
```

- **整个节点的内存布局在编译期完全确定**；
- 字段不是“变量”，而是 **结构中的命名槽位**；
- 无栈帧概念：所有状态驻留在节点实例的 data 段中；
- 生命周期 = 节点存在性，由 `push`/`pop`/`finish` 显式控制。

**优势**：
- **零内存碎片**：节点分配可对齐、批量、甚至静态预分配；
- **无栈溢出风险**：递归 = 新节点，非栈增长；
- **缓存友好**：相关字段地址相邻（data 段虚拟地址连续）；
- **安全内建**：字段访问受结构作用域保护。

> **TSL 将“内存是什么样子”从运行时隐式行为变为编译期显式契约**。

---

## 二、重新定义“函数”与“栈帧”：过程抽象的结构化重生

### 2.1 “为什么要有栈帧？”——一个被忽视的根本问题

传统栈帧存在的历史原因：
- 支持递归（通过返回地址 + 局部变量隔离）；
- 实现函数调用约定（ABI）；
- 管理临时状态。

但在现代系统中，栈帧带来：
- 缓存局部性差（跨函数跳转破坏 TLB）；
- 栈溢出风险；
- 难以并行化（栈是线程私有且串行）。

### 2.2 TSL 的回答：用节点替代栈帧

- **每个逻辑单元 = 一个节点**；
- **局部状态 = 节点私有字段**；
- **递归 = push 新节点**（祖链继承上下文）；
- **函数调用 = exe（同节点）或 push（跨节点）**。

> **栈帧被“去中心化”为 TS 节点**，而不再是一个全局线性结构。

这不仅解决了栈的固有问题，还自然支持：
- **结构化并发**（子任务 = 子节点）；
- **安全状态隔离**（无意外共享）；
- **确定性生命周期**（无 GC，无 manual free）。

---

## 三、EDSOS Event：对“回调地狱”的结构性终结

### 4.1 传统回调的致命缺陷

在 C/Rust/JS 中，回调函数：
- 在触发方的栈上执行；
- 继承触发方的 capability 和内存上下文；
- 可能修改触发方状态，导致 **逻辑耦合 + 安全漏洞**（如 reentrancy 攻击）。

> **回调 = 在他人家中随意走动**。

### 4.2 EDSOS Event 的革命性设计

借助 EDSOS Event，TSL 实现 **完全隔离的异步协作**：

```tsl
// 触发方
exe (() ()) start_io
signal io_complete all   // 通知所有等待者

// 处理方
wait io_complete
exe (() ()) handle_result
```

- **触发方与处理方属于不同节点** → 不同 capability 域、不同 data 空间；
- **无直接调用** → 无栈帧嵌套；
- **同步由调度器协调** → 无竞争条件；
- **事件无实体** → 零内存开销，仅逻辑屏障。

**这实现了“逻辑关联，物理隔离”**：
- 安全：处理方无法访问触发方私有状态；
- 模块化：双方可独立开发、测试、部署；
- 高效：同核 signal/wait 开销 ≈ 几条指令。

> **这是对“最小权限原则”在并发场景下的完美实践**。

---

## 四、更高维审视：TSL 是“调度器感知语言”（Scheduler-Aware Language）

TSL 的终极创新在于：

> **它将操作系统的调度语义直接编码到语言结构中**。

- `push`/`pop` → 调度单元创建/销毁；
- `wait`/`signal` → 调度屏障；
- `lift` → 跨调度单元/进程共享；
- `capability` → 调度器权限检查依据。

这使得：
- **编译器可生成调度器友好的代码**；
- **调度器可基于结构语义优化执行**（如亲和性调度、批处理 wait）；
- **程序员无需理解底层调度细节，即可写出高效并发代码**。
