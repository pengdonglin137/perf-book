### 线程数扩展案例研究 {#sec:ThreadCountScalingStudy}

线程数扩展可能是你可以对多线程应用程序进行的最有价值的分析。它显示了应用程序可以利用现代多核系统的程度。正如你将看到的，你可以在这个过程中学到大量信息。不再做更多介绍，让我们开始吧。在这个案例研究中，我们将分析以下基准测试的线程数扩展，其中一些你应该已经从前几章中熟悉了：

1. Blender 3.4 - 一个开源 3D 创作和建模软件项目。此测试使用 BMW27 混合文件进行 Blender 的 Cycles 性能测试。URL：[https://download.blender.org/release](https://download.blender.org/release)。命令行：`./blender -b bmw27_cpu.blend -noaudio --enable-autoexec -o output.test -x 1 -F JPEG -f 1 -t N`，其中 `N` 是线程数。
2. Clang 17 构建 - 此测试使用 Clang 15 从源代码构建 Clang 17 编译器。URL：[https://www.llvm.org](https://www.llvm.org)。命令行：`ninja -jN clang`，其中 `N` 是线程数。
3. Zstandard v1.5.5，一种快速无损压缩算法。URL：[https://github.com/facebook/zstd](https://github.com/facebook/zstd)。用于压缩的数据集：[http://wanos.co/assets/silesia.tar](http://wanos.co/assets/silesia.tar)。命令行：`./zstd -TN -3 -f -- silesia.tar`，其中 `N` 是压缩工作线程数。
4. CloverLeaf 2018 - 一个拉格朗日-欧拉流体动力学基准测试。使用所有硬件线程。此测试使用输入文件 `clover_bm.in`（问题 5）。URL：[http://uk-mac.github.io/CloverLeaf](http://uk-mac.github.io/CloverLeaf)。命令行：`export OMP_NUM_THREADS=N; ./clover_leaf`，其中 `N` 是线程数。
5. CPython 3.12，Python 编程语言的参考实现。URL：[https://github.com/python/cpython](https://github.com/python/cpython)。我运行了一个用 Python 编写的简单多线程二分搜索脚本，它在 `1,000,000` 个元素的有序列表（干草堆）中搜索 `10,000` 个随机数字（针）。命令行：`./python3 binary_search.py N`，其中 `N` 是线程数。针在各个线程之间平均分配。

基准测试在以下配置的机器上执行：

* 第 12 代 Alder Lake Intel&reg; Core&trade; i7-1260P CPU @ 2.10GHz（4.70GHz Turbo），4P+8E 核心，18MB L3 缓存。
* 16 GB RAM，DDR4 @ 2400 MT/s。
* Clang 15 编译器，使用以下选项：`-O3 -march=core-avx2`。
* 256GB NVMe PCIe M.2 SSD。
* 64 位 Ubuntu 22.04.1 LTS（Jammy Jellyfish，Linux 内核 6.5）。

这显然不是顶级硬件配置，而是主流计算机，不一定设计用于处理媒体、开发者或 HPC 工作负载。然而，它是我们的案例研究的绝佳平台，因为它演示了线程数扩展的各种效果。由于资源有限，即使使用少量线程，应用程序也开始遇到性能障碍。请记住，在更好的硬件上，扩展结果会有所不同。

我的处理器有四个 P 核心和八个 E 核心。P 核心启用 SMT，这意味着此平台上的总线程数为十六。默认情况下，Linux 调度器将首先尝试使用空闲的物理 P 核心。前四个线程将利用四个空闲 P 核心上的四个线程。当它们被完全利用时，它将开始在 E 核心上调度线程。因此，接下来的八个线程将被调度在八个 E 核心上。最后，剩余的四个线程将被调度在 P 核心的 4 个兄弟 SMT 线程上。

我使用上述方案在固定线程亲和性的情况下运行基准测试，`Zstd` 和 `CPython` 除外。不运行亲和性可以更好地代表真实场景，但线程亲和性使线程数扩展分析更清晰。由于性能数字非常相似，在本案例研究中，我展示了使用线程亲和性时的结果。

基准测试执行固定数量的工作。无论线程数如何，退休指令数几乎相同。在所有基准测试中，算法的最大部分使用分治范式实现，其中工作被分成相等的部分，每个部分可以独立处理。理论上，这允许应用程序随着核心数量的增加而良好扩展。然而，在实践中，扩展通常远非最优。

图 @fig:ScalabilityMainChart 显示了所选基准测试的线程数可扩展性。x 轴表示线程数，y 轴显示相对于单线程执行的加速比。加速比计算为单线程执行的执行时间除以多线程执行的执行时间。加速比越高，应用程序随线程数的扩展越好。

![五个选定基准测试的线程数可扩展性图表。](../../img/mt-perf/ScalabilityMainChart.png){#fig:ScalabilityMainChart width=100%}

如你所见，大多数都远低于线性扩展，这相当令人失望。在本案例研究中扩展性最好的基准测试 Blender 在使用 16 个线程时仅实现 6 倍加速。例如，CPython 根本不享受线程数扩展。Clang 和 Zstd 的性能在超过 10 个线程时会下降。为了理解为什么会发生这种情况，让我们深入了解每个基准测试的细节。

### Blender {.unlisted .unnumbered}

Blender 是我们套件中唯一一个持续扩展到系统中所有 16 个线程的基准测试。原因是工作负载高度可并行化。渲染过程被分成小块，每个块可以独立渲染。然而，即使具有这种高水平的并行性，扩展也只有 `6.1 倍加速 / 16 个线程 = 38%`。这种次优扩展的原因是什么？

根据 [@sec:PerfMetricsCaseStudy]，我们知道 Blender 的性能受浮点计算限制。它还具有相对较高比例的 SIMD 指令。P 核心在处理此类指令方面比 E 核心好得多。这就是为什么我们看到在 4 个线程之后，随着 E 核心开始被使用，加速曲线的斜率下降。性能扩展以相同的速度持续到 12 个线程，然后开始再次下降。这是使用 SMT 兄弟线程的效果。两个活动的兄弟 SMT 线程争夺有限数量的 FP/SIMD 执行单元。要度量 SMT 扩展，我们需要将两个 SMT 线程（2T1C - 两个线程一个核心）的性能除以单个 P 核心（1T1C）的性能。[^5] 对于 Blender，SMT 扩展约为 `1.3 倍`。

还有另一个也适用于 Blender 的扩展降级方面，我们将在讨论 Clang 的线程数扩展时讨论。
