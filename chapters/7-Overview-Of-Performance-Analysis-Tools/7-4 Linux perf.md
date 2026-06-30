## Linux Perf

Linux Perf 可能是世界上使用最多的性能分析器，因为它在大多数 Linux 发行版上可用，这使其可被广泛的用户访问。Perf 在许多流行的 Linux 发行版中被原生支持，包括 Ubuntu、Red Hat 和 Debian。它包含在内核中，因此你可以在任何运行 Linux 的系统上获得操作系统级统计信息（页面错误、CPU 迁移等）。截至 2024 年中，该分析器支持 x86、ARM、PowerPC64、UltraSPARC 和其他几种 CPU 类型。[^2] 在这些平台上，`perf` 提供对硬件性能监控功能的访问，例如性能计数器。有关 Linux `perf` 的更多信息，请参阅其 [wiki 页面](https://perf.wiki.kernel.org/index.php/Main_Page)[^1]。

### 如何配置 {.unlisted .unnumbered}

安装 Linux perf 非常简单，可以用一条命令完成：

```bash
$ sudo apt-get install linux-tools-common linux-tools-generic linux-tools-`uname -r`
```

另外，除非安全是问题，否则考虑更改以下默认值：

```bash
# 允许非特权用户进行内核分析和访问 CPU 事件
$ echo 0 | sudo tee /proc/sys/kernel/perf_event_paranoid
$ echo kernel.perf_event_paranoid=0 | sudo tee -a /etc/sysctl.d/local.conf
# 启用非特权用户的内核模块符号解析
$ echo 0 | sudo tee /proc/sys/kernel/kptr_restrict
$ echo kernel.kptr_restrict=0 | sudo tee -a /etc/sysctl.d/local.conf
```

### 你能用它做什么： {.unlisted .unnumbered}

通常，Linux `perf` 可以做其他分析器可以做的大多数事情。硬件供应商优先在 Linux `perf` 中启用其功能，以便在新 CPU 上市时，`perf` 已经支持它。大多数人使用两个主要命令。第一个 `perf stat` 报告指定性能事件的计数。第二个 `perf record` 在采样模式下分析应用程序或系统，通常后跟 `perf report` 从采样数据生成报告。

`perf record` 命令的输出是样本的原始转储。许多建立在 Linux `perf` 之上的工具解析原始转储文件并提供新的分析类型。以下是最值得注意的：

- 火焰图，在 [@sec:secFlameGraphs] 中讨论。
- [KDAB Hotspot](https://github.com/KDAB/hotspot)，[^3] 一个使用与 Intel VTune 非常相似的界面可视化 Linux `perf` 数据的工具。如果你使用过 Intel VTune，KDAB Hotspot 会看起来非常熟悉。
- Netflix [Flamescope](https://github.com/Netflix/flamescope)。[^4] 此工具显示应用程序运行时采样事件的热图。你可以观察工作负载行为中的不同阶段和模式。Netflix 工程师使用此工具发现了一些非常细微的性能错误。此外，你可以在热图上选择时间范围并为该时间范围生成火焰图。

### 你不能用它做什么： {.unlisted .unnumbered}

Linux perf 是一个命令行工具，缺少图形用户界面（GUI），这使得过滤数据、观察工作负载行为随时间的变化、放大运行时的一部分等变得困难。通过 `perf report` 命令提供了有限的控制台输出，这对于快速分析来说虽然不如其他 GUI 分析器方便，但已经足够了。幸运的是，正如我们刚才提到的，有一些 GUI 工具可以后处理和可视化 Linux `perf` 的原始输出。

[^1]: Linux perf wiki - [https://perf.wiki.kernel.org/index.php/Main_Page](https://perf.wiki.kernel.org/index.php/Main_Page)。
[^2]: RISCV 尚未作为官方内核的一部分支持，尽管供应商存在自定义工具。
[^3]: KDAB Hotspot - [https://github.com/KDAB/hotspot](https://github.com/KDAB/hotspot)。
