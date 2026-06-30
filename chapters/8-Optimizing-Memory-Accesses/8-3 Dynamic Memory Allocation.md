## 动态内存分配

开发人员应该意识到动态内存分配有许多相关成本。在堆栈上分配对象是即时完成的，无论对象大小如何：你只需要移动堆栈指针。但是，动态内存分配是一个更复杂的操作。它涉及调用标准库函数，如 `malloc`，这可能会将分配委托给操作系统。避免不必要的动态内存分配是避免这些成本的第一步。这个过程中的简单目标是临时分配，即直接跟随其释放的分配。在 [@sec:HeaptrackCaseStudy] 中，我们展示了如何使用 `heaptrack` 来查找动态内存分配的来源。

你可以通过分配一个大块来摊薄许多小分配的成本。这是 [arena 分配器](https://en.wikipedia.org/wiki/Region-based_memory_management)[^16] 和内存池背后的核心思想。它为手动内存管理提供了更多灵活性。你可以获取操作系统分配的内存区域，并在该区域之上设计自己的分配策略。一个简单的策略可能是将该区域分成两部分：一部分用于热数据，一部分用于冷数据。并提供两种分配方法，它们将使用各自的 arena。将热数据放在一起为更好的缓存利用创造了机会。它也可能改善 TLB 利用率，因为热数据将更紧凑，占用更少的内存页。

动态内存分配的另一个成本出现在应用程序使用多个线程时。当两个线程同时尝试分配内存时，操作系统必须同步它们。在高度并发的应用程序中，线程可能花费大量时间等待公共锁来分配内存。内存释放也是如此。同样，自定义分配器可以帮助避免这个问题，例如，为每个线程使用单独的 arena。

有许多标准动态内存分配例程（`malloc` 和 `free`）的直接替换品，它们更快、更具可扩展性，并且更好地解决碎片问题。一些最流行的内存分配库是 [jemalloc](http://jemalloc.net/)[^17] 和 [tcmalloc](https://github.com/google/tcmalloc)[^18]。一些项目采用 `jemalloc` 和 `tcmalloc` 作为其默认内存分配器，并且它们看到了显著的性能改进。

最后，动态内存分配的一些成本是隐藏的[^20]，不容易度量。在所有主要操作系统中，`malloc` 返回的指针只是一个承诺——操作系统承诺当页面被触及时它将提供所需的内存，但实际的物理页面在虚拟地址被访问之前不会被分配。这称为*请求调页*，它对每个新分配的页面产生一个小页面错误的成本。我们将在 [@sec:AvoidPageFaults] 中讨论如何减轻这种成本。此外，出于安全原因，所有现代操作系统在将页面交给下一个进程之前都会擦除其内容（写零）。操作系统维护一个已清零页面的池，以便为分配做好准备。但是当这个池用完可用的已清零页面时，操作系统必须按需清零页面。这个过程不是超级昂贵，但也不是免费的，可能会增加内存分配调用的延迟。

[^16]: 基于区域的内存管理 - [https://en.wikipedia.org/wiki/Region-based_memory_management](https://en.wikipedia.org/wiki/Region-based_memory_management)
[^17]: jemalloc - [http://jemalloc.net/](http://jemalloc.net/)。
[^18]: tcmalloc - [https://github.com/google/tcmalloc](https://github.com/google/tcmalloc)
[^20]: Bruce Dawson：内存分配的隐藏成本 - [https://randomascii.wordpress.com/2014/12/10/hidden-costs-of-memory-allocation/](https://randomascii.wordpress.com/2014/12/10/hidden-costs-of-memory-allocation/)。
