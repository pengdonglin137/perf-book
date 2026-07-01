## Roofline 性能模型 {#sec:roofline}

Roofline 性能模型是一种面向吞吐量的性能模型，在 HPC 领域被广泛使用。它于 2009 年在加州大学伯克利分校开发 [@RooflinePaper]。该模型中的"屋顶线"一词表达了应用程序的性能不能超过机器能力的事实。程序中的每个函数和每个循环都受到机器计算或内存带宽容量的限制。这个概念在@fig:RooflineIntro 中表示。应用程序的性能将始终受到某个"屋顶线"函数的限制。

![Roofline 性能模型。应用程序的最大性能受峰值 FLOPS（水平线）和平台带宽乘以算术强度（对角线）的最小值限制。](../../img/perf-analysis/Roofline-intro.png){#fig:RooflineIntro width=80%}

硬件有两个主要限制：它能多快进行计算（峰值计算性能，FLOPS）和它能多快移动数据（峰值内存带宽，GB/s）。应用程序的最大性能受峰值 FLOPS（水平线）和平台带宽乘以算术强度（对角线）的最小值限制。@fig:RooflineIntro 中的 Roofline 图表绘制了两个应用程序 `A` 和 `B` 相对于硬件限制的性能。应用程序 `A` 具有较低的算术强度，其性能受内存带宽限制，而应用程序 `B` 更计算密集，不会像那样受到内存瓶颈的影响。类似地，`A` 和 `B` 可以表示程序中的两个不同的函数，并具有不同的性能特征。Roofline 性能模型考虑了这一点，可以在同一个图表上显示应用程序的多个函数和循环。但是，请记住，Roofline 性能模型主要适用于具有少量计算密集型循环的 HPC 应用程序。我不建议将其用于通用应用程序，如编译器、Web 浏览器或数据库。

*算术强度*是浮点操作（FLOPs）[^7] 和字节之间的比率，可以为程序中的每个循环计算。让我们计算 [@lst:BasicMatMul] 中代码的算术强度。在最内层循环体中，我们有一个浮点加法和一个乘法；因此，我们有 2 个 FLOP。另外，我们有三个读操作和一个写操作；因此，我们传输 `4 操作 * 4 字节 = 16` 字节。该代码的算术强度为 `2 / 16 = 0.125`。算术强度是 Roofline 图表的 X 轴，而 Y 轴度量给定程序的性能。

Listing: 朴素并行矩阵乘法。

~~~~ {#lst:BasicMatMul .cpp .numberLines}
void matmul(int N, float a[][2048], float b[][2048], float c[][2048]) {
    #pragma omp parallel for
    for(int i = 0; i < N; i++) {
        for(int j = 0; j < N; j++) {
            for(int k = 0; k < N; k++) {
                c[i][j] = c[i][j] + a[i][k] * b[k][j];
            }
        }
    }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

加速应用程序性能的传统方法是充分利用机器的 SIMD 和多核能力。通常，我们需要优化许多方面：向量化、内存和线程。Roofline 方法可以帮助评估应用程序的这些特征。在 Roofline 图表上，我们可以绘制标量单核、SIMD 单核和 SIMD 多核性能的理论最大值（参见@fig:RooflineIntro2）。这将使我们了解改进应用程序性能的范围。如果我们发现我们的应用程序是计算受限的（即具有高算术强度）并且低于峰值标量单核性能，我们应该考虑强制向量化（参见 [@sec:Vectorization]）并将工作分配给多个线程。相反，如果应用程序具有低算术强度，我们应该寻求改善内存访问的方法（参见 [@sec:MemBound]）。使用 Roofline 模型优化性能的最终目标是将点在图表上向上移动。向量化和线程将点向上移动，而通过增加算术强度来优化内存访问将点向右移动，并且也可能提高性能。

![程序的 Roofline 分析及其性能改进的潜在方法。](../../img/perf-analysis/Roofline-intro2.jpg){#fig:RooflineIntro2 width=70%}

理论最大值（屋顶线）通常在设备规格中给出，可以轻松查找。此外，可以根据你使用的机器特性计算理论最大值。通常，一旦你知道机器的参数，这并不难做到。对于 Intel Core i5-8259U 处理器，使用 AVX2 和 2 个融合乘加（FMA）单元的最大 FLOPS（单精度浮点数）可以计算为：

$$
\begin{aligned}
\textrm{Peak FLOPS} =& \textrm{ 8 (number of logical cores)}~\times~\frac{\textrm{256 (AVX bit width)}}{\textrm{32 bit (size of float)}} ~ \times ~ \\
& \textrm{ 2 (FMA)} \times ~ \textrm{3.8 GHz (Max Turbo Frequency)} \\
& = \textrm{486.4 GFLOPS}
\end{aligned}
$$

Intel NUC Kit NUC8i5BEH 的最大内存带宽（我用于实验的设备）可以如下计算。记住，DDR 技术允许每次内存访问传输 64 位或 8 字节。
