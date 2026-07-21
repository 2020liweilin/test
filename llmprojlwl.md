# 任务目标：基于 TinyRISCV 的大模型动态负载功耗特征仿真与对比分析

Hello Claude, 请作为一名高级数字 IC 设计与嵌入式软件工程师，严格按照以下步骤执行任务。该任务旨在通过开源 RISC-V 处理器仿真，对比“传统平稳负载”与“模拟 LLM 动态负载”下的芯片翻转率（功耗平替），以验证基于大模型动态负载进行硬件极限压测的优势。

## 一、 环境准备与项目克隆

1. **获取源码**：克隆开源项目 `tinyriscv` (https://gitee.com/liangkangnan/tinyriscv.git)。
2. **工具链要求**：请确保系统已安装 `riscv64-unknown-elf-gcc` (用于编译裸机 C 代码) 和 `iverilog` / `verilator` (用于 RTL 仿真及生成 VCD 波形文件)。
3. **了解仿真架构**：分析 `tinyriscv` 的 testbench，确认如何通过修改 Makefile 或 testbench 开启 VCD (Value Change Dump) 波形导出功能。

---

## 二、 编写与编译测试用例 (核心逻辑)

你需要在 `tinyriscv` 的固件目录下准备两个核心测试用例。这两个 C 程序必须以裸机（Bare-metal）形式运行，依赖对应的 `start.S` 启动代码和链接脚本 `link.ld`。

### 测试用例 1：传统平稳负载 (baseline.c)
* **目的**：模拟传统的 CPU 烤机工具（类似于 GPU_BURN[cite: 1]）。
* **要求**：实现一个死循环的整数矩阵乘法或移植 Dhrystone 测试。确保指令流水线和 ALU 处于高度规律的稳态。

### 测试用例 2：模拟 Qwen/vLLM 动态激变负载 (llm_dynamic_sim.c)
* **目的**：通过底层 C 代码，模拟 LLM 在 Prefill（预填充）和 Decode（解码）阶段交替时的硬件微架构行为[cite: 1]。
* **RISC-V 生态深度结合要求**：
    1. **使用内联汇编读取 CSR**：通过 `rdcycle` (或 `mcycle`) 和 `rdinstret` 获取时钟周期和指令退休数量，用于精准控制两段负载的执行时间，而不是简单的 `for` 循环延时。
    2. **使用内存屏障**：在随机访存阶段，加入 `asm volatile("fence i, i");` 或相关内存屏障指令，防止编译器或硬件流水线过度优化乱序访存。
    3. **裸机生态**：代码直接操作分配好的 SRAM 物理地址空间，不依赖任何标准库。
* **算法逻辑 (死循环交替执行)**：
    * **阶段 A (模拟 Prefill 阶段)**：密集计算。编写大规模的稠密矩阵乘法 (GEMM)[cite: 1]。这会让 ALU 满载，模拟极高的计算峰值。
    * **阶段 B (模拟 Decode 阶段)**：访存密集 (Memory-Bound)。基于伪随机数生成器，编写大范围的内存指针跳跃读取代码，模拟对 KV Cache 的不规则访存[cite: 1]。这会导致流水线频繁 Stall。

---

## 三、 执行 RTL 仿真与波形生成

1. 修改 `tinyriscv` 的编译脚本，分别将 `baseline.c` 和 `llm_dynamic_sim.c` 编译为 `.bin` 或 `.hex` 文件，作为 ROM 初始数据。
2. 使用 `iverilog` 运行 testbench。
3. **输出要求**：确保仿真过程生成了两个完整的波形文件：`baseline_wave.vcd` 和 `llm_dynamic_wave.vcd`。

---

## 四、 功耗平替（翻转率）解析与图表绘制

由于是 RTL 仿真，我们使用顶层模块信号的**翻转率 (Toggle Rate)** 作为动态功耗的替代指标。

1. **编写 Python 解析脚本**：
    * 请编写一个 Python 脚本（使用 `vcdvcd` 库或正则表达式解析 VCD 文件）。
    * 将时间轴划分为固定的时间窗口（例如每 1000 个时钟周期为一个 Window）。
    * 统计在每个 Window 内，处理器的核心数据总线、ALU 关键信号或整体寄存器的电平跳变总次数。
2. **绘制对比曲线**：
    * 使用 `matplotlib` 绘制两张折线图。横轴为时间（或周期），纵轴为翻转率（Toggle Rate / 模拟功耗）。
    * **图表 1**：`baseline.c` 的翻转率曲线。预期为一条相对平缓的直线。
    * **图表 2**：`llm_dynamic_sim.c` 的翻转率曲线。预期出现极其剧烈的锯齿状跳变（高 $di/dt$）。
3. **输出结果**：将两张图表保存为 `power_comparison.png`，并输出一段简短的 Markdown 分析报告，对比两者的峰谷差值。