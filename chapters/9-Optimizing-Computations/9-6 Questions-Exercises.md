## 问题和练习 {.unlisted .unnumbered}

\markright{问题和练习}

1. 使用我们在本章中讨论的技术解决以下实验室作业：
- `perf-ninja::function_inlining_1` 
- `perf-ninja::vectorization` 1 和 2
- `perf-ninja::dep_chains` 1 和 2
- `perf-ninja::compiler_intrinsics` 1 和 2
- `perf-ninja::loop_interchange` 1 和 2
- `perf-ninja::loop_tiling_1`
2. 描述你将采取的步骤来找出应用程序是否正在利用利用 SIMD 代码的所有机会。
3. 在真实代码上练习手动进行循环优化。确保所有测试仍然通过。
4. 假设你正在处理一个具有非常低 IpCall（每调用指令数）指标的应用程序。你将尝试应用/强制哪些优化？
5. 运行你日常使用的应用程序。找到最热的循环。它是否被向量化？是否可以强制编译器自动向量化？循环是否受依赖链或执行吞吐量限制？
