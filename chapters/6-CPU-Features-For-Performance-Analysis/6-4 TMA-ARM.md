### Arm 平台上的 TMA

Arm CPU 架构师也为他们的处理器开发了 TMA 性能分析方法，我们接下来将讨论。Arm 在其文档中称其为"Topdown"[@ARMNeoverseV1TopDown]，因此我们将使用他们的命名。在撰写本文时（2023 年底），Topdown 仅在 Arm 设计的核心上支持，例如 Neoverse N1 和 Neoverse V1，[^5] 以及它们的衍生产品，例如 Ampere Altra 和 AWS Graviton3。如果你需要刷新对 Arm 芯片系列的记忆，请参阅本书末尾的主要 CPU 微架构列表。Apple 设计的处理器尚不支持 Arm Topdown 性能分析方法。

Arm Neoverse V1 是 Neoverse 系列中第一个支持完整第 1 级 Topdown 指标集的 CPU：`Bad Speculation`、`Frontend Bound`、`Backend Bound` 和 `Retiring`。在 V1 核心之前，Neoverse N1 仅支持两个 L1 类别：`Frontend Stalled Cycles` 和 `Backend Stalled Cycles`。[^6]

为了演示 Arm 在基于 V1 的处理器上的 Topdown 分析，我启动了一个由 AWS Graviton3 提供支持的 AWS EC2 `m7g.metal` 实例。请注意，由于虚拟化，Topdown 可能无法在其他非 `metal` 实例类型上工作。我使用了由 AWS 管理的 64 位 ARM `Ubuntu 22.04 LTS` 和 `Linux kernel 6.2`。提供的 `m7g.metal` 实例具有 64 个 vCPU 和 256 GB RAM。

我将 Topdown 方法应用于 [AI Benchmark Alpha](https://ai-benchmark.com/alpha.html)，[^1] 这是一个用于评估各种硬件平台（包括 CPU、GPU 和 TPU）AI 性能的开源 Python 库。该基准测试依赖于 TensorFlow 机器学习库来测量关键深度学习模型的推理和训练速度。AI Benchmark Alpha 总共包含 42 个测试，包括分类、图像分割、文本翻译等。

Arm 工程师开发了 [topdown-tool](https://learn.arm.com/install-guides/topdown-tool/)[^2]，我们将在下面使用。该工具在 ARM 上的 Linux 和 Windows 上都能工作。在 Linux 上，它利用标准的 perf 工具，而在 Windows 上，它使用 [WindowsPerf](https://gitlab.com/Linaro/WindowsPerf/windowsperf)[^3]，这是一个 Windows on ARM 性能分析工具。与 Intel 的 TMA 类似，Arm 方法采用"深入"概念，即你首先确定高级性能瓶颈，然后深入进行更细致的根本原因分析。以下是我们使用的命令：

```bash
$ topdown-tool --all-cpus -m Topdown_L1 -- python -c "from ai_benchmark import AIBenchmark; results = AIBenchmark(use_CPU=True).run()"
Stage 1 (Topdown metrics)
=========================
[Topdown Level 1]
Frontend Bound... 16.48% slots
Backend Bound.... 54.92% slots
Retiring......... 27.99% slots
Bad Speculation..  0.59% slots
```

其中 `--all-cpus` 选项启用所有 CPU 的系统级收集，`-m Topdown_L1` 收集 Topdown 第 1 级指标。`--` 之后的所有内容是要运行的 AI Benchmark Alpha 套件的命令行。

从上面的输出中，我们得出结论基准测试没有遭受分支预测错误。此外，如果没有对所涉及工作负载的更深入理解，很难说 16.5% 的 `Frontend Bound` 是否值得研究，因此我们将注意力转向 `Backend Bound` 指标，这是停顿周期的主要来源。基于 Neoverse V1 的芯片没有二级分解，相反，该方法建议通过收集一组相应的指标来进一步探索有问题的类别。以下是我们如何深入更详细的 `Backend Bound` 分析：

```bash
$ topdown-tool --all-cpus -n BackendBound -- python -c "from ai_benchmark import AIBenchmark; results = AIBenchmark(use_CPU=True).run()"
Stage 1 (Topdown metrics)
=========================
[Topdown Level 1]
Backend Bound......................... 54.70% slots

Stage 2 (uarch metrics)
=======================
  [Data TLB Effectiveness]
  DTLB MPKI........................... 0.413 misses per 1,000 instructions
  L1 Data TLB MPKI.................... 3.779 misses per 1,000 instructions
  L2 Unified TLB MPKI................. 0.407 misses per 1,000 instructions
```
