## 内存延迟和带宽 {#sec:MemLatBw}

低效的内存访问通常是现代环境中的主要性能瓶颈。因此，处理器从内存子系统获取数据的速度是决定应用程序性能的关键因素。内存性能有两个方面：1) CPU 从内存获取单个字节的速度（延迟），2) 每秒可以获取多少字节（带宽）。这两个方面在各种场景中都很重要；我们稍后将看一些例子。在本节中，我们将专注于度量内存子系统组件的峰值性能。

在 x86 平台上可能有用的工具之一是 Intel 内存延迟检查器（MLC），[^1] 它在 Windows 和 Linux 上免费可用。MLC 可以使用不同的访问模式和负载来度量缓存和内存延迟和带宽。在基于 ARM 的系统上没有类似的工具，但是用户可以从源代码下载和构建内存延迟和带宽基准测试。此类项目的例子有 [lmbench](https://sourceforge.net/projects/lmbench/)[^2]、[bandwidth](https://zsmith.co/bandwidth.php)[^4] 和 [Stream](https://github.com/jeffhammond/STREAM)。[^3]

我们将只关注一部分指标，即空闲读取延迟和读取带宽。让我们从读取延迟开始。空闲意味着在我们进行度量时，系统是空闲的。这将给我们从内存系统组件获取数据所需的最短时间，但是当系统被其他"内存饥渴"应用程序加载时，由于在各个点可能有更多资源排队，访问延迟会增加。MLC 通过执行依赖加载（也称为指针追逐）来度量空闲延迟。一个度量线程分配一个非常大的缓冲区并初始化它，使缓冲区中的每个（64 字节）缓存行包含指向缓冲区中另一个但不相邻的缓存行的指针。通过适当调整缓冲区大小，我们可以确保几乎所有的加载都命中缓存的某个级别或在主内存中。

我的被测系统是一台 Intel Alder Lake 计算机，配备 Core i7-1260P CPU 和 16GB DDR4 @ 2400 MT/s 双通道内存。该处理器具有 4 个 P（性能）超线程核心和 8 个 E（高效）核心。每个 P 核心有 48 KB 的 L1 数据缓存和 1.25 MB 的 L2 缓存。每个 E 核心有 32 KB 的 L1 数据缓存，四个 E 核心形成一个集群，可以访问共享的 2 MB L2 缓存。系统中的所有核心都由 18 MB 的 L3 缓存支持。如果我们使用 10 MB 的缓冲区，我们可以几乎确定对该缓冲区的重复访问会在 L2 中未命中但在 L3 中命中。以下是示例 `mlc` 命令：

```bash
$ sudo ./mlc --idle_latency -c0 -L -b10m
Intel(R) Memory Latency Checker - v3.10
Command line parameters: --idle_latency -c0 -L -b10m
Using buffer size of 10.000MiB
Each iteration took 31.1 base frequency clocks (	12.5	ns)
```

选项 `--idle_latency` 在不加载系统的情况下度量读取延迟。此外，MLC 有 `--loaded_latency` 选项来度量当其他线程产生内存流量时的延迟。选项 `-c0` 将度量线程固定到逻辑 CPU 0，该 CPU 位于 P 核心上。选项 `-L` 启用大页以限制我们度量中的 TLB 影响。选项 `-b10m` 告诉 MLC 使用 10MB 缓冲区，该缓冲区将适合我们系统上的 L3 缓存。

图 @fig:MemoryLatenciesCharts 显示了 L1、L2 和 L3 缓存的读取延迟。图表上有四个不同的区域。左侧从 1 KB 到 48 KB 缓冲区大小的第一个区域对应于 L1 D-cache，它是每个物理核心私有的。我们可以观察到 E 核心的延迟为 0.9 纳秒，P 核心略高为 1.1 纳秒。此外，我们可以使用此图表来确认缓存大小。注意 E 核心延迟在缓冲区大小超过 32 KB 后开始攀升，而 P 核心延迟在 48 KB 之前保持恒定。这确认了 E 核心中的 L1 D-cache 大小为 32 KB，P 核心中为 48 KB。

![Intel Core i7-1260P 上的 L1/L2/L3 缓存读取延迟（越低越好），使用 MLC 工具测量，启用大页。](../../img/terms-and-metrics/MemLatencies.png){#fig:MemoryLatenciesCharts width=100% }

第二个区域显示 L2 缓存延迟，E 核心的延迟几乎是 P 核心的两倍（5.9 纳秒 vs. 3.2 纳秒）。对于 P 核心，延迟在超过 1.25 MB 缓冲区大小后增加，这是预期的。我们预计 E 核心延迟在达到 2 MB 之前保持不变，但是根据我们的测量，它发生得更早。

从 2 MB 到 14 MB 的第三个区域对应于 L3 缓存延迟，两种核心类型大约为 12 纳秒。系统中所有核心共享的 L3 缓存总大小为 18 MB。有趣的是，我们从 15 MB 开始看到一些意外的动态，而不是 18 MB。这很可能与某些访问在 L3 中未命中并需要从主内存获取有关。

我没有显示图表中对应于内存延迟的部分，该部分在超过 18MB 边界后开始。延迟开始急剧攀升，E 核心在 24 MB 处趋于平稳，P 核心在 64 MB 处趋于平稳。使用更大的缓冲区大小，例如 500 MB，E 核心访问延迟为 45 纳秒，P 核心为 90 纳秒。这度量的是内存延迟，因为几乎没有加载命中 L3 缓存。

使用类似的技术，我们可以度量内存层次结构各种组件的带宽。对于度量带宽，MLC 执行加载请求，其结果不被任何后续指令使用。这允许 MLC 生成最大可能的带宽。MLC 在每个配置的逻辑处理器上生成一个软件线程。每个线程访问的地址是独立的，线程之间没有数据共享。与延迟实验一样，线程使用的缓冲区大小决定了 MLC 是度量 L1/L2/L3 缓存带宽还是内存带宽。

```bash
$ sudo ./mlc --max_bandwidth -k0-15 -Y -L -u -b18m
Measuring Maximum Memory Bandwidths for the system
Bandwidths are in MB/sec (1 MB/sec = 1,000,000 Bytes/sec)
Using all the threads from each core if Hyper-threading is enabled
Using traffic with the following read-write ratios
ALL Reads        :      349670.42
```
