## 软件和硬件计时器 {#sec:timers}

为了对执行时间进行基准测试，工程师通常使用两种不同的计时器，所有现代平台都提供：

 - **系统级高分辨率计时器**：这是一个系统计时器，通常实现为自称为[纪元](https://en.wikipedia.org/wiki/Epoch_(computing))[^1] 的任意起始日期以来经过的刻度数的简单计数。这个时钟是单调的；即它总是向上走。系统时间可以通过系统调用从操作系统获取。在 Linux 系统上访问系统计时器可以通过 `clock_gettime` 系统调用。系统计时器具有纳秒分辨率，在所有 CPU 之间一致，并且独立于 CPU 频率。尽管系统计时器可以返回纳秒精度的时间戳，但它不适用于度量短时间运行的事件，因为通过 `clock_gettime` 系统调用获取时间戳需要很长时间。但对于持续时间超过一微秒的事件是可以的。在 C++ 中访问系统计时器的标准方法是使用 `std::chrono`，如 [@lst:Chrono] 所示。

   Listing: 使用 C++ std::chrono 访问系统计时器
   
   ~~~~ {#lst:Chrono .cpp}
   #include <cstdint>
   #include <chrono>

   // 返回以纳秒为单位的经过时间
   uint64_t timeWithChrono() {
     using namespace std::chrono;
     auto start = steady_clock::now();
     // 运行某些内容
     auto end = steady_clock::now();
     uint64_t delta = duration_cast<nanoseconds>(end - start).count();
     return delta;
   }
   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   
 - **时间戳计数器（TSC）**：这是一个实现为硬件寄存器的硬件计时器。TSC 是单调的并且具有恒定速率，即它不考虑频率变化。每个 CPU 都有自己的 TSC，它只是经过的参考周期数（参见 [@sec:secRefCycles]）。它适用于度量持续时间从纳秒到一分钟的短事件。在 x86 平台上，TSC 的值可以通过使用编译器的内置函数 `__rdtsc` 来获取，如 [@lst:TSC] 所示，它在底层使用 `RDTSC` 汇编指令。有关使用 `RDTSC` 汇编指令对代码进行基准测试的更多低级详细信息，请参见白皮书 [@IntelRDTSC]。在 ARM 平台上，你可以读取 `CNTVCT_EL0`，计数器-定时器虚拟计数寄存器。

   Listing: 使用 __rdtsc 编译器内置函数访问 TSC

   ~~~~ {#lst:TSC .cpp}
   #include <x86intrin.h>
   #include <cstdint>

   // 返回经过的参考时钟数
   uint64_t timeWithTSC() {
       uint64_t start = __rdtsc();
       // 运行某些内容
       return __rdtsc() - start;
   }
   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

选择使用哪个计时器非常简单，取决于你要度量的东西的持续时间。如果你度量非常短时间内的内容，TSC 将给你更好的精度。相反，使用 TSC 度量运行数小时的程序是没有意义的。除非你需要周期精度，否则系统计时器对于大多数情况应该足够了。重要的是要记住，访问系统计时器通常比访问 TSC 具有更高的延迟。进行 `clock_gettime` 系统调用可能比执行 `RDTSC` 指令慢得多。后者大约需要 5 纳秒（20 个 CPU 周期），而前者大约需要 500 纳秒。这对于最小化测量开销可能很重要，特别是在生产环境中。CppPerformanceBenchmarks 仓库的 wiki 页面上提供了不同平台上访问计时器的不同 API 的性能比较。[^3]

[^1]: Unix 纪元从 1970 年 1 月 1 日 00:00:00 UT 开始：[https://en.wikipedia.org/wiki/Unix_epoch](https://en.wikipedia.org/wiki/Unix_epoch)。
[^3]: CppPerformanceBenchmarks wiki - [https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis](https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis)
