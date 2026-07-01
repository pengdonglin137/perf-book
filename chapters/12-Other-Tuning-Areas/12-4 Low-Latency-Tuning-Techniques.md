## 低延迟调优技术 {#sec:LowLatency}

到目前为止，我们已经讨论了旨在提高应用程序整体性能的各种软件优化。在本节中，我们将讨论在低延迟系统中使用的其他调优技术，例如实时处理和高频交易（HFT）。在这种环境中，主要优化目标是使程序的特定部分尽可能快地运行。当你在 HFT 行业工作时，每一微秒和纳秒都很重要，因为它直接影响利润。通常，低延迟部分实现实时或 HFT 系统的关键循环，例如移动机械臂或向交易所发送订单。优化关键路径的延迟有时是以牺牲程序的其他部分为代价完成的。一些技术甚至牺牲了系统的整体吞吐量。

当开发人员为延迟进行优化时，他们会避免在热路径上需要支付的任何不必要的成本。这通常涉及系统调用、内存分配、I/O 以及其他具有非确定性延迟的任何内容。为了达到尽可能低的延迟，热路径需要所有资源立即就绪和可用。

一个相对简单的技术是预计算你在热路径上执行的一些操作。这会带来使用更多内存的成本，这些内存将对系统中的其他进程不可用，但它可能会在关键路径上为你节省一些宝贵的周期。然而，请记住，有时计算比从内存中获取结果更快。

由于这是一本关于底层 CPU 性能的书，我们将跳过讨论类似于我们刚才提到的高级技术。相反，我们将讨论如何避免页面错误、缓存未命中、TLB 射击和关键路径上的核心节流。

### 避免小页面错误 {#sec:AvoidPageFaults}

虽然该术语包含"小"这个词，但小页面错误对运行时延迟的影响一点也不小。回想一下，当用户代码分配内存时，操作系统只承诺提供一个页面，但它不会立即通过给我们一个清零的物理页面来执行承诺。相反，它将等待用户代码第一次访问它，然后操作系统才履行其职责。对新分配页面的第一次写入会触发小页面错误，这是由操作系统处理的硬件中断。小错误的延迟影响从不到一微秒到几微秒不等，特别是如果你使用具有 5 级页表而不是 4 级页表的 Linux 内核。

如何检测应用程序中的运行时小页面错误？一个简单的方法是使用 `top` 工具（添加 `-H` 选项以获取线程级视图）。将 `vMn` 字段添加到显示列的默认选择中，以查看每次显示刷新间隔内发生的小页面错误数量。[@lst:DumpTopWithMinorFaults] 显示了在编译大型 C++ 项目时 `top` 命令的转储，其中包含前 10 个进程。额外的 `vMn` 列显示在最后 3 秒内发生的小页面错误数量。

Listing: 编译大型 C++ 项目时带有额外 vMn 字段的 Linux top 命令转储。

~~~~ {#lst:DumpTopWithMinorFaults .cpp}
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  vMn
341763 dendiba+  20   0  303332 165396  83200 R  99.3   1.0   0:05.09 c++      13k
341705 dendiba+  20   0  285768 153872  87808 R  99.0   1.0   0:07.18 c++       5k
341719 dendiba+  20   0  313476 176236  83328 R  94.7   1.1   0:06.49 c++       8k
341709 dendiba+  20   0  301088 162800  82944 R  93.4   1.0   0:06.46 c++       2k
341779 dendiba+  20   0  286468 152376  87424 R  92.4   1.0   0:03.08 c++      26k
341769 dendiba+  20   0  293260 155068  83072 R  91.7   1.0   0:03.90 c++      22k
341749 dendiba+  20   0  360664 214328  75904 R  88.1   1.3   0:05.14 c++      18k
341765 dendiba+  20   0  351036 205268  76288 R  87.1   1.3   0:04.75 c++      18k
341771 dendiba+  20   0  341148 194668  75776 R  86.4   1.2   0:03.43 c++      20k
341776 dendiba+  20   0  286496 147460  82432 R  76.2   0.9   0:02.64 c++      25k
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

检测运行时小页面错误的另一种方法是使用 `perf stat -e page-faults` 附加到正在运行的进程。

在 HFT 世界中，任何超过 `0` 的都是问题。但对于其他业务领域中的低延迟应用程序，每秒 100-1000 次错误的恒定发生应该促使进一步调查。调查运行时小页面错误的根本原因可以简单到启动 `perf record -e page-faults`，然后 `perf report` 来定位有问题的源代码行。

为了避免运行时的页面错误惩罚，你应该在启动时为应用程序预取所有内存。一个简单的示例可能如下所示：

```cpp
char *mem = malloc(size);
int pageSize = sysconf(_SC_PAGESIZE)
for (int i = 0; i < size; i += pageSize)
  mem[i] = 0;
```

首先，此示例代码照常在堆上分配 `size` 大小的内存。然而，之后立即步进并写入新分配内存的每一页的第一个字节，以确保每一页都被引入 RAM。此方法有助于避免未来访问时由小页面错误引起的运行时延迟。

看看 [@lst:LockPagesAndNoRelease]，其中包含一种更全面的方法来调整 glibc 分配器，结合 `mlock/mlockall` 系统调用（取自 "Real-time Linux Wiki" [^1]）。

Listing: 调整 glibc 分配器以将页面锁定在 RAM 中并防止释放给操作系统。

~~~~ {#lst:LockPagesAndNoRelease .cpp}
#include <malloc.h>
#include <sys/mman.h>

mallopt(M_MMAP_MAX, 0);
mallopt(M_TRIM_THRESHOLD, -1);
mallopt(M_ARENA_MAX, 1);

mlockall(MCL_CURRENT | MCL_FUTURE);

char *mem = malloc(size);
for (int i = 0; i < size; i += sysconf(_SC_PAGESIZE))
    mem[i] = 0;
//...
free(mem);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[@lst:LockPagesAndNoRelease] 中的代码调整了三个 glibc malloc 设置：`M_MMAP_MAX`、`M_TRIM_THRESHOLD` 和 `M_ARENA_MAX`。

- 将 `M_MMAP_MAX` 设置为 `0` 禁用大型分配的底层 `mmap` 系统调用——这是必要的，因为 `mlockall` 可以被库使用 `munmap` 撤销，当它尝试将 `mmap` 映射的段释放回操作系统时，会破坏我们的努力。
- 将 `M_TRIM_THRESHOLD` 设置为 `-1` 阻止 glibc 在调用 `free` 后将内存返回给操作系统。如前所述，此选项对 `mmap` 映射的段没有影响。
- 最后，将 `M_ARENA_MAX` 设置为 `1` 阻止 glibc 通过 `mmap` 分配多个 arena 以适应多核。请记住，后者会阻碍 glibc 分配器的多线程可扩展性功能。

综合起来，这些设置强制 glibc 进行堆分配，在应用程序结束之前不会将内存释放回操作系统。因此，在上面代码中最后一次调用 `free(mem)` 后，堆将保持相同大小。任何后续的运行时 `malloc` 或 `new` 调用将简单地重用此预分配/预取页堆区域中的空间（如果它在初始化时足够大）。

更重要的是，由于之前的 `mlockall` 调用，在 `for` 循环中预取页的所有堆内存将持久存在于 RAM 中——选项 `MCL_CURRENT` 锁定当前映射的所有页面，而 `MCL_FUTURE` 锁定将来将映射的所有页面。以这种方式使用 `mlockall` 的额外好处是，此进程生成的任何线程的堆栈也将被预取页和锁定。为了更精细地控制页面锁定，开发人员应使用 `mlock` 系统调用，它可以选择哪些页面应持久存在于 RAM 中。此技术的缺点是它减少了系统上其他进程可用的内存量。

Windows 应用程序的开发人员应查看以下 API：使用 `VirtualLock` 锁定页面，使用带有 `MEM_DECOMMIT` 而非 `MEM_RELEASE` 标志的 `VirtualFree` 避免立即释放内存。

这些只是防止运行时小错误的两种示例方法。其中一些或全部技术可能已经集成到内存分配库中，如 jemalloc、tcmalloc 或 mimalloc。请检查你的库文档以了解可用内容。

### 缓存预热 {#sec:CacheWarm}

在某些应用程序中，对延迟最敏感的代码部分是最少执行的。此类应用程序的一个示例可能是 HFT 应用程序，它持续从证券交易所读取市场数据信号，一旦检测到有利的市场信号，就向交易所发送订单。在上述工作负载中，读取市场数据涉及的代码路径最常执行，而执行订单的代码路径很少执行。

由于市场中的其他参与者可能捕捉到相同的市场信号，策略的成功很大程度上取决于我们的反应速度，换句话说，我们向交易所发送订单的速度。当我们希望订单尽快到达交易所以利用市场数据中检测到的有利信号时，我们最不希望的是在决定起飞的时刻遇到障碍。

当某个代码路径一段时间未被执行时，其指令和相关数据可能被从 I-cache 和 D-cache 中驱逐。然后，就在我们需要运行那段关键的很少执行的代码时，我们遭遇 I-cache 和 D-cache 未命中惩罚，这可能导致我们在竞争中失败。这就是*缓存预热*技术有用的地方。

缓存预热涉及定期执行延迟敏感的代码以将其保持在缓存中，同时确保它不会完全执行任何不需要的操作。执行延迟敏感的代码还通过将延迟敏感数据引入 D-cache 来"预热"D-cache。此技术经常用于 HFT 应用程序。虽然我不会提供示例实现，但你可以在 [CppCon 2018 lightning talk](https://www.youtube.com/watch?v=XzRxikGgaHI)[^4] 中了解它。

### 避免 TLB 射击

我们从前面的章节中了解到，TLB 是一个快速但有限的每核缓存，用于虚拟到物理内存地址转换，减少了耗时的内核页表遍历的需要。与基于 MESI 的协议和每核 CPU 缓存（即 L1、L2 和 LLC）的情况不同，硬件本身不维护核间 TLB 一致性。因此，此任务必须由操作系统在软件中执行。

在多线程应用程序中，进程线程共享虚拟地址空间。因此，内核必须在执行参与线程的核心的 TLB 之间传达对该共享地址空间的特定类型的更新。例如，常用的系统调用如 `munmap`（可以从 glibc 分配器使用中禁用，参见 [@sec:AvoidPageFaults]）、`mprotect` 和 `madvise` 可能使 TLB 条目无效。这些更新必须在进程的组成线程之间传达。内核使用特定类型的处理器间中断（IPI）执行此工作，称为*TLB 射击*，在 x86 平台上通过 `INVLPG` 汇编指令实现。TLB 射击是使用多线程应用程序实现低延迟时最容易被忽视的陷阱之一。

尽管开发人员可能避免在代码中显式使用这些系统调用，TLB 射击仍可能从外部来源爆发——例如，内存分配共享库或操作系统工具。这种类型的 IPI 不仅会中断运行时应用程序性能，而且其影响程度随着涉及的线程数量增加而增长，因为中断是在软件中传递的。

如何在多线程应用程序中检测 TLB 射击？一个简单的方法是检查 `/proc/interrupts` 中的 TLB 行。检测运行时持续 TLB 中断的一个有用方法是在查看此文件时使用 `watch` 命令。例如，你可以运行 `watch -n5 -d 'grep TLB /proc/interrupts'`，其中 `-n 5` 选项每 5 秒刷新一次视图，而 `-d` 突出显示每次刷新输出之间的差异。

[@lst:ProcInterrupts] 显示了 `/proc/interrupts` 的转储，在运行延迟关键线程的 `CPU2` 处理器上有大量 TLB 射击。注意与其他核心之间的数量级差异。在该场景中，此行为的罪魁祸首是一个名为 Automatic NUMA Balancing 的 Linux 内核功能，可以通过 `sysctl -w numa_balancing=0` 轻松禁用。

Listing: 显示 CPU2 上大量 TLB 射击的 /proc/interrupts 转储

~~~~ {#lst:ProcInterrupts .cpp}
           CPU0       CPU1       CPU2       CPU3       
...
NMI:          0          0          0          0   Non-maskable interrupts
LOC:     552219    1010298    2272333    3179890   Local timer interrupts
SPU:          0          0          0          0   Spurious interrupts
...
IWI:          0          0          0          0   IRQ work interrupts
RTR:          7          0          0          0   APIC ICR read retries
RES:      18708       9550        771        528   Rescheduling interrupts
CAL:        711        934       1312       1261   Function call interrupts
TLB:       4493       6108      73789       5014   TLB shootdowns
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

但这不是 TLB 射击的唯一来源。其他来源包括透明大页、内存压缩、页面迁移和页面缓存写回。垃圾收集器也可以发起 TLB 射击。这些功能在履行职责的过程中要么重新定位页面，要么更改页面权限，这需要页表更新，从而需要 TLB 射击。

防止 TLB 射击需要限制对共享进程地址空间的更新次数。在源代码级别，你应该避免运行时执行上述系统调用列表，即 `munmap`、`mprotect` 和 `madvise`。在操作系统级别，禁用作为其功能结果而诱发 TLB 射击的内核功能，如透明大页和 Automatic NUMA Balancing。有关 TLB 射击的更细致讨论，以及其检测和预防，请阅读 JabPerf 博客上的相关文章[^5]。

### 防止意外核心节流

C/C++ 编译器是工程的奇迹。然而，它们有时会产生令人惊讶的结果，可能导致你徒劳无功。一个真实的例子是编译器优化器发出你从未打算使用的重型 AVX512 指令。虽然在较新的芯片上问题较小，但许多旧代 CPU（在本地和云中仍在积极使用）在执行重型 AVX512 指令时表现出严重的核心节流/降频。如果你的编译器在你不知情或未同意的情况下产生这些指令，你可能在应用程序运行时遇到无法解释的延迟异常。

对于此特定情况，如果不希望使用重型 AVX512 指令，请在编译标志中包含 `-mprefer-vector-width=###` 以将最高宽度指令集固定为 128 或 256。同样，如果你的整个服务器群都运行在最新芯片上，那么这就不那么令人担忧了，因为 AVX 指令集的节流影响如今可以忽略不计。

[^1]: The Linux Foundation Wiki: Memory for Real-time Applications - [https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory)
[^4]: 缓存预热技术 - [https://www.youtube.com/watch?v=XzRxikGgaHI](https://www.youtube.com/watch?v=XzRxikGgaHI)
[^5]: JabPerf 博客：TLB 射击 - [https://www.jabperf.com/how-to-deter-or-disarm-tlb-shootdowns/](https://www.jabperf.com/how-to-deter-or-disarm-tlb-shootdowns/)
