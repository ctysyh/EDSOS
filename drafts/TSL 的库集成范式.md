# Arxil 的库集成范式

## 一、模型定义：三位一体的库抽象

### 1.1 核心组件

| 组件 | 角色 | Arxil 表达 |
|------|------|--------|
| **库类型（Library Type）** | 封装运行时状态（上下文、缓存、资源句柄等） | `.arxtype MyLibCtx { ... }` |
| **库函数集合（`lib`）** | 提供改变结构/触发事件/复杂逻辑的操作 | `lib mylib { fn init(); fn render(); ... }` |
| **库托管节点（Lib Managed Type Node, LMTN）** | 实例化库类型 + 挂载库函数 + 管理生命周期 | `lmtn { data { (MyLibCtx) ctx; } code { instruct { cycl(ctx.exit); fnsh; } } }` |

### 1.2 调用方视角

当我们的代码的某个节点需要调用lib，我们就在代码里 `psh` 并 `lft` 一个 LMTN，它其中的字段正是库类型的，其中的代码正是前面提到的 `lib`；而它的主逻辑 `instruct` 里边是一个 `cycl` 等待在自己专门的“退出”字段，这个字段在我们的代码认为这个节点不再需要的时候就可以通过 `data`->`ance` 解析赋值为 `true` ，这样它就执行下一条Arxil指令即 `fnsh`。
我们区分两种操作：一种是改变现有库字段的值、或者是从中获取值存到我们自己节点的字段，这是应该集成在库类型的运算符里的；另一种是改变现有库字段的结构、或者需要扩容/减容，这是应该包装在 lib 的 `fn` 里的。
这样，对于运算符，它不存在节点的切换，而且有很好的形参和返回槽基础，执行效率比传统的库调用高；对于 lib 的 `fn`，它在当前节点的上下文和权限里执行，但是内部自定义地解析自有库类型、或者自定义地触发自有 Semaphore：从而解决了性能问题，并且既保持 lib 的封装性、又维护所有节点的边界。

---

## 二、为什么这个模型是“良性”的？

### 2.1 严格维护节点边界

- **状态隔离**：库状态完全位于 `lib_mt.ctx` 字段中，调用方无法直接修改（除非显式 `ance` 绑定，且仅限 `publ` 字段）；
- **执行隔离**：`lib` 中的 `fn` 在 **LMTN 节点的上下文** 中执行，其指令只能访问：
  - 自身 `data` 字段（即 `ctx`）；
  - 通过 `ance` 声明的祖先字段（如调用方传入的 `scene`）；
- **无越权访问**：分离逻辑依然成立——LMTN 拥有 `ctx`，调用方拥有 `scene`/`output`，共享通过绑定显式声明。

---

### 2.2 性能分层清晰

| 操作类型 | 实现方式 | 开销 | 适用场景 |
|--------|--------|-----|--------|
| **查询/简单变换** | 类型运算符（如 `img.blur()`） | 零节点切换，寄存器优化 | 高频、无状态 |
| **复杂逻辑/状态变更** | lib `fn`（在 LMTN 中执行） | 一次节点切换（EDSOS 可优化） | 低频、有状态、需扩容 |
| **生命周期管理** | `ctx.exit = true` + `cycl` | 无额外开销 | 按需释放 |

- **运算符优势**：利用 Ordinary Opcodes 的寄存器豁免，可生成 SIMD/向量化代码；
- **lib `fn` 优势**：可在内部自由使用 `Push`/`Merge` 扩容自己的 `ctx` 子结构（如加载新纹理时 `Push` 一个 `TextureNode` 到 `ctx.textures` 下）。

---

### 2.3 动态性与静态性的统一

- **静态部分**：
  - 库类型大小固定（`.arxtype` 要求）；
  - 运算符签名固定；
  - lib `fn` 的 Param/RetList 固定。
- **动态部分**：
  - 库类型内部可通过 **子节点树** 实现无限扩展（如 `ctx.textures` 可 `Push` 任意多纹理）；
  - lib `fn` 可在运行时创建/销毁子节点，实现真正的动态资源管理；
  - 生命周期由调用方控制（`end` 字段），实现 RAII 式资源释放。

---

## 三、与传统 DLL/动态库的映射

**Arxil 的 `lib` 可直接对应传统动态库**，只需附加元信息：

| 传统 DLL | Arxil 对应 |
|--------|--------|
| 导出函数（`__declspec(dllexport)`） | `lib { fn ... }` |
| 全局状态（如 TLS、static var） | 封装进 `.arxtype`，由 LMTN 实例化 |
| 初始化/清理函数 | `fn init()` / `fn deinit()`，由 LMTN 的 `instruct` 调用 |
| ABI 兼容性 | 通过 `.arxtype`（Arxil Type）文件描述类型布局 |

> **这使 Arxil 成为传统生态的“结构化封装层”**，而非替代品。

---

## 四、潜在挑战与应对

| 挑战 | 解决方案 |
|------|--------|
| **跨平台类型布局** | `.arxtype` 文件包含目标平台的字段偏移、对齐信息 |
| **异常/错误处理** | 返回槽中包含 `err: ErrorCode` |
| **多线程访问** | LMTN 默认单线程（Arxil 节点天然串行）；若需并发，创建多个 LMTN 实例 |
| **调试符号** | `.arxtype` + DWARF 映射，支持源码级调试 |

---

## 五、示例片段

```arxil
node libcall_example_main {
    meta_data {}
    data {
        imme {
            // some field
        }
        futu {
            ance {
                MyLibCtx ctx;
                std.bool sdl_end;
            }
        }
    }
    code {
        instruct {
            psh sdl_mtn (libcall_example_sdl () () ());
            lft sdl_mtn ((ctx=>ctx) (end=>sdl_end));
            // do something, like:
            sdl_example_op (ctx::a) (ctx::a, ctx::b, ctx::c)
            // when logical flow is end
            set (sdl_end) (true);
            fnsh;
        }
        ance {
            lib sdl;
        }
    }
}


node libcall_example_sdl {
    meta_data {
        @lmtn lib sdl;
    }
    data {
        imme {
            publ {
                MyLibCtx ctx;
                std.bool end;
            }
            priv {
                // some internal variables
            }
        }
        futu {
            priv {
                // some internal variables
            }
        }
    }
    code {
        instruct {
            exec ((ctx) (end)) init_sdl;
            cycl (end) (exec ((ctx) ()) on_notify);
            fnsh;
        }
        publ {
            // some lib func
        }
        priv {
            fn init_sdl ((MyLibCtx)ctx) => ((std.bool)end) {
                'inst_scri' {
                    // some actions to init
                }
            }
            fn on_notify ((MyLibCtx)ctx) => () {
                'inst_scri' {
                    wait (sdl_call_event) ();
                    // some actions to deal
                    // here can also push some son nodes, dealing tasks in parallel
                    sgnl (sdl_ready_event) ();
                }
            }
        }
    }
}

```