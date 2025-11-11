# TSL 的 `fn` 内部 High Level Language 的使用

---

## 一、设计哲学重述：TSL 的“双层抽象”模型

TSL 的核心设计可概括为：

> **“语法自由，语义封闭。”**

- **上层**（Syntax Layer）：允许在 `'lang'` 块中使用任意高级语言（C/C++/Rust 等），包括类型定义、函数定义、标准库调用。
- **下层**（Semantic Layer）：所有代码*最终*必须编译为 **纯 TSL-instruct 序列**，且该序列：
  - 仅引用 `fn` 接口声明的符号（参数 + 返回值）；
  - 不引入任何未声明的局部状态；
  - 可被静态验证满足 TS Node 封闭性。

> 关键桥梁：frontend 编译器负责将上层自由语法“坍缩”为下层封闭语义。

---

## 二、绑定机制详解：从高级变量到字段符号的映射

### 2.1 `fn` 接口：逻辑绑定槽（Logical Binding Slots）

一个 `fn` 声明：

```tsl
fn F ((τ₁)x₁, …, (τₘ)xₘ) ⇒ ((σ₁)y₁, …, (σₙ)yₙ)
```

定义了一个 **符号环境**：

$$
\Gamma_F = \{ x_1 : τ_1, \dots, x_m : τ_m, y_1 : σ_1, \dots, y_n : σ_n \}
$$

这些符号 **不是内存地址**，而是 **命名占位符**（named placeholders）。

### 2.2 高级语言块的 lowering 过程

考虑 `'c' { int t = x1 + x2; y1 = t * 2; }`。

#### 步骤 1：前端解析为 AST + SSA

```llvm
%t = add i32 %x1, %x2
%y1 = mul i32 %t, 2
```

#### 步骤 2：临时变量分配策略

临时使用寄存器或复用 $\Gamma_F$ 范围内的字段（如 `__tmp0`）。

#### 步骤 3：生成 TSL-instruct

```tsl
add (__tmp0) ((x1) (x2));
mul (y1) ((__tmp0) (0x2));
```

### 2.3 递归绑定：嵌套函数调用的结构化模拟

考虑：

```c
int helper(int a) { return a + 1; }
int foo() { int x1 = 1; return helper(x1); }
```

#### Lowering 策略 A：内联

```tsl
set (x1) (0x1)
add (y1) (x1, 0x1);
```

#### Lowering 策略 B：显式 `exe` 调用

前提：`helper` 已作为顶层 `fn` 或在当前 `'lang'` 块内存在：

```tsl
fn example_fn (...) => (...) {
    @lowered 'c' {
        fn foo ((int)x1) => ((int)y1);
        fn helper ((i32)a) => ((i32)ret) {
            add (ret) ((a) (0x1));
        }
    }
}
```

则 `foo` 编译为：

```tsl
set (x1) (0x1)
exe ((x1) (y1)) helper;
```

> **绑定传递**：
> - `foo.x1` → 绑定到调用方字段 `A`
> - `helper.a` → 通过 `exe` 绑定到 `foo.x1`（即 `A`）
> - `helper.ret` → 绑定到 `foo.y1`（即调用方字段 `B`）

这形成了 **符号绑定链**：

$$
\texttt{helper.a} \xrightarrow{\text{bind}} \texttt{foo.x1} \xrightarrow{\text{bind}} A
$$

---

## 三、初步形式化：从高级表达式到 instruct 的语义映射

我们定义一个 **lowering 函数**：

$$
\llbracket \cdot \rrbracket : \text{HighLevelExpr} \to \text{TSL-Instruct}^*
$$

其定义依赖于当前 `fn` 的符号环境 $\Gamma$。

### 3.1 符号环境与字段谓词

节点相关谓词：

$$
\texttt{node}(n, \mathcal{D}, \mathcal{I}) \quad \text{其中} \quad \mathcal{D} : \text{FieldName} \rightharpoonup \text{Value}
$$

而 `fn` 执行时，通过 `exe` 建立 **符号到字段名的映射**：

$$
\theta : \text{Symbol} \to \text{FieldName}
$$

例如：$\theta(\texttt{x1}) = \texttt{input\_field\_A}$

### 3.2 Instruct 的语义解释

一条 instruct 如 `add z (x, y)` 的语义为：

$$
\frac{
  v_x = \mathcal{D}(\theta(x)), \quad
  v_y = \mathcal{D}(\theta(y))
}{
  \mathcal{D}' = \mathcal{D}[\theta(z) \mapsto v_x + v_y]
}
$$

### 3.3 HighLevelExpr 的 lowering 规则（示例）

#### 赋值语句

$$
\frac{
  \llbracket e \rrbracket_{\Gamma} = (v, I_e) \quad \text{// $I_e$: compute $e$ into temp $v$} \\
  v \in \Gamma \quad \text{// temporary must be in interface}
}{
  \llbracket \texttt{var = e;} \rrbracket_{\Gamma} = I_e \mathbin{+\!+} [\texttt{cpy } \texttt{var} \texttt{ } v]
}
$$

#### 函数调用（内联）

<TODO>

#### 函数调用（外联）

<TODO>

---

## 四、同一 `fn` 中多个 `'lang'` 块的绑定一致性

考虑：
```tsl
fn F ((i32)a) ⇒ ((i32)r) {
  'c' { int t = a + 1; }
  'rust' { r = t + 1; }
}
```

关键问题：`t` 是否跨块可见？

**不直接可见**，除非：
- `t` 必须被 lowering 到 $\Gamma_F$ 中的一个符号（如 `__t`）；
- 所有 `'lang'` 块共享同一个 $\Gamma_F$。

---

## 五、总结：绑定机制的形式化骨架

| 概念 | 形式化表示 | 说明 |
|------|----------|------|
| `fn` 接口 | $\Gamma = \text{Params} \cup \text{Returns} \cup \text{Temps}$ | 逻辑符号集合 |
| 物理绑定 | $\theta : \Gamma \to \text{FieldName}$ | 由 `exe` 提供 |
| Instruct 语义 | $\langle \texttt{inst}, \mathcal{D}, \theta \rangle \Downarrow \mathcal{D}'$ | 字段更新 |
| Lowering | $\llbracket \text{HL} \rrbracket_\Gamma : \text{Seq(Instruct)}$ | 编译期转换 |

> **最终产物**：一个纯 instruct 序列，其所有符号引用均在 $\Gamma$ 中，且可通过 $\theta$ 映射到物理字段。

---