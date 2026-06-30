## 任务调度

随着混合处理器的出现，任务调度变得非常具有挑战性。例如，最新的 Intel Meteor Lake 芯片有三种类型的核心；都具有不同的性能特征。正如你将在本节中看到的，通过次优地调度任务，很容易降低多线程应用程序的性能。实现通用任务调度策略很棘手，因为它在很大程度上取决于运行任务的性质。以下是一些示例：

* 计算密集型轻线程工作负载（例如数据压缩）必须仅在 P 核心上服务。
* 后台任务（例如视频通话）可以在 E 核心上运行以节省功耗。
* 对于需要高响应性的突发应用程序（例如生产力软件），系统应仅使用 P 核心。
* 具有持续性能需求的多线程程序（例如视频渲染）应同时利用 P 和 E 核心。

在大多数情况下，现代操作系统中的任务调度器会处理这些以及许多其他边缘情况。例如，Intel 的 Thread Director 帮助实时监控和分析性能数据，以将正确的应用程序线程无缝放置在正确的核心上。我在这里的总体建议是让操作系统完成其工作，不要过度限制它。操作系统知道如何调度任务以最小化争用、最大化缓存中的数据重用，并最终最大化性能。如果你正在开发旨在不同硬件配置上运行的跨平台软件，这将发挥重要作用。

下面我展示了非对称系统中任务调度的一些典型陷阱。我使用了与前一个案例研究中相同的系统：第 12 代 Alder Lake Intel&reg; Core&trade; i7-1260P CPU，它有四个 P 核心和八个 E 核心。为简单起见，我只启用了两个 P 核心和两个 E 核心；其余核心被暂时禁用。我还禁用了两个活动 P 核心上的 SMT 兄弟线程。我编写了一个简单的 OpenMP 应用程序，其中每个工作线程在大型数组的每个 32 位整数元素上执行几个位操作。工作线程完成处理后，它遇到一个屏障并被迫等待其他线程完成其部分。之后，主数组清理并重复处理。该程序使用 GCC 13.2 和 `-O3 -march=core-avx2` 编译，这启用了向量化。

图 @fig:OmpScheduling 显示了三种策略，它们突出了我在实践中经常看到的常见问题。这些屏幕截图是使用 Intel VTune 捕获的。时间线上的条形图表示 CPU 时间，即线程正在运行的时间段。对于每个软件线程，有一个或两个对应的 CPU 核心。使用此视图，我们可以看到每个线程在任何给定时刻在哪个核心上运行。

\begin{figure}[htbp]
\centering

\subfloat[静态分区，将线程固定到核心：
\passthrough{\lstinline!\#pragma omp for schedule(static)!} 配合
\passthrough{\lstinline!OMP\_PROC\_BIND=true!}。]{\includegraphics[width=0.8\textwidth,height=\textheight]{../../img/mt-perf/OmpAffinity.png}\label{fig:OmpAffinity}}

\subfloat[静态分区，无线程亲和性：
\passthrough{\lstinline!\#pragma omp for schedule(static)!}。]{\includegraphics[width=0.8\textwidth,height=\textheight]{../../img/mt-perf/OmpStatic.png}\label{fig:OmpStatic}}

\subfloat[动态分区，16 个块：
\passthrough{\lstinline!\#pragma omp for schedule(dynamic, N/16)!}。]{\includegraphics[width=0.8\textwidth,height=\textheight]{../../img/mt-perf/OmpDynamic.png}\label{fig:OmpDynamic}}

\caption{典型的任务调度陷阱：核心亲和性阻止线程
迁移，大粒度分区作业无法最大化
CPU 利用率。}

\label{fig:OmpScheduling}

\end{figure}

我们的第一个示例使用静态分区，它将大型数组的处理分成四个相等的块（因为我启用了四个核心）。对于每个块，OpenMP 运行时生成一个新线程。此外，我使用了 `OMP_PROC_BIND=true`，它指示 OpenMP 运行时将生成的线程固定到 CPU 核心。图 @fig:OmpAffinity 演示了效果：P 核心在处理 SIMD 指令方面比 E 核心好得多，它们完成工作快两倍（参见*线程 1*和*线程 2*）。但是，线程亲和性不允许*线程 3*和*线程 4*迁移到等待屏障的 P 核心。这导致高延迟，其速度受 E 核心限制。

我的建议是避免将线程固定到核心。在工作不平衡的情况下，固定可能会限制工作窃取，将长执行尾部留给 E 核心。在 macOS 上，无法将线程固定到核心，因为操作系统不提供相应的 API。
