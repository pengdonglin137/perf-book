## 减少 DTLB 未命中 {#sec:secDTLB}

如 [@sec:TLBs] 中所讨论的，TLB 是一个快速但有限的每核心缓存，用于虚拟到物理地址的内存地址转换。没有它，应用程序的每次内存访问都将需要耗时的内核页表遍历来计算每个引用虚拟地址的正确物理地址。在具有 5 级页表的系统中，它将需要访问至少 5 个不同的内存位置来获取地址转换。在第 [@sec:FeTLB] 节中，我们将讨论如何将大页用于代码。这里我们将看到如何将它们用于数据。

任何对大内存区域进行随机访问的算法都可能遭受 DTLB 未命中。此类应用程序的例子包括大数组中的二分查找、访问大型哈希表和遍历图。使用大页有潜力加速此类应用程序。

在 x86 平台上，默认页面大小为 4KB。考虑一个频繁引用 20MB 内存空间的应用程序。使用 4KB 页面，操作系统需要分配许多小页面。此外，该进程将触及许多 4KB 大小的页面，每个页面将争夺有限数量的 TLB 条目。相比之下，使用 2MB 大页，20MB 的内存可以用仅仅 10 个页面映射，而使用 4KB 页面，你需要 5120 个页面。这意味着使用大页时需要更少的 TLB 条目，这反过来减少了 TLB 未命中的数量。由于 2MB 条目的数量少得多，减少不会按 512 的比例。例如，在 Intel 的 Skylake 核心系列中，L1 DTLB 有 64 个 4KB 页面的条目，只有 32 个 2MB 页面的条目。除了 2MB 大页之外，AMD 和 Intel 的 x86 芯片还支持 1GB 巨页用于数据，但不用于指令。使用 1GB 页面而不是 2MB 页面进一步减少了 TLB 压力。

利用大页通常会导致更少的页面遍历，并且由于表本身更紧凑，在 TLB 未命中事件中遍历内核页表的惩罚会降低。利用大页的性能提升有时可以高达 30%，具体取决于应用程序经历的 TLB 压力量。期望 2 倍加速要求太高，因为 TLB 未命中很少是主要瓶颈。论文 [@Luo2015] 介绍了在 SPEC2006 基准测试套件上使用大页的评估。结果可以总结如下。在套件中的 29 个基准测试中，15 个的加速在 1% 以内，可以作为噪声丢弃。六个基准测试的加速在 1%-4% 范围内。四个基准测试的加速在 4% 到 8% 范围内。两个基准测试的加速为 10%，另外两个基准测试分别获得了 22% 和 27% 的加速。

许多实际应用程序已经利用了大页，例如 KVM、MySQL、PostgreSQL、Java 的 JVM 等。通常，这些软件包提供启用该功能的选项。每当你使用类似的应用程序时，请检查其文档以查看是否可以启用大页。

Windows 和 Linux 都允许应用程序建立大页内存区域。如何在 Windows 和 Linux 上启用大页的说明可以在附录 B 中找到。在 Linux 上，有两种方式在应用程序中使用大页：显式大页和透明大页。Windows 的支持不如 Linux 丰富，将在后面讨论。

### 显式大页

显式大页（EHP）作为系统内存的一部分可用，并作为大页文件系统 `hugetlbfs` 暴露。EHP 应在系统启动时或在应用程序启动之前保留。有关如何操作的说明，请参见附录 B。在启动时保留 EHP 增加了成功分配的可能性，因为内存尚未被严重碎片化。显式预分配的页面驻留在物理内存的保留块中，在内存压力下不能被换出。此外，此内存空间不能用于其他目的，因此用户应小心，只保留他们需要的页面数量。

在 Linux 应用程序中使用 EHP 的最简单方法是使用 `MAP_HUGETLB` 调用 `mmap`，如 [@lst:ExplicitHugepages1] 所示。在此代码中，指针 `ptr` 将指向为 EHP 显式保留的 2MB 内存区域。注意，如果 EHP 没有提前保留，分配可能会失败。在用户代码中使用 EHP 的其他不太流行的方式在附录 B 中提供。此外，开发人员可以编写自己的基于 arena 的分配器来使用 EHP。

Listing: 从未分配的大页映射内存区域。

~~~~ {#lst:ExplicitHugepages1 .cpp}
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};                
...
munmap(ptr, size);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

过去，有一个选项可以使用 [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs)[^1] 库，它覆盖了现有动态链接可执行文件中使用的 `malloc` 调用，以在 EHP 中分配内存。不幸的是，这个项目不再维护了。它不需要用户修改代码或重新链接二进制文件。他们只需在命令行前加上 `LD_PRELOAD=libhugetlbfs.so HUGETLB_MORECORE=yes <your app command line>` 即可使用它。但幸运的是，其他库支持使用 `malloc` 的大页（不是 EHP），我们接下来将讨论。

### 透明大页

Linux 还提供透明大页支持（THP），它有两种操作模式：系统范围和每进程。当 THP 在系统范围启用时，内核自动管理大页，对应用程序透明。当需要大块内存且可以分配时，操作系统内核会尝试将大页分配给任何进程，因此大页不需要手动保留。如果 THP 按进程启用，内核仅将大页分配给归因于 `madvise` 系统调用的单个进程的内存区域。你可以通过以下方式检查 THP 是否在系统中启用：

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

方括号中显示的值是当前设置。如果此值为 `always`（系统范围）或 `madvise`（每进程），则 THP 可用于你的应用程序。每个选项的详细规范可以在 Linux 内核关于 THP 的[文档](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)[^2] 中找到。

当 THP 在系统范围启用时，大页自动用于普通内存分配，无需应用程序的显式请求。要观察大页对其应用程序的影响，用户只需使用 `echo "always" | sudo tee /sys/kernel/mm/transparent_hugepage/enabled` 启用系统范围的 THP。它将自动启动一个名为 `khugepaged` 的守护进程，开始扫描应用程序的内存空间以将常规页面提升为大页。有时内核可能无法将多个常规页面合并为大页，因为它找不到连续的 2MB 内存块。

系统范围 THP 模式适合快速实验以检查大页是否可以提高性能。它自动工作，甚至对不了解 THP 的应用程序也是如此，因此开发人员不必更改代码即可看到大页对其应用程序的好处。当大页在系统范围启用时，应用程序可能最终分配比所需更多的内存资源。这就是系统范围模式默认禁用的原因。完成实验后不要忘记禁用系统范围的 THP，因为它可能会损害整体系统性能。

使用 `madvise`（每进程）选项，THP 仅在通过带有 `MADV_HUGEPAGE` 标志的 `madvise` 系统调用归因的内存区域内启用。如 [@lst:TransparentHugepages1] 所示，指针 `ptr` 将指向内核动态分配的 2MB 匿名（透明）内存区域。如果内核找不到连续的 2MB 内存块，`mmap` 调用将失败。

Listing: 将内存区域映射到透明大页。

~~~~ {#lst:TransparentHugepages1 .cpp}
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE | PROT_EXEC,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};
madvise(ptr, size, MADV_HUGEPAGE);
// use the memory region `ptr`
munmap(ptr, size);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

开发人员可以基于 [@lst:TransparentHugepages1] 中的代码构建自定义 THP 分配器。但也可以在其应用程序进行的 `malloc` 调用中使用 THP。许多内存分配库通过覆盖 `libc` 的 `malloc` 实现来提供此功能。以下是使用 `jemalloc` 的示例，它是最流行的选项之一。如果你可以访问应用程序的源代码，可以使用额外的 `-ljemalloc` 选项重新链接二进制文件。这将使你的应用程序动态链接到 `jemalloc` 库，它将处理所有 `malloc` 调用。然后使用以下选项为堆分配启用 THP：

```bash
$ MALLOC_CONF="thp:always" <your app command line>
```

如果你无法访问源代码，仍然可以通过预加载动态库来使用 `jemalloc`：

```bash
$ LD_PRELOAD=/usr/local/libjemalloc.so.2 MALLOC_CONF="thp:always" <your app command line>
```

Windows 仅提供通过 `VirtualAlloc` 系统调用以类似于 Linux THP 每进程模式的方式使用大页。详情请参见附录 B。

### 显式大页 vs. 透明大页

Linux 用户可以使用三种不同的模式来使用大页：

* 显式大页（Explicit Huge Pages）
* 系统范围透明大页（System-wide Transparent Huge Pages）
* 每进程透明大页（Per-process Transparent Huge Pages）

让我们比较这些选项。首先，EHP 在虚拟内存中预先保留，THP 则不是。这使得使用 EHP 的软件包更难分发，因为它们依赖于机器管理员进行的特定配置设置。此外，EHP 静态存在于内存中，占用宝贵的 DRAM，即使它们未被使用。

系统范围透明大页非常适合快速实验。无需修改用户代码即可测试大页在应用程序中的好处。然而，向客户交付软件包并要求他们启用系统范围 THP 是不明智的，因为它可能会对系统上其他运行的程序产生负面影响。通常，开发人员识别代码中可以从大页受益的分配，并在这些地方使用 `madvise` 提示（每进程模式）。

每进程 THP 没有上述两种缺点，但它有另一个缺点。前面我们讨论了 THP 的内核分配对用户是透明的。分配过程可能涉及负责在虚拟内存中腾出空间的多个内核进程，这可能包括将内存换出到磁盘、碎片整理或提升页面。透明大页的后台维护会产生来自内核的非确定性延迟开销，因为它管理不可避免的碎片和换页问题。EHP 不受内存碎片的影响且不能被换出到磁盘，因此它们产生的延迟开销要小得多。

总而言之，THP 更容易使用，但会产生更大的分配延迟开销。这就是为什么 THP 在高频交易和其他超低延迟行业中不流行的原因；它们更倾向于使用 EHP。另一方面，虚拟机提供商和数据库倾向于使用每进程 THP，因为要求额外的系统配置可能成为其用户的负担。

[^1]: libhugetlbfs - [https://github.com/libhugetlbfs/libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs).
[^2]: Linux 内核 THP 文档 - [https://www.kernel.org/doc/Documentation/vm/transhuge.txt](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)
