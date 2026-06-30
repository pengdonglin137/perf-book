## 微基准测试

微基准测试是人们编写的小型自包含程序，用于快速测试假设。通常，微基准测试用于选择某个相对较小的算法或功能的最佳实现。几乎所有现代语言都有基准测试框架。在 C++ 中，你可以使用 Google [benchmark](https://github.com/google/benchmark)[^3] 库，C# 有 [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)[^4] 库，Julia 有 [BenchmarkTools](https://github.com/JuliaCI/BenchmarkTools.jl)[^5] 包，Java 有 [JMH](http://openjdk.java.net/projects/code-tools/jmh/etc)[^6]（Java 微基准测试工具），Rust 有 Criterion[^8] 包等。

编写微基准测试时，确保你想测试的场景在运行时确实由你的微基准测试执行非常重要。优化编译器可能会消除重要的代码，使实验无用，甚至更糟，引导你得出错误的结论。在下面的示例中，现代编译器可能会消除整个循环：

```cpp
// foo 不对字符串创建进行基准测试
void foo() {
  for (int i = 0; i < 1000; i++)
    std::string s("hi");
}
```

像这样的错误在论文"Always Measure One Level Deeper" [@MeasureOneLevelDeeper] 中得到了很好的捕捉，作者倡导更科学的方法，并从不同角度度量性能。遵循论文的建议，我们应该检查基准测试的性能概况，并确保预期的代码作为热点突出显示。有时可以立即发现异常计时，因此在分析和比较基准测试运行时使用常识。

防止编译器优化掉重要代码的流行方法之一是使用 [`DoNotOptimize`](https://github.com/google/benchmark/blob/c078337494086f9372a46b4ed31a3ae7b3f1a6a2/include/benchmark/benchmark.h#L307)-like[^7] 辅助函数，它们在底层执行必要的内联汇编魔法：

```cpp
// foo 对字符串创建进行基准测试
void foo() {
  for (int i = 0; i < 1000; i++) {
    std::string s("hi");
    DoNotOptimize(s);
  }
}
```

如果编写得当，微基准测试可以成为性能数据的良好来源。它们通常用于比较关键函数的不同实现的性能。好的基准测试在现实条件下测试性能。相反，如果基准测试使用与实践中不同的合成输入，那么基准测试可能会误导你并引导你得出错误的结论。除此之外，当基准测试在没有其他高需求进程的系统上运行时，它拥有所有可用资源，包括 DRAM 和缓存空间。这样的基准测试可能会拥护更快版本的函数，即使它比另一个版本消耗更多内存。但是，如果有邻居进程消耗了 DRAM 的很大一部分，导致属于基准测试进程的内存区域被交换到磁盘，结果可能会相反。

出于同样的原因，在得出从函数单元测试获得的结果时要小心。现代单元测试框架（如 GoogleTest）提供每个测试的持续时间。但是，这些信息不能替代精心编写的基准测试，该基准测试在实际条件下使用现实输入测试函数（更多内容请参见 [@fogOptimizeCpp, chapter 16.2]）。并不总是能够复制与实践中完全相同的输入和环境，但这是开发者在编写好的基准测试时应该考虑的。

[^3]: Google benchmark 库 - [https://github.com/google/benchmark](https://github.com/google/benchmark)
[^4]: BenchmarkDotNet - [https://github.com/dotnet/BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)
[^5]: Julia BenchmarkTools - [https://github.com/JuliaCI/BenchmarkTools.jl](https://github.com/JuliaCI/BenchmarkTools.jl)
[^6]: Java Microbenchmark Harness - [http://openjdk.java.net/projects/code-tools/jmh/etc](http://openjdk.java.net/projects/code-tools/jmh/etc)
[^7]: 对于 JMH，这被称为 `Blackhole.consume()`。
[^8]: Criterion.rs - [https://github.com/bheisler/criterion.rs](https://github.com/bheisler/criterion.rs)
