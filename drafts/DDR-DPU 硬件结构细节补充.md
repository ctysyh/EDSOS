# DDR-DPU 硬件结构细节补充

## 一、架构对齐：基于 MRDIMM

### 1. **MRDIMM 的核心特征回顾**
- **子通道（Sub-Channel）架构**：将 64-bit DQ 通道拆分为两个 32-bit 子通道（SubCH A/B），每个子通道可独立访问不同 Rank/Bank Group。
- **RCD + DB（Data Buffer）协同**：RCD 负责 CA 分发，DB 负责 DQ 缓冲与重驱动。
- **支持 BL16 突发**：每个子通道可独立完成 BL16（512b）传输。
- **JEDEC 标准化**：DDR5 MRDIMM 已纳入 JEDEC JESD79-5B，是未来高带宽服务器内存的主流形态。

### 2. **本方案如何对齐 MRDIMM？**
| 要素 | 本方案实现 |
|------|-----------|
| **CPU 侧接口** | 完全兼容 MRDIMM：64-bit CA + 64-bit DQ（2×32b SubCH） |
| **Sub PCB 形态** | 封装为标准 MRDIMM 模组（含 RCD + 2×DB） |
| **DPU 接口** | 通过 **私有链路连接 RCD/DB 内部扩展逻辑**，不暴露于 DIMM 金手指 |
| **DRAM Die** | 采用 **主从式双端口 DRAM**，每个 Die 支持双 LIO/GIO 路径 |

> **关键结论**：Sub PCB 是 **“增强型 MRDIMM”** ——对外是标准 MRDIMM，对内集成 DPU 访问能力。

---

## 二、控制面细化：CAIU 注入逻辑与两阶段提交协议

### 1. **CAIU 的核心职责升级**
传统 CAIU 仅做门控，现在需承担 **动态调度器 + 协议引擎** 角色：

#### （1）**两阶段提交协议（Two-Phase Commit for DPU）**
- **Phase 1（请求）**：DPU 发送：
  ```text
  {addr, len, max_latency, access_key, priority}
  ```
  - `max_latency`：DPU 可接受的最晚完成时间（如 100 ns）。
- **Phase 2（授权）**：CAIU 返回：
  ```text
  {granted, exact_tCK_window, subch_id, bg_id}
  ```
  - `exact_tCK_window`：精确到 tCK 的执行窗口（如 “t=128–135”）；
  - 若无法满足 `max_latency`，返回 `granted=false`。

> **意义**：DPU 可据此安排内部流水线（如提前加载指令），避免盲目等待。

#### （2）**内部 CA 处理频率翻倍（2×fCK）**
- **动机**：在不增加外部引脚的前提下，提升 CA 吞吐。
- **实现**：
  - RCD 内部 PLL 生成 **2×fCK 时钟**（如 DDR5-4800 → 1.6 GHz 内部）；
  - **上升沿**：处理 CPU CA（JEDEC 标准行为）；
  - **下降沿**：处理 DPU CA（经 CAIU 授权）；
  - **物理实现**：类似 DDR5 的 **dual-edge command sampling**（已有先例，如某些 RCD 的 test mode）。

> **注意**：对外 CK 仍为 fCK，CPU 无感知；仅 RCD 内部逻辑提速。

#### （3）**BG 负载感知调度**
- CAIU 维护每个 BG 的状态机：
  ```text
  {idle, active, precharging, busy_until_tCK}
  ```
- 结合 tCCD_L/tCCD_S、tRCD、tRP 等参数，动态计算可用窗口。
- 优先分配 **空闲 BG** 或 **即将空闲 BG** 给 DPU。

---

## 三、数据面协同：子通道模式 + 双端口 DRAM + BL16 合并

### 1. **CPU 侧：标准 MRDIMM 子通道行为**
- CPU Controller 发送 **64-bit CA**，RCD 自动拆分为 SubCH A/B；
- 每个子通道可访问 **独立 Rank/BG**；
- 支持 **BL16 突发**（32b × 16 = 512b per subch）。

### 2. **DPU 侧：利用双端口 DRAM 实现“隐形子通道”**
- **DRAM Die 内部**：
  - Port A（主）→ SubCH A（CPU）
  - Port B（从）→ **DPU 专用路径**（不经过 DB 金手指）
- **关键创新**：当 CPU 与 DPU **并发访问同一 BG 的不同列**：
  - DRAM 微架构支持 **双列选择（Dual CSL）**；
  - SA 同时驱动 LIO_A 和 LIO_B；
  - 数据分别送至 SubCH A 和 DPU DQ'。

### 3. **BL16 合并与结果同步**
- **场景**：DPU 请求 64B，需跨两个子通道（或两次 BL16）。
- **DRAM 微架构支持**：
  - 内部 **结果 FIFO + 合并引擎**；
  - 在 DPU DQ' 接口输出 **连续 64B**，而非分段 BL16。
- **时序对齐**：通过内部 2×fCK 时钟确保两次读取的相位对齐。

---

## 四、时序训练与同步：CPU 黑盒下的精密对齐

### 1. **CPU 侧训练机制（完全标准）**
- **Write Leveling**、**Read/Write Preamble Training**、**CA Training** 均按 JEDEC 执行；
- RCD 对外表现为 **标准 MRDIMM RCD**，训练结果仅用于 CPU 路径。

### 2. **DPU 侧训练机制（私有）**
- **CK 相位训练**：
  - Main PCB 采样 Sub PCB 回传的 CK_ref；
  - 执行 **DPU Write Leveling-like** 流程，校准 DQ'/DQS' 相位；
  - 结果存储于 CAIU 内部寄存器。
- **CA 注入延迟校准**：
  - DPU 发送测试 CA，测量从注入到 DQ' 返回的环回延迟；
  - 动态调整 `exact_tCK_window` 补偿 skew。

### 3. **Timing Margin 的双重利用**
| Margin 类型 | CPU 利用 | DPU 利用 |
|------------|--------|--------|
| **tCCD_L 冗余** | 保留 | 用于插入 DPU 命令 |
| **tDQSCK skew** | 由 DB 校准 | 由私有 DLL 校准 |
| **Command Bus Setup/Hold** | 按 JEDEC | 下降沿利用“空闲边沿” |
| **Bank Precharge Idle** | 无利用 | 用于 DPU 预激活 |

---

## 五、DRAM 微架构适配：频率翻倍与双端口协同

### 1. **DRAM Die 频率翻倍（2×fCK）的必要性**
- **原因**：若 RCD 内部以 2×fCK 处理 CA，则 DRAM 必须能响应更密集的命令。
- **实现方式**：
  - **内部 DLL 生成 2×fCK**；
  - **Column Decoder、IOG-MUX、LIO 驱动器** 均工作在 2×fCK；
  - **Cell Array 与 SA 仍工作在 fCK**（因 tRCD/tRP 限制）。

> 这不是“DRAM 频率翻倍”，而是 **I/O 逻辑频率翻倍**，类似 GDDR6 的 2N 模式。

### 2. **主从端口的同步动作**
- **并发读示例**：
  ```text
  t=0 (↑): CPU CA → Port A → CSL_A = 1
  t=0.5 (↓): DPU CA → Port B → CSL_B = 1
  t=4: SA 同时驱动 LIO_A 和 LIO_B
  t=4.5: GIO_A/GIO_B 输出数据
  ```
- **关键电路**：IOG-MUX 的 NMOS_A/NMOS_B 栅极由 **独立 CSL 信号** 控制，互不干扰。

### 3. **功耗与面积影响**
- **面积**：增加一套 LIO/GIO/DQ' 逻辑 ≈ +8–12% die area；
- **功耗**：2×fCK I/O 逻辑动态功耗翻倍，但仅在 DPU 活跃时开启（可电源门控）；
- **热设计**：DPU 访问通常 bursty，平均功耗可控。
