## 减少 ITLB 未命中 {#sec:FeTLB}

调优前端效率的另一个重要领域是内存地址的虚拟到物理地址转换。这些转换主要由 TLB（参见 [@sec:TLBs]）提供服务，TLB 在专用条目中缓存最近使用的内存页转换。当 TLB 无法服务转换请求时，会进行耗时的内核页表遍历，以计算每个引用虚拟地址的正确物理地址。每当你在 TMA 摘要中看到高百分比的 ITLB 开销时，本节中的建议可能会派上用场。

通常，相对较小的应用程序不容易受到 ITLB 未命中的影响。例如，Golden Cove 微架构可以在其 ITLB 中覆盖多达 1MB 的内存空间。如果你的应用程序的机器代码适合 1MB，你应该不会受到 ITLB 未命中的影响。当应用程序的频繁执行部分分散在内存中时，问题开始出现。当许多函数开始频繁相互调用时，它们开始争夺 ITLB 中的条目。其中一个例子是 Clang 编译器，在撰写本文时，其代码段约为 60MB。在配备主流 Intel Coffee Lake 处理器的笔记本电脑上运行的 ITLB 开销约为 7%，这意味着 7% 的周期用于处理 ITLB 未命中：执行需要的页面遍历和填充 TLB 条目。

另一组经常受益于使用大页的大型内存应用程序包括关系数据库（例如 MySQL、PostgreSQL、Oracle）、托管运行时（例如 JavaScript V8、Java JVM）、云服务（例如 Web 搜索）、Web 工具（例如 node.js）。

减少 ITLB 压力的总体思想是将应用程序的性能关键代码部分映射到 2MB（大）页上。通常，为了简单起见，应用程序的整个代码段会被重新映射。实现此转换的关键要求是代码段在 2MB 边界上对齐。在 Linux 上，这可以通过两种不同的方式实现：使用额外的链接器选项重新链接二进制文件，或在运行时重新映射代码段。两种选项都在 Easyperf[^1] 博客中进行了展示。据我所知，在 Windows 上不可能，所以我将只展示如何在 Linux 上执行此操作。

第一个选项可以通过使用以下选项链接二进制文件来实现：`-Wl,-zcommon-page-size=2097152` `-Wl,-zmax-page-size=2097152`。这些选项指示链接器将代码段放置在 2MB 边界处，以便在启动时由加载器放置在 2MB 页上。这种放置的缺点是链接器将被迫插入多达 2MB 的填充（浪费）字节，使二进制文件更加膨胀。在 Clang 编译器的示例中，它将二进制文件的大小从 111 MB 增加到 114 MB。重新链接二进制文件后，我们在 ELF 二进制文件头中设置一个特殊位，该位决定文本段是否应默认使用大页支持。最简单的方法是使用 [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO)[^12] 包中的 `hugeedit` 或 `hugectl` 工具。例如：

```bash
# 在 ELF 二进制文件头中永久设置特殊位。
$ hugeedit --text /path/to/clang++
# 代码段将默认使用大页加载。
$ /path/to/clang++ a.cpp

# 在运行时覆盖默认行为。
$ hugectl --text /path/to/clang++ a.cpp
```

第二个选项是在运行时重新映射代码段。此选项不需要代码段与 2MB 边界对齐，因此无需重新编译应用程序即可工作。当你无法访问源代码时，这特别有用。此方法背后的思想是在程序启动时分配大页并将代码段转移到那里。此方法的参考实现在 [iodlr](https://github.com/intel/iodlr)[^2] 库中实现。一个选项是从你的 `main` 函数调用该功能。另一个更简单的选项是构建动态库并在命令行中预加载它：

```bash
$ LD_PRELOAD=/usr/lib64/liblppreload.so clang++ a.cpp
```

虽然第一种方法仅适用于显式大页，但第二种使用 `iodlr` 的方法同时适用于显式和透明大页。有关如何在 Windows 和 Linux 上启用大页的说明，请参见附录 B。

将代码段映射到大页可以将 ITLB 未命中减少多达 50% [@IntelBlueprint]，从而为某些应用程序带来高达 10% 的加速。然而，与许多其他功能一样，大页并不适用于所有应用程序。可执行文件只有几 KB 大小的小程序最好使用常规 4KB 页面而不是 2MB 大页；这样可以更有效地使用内存。

除了使用大页之外，标准的 I-cache 性能优化技术也可用于改善 ITLB 性能。即，重排序函数以使热函数更好地共位，通过链接时优化（LTO/IPO）减小热区域的大小，使用配置文件引导优化（PGO）和 BOLT，以及较少激进的内联。

BOLT 提供 `-hugify` 选项，根据配置文件数据自动使用大页处理热代码。使用此选项时，`llvm-bolt` 将注入代码以在运行时将热代码放在 2MB 页上。该实现利用了 Linux 透明大页（THP）。这种方法的好处是只有一小部分代码被映射到大页，并且所需的大页数量被最小化，因此页面碎片减少了。

[^1]: "使用大页处理代码的性能优势" - [https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code](https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code)。
[^2]: iodlr 库，仅限 Linux - [https://github.com/intel/iodlr](https://github.com/intel/iodlr)。
[^12]: libhugetlbfs - [https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO)。
