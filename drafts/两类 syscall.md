# 两类 syscall

---

## 一、第一类：**调度器本地处理型 syscall（Scheduler-Local Syscall）**

### 1.1 用户态指令表现

这类 syscall **不涉及特权指令切换**，而是直接调用调度器的 **常态开放接口（Normal Open Interface, NOI）**。例如：

```c
// 示例：等待事件
edsos_wait("io_complete_123");

// 示例：查询自身 TS 路径索引
uint64_t self_path = edsos_self_path();

// 示例：在当前 TS 中 push 一个新节点
edsos_push(stack_size, entry_point);

// 示例：将当前子树 lift 到父节点下
edsos_lift(target_parent_path);
```

这些调用在进程代码加载期间由 EDSOS Loader 内联展开为对调度器 per-core 数据结构的访问（如 Ready Forest、PCB 缓存）。

> **关键优势**：
> - **零特权切换**（no ring0/ring3 transition）；
> - **O(1) 时间复杂度**；
> - **无 IPC 开销**；
> - **天然原子性**（调度器单核串行执行）。

### 1.2 属于第一类 syscall 的操作

- **`wait(event)`**：将当前节点从 Ready Forest 移除，加入本地 Wait Graph（event_name → node）；
- **`push()`/`lift()`/`pop()`等 TS 结构操作**：执行后调度器 pause 受影响的节点使其重新被加载；
- **`yield()`**：主动让出 CPU，调度器直接切换；
- **`get_sched_info()`**：查询当前节点的调度策略、优先级、CPU 亲和性；
- **`set_timer(event, deadline)`**：设置本地定时器，到期后 signal(event)；
- **`get_nowtime()`**：根据请求节点的身份返回对应的模糊时钟值（防时序侧信道攻击）；
- **`query_capability(path)`**：检查是否能访问某 TS 路径（仅需遍历祖先链）；
- **`get_tlb_context()`**：获取当前 PCID 或 TLB 标签（用于调试或性能分析）。

> **共性**：这些操作 **仅依赖调度器本地状态 + 当前 TS 结构**，无需访问全局服务。

### 

---

## 二、第二类：**KPN 中转型 syscall（Kernel Proxy Node Mediated Syscall, KPN-Mediated Syscall）**

### 2.1 用户态指令表现

用户代码调用标准 syscall 接口（如 `read(fd, buf, len)`），但在进程代码加载期 EDSOS Loader 将指令内联展开而变量地址被重定向到 **所属 TS 的 KPN（Kernel Proxy Node）**：

```c
// 用户代码
ssize_t n = read(fd, buf, 4096);

// EDSRT 内部实现大致流程（伪代码）
{
    // 1. 将参数写入 KPN 的共享缓冲区（数据段）
    kpn->args.fd = fd;
    memcpy(kpn->args.buf_shadow, buf, len);

    // 2. 设置 syscall 类型
    kpn->syscall_id = SYS_READ;

    // 3. 触发“请求”：通过 CTRN 发送消息到内核服务节点
    edsos_ct_call(kpn->service_ref, kpn->req_event);

    // 4. 等待响应
    edsos_wait(kpn->resp_event);

    // 5. 从 KPN 缓冲区读取结果
    memcpy(buf, kpn->result.buf, kpn->result.size);
    return kpn->result.ret;
}
```

> 注意：`edsos_ct_call` 是轻量级跨 TS 调用原语，**不切换特权级**，仅构造消息并触发调度器阻塞。

### 2.2 KPN 的角色与结构

KPN 是一个 **CTRN（Cross TS Referring Node）**，同时属于：
- **用户 TS**：作为其根节点的直接子节点（如 `/chrome_instance_01/kpn`）；
- **内核 TS**：作为某内核服务（如 `vfs`）的客户端代理。

### 2.3 处理流程（跨 TS 协作）

1. **用户节点** 调用 syscall 写入 KPN 缓冲区；
2. **调度器** 执行 `edsos_ct_call`：
   - 构造消息：`{target=GVA_of_vfs_node, event=req_event, payload=KPN_GVA}`;
   - 通过 **调度器间链式消息总线** 发送给目标 PM（若跨核）；
   - 将当前节点状态设为 **BLOCK**，加入 `resp_event` 的 Wait Graph；
3. **目标系统服务所处的调度器**（如 VFS 服务所在核）：
   - 收到消息，将 `vfs_node` 加入其 Ready Forest；
   - 调度执行 `vfs_node`，其代码读取 KPN 缓冲区；
   - 执行文件 I/O（可能进一步调用磁盘驱动 TS 节点）；
   - 完成后写入结果到 KPN，`signal(resp_event)`；
4. **用户调度器** 收到 `resp_event` 信号：
   - 唤醒原节点；
   - 用户代码从 KPN 读取结果，返回。

> **整个过程无传统 IPC 拷贝**，仅通过 **内存共享 + Event 同步** 实现；
> **中间过程对用户进程透明**，内部的实质**异步委托**对用户代码是完全无感的，不会在此处因为“陷入内核”而阻碍排队等待当前 CPU core 执行的其他节点、更不会因为处于同一进程空间而被恶意代码攻击内核资源。

### 2.4 属于第二类 syscall 的操作

- **进程管理**：具有 TS 根节点的 TS 创建；
- **文件I/O**：`open`, `read`, `write`, `stat`（由 VFS 服务节点处理）；
- **网络I/O**：`socket`, `send`, `recv`（由 netd 服务节点处理）；
- **权限申请**：`request_capability(path)`（需内核 capability 服务审批）；
- **全局配置**：`set_system_time`, `mount`（需 root 权限）。

### 2.5 安全与权限模型

- **KPN 是 capability 的载体**：
  - 用户只能访问其 KPN 中暴露的 syscall；
  - KPN 本身由内核在创建 TS 时按策略注入；
- **内核服务节点执行深度验证**：
  - 检查 KPN 是否合法；
  - 检查调用源是否有权访问该资源；
  - 检查参数是否越界。

---

## 三、与传统 syscall 模型的本质区别

| 维度 | 传统 OS（Linux） | EDSOS |
|------|------------------|-------|
| **特权切换** | 在当前栈上生长出特权帧（syscall → ring0） | 仅第二类通过异步委托请求，当前节点内无特权切换 |
| **接口位置** | 全局 syscall 表 | 每个 TS 有独立 KPN，接口动态定制 |
| **阻塞机制** | 内核 sleep_on 等待队列 | 调度器 Wait Graph + Event |
| **参数传递** | 寄存器/栈 → 内核栈拷贝 | KPN 作为 CTRN 无拷贝共享 |
| **权限模型** | UID + DAC | TS 路径 + KPN capability |
| **扩展性** | 需修改内核 | 新增系统 TS 服务节点即可，呈现微内核特征 |

---

## 四、syscall 体现出 EDSOS Scheduler 的本质及其内部模块化

### 4.1 它不是一个“实体”，而是一种“运行时契约”

- **无全局调度器进程**，无调度器线程；
- **每个 CPU Core** 拥有独立的：
  - `Ready Linked List`（就绪节点链表）
  - `Wait Graph`（事件 → 节点映射）
- **用户进程二进制中**，在链接阶段由 EDSOS Linker **静态展开** 调度器 ABI stub：
  - 如 `__edsos_wait`, `__edsos_push` 等；
  - 这些 stub 是 **只读、不可修改、不可绕过** 的指令序列；
  - 它们直接操作当前 Core 的调度器数据结构（通过 TLS 或固定寄存器约定）。
- **EDSOS Scheduler** 的本质是与调度相关的 per core 独立的 **数据结构** 和访问这些数据结构的内联展开到用户进程指令流中的 **指令段** 这两者的 **统称**。
- 在代码组织上，EDSOS Scheduler 对应于一个专门的文件，其中列举了与此相关的数据结构声明和 ABI → 内联指令段的映射。

> 这类似于 **unikernel 中的“内联系统调用”**，但更进一步：
> 它不是“应用包含内核”，而是 **“每个进程包含其调度器视图”**。

### 4.2 ABI 的版本化与兼容模式绑定

- 在进程加载时，根据其声明的 **兼容模式**（POSIX / Win32 / EDSOS-native）：
  - 选择对应的 syscall stub 集合；
  - 静态绑定到二进制中；
  - 例如：`edsos_wait` 在 POSIX 模式下可能映射为 `pthread_cond_wait` 语义，在 Win32 模式下映射为 `WaitForSingleObject`。
- **运行时不可切换模式** → ABI 稳定 → 无需动态分派 → 零运行时开销。

> 安全性：用户代码无法替换或 hook 这些 stub，因为它们是 **只读段的一部分**，且 GVA 映射由 TLB 验证。

### 4.3 内部模块化：最小化权责，明确边界

- 核心内环（必须由 Scheduler 直接处理）：

| 功能 | 理由 |
|------|------|
| **节点状态管理**（ready/block/run） | 调度决策的基础 |
| **Ready Forest 维护** | 调度器的“就绪队列” |
| **Wait Graph 管理** | 同步原语的实现基础 |
| **TLB Context 预热/切换** | 安全访问控制的关键 |
| **本地事件 signal/wait** | O(1) 同步的保障 |
| **TS 节点结构操作**（push/pop/lift） | 调度器需知晓拓扑以预热 TLB |

> 这些操作 **无法解耦**，若交由其他服务，将破坏确定性、增加延迟。

- 可疏散外环（不应由 Scheduler 处理）：

| 功能 | 应归属模块 |
|------|----------|
| 文件 I/O 路径解析 | VFS 服务节点 |
| 网络协议栈处理 | Netd 服务节点 |
| Capability 审批 | Auth 服务节点 |
| 全局资源分配 | Registry 服务节点 |

> **Scheduler 不应包含**：
> - 任何业务逻辑（如“什么是合法的文件路径”）；
> - 任何全局状态（如“下一个可用 PID”）。

- **Scheduler 的唯一职责**：
  - **“在正确的时刻，让正确的 TS 节点，在正确的 Core 上，以正确的 TLB 上下文运行。”**