
## Arxil 建设任务

### 试验阶段
1. 开展形式化验证。
2. 补充 Arxil 中的几个基本opcode，开展Node内部演化（尤其是 $\mathcal{D}$）的形式化工作。
3. 过程式映射（Imperative → TSL）：构建从典型过程式语言（如 C 或简化 Imperative Core）到 Arxil 的语义保持编译方案，证明控制流、状态更新、栈帧等可被精确编码。
4. 函数式映射（Functional → Arxil）：构建从纯函数式语言（如 STLC 或带递归的 λ-演算）到 Arxil 的结构化嵌入方案，重点处理闭包、高阶函数、尾递归等特性。
5. 探索 Arxil 与并发模型（如 π-calculus、Actor 模型）的映射。
6. 开展 Arxil 作为中间表示（IR）的系统化完善工作。

### 工程化阶段
1. 对应于新的语言规范，更新语言形式化文档的内容。
2. 开展第一组 `.arxtype` 的编写。
3. 根据形式化文档，制作第一版简单的编译器，只支持 `instruct` 和 `'inst_scri'`。
4. 结合 EDSOS 的进程子系统文档，制作第一版 ArxilVM。
5. 展开分析和设计 `.arxlib`。

---

## .arxtype建设路径
短期: 基于 .NET BCL 和 NativeAOT，建立 `.arxtype` 生成器；基于 LLVM Intrinsic，提取出部分关键路径上的硬件指令优化实现，补充强化输出的 Target Code 性能。
中期: 深入提取、整理、融合 LLVM Target Description，使原生级编译能接近达到手写汇编的性能；设计并实现 ArxilRT，验证兼容级模型。
长期: 构建完整的 ArxilVM，并确保其 JIT 编译器能充分利用 .arxtype 中的信息进行优化。最终形成一个闭环：开发者只需关心 Arxil 源码和 .arxtype 接口，编译器和运行时会根据目标环境自动选择最优的执行路径。