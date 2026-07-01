## 缓存一致性 {#sec:TrueFalseSharing}

多处理器系统包含确保每个核心在共享内存使用期间数据一致性的方法，每个核心包含自己的独立缓存实体。如果没有这样的协议，如果 CPU `A` 和 `B` 都将内存位置 `L` 读入它们各自的缓存，然后 CPU `B` 随后修改其缓存值 `L`，那么 CPU 将具有相同内存位置 `L` 的不一致值。缓存一致性协议确保对缓存条目的任何更新都会在相同位置的任何其他缓存条目中被忠实地更新或无效化。

### 缓存一致性协议

最著名的缓存一致性协议之一是 MESI（**M**odified **E**xclusive **S**hared **I**nvalid），它用于支持现代 CPU 中使用的写回缓存。它的首字母缩写表示缓存行可以被标记的四种状态（参见@fig:MESI）：

* **已修改**：缓存行仅存在于当前缓存中，并且已从 RAM 中的值修改
* **独占**：缓存行仅存在于当前缓存中，并且与 RAM 中的值匹配
* **共享**：缓存行存在于此处和其他缓存行中，并且与 RAM 中的值匹配
* **无效**：缓存行未使用（即不包含任何 RAM 位置）

![MESI 状态图。*© 来源：华盛顿大学 via courses.cs.washington.edu.*](../../img/mt-perf/MESI_Cache_Diagram.jpg){#fig:MESI width=60%}

从内存获取时，每个缓存行都有一个状态编码到其标签中。然后缓存行状态从一个状态转换到另一个状态。[^25] 实际上，CPU 供应商通常实现稍作改进的 MESI 变体。例如，Intel 使用 [MESIF](https://en.wikipedia.org/wiki/MESIF_protocol)，[^26] 它添加了转发（F）状态，而 AMD 使用 [MOESI](https://en.wikipedia.org/wiki/MOESI_protocol)，[^27] 它添加了拥有（O）状态。然而，这些协议仍然保持基本 MESI 协议的本质。

缺乏缓存一致性可能导致顺序不一致的程序。这个问题可以通过让_snoop_缓存监视所有内存事务并相互协作以保持内存一致性来缓解。不幸的是，这需要付出代价，因为一个核心所做的修改会使另一个核心缓存中的相应缓存行无效。这会导致内存停顿并浪费系统带宽。与只能为应用程序性能设定上限的序列化和锁定问题相比，一致性问题可能导致 USL 在 [@sec:secAmdahl] 中归因的倒退效应。两种广泛知晓的一致性问题是*真共享*和*假共享*，我们接下来将探讨。

### 真共享 {#sec:secTrueSharing}

当两个不同的核心访问同一个变量时，就会发生真共享（参见 [@lst:TrueSharing]）。

Listing: 真共享示例。

~~~~ {#lst:TrueSharing .cpp}
unsigned int sum; // 在所有线程之间共享
{ // 由线程 A 执行的代码      │ { // 由线程 B 执行的代码
  for (int i = 0; i < N; i++)       │   for (int i = 0; i < N; i++)
    sum += a[i];                    │     sum += b[i];
}                                   │ }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

首先，除了真共享之外，我们还有一个更大的问题。我们实际上有一个*数据竞争*，这有时可能很难检测。注意，我们没有适当的同步机制，这可能导致不可预测或不正确的程序行为，因为对共享数据的操作可能相互干扰。幸运的是，有一些工具可以帮助识别此类问题。来自 Clang 的 [Thread sanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html)[^30] 和 [helgrind](https://www.valgrind.org/docs/manual/hg-manual.html)[^31] 就是此类工具。为了防止 [@lst:TrueSharing] 中的数据竞争，你应该将 `sum` 变量声明为 `std::atomic<unsigned int> sum`。

使用 C++ 原子类型可以帮助解决真共享发生时的数据竞争。但是，它有效地序列化了对原子变量的访问，这可能会损害性能。解决我们真共享问题的更好方法是使用线程本地存储（TLS）。TLS 是给定多线程进程中每个线程可以分配内存来存储线程特定数据的方法。通过这样做，线程修改它们的本地副本，而不是争夺全局可用的内存位置。[@lst:TrueSharing] 中的示例可以通过使用 TLS 类说明符声明 `sum` 来修复：`thread_local unsigned int sum`（自 C++11 起）。然后主线程应该合并每个工作线程的所有本地副本的结果。

### 假共享 {#sec:secFalseSharing}

如果不小心，你可能会尝试如 [@lst:FalseSharing] 所示解决真共享问题。此解决方案引入了另一个问题：*假共享*。当两个不同的核心修改恰好位于同一缓存行上的不同变量时，就会发生这种情况。在 [@lst:FalseSharing] 中显示的代码示例中，即使线程 `A` 和 `B` 更新结构体 `S` 的不同字段，它们也很可能位于同一缓存行上，这将触发假共享问题。@fig:FalseSharing 说明了这个问题。

Listing: 假共享示例。

~~~~ {#lst:FalseSharing .cpp}
struct S {
  int sumA; // sumA 和 sumB 很可能
  int sumB; // 位于同一缓存行上
};
S s;

{ // 由线程 A 执行的代码     │     { // 由线程 B 执行的代码
  for (int i = 0; i < N; i++)      │       for (int i = 0; i < N; i++)
    s.sumA += a[i];                │         s.sumB += b[i];
}                                  │     }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![假共享：两个线程访问同一缓存行。](../../img/mt-perf/FalseSharing.jpg){#fig:FalseSharing width=60%}

假共享是多线程应用程序性能问题的常见来源。因此，现代分析工具内置了对检测此类情况的支持。对于经历真/假共享的应用程序，TMA 可能会显示较高的 `Memory Bound` &rarr; `L3 Bound` &rarr; `Contested Accesses` 指标。[^18]

使用 Intel VTune Profiler 时，我建议运行两种类型的分析来查找和消除假共享问题。首先，运行*微架构探索*分析，该分析实现 TMA 方法论以检测应用程序中是否存在假共享。如前所述，*Contested Accesses* 指标的高值促使我们更深入地挖掘，并运行启用*分析动态内存对象*复选框的*内存访问*分析。此分析有助于找出导致争用问题的数据结构的内存访问。通常，此类内存访问具有高延迟，分析将揭示这一点。有关使用 Intel VTune Profiler 修复假共享问题的示例，请参见 [Intel Developer Zone](https://software.intel.com/en-us/vtune-cookbook-false-sharing)。[^20]

Linux `perf` 也支持查找假共享。与 Intel VTune Profiler 一样，首先运行 TMA（参见 [@sec:secTMA_Intel]）以查看程序是否存在假/真共享问题。如果是这样，请使用 `perf c2c` 工具检测具有高缓存一致性成本的内存访问。`perf c2c` 匹配不同线程的存储/加载地址，并检查是否发生了对已修改缓存行的命中。读者可以在专门的 [博客文章](https://joemario.github.io/blog/2016/09/01/c2c-blog/)[^21] 中找到该过程的详细解释以及如何使用该工具。

可以通过对齐/填充内存对象来消除假共享。[@sec:secTrueSharing] 中的示例可以通过确保 `sumA` 和 `sumB` 不共享同一缓存行来修复，如 [@lst:PadFalseSharing] 所示。[^32]

Listing: 数据填充以避免假共享。

~~~~ {#lst:PadFalseSharing .cpp}
                              constexpr int CacheLineAlign = 64;
struct S {                    struct S {
  int sumA;        =>           int sumA; 
  int sumB;                     alignas(CacheLineAlign) int sumB;
};                            };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

假共享不仅可以在 C 和 C++ 等原生语言中观察到，也可以在 Java 和 C# 等托管语言中观察到。从一般性能角度来看，最重要的考虑因素是可能的状态转换成本。在所有缓存状态中，唯一在 CPU 读/写操作期间不涉及昂贵的跨缓存子系统通信和数据传输的是已修改（M）和独占（E）状态。因此，缓存行保持 `M` 或 `E` 状态的时间越长（即跨缓存的数据共享越少），多线程应用程序产生的相干性成本就越低。Nitsan Wakart 的博客文章 "[Diving Deeper into Cache Coherency](http://psy-lob-saw.blogspot.com/2013/09/diving-deeper-into-cache-coherency.html)"[^28] 中可以找到演示如何使用此属性的示例。

[^18]: 有关 *Contested Accesses* 指标的描述，请参阅 Intel VTune 用户指南。
[^20]: VTune cookbook: false-sharing - [https://software.intel.com/en-us/vtune-cookbook-false-sharing](https://software.intel.com/en-us/vtune-cookbook-false-sharing)。
[^21]: 关于 `perf c2c` 的文章 - [https://joemario.github.io/blog/2016/09/01/c2c-blog/](https://joemario.github.io/blog/2016/09/01/c2c-blog/)。
[^25]: 有一个 MESI 协议的动画演示 - [https://www.scss.tcd.ie/Jeremy.Jones/vivio/caches/MESI.htm](https://www.scss.tcd.ie/Jeremy.Jones/vivio/caches/MESI.htm)。
[^26]: MESIF - [https://en.wikipedia.org/wiki/MESIF_protocol](https://en.wikipedia.org/wiki/MESIF_protocol)
[^27]: MOESI - [https://en.wikipedia.org/wiki/MOESI_protocol](https://en.wikipedia.org/wiki/MOESI_protocol)
[^28]: 博客文章 "Diving Deeper into Cache Coherency" - [http://psy-lob-saw.blogspot.com/2013/09/diving-deeper-into-cache-coherency.html](http://psy-lob-saw.blogspot.com/2013/09/diving-deeper-into-cache-coherency.html)
[^30]: Clang 的线程消毒器工具：[https://clang.llvm.org/docs/ThreadSanitizer.html](https://clang.llvm.org/docs/ThreadSanitizer.html)。
[^31]: Helgrind，线程错误检测器工具：[https://www.valgrind.org/docs/manual/hg-manual.html](https://www.valgrind.org/docs/manual/hg-manual.html)。
[^32]: 不要将缓存行大小视为常量值。例如，在 Apple 处理器如 M1、M2 及更高版本中，L2 缓存以 128B 缓存行运行。
