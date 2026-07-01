## 内存分析 {#sec:MemoryProfiling}

到目前为止，在本章中，我们讨论了识别程序花费大部分时间的位置的工具。在本节中，我们将专注于程序与内存的交互。这通常称为*内存分析*。特别是，我们将学习如何收集内存使用情况、分析堆分配和度量内存占用。内存分析帮助你了解应用程序如何随时间使用内存，并帮助你构建程序与内存交互的准确心智模型。以下是它可以回答的一些问题：

* 程序的总虚拟内存消耗是多少，它如何随时间变化？
* 程序在何处以及何时进行堆分配？
* 分配内存最多的代码位置在哪里？
* 程序每秒访问多少内存？
* 程序的总内存占用是多少？

### 内存使用

内存使用通常由虚拟内存大小（VSZ）和常驻集大小（RSS）描述。VSZ 包括进程可以访问的所有内存，例如堆栈、堆、用于编码可执行文件指令的内存以及来自链接共享库的指令，包括换出到磁盘/SSD 的内存。另一方面，RSS 度量分配给进程的内存量驻留在 RAM 中。因此，RSS 不包括换出的内存或该进程从未触及的内存。RSS 也不包括未加载到内存中的共享库的内存。使用 `mmap` 映射到内存的文件也对 VSZ 和 RSS 使用有贡献。

考虑一个示例。进程 `A` 有 200K 的堆栈和堆分配，其中 100K 驻留在主内存中；其余被换出或未使用。它有一个 500K 的二进制文件，其中只有 400K 被触及。进程 `A` 链接到 2500K 的共享库，只加载了 1000K 到主内存中。

```
VSZ: 200K + 500K + 2500K = 3200K
RSS: 100K + 400K + 1000K = 1500K
```

开发人员可以使用标准 `top` 工具在 Linux 上观察 RSS 和 VSZ，但是，这两个指标可能变化得非常快。幸运的是，一些工具可以记录和可视化随时间变化的内存使用情况。@fig:MemoryUsageAIBench 显示了 PSPNet 图像分割算法的内存使用情况，它是 [AI Benchmark Alpha](https://ai-benchmark.com/alpha.html) 的一部分。[^5] 此图表是基于名为 [memory_profiler](https://github.com/pythonprofilers/memory_profiler)[^6] 的工具的输出创建的，这是一个建立在跨平台 [psutil](https://github.com/giampaolo/psutil)[^7] 包之上的 Python 库。

![AI_bench PSPNet 图像分割的 RSS 和 VSZ 内存利用率。](../../img/memory-access-opts/MemoryUsageAIBench.png){#fig:MemoryUsageAIBench width=100%}

除了标准的 RSS 和 VSZ 指标外，人们还开发了一些更复杂的指标。由于 RSS 包括进程独有的内存和与其他进程共享的内存，因此不清楚进程自己拥有多少内存。USS（唯一集大小）是进程独有的内存，如果进程现在被终止，这些内存将被释放。PSS（比例集大小）表示唯一内存加上共享内存量，在共享它的进程之间均匀分配。例如，如果一个进程有 10 MB 全部给自己（USS）和 10 MB 与另一个进程共享，它的 PSS 将是 15 MB。`psutil` 库支持度量这些指标（仅限 Linux），可以由 `memory_profiler` 可视化。

在 Windows 上，类似的概念由已提交内存大小和工作集大小定义。它们不是 VSZ 和 RSS 的直接等效项，但可用于有效估计 Windows 应用程序的内存使用情况。[RAMMap](https://learn.microsoft.com/en-us/sysinternals/downloads/rammap)[^8] 工具提供了有关系统和单个进程内存使用的丰富信息。

当开发人员谈论内存消耗时，他们隐含地指的是堆使用。堆实际上是大多数应用程序中最大的内存消耗者，因为它容纳所有动态分配的对象。但堆不是唯一的内存消耗者。为了完整性，让我们提及其他方面：

* 堆栈：应用程序中堆栈帧使用的内存。应用程序内的每个线程都有自己 的堆栈内存空间。通常，堆栈大小只有几 MB，如果超过限制，应用程序将崩溃。例如，Linux 上堆栈内存的默认大小通常为 8MB，尽管它可能因发行版和内核设置而异。macOS 上的默认堆栈大小也是 8MB，但在 Windows 上只有 1 MB。总堆栈内存消耗与系统中运行的线程数成正比。
* 代码：用于存储应用程序及其库的代码（指令）的内存。在大多数情况下，它对内存消耗的贡献不大，但也有例外。例如，Clang 17 C++ 编译器有 33 MB 的代码段，而最新的 Windows Chrome 浏览器在其 219MB 的 `chrome.dll` 中有 187MB 专用于代码。但是，并非所有代码部分在程序运行时都会被频繁使用。我们将在 [@sec:CodeFootprint] 中展示如何度量代码占用。

由于堆通常是内存资源的最大消耗者，因此开发人员在分析应用程序的内存使用情况时关注内存的这一部分是有意义的。在下一节中，我们将研究流行的实际应用程序中的堆消耗和内存分配。

### 案例研究：分析 Stockfish 的堆分配 {#sec:HeaptrackCaseStudy}

在这个案例研究中，我使用 [heaptrack](https://github.com/KDE/heaptrack)[^2]，这是 KDE 开发的开源 Linux 堆内存分析器。Ubuntu 用户可以非常容易地使用 `apt install heaptrack heaptrack-gui` 安装它。Heaptrack 可以找到代码中最大和最频繁分配发生的位置以及其他许多事情。在 Windows 上，你可以使用 [Mtuner](https://github.com/milostosic/MTuner)[^3]，它具有与 Heaptrack 类似的功能。
