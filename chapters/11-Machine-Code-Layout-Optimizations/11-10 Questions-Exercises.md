## 问题和练习 {.unlisted .unnumbered}

\markright{问题和练习}

1. 解决 `perf-ninja::pgo` 和 `perf-ninja::lto` 实验室作业。
2. 尝试为代码段使用大页。取一个大型应用程序（访问源代码是加分项但不是必需的），二进制文件大小超过 100MB。尝试使用 [@sec:FeTLB] 中描述的方法之一将其代码段重新映射到大页上。观察性能、`/proc/meminfo` 中的大页分配以及度量 ITLB 加载和未命中的 CPU 性能计数器的任何变化。
3. 运行你日常使用的应用程序。应用 PGO、llvm-bolt 或 Propeller 并检查结果。比较"之前"和"之后"的配置文件以了解加速来自哪里。
