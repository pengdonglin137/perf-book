## 高级分析工具

已经开发了许多工具来解决传统分析器无法提供足够可见性的特定用例。在本节中，我们将介绍 Coz 和 eBPF 工具。我们鼓励你对这些和其他工具进行进一步研究。

### Coz {#sec:COZ}

在 [@sec:secAmdahl] 中，我们定义了识别影响多线程程序整体性能的代码部分的挑战。由于各种原因，优化多线程程序的一部分可能并不总是产生可见的结果。传统的基于采样的分析器仅显示大部分时间花费的代码位置。然而，这不一定对应于程序员应集中优化工作的地方。

[Coz](https://github.com/plasma-umass/coz)[^16] 是一个解决此问题的分析器。它使用一种称为*因果分析*的新技术，通过在应用程序运行时通过虚拟加速代码段来预测某些优化的总体效果来进行实验。它通过插入暂停来减慢所有其他同时运行的代码来实现这些"虚拟加速"。此外，Coz 量化了优化的潜在影响。[@CozPaper]

图 @fig:CozProfile 显示了将 Coz 分析器应用于 [C-Ray](https://github.com/jtsiomb/c-ray)[^15] 基准测试的示例。根据图表，如果我们将 `c-ray-mt.c` 中第 540 行的性能提高 20%，Coz 预计 C-Ray 基准测试的整体应用程序性能将相应增加约 17%。一旦我们在该行上达到约 45% 的改进，根据 Coz 的估计，对应用程序的影响将趋于平稳。有关此示例的更多详细信息，请参见 Easyperf 博客上的[文章](https://easyperf.net/blog/2020/02/26/coz-vs-sampling-profilers)[^17]。

![C-Ray 基准测试的 Coz 配置文件。](../../img/mt-perf/CozProfile.png){#fig:CozProfile width=60%}

[^15]: C-Ray 基准测试 - [https://github.com/jtsiomb/c-ray](https://github.com/jtsiomb/c-ray)。
[^16]: COZ 源代码 - [https://github.com/plasma-umass/coz](https://github.com/plasma-umass/coz)。
[^17]: 博客文章"COZ vs 采样分析器" - [https://easyperf.net/blog/2020/02/26/coz-vs-sampling-profilers](https://easyperf.net/blog/2020/02/26/coz-vs-sampling-profilers)。

### eBPF 和 GAPP {#sec:secEBPF}

Linux 支持各种线程同步原语：互斥锁、信号量、条件变量等。内核通过 `futex` 系统调用支持这些线程原语。因此，通过在内核中跟踪 `futex` 系统调用的执行，同时从相关线程收集有用的元数据，可以更容易地识别争用瓶颈。Linux 提供了使这成为可能的内核跟踪和分析工具，其中最强大的是 [Extended Berkeley Packet Filter](https://prototype-kernel.readthedocs.io/en/latest/bpf/)[^22]（eBPF）。

eBPF 基于在内核中运行的沙箱虚拟机，允许在内核内安全高效地执行用户定义的程序。用户定义的程序可以用 C 编写，并由 [BCC 编译器](https://github.com/iovisor/bcc)[^23] 编译为 BPF 字节码，准备加载到内核虚拟机中。这些 BPF 程序可以被编写为在某些内核事件执行时启动，并通过各种方式将原始或处理后的数据传回用户空间。

开源社区提供了许多通用的 eBPF 程序。其中一个工具是通用自动并行分析器（GAPP），[^25] 它有助于跟踪多线程争用问题。GAPP 使用 eBPF 通过识别序列化瓶颈的关键性来跟踪多线程应用程序的争用开销，并收集被阻塞线程和导致阻塞的线程的堆栈跟踪。GAPP 最好的地方是它不需要代码更改、昂贵的检测或重新编译。GAPP 分析器的创建者能够确认已知瓶颈，并在 [Parsec 3.0 Benchmark Suite](https://parsec.cs.princeton.edu/index.htm)[^24] 和一些大型开源项目中发现新的、以前未报告的瓶颈。[@GAPP]

作为结束语，我想再次强调优化多线程应用程序的重要性。从我们在本书中讨论的所有内容来看，本章的建议可能会带来最显著的性能改进。在多线程应用程序中，魔鬼在于细节。微妙的同步问题或数据共享中的小低效可能导致显著的性能降级。展望未来，多核处理器和并行工作负载的趋势只会加速。多线程优化的复杂性会增长，但掌握它的机会也会增长。

[^22]: eBPF 文档 - [https://prototype-kernel.readthedocs.io/en/latest/bpf/](https://prototype-kernel.readthedocs.io/en/latest/bpf/)
[^23]: BCC 编译器 - [https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)
[^24]: Parsec 3.0 基准测试套件 - [https://parsec.cs.princeton.edu/index.htm](https://parsec.cs.princeton.edu/index.htm)
[^25]: GAPP - [https://github.com/RN-dev-repo/GAPP/](https://github.com/RN-dev-repo/GAPP/)
