## 章节总结 {.unlisted .unnumbered}

\markright{总结}

* 我们快速概述了三大主要平台（Linux、Windows 和 MacOS）上可用的最流行工具。根据 CPU 供应商的不同，分析工具的选择会有所不同。对于具有 Intel 处理器的系统，我们建议使用 VTune；对于具有 AMD 处理器的系统，使用 uProf；在 Apple 平台上，使用 Xcode Instruments。
* Linux perf 可能是 Linux 上最常用的分析工具。它支持所有主要 CPU 供应商的处理器。它没有图形界面。但是，有一些工具可以可视化 `perf` 的分析数据。
* 我们还讨论了 Windows 事件跟踪（ETW），它旨在观察运行系统中的软件动态。Linux 有一个类似的工具叫 [KUtrace](https://github.com/dicksites/KUtrace)，[^1] 它在 [@DickSitesBook] 一书中有介绍。
* 有一些混合分析器结合了代码检测、采样和跟踪等技术。这吸取了这些方法的优点，允许用户获得关于特定代码片段的非常详细的信息。在本章中，我们研究了 Tracy，它在游戏开发者中相当流行。
* 内存分析器提供有关内存使用、堆分配、内存占用和其他指标的信息。内存分析帮助你了解应用程序如何随时间使用内存。
* 持续分析工具已经成为生产环境中监控性能的重要组成部分。它们收集系统范围的性能指标和调用栈，持续数天、数周甚至数月。这些工具使发现性能变化开始的时间点并确定问题的根本原因变得更容易。

[^1]: KUtrace - [https://github.com/dicksites/KUtrace](https://github.com/dicksites/KUtrace)

\sectionbreak
