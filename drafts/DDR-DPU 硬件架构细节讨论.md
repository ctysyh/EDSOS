# DDR-DPU 硬件架构设计细节讨论

> **在 JEDEC DDR5 物理边界内、不改动通用 CPU/主板、支持 CPU 与 DPU 共享同一物理内存、且能实现百纳秒级远程访问**的可行硬件架构。

---

## 一、CA 路径：双 CA Buffer + CAIU

### 采用方案：
- **RCD 内部集成两个 CA Buffer**：
  - **CA Buffer A**：接收 CPU CA（标准路径）；
  - **CA Buffer B**：接收 Main PCB CA（经 CAIU 检查后）。
- **CAIU（Command/Address Injection Unit）**：
  - 位于 RCD 外部或集成于 RCD；
  - 功能：验证 `access_key`、检查授权窗口、在 CPU CK 上升沿采样 Main PCB CA；
  - 输出：仅在合法窗口内将 Main PCB CA 传递给 CA Buffer B；
  - **作为使能门控**，避免总线冲突。

### 关键特性：
- **CK 同源**：Main PCB 通过 Sub PCB 回传的 CK（或训练锁定）重建本地时钟；
- **命令正交**：CPU 与 DPU 可并发发送 CA 到不同 BG；
- **JEDEC 合规**：RCD 对外表现为单一 CA 输入，内部扩展不暴露。

---

## 二、DQ 路径：独立 I/O 旁路

### 采用方案：
- **DRAM 芯片内部增加第二套 I/O 路径**：
  - **LIO'**：独立于主 LIO，在每个 Bank 的 Column MUX 和 Local I/O Lines 处设置双路分配器，分别连接到主 LIO 和旁路 LIO'；
  - **GIO' Mux**：直连 LIO'，和主 GIO Mux 完全分离；
  - **DQ' Buffer + DQS' I/O**：专用 DQ/DQS 驱动/接收器；
  - **物理连接**：DQ' 引脚通过 Sub PCB 走线直连 Main PCB（不引出到 DIMM 金手指）。
- **共享资源**：
  - **Cell Array、Sense Amplifier**：完全共享（保证数据一致性）。

### 关键优势：
- **DQ 总线无冲突**：CPU 与 DPU 各自拥有独立 DQ 路径；
- **支持并发读写**：通过分离的 I/O Bus 压缩 Banks 实际所需的 tCCD_L 和 tCCD_S，利用它们和标称的差值作为 DPU 访问窗口，和 CPU 并发访问不会产生冲突。

---

## 三、DRAM 微架构优化：聚焦 LIO 与调度

### 采用方案：

- **CA 预解码支持**：
   - Main PCB 提前发送 CA 命令，进入 RCD 内部预解码 FIFO；
   - 在窗口到来时直接触发列选择；
   - 隐藏 CA 解码延迟。

---

## 四、时序调度：利用 tCCD 冗余“偷”周期

### 调度策略：
- 基础假设：JEDEC tCCD_L = 8 tCK，但实际物理延迟 = 4 tCK；
- **DPU 访问窗口**：DPU 可在 4–8 tCK 窗口内访问 BG；
- **动态窗口协商**：通过 ACIU 来仲裁、批准 DPU 的访问请求，并划定窗口期。

### 性能预期（DDR5-4800, tCK=312.5 ps）：
- 每 8 tCK（2.5 ns）可插入 1 次 32B DQ + 8B ECC DPU 访问；
- 理论 DPU 带宽 ≈ 32B / 2.5ns ≈ **12.8 GB/s per DIMM**，远超 RDMA 需求。

---

## 五、控制与软件协同

### 驱动/OS 职责：
- **访问协调**：
   - 当预知 DPU 需要访问某段内存时，等待对应的缓存写回，避免数据不一致；
   - 同时避免和 DPU 并发访问同一段内存，以减少对 DPU 访存的阻滞。

### Main PCB 职责：
- 支持 CK 相位训练（Write Leveling 类机制）；
- 处理私有链路协议（CA + DQ' + 控制命令）。

---

## 六、整体 Sub PCB 拓扑图

```
[CPU]
  │
  ├── CA → [RCD: CA Buffer A]
  └── DQ/DQS → [DRAM: GIO Mux → DQ Buffer] ←→ BG

[Main PCB]
  │
  ├── CA → [CAIU] → [RCD: CA Buffer B]
  └── DQ'/DQS' ←→ [DRAM: GIO' Mux → DQ' Buffer] ←→ BG

[DRAM Chip Internal]
  │
  ├── Cell Array (shared)
  ├── Sense Amplifier (per bank, shared)
  ├── (LIO,LIO') ← MUX-IOG ← Sense Amplifier

[Sub PCB Board]
  │
  ├── Private Link ↔ Main PCB
  ├── CK Ref Out → Main PCB (for phase lock)
  └── All DQ' signals routed internally (not to DIMM edge)
```

MUX-IOG inside
```
[SA]
 │
 ├──[NMOS_A]── LIO_A
 │     │
 │   Gate = CSL_driven_by_PortA
 │
 └──[NMOS_B]── LIO_B
       │
     Gate = CSL_driven_by_PortB

CSL_driven_by_PortX = Col_Decoder_Output ∧ PORT_EN_X
```

> 最终，Sub PCB Board 呈现双边金手指外形，一边连接标准 DDR5 插槽，另一边连接 Main PCB。
