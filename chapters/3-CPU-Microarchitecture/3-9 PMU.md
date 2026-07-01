


## 性能监控单元 {#sec:PMU}

每款现代 CPU 都提供了用于监控性能的设施，这些设施被整合到性能监控单元（Performance Monitoring Unit，PMU）中。该单元集成了帮助开发者分析应用性能的各种功能。图 @fig:PMU 展示了现代 Intel CPU 中 PMU 的示例。大多数现代 PMU 都有一组性能监控计数器（Performance Monitoring Counter，PMC），可用于收集程序执行过程中发生的各种性能事件。我们将在 [@sec:counted] 中讨论如何使用 PMC 进行性能分析。此外，PMU 还有其他增强性能分析的功能，如 LBR、PEBS 和 PT，[@sec:PmuChapter] 专门讨论这些主题。

![现代 Intel CPU 的性能监控单元。](../../img/uarch/PMU.png){#fig:PMU width=100%}

随着每一代新 CPU 的设计演进，其 PMU 也在不断发展。在 Linux 上，可以使用 `cpuid` 命令确定 CPU 中 PMU 的版本，如 [@lst:QueryPMU] 所示。类似的信息也可以通过检查 `dmesg` 命令输出的内核消息缓冲区来获取。每个 Intel PMU 版本的特性以及与前一版本的变更，可以在 [@IntelOptimizationManual, Volume 3B, Chapter 20] 中找到。

Listing: 查询你的 PMU

~~~~ {#lst:QueryPMU .bash}
$ cpuid
...
Architecture Performance Monitoring Features (0xa/eax):
      version ID                               = 0x4 (4)
      number of counters per logical processor = 0x4 (4)
      bit width of counter                     = 0x30 (48)
...
Architecture Performance Monitoring Features (0xa/edx):
      number of fixed counters    = 0x3 (3)
      bit width of fixed counters = 0x30 (48)
...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### 性能监控计数器 {#sec:PMC}

如果我们想象一个简化的处理器视图，它可能如图 @fig:PMC 所示。正如我们在本章前面讨论的，现代 CPU 有缓存、分支预测器、执行流水线和其他单元。当 PMC 连接到多个单元时，可以从中收集有用的统计信息。例如，它可以统计经过了多少个时钟周期、执行了多少条指令、发生了多少次缓存未命中或分支误预测等性能事件。

![带有性能监控计数器的 CPU 简化视图。](../../img/uarch/PMC.png){#fig:PMC width=60%}

PMC 通常为 48 位宽，这使得分析工具可以长时间运行而无需中断程序执行。[^2] 性能计数器是一种作为型号特定寄存器（Model-Specific Register，MSR）实现的硬件寄存器。这意味着计数器的数量和宽度可能因型号而异，你不能依赖 CPU 中计数器的数量始终相同。你应该始终先查询，例如使用 `cpuid` 等工具。PMC 通过 `RDMSR` 和 `WRMSR` 指令访问，这些指令只能在内核空间中执行。幸运的是，只有当你在开发性能分析工具（如 Linux `perf` 或 Intel VTune Profiler）时才需要关心这一点。这些工具处理了 PMC 编程的所有复杂性。

工程师在分析应用时，收集已执行指令数和经过的周期数是非常常见的。这就是为什么某些 PMU 有专门的 PMC 来收集这类事件。固定计数器（Fixed Counter）始终测量 CPU 核心内的同一指标。可编程计数器（Programmable Counter）则由用户选择要测量的内容。

例如，在 Intel Skylake 架构（PMU 版本 4，参见 [@lst:QueryPMU]）中，每个物理核心有三个固定计数器和八个可编程计数器。三个固定计数器分别设置为核心时钟、参考时钟和退休指令数（详见 [@sec:secMetrics] 中关于这些指标的说明）。AMD Zen4 和 Arm Neoverse V1 核心每个处理器核心支持 6 个可编程性能监控计数器，没有固定计数器。

PMU 提供超过一百个可监控事件的情况并不罕见。图 @fig:PMU 只展示了现代 Intel CPU 上可用于监控的性能监控事件的一小部分。不难注意到，可用 PMC 的数量远少于性能事件的数量。不可能同时统计所有事件，但分析工具通过在程序执行期间在性能事件组之间进行多路复用来解决这个问题（参见 [@sec:secMultiplex]）。

* 对于 Intel CPU，完整的性能事件列表可以在 [@IntelOptimizationManual, Volume 3B, Chapter 20] 或 [perfmon-events.intel.com](https://perfmon-events.intel.com/) 上找到。
* AMD 没有为每款 AMD 处理器发布性能监控事件列表。好奇的读者可以在 Linux `perf` 源代码[^3]中找到一些信息。此外，你可以使用 AMD uProf 命令行工具列出可用于监控的性能事件。关于 AMD 性能计数器的一般信息可以在 [@AMDProgrammingManual, 13.2 Performance Monitoring Counters] 中找到。
* 对于 ARM 芯片，性能事件的定义不那么统一。各厂商按照 ARM 架构实现核心，但性能事件在含义和支持的事件上都有所不同。对于 Arm 自行设计的 Neoverse V1 核心，性能事件列表可以在 [@ARMNeoverseV1] 中找到。对于 Arm Neoverse V2 和 V3 微架构，性能事件列表可在 Arm 官网找到。[^4]

[^2]: 当 PMC 的值溢出时，必须中断程序的执行。性能分析工具随后应保存溢出的事实。我们将在 [@sec:sec_PerfApproaches] 中更详细地讨论这一点。
[^3]: AMD 核心的 Linux 源代码——[https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c](https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c)
[^4]: Arm 遥测——[https://developer.arm.com/telemetry](https://developer.arm.com/telemetry)
