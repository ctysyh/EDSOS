# 函数式和部分高级语言的一些关键概念到 TSL 的语义保持编译方案与形式化

> 主要考虑了 Rust 所有权和闭包

---

## 一、Rust 所有权模式的 TSL 编码

### 1.1 Rust 所有权的核心语义

Rust 的所有权规则可形式化为：
- 每个值有**唯一所有者**；
- 所有权可**移动（move）** 或**克隆（clone）**；
- **借用（borrow）** 分为不可变（`&T`）和可变（`&mut T`），受生命周期约束；
- 编译期静态检查，无运行时开销。

### 1.2 TSL 的结构化内存 vs Rust 栈/堆

| 概念 | Rust | TSL |
|------|------|-----|
| 值存储 | 栈（局部） / 堆（Box） | 节点 `priv` / `publ` 字段 |
| 所有权转移 | move 语义（源失效） | `Divide` + `Merge`（字段物理迁移） |
| 共享只读 | `&T` | 绑定为 `ance` 字段并额外声明访问权限 |
| 独占可写 | `&mut T` | 从共享数据节点中临时 `Divide` 出字段，独占操作后 `Merge` 回 |

### 1.3 所有权操作的 TSL 映射

#### （1）Move 语义：字段的物理迁移

Rust:
```rust
let x = vec![1,2,3];
let y = x; // x moved, no longer usable
```

TSL 编码：
```tsl
// 当前节点 n 拥有 priv.vec_x
divide temp_owner (VecType () (vec_x => vec_y));
// 此刻 n.priv 不再含 vec_x；temp_owner.priv.vec_y = 原值
merge target_node ((vec_y => vec_z)); // 转移给其他节点
```

- `Divide` 将字段从原节点移出 → 模拟“源失效”；
- `Merge` 将字段注入目标节点 → 模拟“接收所有权”；
- **无复制**，符合 move 语义。

#### （2）Clone 语义：显式复制

Rust:
```rust
let y = x.clone();
```

TSL:
```tsl
cpy vec_y vec_x;
```

- 依赖类型环境 $\Theta(\text{Vec}).\text{ops}$ 包含 `cpy` 操作；
- TSL 没有所谓“深拷贝”“浅拷贝”区分，它像 C 一样，对于浅拷贝的模拟可以将数据放置在共享祖先中、并在具体编码中通过一个额外的计数字段来实现。

#### （3）不可变借用：`ance` 绑定

Rust:
```rust
let r = &x;
```

TSL:
```tsl
// x 在当前节点 publ 中
push borrower (Borrower () (x => ref_x) ()); // ref_x 是 ance 字段
// borrower 内可通过 ref_x 只读访问 x
```

- 由于 `x` 是 `publ`，绑定安全；
- 多个 borrower 可同时绑定 → 共享只读；
- 原所有者仍可读写 `x`（Rust 不允许，但 TSL 不强制），或者通过增加读写权限来限制。

#### （4）可变借用：临时独占

Rust:
```rust
let r = &mut x;
// x 不可用，直到 r 超出作用域
```

TSL 模拟完全同对 Move 的模拟，只是增加一次“归还”（即用完之后再 Move 回去）。

---

## 二、闭包（Closure）的 TSL 编码挑战与策略

### 2.1 闭包的本质

闭包 = **代码指针 + 捕获的环境（captured variables）**

在 λ-演算中：
$$
(\lambda x. e)\ \rho \quad \text{其中 } \rho \text{ 是环境映射}
$$

问题：TSL 的 `fn` **没有环境**，只有固定标签。

### 2.2 核心障碍

- TSL 的 `exe L` 跳转仅改变 `pc`，不改变数据上下文，即使 exe 的是祖先节点中的指令块也仍然使用本节点自己的数据上下文；
- 指令块 `L` 中的字段引用必须在**编译期确定作用域**；
- 若 `L` 引用祖先节点的字段，这些字段必须通过 `ance` 显式传入。

因此，**TSL 无法直接表示高阶函数**，但可**通过结构化环境模拟闭包应用**。

### 2.3 闭包的结构化编码策略

我们采用 **“闭包 = 数据节点 + 代码标签”** 的经典方案（类似 closure conversion）。
实际上将**函数式闭包降级为面向对象风格的对象**（数据+方法），这是结构化编程对高阶抽象的自然编码方式。函数式特性被“编译掉”，转化为结构演化。
本质是先映射为过程式编码，再进一步映射为 TSL。

