# DDR-DPU 硬件架构设计概述

## 一、核心目标与定位

DDR-DPU 是一种**面向极致性能、内存原生、Linux 兼容的分布式内存访问架构**，其核心目标是：

- **实现百纳秒级远程内存访问延迟**（< 300 ns 端到端）；
- **兼容 RDMA RC Verbs API**，但底层机制完全重构；
- **将内存本身作为网络端点**（Memory-as-Endpoint），而非附加设备；
- **服务于 EDSOS 的高性能需求，同时为 Linux 提供标准驱动接口**；
- **避免 SmartNIC 的固化逻辑与小内存瓶颈**，通过双 PCB 架构实现灵活性与扩展性。

> 本质：**不是“带内存的 NIC”，而是“可远程访问的内存系统”**。

---

## 二、硬件拓扑：Main PCB + Sub PCB 双 PCB 架构

### 2.1 基本组成
- **Main PCB**：含 DPU ASIC，负责网络协议处理、远程访问执行、校验、调度；
- **Sub PCB**：标准 DDR5 DIMM 形态，含 DRAM + RCD + CAIU（CA Injection Unit），提供大容量共享内存；
- **连接方式**：Main PCB 与 Sub PCB 通过**私有高速电气链路**（如私有金手指）互联，延迟 < 50 ns。

### 2.2 关键创新
- **内存与逻辑分离**：DRAM 在 Sub PCB，控制逻辑在 Main PCB；
- **数据路径极简**：远程 WRITE → Main PCB → **Sub PCB = DRAM**；
- **控制路径内存化**：QP/CQ/MTT 等结构直接驻留 Sub PCB DRAM，CPU 通过标准内存访问操作。

---

## 三、控制面设计：内存原生控制结构

### 3.1 控制结构布局（位于 Sub PCB DRAM）
| 结构 | 用途 | 访问方式 |
|------|------|--------|
| **QP / CQ** | 存放 Work Request (WR) 与 Work Completion (WC) | CPU 写 WR，DPU 写 WC |
| **MTT**(Memory Translation Table) | 存储 MR 注册信息（含物理偏移、权限、校验） | DPU 查表执行远程访问 |
| **SPSC 控制通道缓存** | 特殊 DB 对应的 FIFO，用于指令下发与完成通知 | 双向内存 FIFO |
| **CAIU 寄存器镜像区**(MMCR) | CPU 配置 MTT 基址、中断使能等，支持三态门控制、仅在授权窗口注入 CA | 类似 DDR5 模式寄存器 |

### 3.2 控制通道优化
- **特殊 DB**（如 DB3）被配置为 **带物理缓存的 FIFO**，非纯直通；
- CPU 写入即入 FIFO，DPU Main PCB 通过私有链路读取；
- 支持 **DPU → CPU 中断**（通过轻量 PCIe MSI 或专用中断线）。

---

## 四、远程访问协议：三大操作正交解耦

远程内存访问被分解为三个正交阶段：

### 4.1 标识可访问性（Permission Binding）
- 由 EDSOS 或 Linux 驱动在本地 Sub PCB 注册内存区域；
- 生成唯一 **access_key**，并写入 **权限标识结构**；
- Main PCB 通过 **Access Key Table **(AKT) 快速查表（CAM/哈希）。

### 4.2 校验（Legitimacy & Integrity）
- **钥匙校验**：验证 `access_key` 是否有效（合法性）；
- **数据校验**：CRC32/64 覆盖包头 + payload（完整性）；
- **权限校验**：由 AKT 条目中的 flags 控制（READ/WRITE）。

> **关键区分**：  
> - **传输层校验**（钥匙 + CRC）由 Main PCB 硬件完成；  
> - **内存访问权限**由 EDSOS 权限结构管理，通过 AKT 投影到硬件。

### 4.3 数据落存 / 飞行（Execution）
- **WRITE**：Main PCB 直接写入目标 Sub PCB DRAM（经 CAIU）；
- **READ**：Main PCB 从 Sub PCB DRAM 直读，封装响应包飞回；
- **零拷贝、无中间 buffer、无 CPU 干预**。

---

## 五、缓存一致性保障：Linux 驱动下的正确性

### 5.1 核心挑战
- DPU 写入绕过 CPU 缓存，但本地 CPU 可能缓存同一物理页 → **数据不一致**。

### 5.2 解决方案
- **强制共享内存区域为 Write-Combining **(WC)：
  - 驱动在 `mmap()` 时设置 `pgprot_writecombine()`；
  - CPU 访问直通内存，不缓存；
  - DPU 写入后立即对 CPU 可见。
- **不依赖硬件 snoop**（因 DPU 非 CPU agent）；
- **EDSOS 可使用更激进策略**，Linux 使用 WC 保证兼容性。

> 这是正确性基石：**所有远程可访问内存必须为 WC 类型**。

---

## 六、多 Sub PCB 扩展：单出口、大容量、高带宽

### 6.1 架构模式
- **多个 Sub PCB**（标准 DDR5 DIMM）连接到**同一 Main PCB**；
- **单网络出口**，避免多 DPU 出口带来的网络复杂性爆炸。

### 6.2 地址空间统一
- 每个 Sub PCB 分配**不重叠的全局物理地址段**；
- Main PCB 维护 `offset → sub_id + local_offset` 映射；
- OS 通过 `dev_dax` / `ZONE_DEVICE` 将多 Sub PCB 视为连续内存。

### 6.3 隔离性设计
- **电气隔离**：独立 CA/CK/DQ 通道；
- **错误域隔离**：独立 ECC、错误 FIFO、状态寄存器；
- **访问引擎隔离**：Main PCB 为每个 Sub PCB 实例化独立 CAIU 引擎。

---

## 七、带宽争用与 QoS

- **控制面**（特殊 DB）：高优先级，低延迟保障；
- **数据面**（所有普通 DB）：公平争用（Round Robin / WFQ）；
- **仲裁器位于 Main PCB**，支持抢占但限流，防饿死。

> **单出口 + 分层 QoS = 网络简洁性与性能兼顾**。

---

## 八、容错与高可用

### 8.1 控制面故障迁移
- **所有 Sub PCB 均具备“特殊 DB”硬件能力**（FIFO + 中断）；
- 默认仅 slot 0 启用；
- 若 slot 0 故障，驱动可通过 MMCR **动态启用 slot 1**，迁移 QP/CQ/MTT。

### 8.2 硬件设计哲学
- **特殊 DB 是“具有特殊能力的 DB”，而非“专用 DB”**；
- 能力通过寄存器配置，非固化；
- **硬件对称性 → 软件可演进性**。

---

## 九、与 Linux 驱动协同

### 9.1 驱动职责
- 初始化时 mmap Sub PCB 控制区域（QP/CQ/MTT/MMCR）；
- 实现 `ib_device` 接口：`reg_mr`, `post_send`, `poll_cq` 等；
- 强制共享内存为 WC 类型；
- 处理中断与故障迁移。

### 9.2 用户态友好
- 应用可通过 `libibverbs` 无缝使用；
- QP/CQ 可用户态 mmap，实现 **kernel-bypass**；
- 控制面操作均为内存读写，无 syscall 开销。

---

## 十、与 SmartNIC 的本质区别

| 维度 | SmartNIC | DDR-DPU |
|------|--------|--------|
| **内存位置** | 小容量片上 SRAM/DRAM | 大容量标准 DDR5（TB 级） |
| **控制逻辑** | 固化 ASIC，微码有限 | Main PCB 可编程 + Sub PCB 软件可更新 |
| **数据路径** | Host Mem ↔ PCIe ↔ NIC ↔ Network | DPU DDR as Host Mem ↔ DPU ASIC ↔ Network |
| **生命周期** | 与 ASIC 绑定 | Sub PCB 可独立升级，Main PCB 微码可演进 |
| **扩展性** | 单设备 | 多 Sub PCB 聚合，容量/带宽线性扩展 |

> DDR-DPU 不是“更快的 NIC”，而是“**可远程访问的内存系统**”，
> 通过 **双 PCB 分离、控制面内存化、权限与传输解耦、多 Sub PCB 聚合**，实现了：

> - **百纳秒级延迟**（性能跨代）；
> - **TB 级共享内存**（容量跨代）；
> - **Linux 兼容 + EDSOS 优化**（生态兼容）；
> - **硬件对称 + 软件可演进**（生命周期跨代）。

---

## 补充

- 前面提到使用轻量 PCIe MSI，但这一点需要仔细分析。因为 MSI-X 本身走 PCIe 通道，传过来的延迟都至少是 1 μs，这个部分是否能够抵得上 CPU 自己轮询 DDR，还真的不好说。