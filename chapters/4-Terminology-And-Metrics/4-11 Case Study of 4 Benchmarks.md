## 案例研究：分析四个基准测试的性能指标 {#sec:PerfMetricsCaseStudy}

为了综合本章到目前为止讨论的所有内容，让我们看一些真实世界的例子。我们运行了来自不同领域的四个基准测试，并计算了它们的性能指标。首先，让我们介绍这些基准测试。

1. Blender 3.4 - 一个开源的 3D 创作和建模软件项目。此测试使用 BMW27 混合文件进行 Blender 的 Cycles 性能测试。使用所有硬件线程。URL：[https://download.blender.org/release](https://download.blender.org/release)。命令行：`./blender -b bmw27_cpu.blend -noaudio --enable-autoexec -o output.test -x 1 -F JPEG -f 1`。
2. Stockfish 15 - 一个高级开源国际象棋引擎。此测试是 stockfish 内置基准测试。使用单个硬件线程。URL：[https://stockfishchess.org](https://stockfishchess.org)。命令行：`./stockfish bench 128 1 24 default depth`。
3. Clang 15 自构建 - 此测试使用 Clang 15 从源代码构建 Clang 15 编译器。使用所有硬件线程。URL：[https://www.llvm.org](https://www.llvm.org)。命令行：`ninja -j16 clang`。
4. CloverLeaf 2018 - 一个拉格朗日-欧拉流体动力学基准测试。使用所有硬件线程。此测试使用 clover_bm.in 输入文件（问题 5）。URL：[http://uk-mac.github.io/CloverLeaf](http://uk-mac.github.io/CloverLeaf)。命令行：`./clover_leaf`。

对于这个练习，我在具有以下特征的机器上运行了所有四个基准测试：

* 第 12 代 Alder Lake Intel&reg; Core&trade; i7-1260P CPU @ 2.10GHz（4.70GHz Turbo），4P+8E 核心，18MB L3 缓存
* 16 GB RAM，DDR4 @ 2400 MT/s
* 256GB NVMe PCIe M.2 SSD
* 64 位 Ubuntu 22.04.1 LTS（Jammy Jellyfish）
* Clang-15 C++ 编译器，使用以下选项：`-O3 -march=core-avx2`

为了收集性能指标，我使用了 Andi Kleen 的 [pmu-tools](https://github.com/andikleen/pmu-tools) 中的 `toplev.py` 脚本：[^1]

```bash
$ ~/workspace/pmu-tools/toplev.py -m --global --no-desc -v -- <app with args>
```

表 {@tbl:perf_metrics_case_study} 提供了我们的四个基准测试的性能指标的并排比较。仅通过查看指标，我们就可以了解这些工作负载的性质。

\small

--------------------------------------------------------------------------
指标名称      核心类型    Blender     Stockfish   Clang15-   CloverLeaf
                               自构建
---------------- ----------- ----------- ----------- ---------- ----------
指令数        P 核心      6.02E+12    6.59E+11    2.40E+13   1.06E+12

核心周期      P 核心      4.31E+12    3.65E+11    3.78E+13   5.25E+12

IPC           P 核心      1.40        1.80        0.64       0.20

CPI           P 核心      0.72        0.55        1.57       4.96

指令数        E 核心      4.97E+12    0           1.43E+13   1.11E+12

--------------------------------------------------------------------------
表：四个基准测试的性能指标比较。{#tbl:perf_metrics_case_study}

\normalsize
