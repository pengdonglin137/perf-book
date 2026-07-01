## 基于硬件的采样功能

主要 CPU 供应商提供一组额外功能来增强采样。由于 CPU 供应商以不同的方式处理性能监控，这些能力不仅在称呼上有所不同，而且在你可以用它们做什么方面也有所不同。在 Intel 处理器中，它被称为处理器基于事件的采样（PEBS），首先在 NetBurst 微架构中引入。AMD 处理器上的类似功能称为基于指令的采样（IBS），从 AMD Opteron Family（10h 代）核心开始可用。接下来，我们将更详细地讨论这些功能，包括它们的相似之处和不同之处。

### Intel 平台上的 PEBS {#sec:secPEBS}

与 Last Branch Record 功能类似，PEBS 在分析程序时用于在每个收集的样本中捕获附加数据。当性能计数器配置为 PEBS 时，处理器保存一组附加数据，该数据具有定义的格式，称为 PEBS 记录。Intel Skylake CPU 的 PEBS 记录格式如@fig:PEBS_record 所示。它包含通用寄存器的状态（`EAX`、`EBX`、`ESP` 等）、`EventingIP`、`Data Linear Address` 和 `Latency value`，以及其他一些字段。PEBS 记录的内容布局因微架构而异，参见 [@IntelOptimizationManual, Volume 3B, Chapter 20 Performance Monitoring]。

![第 6 代、第 7 代和第 8 代 Intel Core 处理器系列的 PEBS 记录格式。*© 来源：[@IntelOptimizationManual, Volume 3B, Chapter 20]。*](../../img/pmu-features/PEBS_record.png){#fig:PEBS_record width=100%}

从 Skylake 开始，PEBS 记录已得到增强，可以收集 XMM 寄存器和 Last Branch Record（LBR）记录。格式已重构，字段分为 Basic 组、Memory 组、GPR 组、XMM 组和 LBR 组。性能分析工具可以选择感兴趣的数据组，从而减少记录开销。默认情况下，PEBS 记录只包含 Basic 组。

使用 PEBS 的一个显著好处是与常规基于中断的采样相比，采样开销更低。回想一下，当计数器溢出时，CPU 生成中断以收集一个样本。频繁生成中断并让分析工具本身在中断服务例程内捕获程序状态是非常昂贵的，因为它涉及操作系统交互。

另一方面，PEBS 维护一个缓冲区来临时存储多个 PEBS 记录。假设我们正在使用 PEBS 采样加载指令。当性能计数器配置为 PEBS 时，计数器中的溢出条件不会触发中断，而是会激活 PEBS 机制。该机制然后将捕获下一个加载，捕获一个新记录，并将其存储在专用的 PEBS 缓冲区区域中。该机制还负责清除计数器溢出状态并用初始值重新加载计数器。只有当专用缓冲区满时，处理器才会发出中断，缓冲区才会刷新到内存。此机制通过触发更少的中断来降低采样开销。

Linux 用户可以通过执行 `dmesg` 来检查是否启用了 PEBS：

```bash
$ dmesg | grep PEBS
[    0.113779] Performance Events: XSAVE Architectural LBR, PEBS fmt4+-baseline,  
AnyThread deprecated, Alderlake Hybrid events, 32-deep LBR, full-width counters, Intel PMU driver.
```

对于 LBR，Linux perf 在每个收集的样本时转储 LBR 堆栈的全部内容。因此，可以分析 Linux perf 收集的原始 LBR 转储。但是，对于 PEBS，Linux `perf` 不会像对 LBR 那样导出原始输出。相反，它处理 PEBS 记录并仅根据特定需求提取数据的子集。因此，无法使用 Linux `perf` 访问原始 PEBS 记录的集合。但是，Linux `perf` 提供了一些从原始样本处理的 PEBS 数据，可以通过 `perf report -D` 访问。要转储原始 PEBS 记录，你可以使用 [`pebs-grabber`](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber)[^1]。

### AMD 平台上的 IBS

基于指令的采样（IBS）是 AMD64 处理器的一项功能，可用于收集与指令获取和指令执行相关的特定指标。AMD 处理器的流水线由两个独立的阶段组成：获取 AMD64 指令字节的前端阶段和执行 `ops` 的后端阶段。由于阶段在逻辑上是分离的，因此有两种独立的采样机制：IBS Fetch 和 IBS Execute。

- IBS Fetch 监控流水线的前端，并提供有关 ITLB（命中或未命中）、I-cache（命中或未命中）、获取地址、获取延迟等信息。
- IBS Execute 监控流水线的后端，并通过跟踪单个 op 的执行来提供有关指令执行行为的信息。例如，分支（已采取或未采取、已预测或未预测）和加载/存储（在 D-cache 和 DTLB 中命中或未命中、线性地址、加载延迟）。

PMC 和 AMD 处理器中的 IBS 有几个重要区别。PMC 计数器是可编程的，而 IBS 充当固定计数器。IBS 计数器只能启用或禁用用于监控，不能编程为任何选择性事件。IBS Fetch 和 Execute 计数器可以独立启用/禁用。使用 PMC，用户必须提前决定要监控哪些事件。使用 IBS，为每条采样指令收集丰富的数据集，然后由用户分析他们感兴趣的数据部分。IBS 选择并标记要监控的指令，然后在执行期间捕获由该指令引起的微架构事件。Intel PEBS 和 AMD IBS 的更详细比较可以在 [@ComparisonPEBSIBS] 中找到。

由于 IBS 集成到处理器流水线中并充当固定事件计数器，因此样本收集开销最小。分析器需要处理 IBS 生成的数据，根据采样间隔、配置的线程数、是否配置 Fetch/Execute 等，这些数据可能非常大。在 Linux 内核版本 6.1 之前，IBS 总是为所有核心收集样本。此限制导致大量数据收集和处理开销。从内核 6.2 开始，Linux perf 仅支持为配置的核心进行 IBS 样本收集。

Linux perf 和 AMD uProf 分析器支持 IBS。以下是收集 IBS Execute 和 Fetch 样本的示例命令：

```bash
```
