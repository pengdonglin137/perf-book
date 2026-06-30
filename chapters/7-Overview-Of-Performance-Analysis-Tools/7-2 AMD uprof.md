## AMD uProf

[uProf](https://www.amd.com/en/developer/uprof.html) 分析器是 AMD 开发的用于监控在 AMD 处理器上运行的应用程序性能的工具。虽然 uProf 也可以在 Intel 处理器上使用，但你只能使用与 CPU 无关的功能。该分析器可免费下载，可用于 Windows、Linux 和 FreeBSD。AMD uProf 可用于在多个虚拟机（VM）上进行分析，包括 Microsoft Hyper-V、KVM、VMware ESXi 和 Citrix Xen，但并非所有功能在所有 VM 上都可用。此外，uProf 支持分析用各种语言编写的应用程序，包括 C、C++、Java、.NET/CLR。

### 如何配置 {.unlisted .unnumbered}

在 Linux 上，uProf 使用 Linux perf 进行数据收集。在 Windows 上，uProf 使用自己的采样驱动程序，在安装 uProf 时安装，不需要额外配置。AMD uProf 支持命令行界面（CLI）和图形界面。CLI 界面需要两个单独的步骤——收集和报告，类似于 Linux perf。

### 你能用它做什么： {.unlisted .unnumbered}

- 查找热点：函数、语句、指令。
- 监控各种硬件性能事件并定位发生这些事件的代码行。
- 过滤特定函数或线程的数据。
- 观察工作负载随时间的行为：在时间线图表中查看各种性能事件。
- 分析热调用路径：调用图、火焰图和自底向上图表。

此外，uProf 可以在 Linux 上监控各种操作系统事件：线程状态、线程同步、系统调用、页面错误等。你可以用它来分析 OpenMP 应用程序以检测线程不平衡，并分析 MPI[^3] 应用程序以检测 MPI 集群节点之间的负载不平衡。有关 uProf 各种功能的更多详细信息，请参阅 [用户指南](https://www.amd.com/en/developer/uprof.html#documentation)[^1]。

### 你不能用它做什么： {.unlisted .unnumbered}

由于工具的采样性质，它最终会错过持续时间非常短的事件。报告的样本是统计估计的数字，大多数时候足以分析性能，但不是事件的确切计数。

### 示例 {.unlisted .unnumbered}

为了演示 AMD uProf 工具的外观和感觉，我们在 AMD Ryzen 9 7950X 上运行了 [Scimark2](https://math.nist.gov/scimark2/index.html)[^2] 基准测试中的密集 LU 矩阵分解组件，运行 Windows 11，配备 64 GB RAM。

![uProf 的函数热点视图。](../../img/perf-tools/uProf_Hopspot.png){#fig:uProfHotspots width=100% }

图 @fig:uProfHotspots 显示了*函数热点*分析（在图像左侧的菜单列表中选择）。在图像顶部，你可以看到一个事件时间线，显示在应用程序执行的各个时间观察到的事件数量。在右侧，你可以选择要绘制的指标；我们选择了 `RETIRED_BR_INST_MISP`。注意在 20s 到 40s 的时间范围内分支预测错误的尖峰。你可以选择此区域以密切分析那里发生了什么。一旦你这样做，它将更新底部面板以仅显示该时间间隔的统计信息。

在时间线图下方，你可以看到函数表中所选函数的自底向上调用栈视图。如我们所见，所选的 `LU_factor` 函数是从 `kernel_measureLU` 调用的，而后者又是从 `main` 调用的。在 Scimark2 基准测试中，这是 `LU_factor` 的唯一调用栈，即使它显示 `Call Stacks [5]`。这是可以忽略的收集伪像。但在其他应用程序中，热函数可以从许多不同的地方调用，因此你也需要检查其他调用栈。

如果你双击任何函数，uProf 将打开该函数的源代码/汇编视图。为简洁起见，我们不显示此视图。在左侧面板中，还有其他可用的视图，如指标、火焰图、调用图视图和线程并发。它们对分析也很有用，但我们决定跳过它们。读者可以自己尝试并查看这些视图。

[^1]: AMD uProf 用户指南 - [https://www.amd.com/en/developer/uprof.html#documentation](https://www.amd.com/en/developer/uprof.html#documentation)
[^2]: Scimark2 - [https://math.nist.gov/scimark2/index.html](https://math.nist.gov/scimark2/index.html)
[^3]: MPI - 消息传递接口，分布式内存系统上并行编程的标准。
