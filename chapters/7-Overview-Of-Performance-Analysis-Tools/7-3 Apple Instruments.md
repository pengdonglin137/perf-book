## Apple Xcode Instruments

在 MacOS 上进行类似性能分析的最方便方法是使用 Xcode Instruments。这是一个随 Xcode 免费提供的应用程序性能分析器和可视化工具。Instruments 分析器建立在从 Solaris 移植到 MacOS 的 DTrace 跟踪框架之上。它有许多工具来检查应用程序的性能，并使我们能够执行其他分析器（如 Intel VTune）可以执行的大多数基本操作。获取分析器的最简单方法是从 Apple App Store 安装 Xcode。该工具不需要配置；安装后即可使用。

在 Instruments 中，你使用称为 instruments 的专用工具来跟踪应用程序、进程和设备随时间的不同方面。Instruments 具有强大的可视化机制。它在分析时收集数据，并实时向你展示结果。你可以收集不同类型的数据并并排查看它们，这使你能够看到执行中的模式，关联系统事件并发现非常细微的性能问题。

在本章中，我们将只展示最与本书相关的"CPU Counters"instrument。Instruments 还可以可视化 GPU、网络和磁盘活动，跟踪内存分配和释放，捕获用户事件（如鼠标点击），提供功耗效率见解等。你可以在 Instruments [文档](https://help.apple.com/instruments/mac/current)中阅读更多关于这些用例的信息。[^1]

### 你能用它做什么： {.unlisted .unnumbered}

- 访问 Apple 处理器上的硬件性能计数器。
- 查找程序中的热点及其调用栈。
- 将生成的 ARM 汇编代码与源代码并排检查。
- 过滤时间线上选定间隔的数据。

### 你不能用它做什么： {.unlisted .unnumbered}

与其他基于采样的分析器类似，Xcode Instruments 具有与 VTune 和 uProf 相同的盲点。

### 示例：分析 Clang 编译 {.unlisted .unnumbered}

在这个示例中，我将展示如何在配备 M1 处理器、macOS 13.5.1 Ventura 和 16 GB RAM 的 Apple Mac mini 上收集硬件性能计数器。我从 LLVM 代码库中取了最大的文件之一，并使用 Clang C++ 编译器的 15.0 版本分析其编译。

![Xcode Instruments：时间线和统计面板。](../../img/perf-tools/XcodeInstrumentsView.jpg){#fig:InstrumentsView width=100% }

以下是我使用的命令行：

```bash
$ clang++ -O3 -DNDEBUG -arch arm64 <other options ...> -c llvm/lib/Transforms/Vectorize/LoopVectorize.cpp
```

图 @fig:InstrumentsView 显示了 Xcode Instruments 的主时间线视图。此截图是在编译完成后拍摄的。我们稍后会回到它，但首先，让我们展示如何启动分析会话。

首先，打开 *Instruments* 并选择 *CPU Counters* 分析类型。你需要做的第一步是配置收集。点击并按住红色目标图标（参见图 @fig:InstrumentsView 中的 \circled{1}），然后从菜单中选择 *Recording Options...*。它将显示图 @fig:InstrumentsDialog 中所示的对话框窗口。这是你可以添加硬件性能监控事件进行收集的地方。Apple 在其手册 [@AppleOptimizationGuide, Section 6.2 Performance Monitoring Events] 中记录了其硬件性能监控事件。

![Xcode Instruments：CPU Counters 选项。](../../img/perf-tools/XcodeInstrumentsDialog.png){#fig:InstrumentsDialog width=70% }

第二步是设置分析目标。为此，点击并按住应用程序的名称（在图 @fig:InstrumentsView 中标记为 \circled{2}）并选择你感兴趣的应用程序。设置参数和环境变量（如果需要）。现在，你可以开始收集了；按红色目标图标 \circled{1}。

Instruments 显示时间线并不断更新关于正在运行的应用程序的统计信息。程序完成后，Instruments 将显示如图 @fig:InstrumentsView 所示的结果。编译花费了 7.3 秒，我们可以看到事件数量随时间的变化。例如，执行的分支指令和预测错误的数量在运行时接近尾声时增加。你可以在时间线上放大到该间隔以检查涉及的函数。
