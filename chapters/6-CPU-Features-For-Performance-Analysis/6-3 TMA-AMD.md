### AMD 平台上的 TMA {#sec:secTMA_AMD}

从 Zen4 开始，AMD 处理器支持第 1 级和第 2 级 TMA 分析。根据 AMD 文档，它被称为"流水线利用率"分析，但思想保持不变。L1 和 L2 桶也与 Intel 的非常相似。从内核 6.2 开始，Linux 用户可以使用 `perf` 工具收集流水线利用率数据。

接下来，我们将研究 [Crypto++](https://github.com/weidai11/cryptopp)[^1] 的 SHA-256（安全哈希算法 256）实现，这是比特币挖矿中的基本加密算法。Crypto++ 是一个开源的 C++ 加密算法类库，包含许多算法的实现，而不仅仅是 SHA-256。然而，对于我们的示例，我通过注释掉 `bench1.cpp` 中 `BenchmarkUnkeyedAlgorithms` 函数中的相应行来禁用其他算法的基准测试。

我在配备 Ubuntu 22.04、Linux 内核 6.5 的 AMD Ryzen 9 7950X 机器上运行了测试。我使用 GCC 12.3 C++ 编译器编译了 Crypto++ 版本 8.9。我使用了默认的 `-O3` 优化选项，但由于代码是用 x86 内置函数编写的（参见 [@sec:secIntrinsics]）并利用了 SHA x86 ISA 扩展，因此它对性能影响不大。

以下是我用于获取 L1 和 L2 流水线利用率指标的命令。输出经过修剪，一些统计数据被删除以消除不必要的干扰。

```bash
$ perf stat -M PipelineL1,PipelineL2 -- ./cryptest.exe b1 10
 0.0 %  bad_speculation_mispredicts        (20.08%) 
 0.0 %  bad_speculation_pipeline_restarts  (20.08%)
 0.0 %  bad_speculation                    (20.08%)
 6.1 %  frontend_bound                     (20.00%)
 6.1 %  frontend_bound_bandwidth           (20.00%)
 0.1 %  frontend_bound_latency             (20.00%)
65.9 %  backend_bound_cpu                  (20.00%)
 1.7 %  backend_bound_memory               (20.00%)
67.5 %  backend_bound                      (20.00%)
26.3 %  retiring                           (20.08%)
20.2 %  retiring_fastpath                  (19.99%)
 6.1 %  retiring_microcode                 (19.99%)
```

在输出中，方括号中的数字表示运行时持续时间的百分比，当监控该指标时。如我们所见，由于多路复用，每个指标只被监控了 20% 的时间。在我们的案例中，由于 SHA256 具有一致的行为，这可能不是问题，但情况并非总是如此。为了最小化多路复用的影响，你可以在单次运行中收集有限的指标集，例如 `perf stat -M frontend_bound,backend_bound`。

上面显示的流水线利用率指标的描述可以在 [@AMDUprofManual, Chapter 2.8 Pipeline Utilization] 中找到。通过查看指标，我们可以看到 SHA256 中没有发生分支预测错误（`bad_speculation` 为 0%）。只有 26.3% 的可用分派槽被使用（`retiring`），这意味着剩余的 73.7% 由于前端和后端停顿而被浪费。

高级加密指令并不简单，因此它们在内部被分解为更小的部分（$\mu$ops）。一旦处理器遇到这样的指令，它就从微代码中检索其 $\mu$ops。微操作从微代码序列器获取的带宽低于常规指令解码器，使其成为潜在的性能瓶颈来源。Crypto++ SHA256 实现大量使用 `SHA256MSG2`、`SHA256RNDS2` 等指令，根据 [uops.info](https://uops.info/table.html)[^2] 网站，这些指令由多个 $\mu$ops 组成。

`retiring_microcode` 指标表明 6.1% 的分派槽被最终退休的微代码操作使用。与同级指标 `retiring_fastpath` 相比，我们可以说大约每 4 条指令中就有一条是微代码操作。如果我们现在查看 `frontend_bound_bandwidth` 指标，我们将看到 6.1% 的分派槽由于 CPU 前端的带宽瓶颈而未被使用。这表明 6.1% 的分派槽被浪费了，因为微代码序列器在后端可以消耗时没有提供 $\mu$ops。在这个例子中，`retiring_microcode` 和 `frontend_bound_bandwidth` 指标紧密相关，然而，它们相等的事实仅仅是巧合。

大多数周期在 CPU 后端停顿（`backend_bound`），但只有 1.7% 的周期在等待内存访问时停顿（`backend_bound_memory`）。所以，我们知道基准测试主要受机器计算能力的限制。正如你将在本书第 2 部分中所知，这可能与数据流依赖或某些加密操作的执行吞吐量有关。它们比传统的 `ADD`、`SUB`、`CMP` 和其他指令更少见，因此通常只能在单个执行单元上执行。大量此类操作可能饱和该特定单元的执行吞吐量。进一步的分析应涉及仔细查看源代码和生成的汇编、检查执行端口利用率、查找数据依赖等。

总之，Crypto++ 在 AMD Ryzen 9 7950X 上的 SHA-256 实现仅使用了 26.3% 的可用分派槽；6.1% 的分派槽由于微代码序列器带宽而被浪费，65.9% 由于缺乏机器计算资源而停顿。该算法确实遇到了一些硬件限制，因此不清楚其性能是否可以改进。

在 Windows 方面，在撰写本文时，TMA 方法仅在 AMD 服务器平台（代号 Genoa）上支持，而不在客户端系统（代号 Raphael）上支持。TMA 支持在 AMD uProf 4.1 版本中添加，但仅在 AMD uProf 安装中的命令行工具 `AMDuProfPcm` 中。你可以查阅 [@AMDUprofManual, Chapter 2.8 Pipeline Utilization] 了解更多关于如何运行分析的详细信息。AMD uProf 的图形版本还没有 TMA 分析功能。
