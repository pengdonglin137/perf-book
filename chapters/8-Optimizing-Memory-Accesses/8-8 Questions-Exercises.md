## 问题和练习 {.unlisted .unnumbered}

\markright{问题和练习}

1. 解决 `perf-ninja::data_packing` 实验室作业，在该作业中你需要使数据结构更紧凑。
2. 使用我们在 [@sec:secDTLB] 中讨论的方法解决 `perf-ninja::huge_pages_1` 实验室作业。观察性能、`/proc/meminfo` 中的大页分配以及度量 DTLB 加载和未命中的 CPU 性能计数器的任何变化。
3. 通过为未来的循环迭代实现显式内存预取来解决 `perf-ninja::swmem_prefetch_1` 实验室作业。
4. 一般性地描述一段代码要缓存友好需要什么。
5. 运行你日常使用的应用程序。度量其内存使用情况，并使用我们在 [@sec:MemoryProfiling] 中讨论的内存分析器分析堆分配。使用 Linux perf、Intel VTune 或其他分析器识别热内存访问。是否有方法可以改进这些访问？
