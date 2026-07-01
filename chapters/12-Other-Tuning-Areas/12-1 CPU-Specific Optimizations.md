## CPU 特定优化

为特定 CPU 微架构优化软件涉及调整代码以利用该微架构的优势并减轻其劣势。当你知道应用程序的确切目标 CPU 时，这样做更容易。但是，大多数应用程序在各种 CPU 上运行。优化具有非常高速度要求的跨平台应用程序的性能可能具有挑战性，因为来自不同供应商的平台具有不同的设计和实现。尽管如此，可以编写在不同供应商的 CPU 上性能合理的代码，同时为特定微架构提供精细调整的版本。

x86（被视为 CISC）和 RISC ISA（如 ARM 和 RISC-V）之间的主要差异总结如下：

* x86 指令是可变长度的，而 ARM 和 RISC-V 指令是固定长度的。这使得解码 x86 指令更加复杂。
* x86 ISA 具有许多寻址模式，而 ARM 和 RISC-V 具有很少的寻址模式。ARM 和 RISC-V 指令中的操作数是寄存器或立即值，而 x86 指令输入也可以来自内存。这增加了 x86 指令的数量，但也允许更强大的单条指令。例如，ARM 需要先加载内存位置，然后执行操作；x86 可以在一条指令中完成两者。

除此之外，在为特定微架构进行优化时，你还应该考虑一些其他差异。截至 2024 年，最新的 x86-64 ISA 有 16 个架构通用寄存器，而最新的 ARMv8 和 RV64 要求 CPU 提供 32 个通用寄存器。额外的架构寄存器减少了寄存器溢出，从而减少了加载/存储的数量。Intel 宣布了一个名为 APX[^1] 的新扩展，它将寄存器数量增加到 32 个。

x86 和 ARM 之间的内存页大小也存在差异。x86 平台的默认页大小为 4 KB，而大多数 ARM 系统（例如 Apple MacBooks）使用 16 KB 页大小，尽管两个平台都支持更大的页大小（参见 [@sec:ArchHugePages] 和 [@sec:secDTLB]）。当这些差异成为瓶颈时，它们都会影响应用程序的性能。

虽然 ISA 差异*可能*对特定应用程序的性能产生切实影响，但大量研究表明，平均而言，两个最流行的 ISA（即 x86 和 ARM）之间的差异不会产生可衡量的性能影响。在整本书中，我小心地避免了任何产品的广告（例如 Intel vs. AMD vs. Apple）和任何宗教 ISA 辩论（x86 vs. ARM vs. RISC-V）。[^5] 以下是一些我认为可以结束辩论的参考：

* 性能或能耗差异不是由 ISA 差异产生的，而是由微架构实现产生的。[@RISCvsCISC2013]
* ISA 对执行指令的数量和类型没有很大影响。[@RISCVvsAArch642023] [@RISCvsCISC2013]
* CISC 代码不比 RISC 代码更密集。[@CodeDensityCISCvsRISC]
* ISA 开销可以通过微架构实现有效缓解。例如，$\mu$op 缓存最小化解码开销；指令缓存最小化代码密度影响。[@RISCvsCISC2013] [@ChipsAndCheesex86]

尽管如此，这并没有消除架构特定优化的价值。在本节中，我们将讨论如何为特定平台进行优化。我们将介绍 ISA 扩展、CPU 分发技术，并讨论如何推理指令延迟和吞吐量。

### ISA 扩展

ISA 演进一直在持续。它专注于加速专门的工作负载，例如加密、AI、多媒体等。利用 ISA 扩展通常会带来显著的性能改进。开发人员不断找到在通用应用程序中利用这些扩展的聪明方法。因此，即使你在这些高度专业化的领域之外，你仍然可能从使用 ISA 扩展中受益。

不可能学习所有特定指令。但我建议你熟悉目标平台上可用的主要 ISA 扩展。例如，如果你正在开发使用 `fp16`（16 位半精度浮点）数据类型的 AI 应用程序，并且你的目标是现代 ARM 处理器之一，请确保你的程序的机器代码包含相应的 `fp16` ISA 扩展。如果你正在开发加密/解密软件，请检查它是否利用了目标 ISA 的加密扩展。等等。

以下是一些著名的 x86 ISA 扩展列表：

* SSE/AVX/AVX2：提供用于浮点和整数操作的 SIMD 指令。
* AVX512：使用 512 位寄存器和许多新指令扩展 AVX2。
* AVX512_FP16/AVX512_BF16：添加对 16 位半精度和 `Bfloat16` 浮点值的支持。
* AES/SHA：提供用于 AES 加密、解密和 SHA 哈希的指令。
* BMI/BMI2：提供用于位操作的指令。
* AVX_VNNI/AVX512_VNNI：用于加速深度学习工作负载的向量神经网络指令。
* AMX：用于加速矩阵乘法的高级矩阵扩展。

以下是一些著名的 ARM ISA 扩展列表：

* Advanced SIMD：也称为 NEON，提供算术 SIMD 指令。
* Cryptographic Instructions：提供加密、哈希和校验和指令。
* FP16/BF16：提供 16 位半精度和 `Bfloat16` 浮点指令。
* UDOT/SDOT：支持点积指令，用于加速机器学习工作负载。
* SVE：启用可扩展向量长度指令。
* SME：用于加速矩阵乘法的可扩展矩阵扩展。

编译应用程序时，请确保启用必要的编译器标志以激活所需的 ISA 扩展。在 GCC 和 Clang 编译器上使用 `-march` 选项。例如，`-march=native` 将激活主机系统的 ISA 特性，即运行编译的系统。或者你可以包含特定版本的 ISA，例如 `-march=armv8.6-a`。在 MSVC 编译器上，使用 `/arch` 选项，例如 `/arch:AVX2`。

我不建议在生产构建中使用 `-march=native`，因为代码生成将取决于你构建代码的机器。许多 CI/CD 系统使用旧机器。在其中一台机器上使用 `-march=native` 构建软件可能导致应用程序在较新机器上运行时性能不佳。相反，使用 `-march` 指定你想要目标的特定微架构。

### CPU 分发

当你想要为特定微架构提供快速路径同时为其他平台保留通用实现时，可以使用 *CPU 分发*。这是一种允许程序检测处理器具有哪些特性，并据此决定执行哪个版本代码的技术。它使你能够在单一代码库中引入平台特定优化。根据经验，最好从通用实现开始，然后逐步引入微架构特定优化，确保没有所需特性的架构有回退方案。例如：

```cpp
if (__builtin_cpu_supports ("avx512f")) {
  avx512_impl();
} else {
  generic_impl();
}
```

这演示了 GCC 和 Clang 编译器中可用的内置函数的使用。除了检测支持的 ISA 扩展外，还有一个 `__builtin_cpu_is` 函数用于检测确切的处理器型号。编写 CPU 分发的编译器无关方式是使用 `CPUID` 指令（仅限 x86）、`getauxval(AT_HWCAP)` Linux 系统调用，或 macOS 上的 `sysctlbyname`。

你通常会看到 CPU 分发构造仅用于优化代码的特定部分，例如热函数或循环。通常，这些平台特定实现使用编译器内建函数（参见 [@sec:secIntrinsics]）编写以生成所需的指令。

尽管 CPU 分发是运行时检查，但其开销并不高。你可以在启动时一次性识别硬件功能并将其保存在某个变量中，因此在运行时，它只是一个可良好预测的单一分支。也许对 CPU 分发更大的担忧是维护成本。每个新的专用分支都需要微调和验证。

### 指令延迟和吞吐量

除了 ISA 扩展外，了解处理器中执行单元的数量和类型也是值得的（例如，处理器每个周期可以发出的加载、存储、除法和乘法的数量）。对于大多数处理器，这些信息由 CPU 供应商在相应的技术手册中发布。然而，特定指令的延迟和吞吐量信息通常不会公开。尽管如此，人们已经对单个指令进行了基准测试，可以在线访问。对于最新的 Intel 和 AMD CPU，指令的延迟、吞吐量、端口使用和 $\mu$ops 数量可以在 [uops.info](https://uops.info/table.html)[^2] 网站上找到。对于 Apple 处理器，类似的数据可以在 [@AppleOptimizationGuide, Appendix A] 中获取。[^6] 除了指令延迟和吞吐量外，开发人员还逆向工程了微架构的其他方面，如分支预测历史缓冲区大小、重排序缓冲区容量、加载/存储缓冲区大小等。

仅根据指令延迟和吞吐量数字得出结论时要非常小心。在许多情况下，指令延迟被乱序执行引擎隐藏，指令的延迟是 4 个周期还是 8 个周期可能并不重要。如果它不阻塞前进进度，这样的指令将在"后台"处理而不损害性能。然而，当指令位于关键依赖链上时，其延迟就变得重要，因为它会延迟依赖操作的执行。

相反，如果你有一个执行大量*独立*操作的循环，你应该关注指令吞吐量而不是延迟。当操作是独立的，它们可以并行处理。在这种情况下，关键因素是每种类型的操作每周期可以执行多少个，即*执行吞吐量*。也有一些"中间"场景，指令延迟和吞吐量都可能影响性能。

当你分析某个热循环的机器代码时，你可能会发现多条指令被分配到同一执行端口。这种情况被称为*执行端口争用*。因此挑战在于找到将其中一些指令替换为不分配到关键端口的指令的方法。例如，在 Intel 处理器上，如果你在 `port5` 上严重瓶颈，那么你可能会发现 `port0` 上的两条指令比 `port5` 上的一条指令更好。通常这不是一项容易的任务，它需要深入的 ISA 和微架构知识。如有疑问，请在专业论坛上寻求帮助。此外，请记住，其中一些内容可能在未来的 CPU 代际中发生变化，因此考虑使用 CPU 分发来隔离代码更改的效果。

### 案例研究：FMA 指令何时损害性能 {.unlisted .unnumbered}

在 [@sec:FMAThroughput] 中，我们看了一个 FMA 指令吞吐量变得关键的例子。现在让我们看另一个涉及 FMA 延迟的例子。在 [@lst:FMAlatency] 左边，我们有 `sqSum` 函数，它计算每个元素平方的和。右边，我们展示了 Clang-18 使用 `-O3 -march=core-avx2` 编译时生成的相应机器代码。注意，我们没有使用 `-ffast-math`，也许因为我们希望在多个平台上保持精确结果。这就是为什么编译器没有自动向量化代码。

Listing: FMA 延迟

~~~~ {#lst:FMAlatency .cpp .numberLines}
float sqSum(float *a, int N) {         │ .loop:
  float sum = 0;                       │  vmovss xmm1, dword ptr [rcx + 4*rdx]
  for (int i = 0; i < N; i++ )         │  vfmadd231ss xmm0, xmm1, xmm1
    sum += a[i] * a[i];                │  inc rdx
  return sum;                          │  cmp rax, rdx
}                                      │  jne .loop
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在循环的每次迭代中，我们有两个操作：计算 `a[i]` 的平方值并将乘积累加到 `sum` 变量中。如果你仔细观察，你可能会注意到乘法彼此独立，因此可以并行执行。生成的机器代码（右边）使用融合乘加（FMA）用单条指令执行两个操作。这里的问题是，通过使用 FMA，编译器将乘法包含在了循环的关键依赖链中。

`vfmadd231ss` 指令计算 `a[i]` 的平方值（在 `xmm1` 中），然后将结果累加到 `xmm0` 中。`xmm0` 上存在数据依赖：处理器在前一条指令完成之前不能发出新的 `vfmadd231ss` 指令，因为 `xmm0` 既是 `vfmadd231ss` 的输入也是输出。即使 FMA 的乘法部分彼此不依赖，这些指令也需要等到所有输入可用。此循环的性能受 FMA 延迟限制，在 Intel 的 Alder Lake 上为 4 个周期。

在这种情况下，融合乘法和加法会损害性能。我们使用两条独立的指令会更好。下面的 `nanobench` 实验证明了这一点：

```
# ran on Intel Core i7-1260P (Alder Lake)
$ sudo ./kernel-nanoBench.sh -f -basic │ $ sudo ./kernel-nanoBench.sh -f -basic
 -loop 100 -unroll 1000                │  -loop 100 -unroll 1000 
 -warm_up_count 10 -asm "              │  -warm_up_count 10 -asm "
vmovss xmm1, dword ptr [R14];          │ vmovss xmm1, dword ptr [R14];
vfmadd231ss xmm0, xmm1, xmm1;"         │ vmulss xmm1, xmm1, xmm1;
-asm_init "<not shown>"                │ vaddss xmm0, xmm0, xmm1;"
                                       │ -asm_init "<not shown>"
Instructions retired: 2.00             │ 
Core cycles: 4.00                      │ Instructions retired: 3.00
                                       │ Core cycles: 2.00
```

左边的版本每次迭代运行四个周期，对应 FMA 延迟。然而，在右边，`vmulss` 指令彼此不依赖，因此可以并行运行。尽管如此，`vaddss` 指令（`FADD`）在 `xmm0` 上仍有循环进位依赖。但 FADD 的延迟仅为两个周期，这就是为什么右边的版本每次迭代只运行两个周期。其他处理器的延迟和吞吐量特性可能有所不同。[^7]

从这个实验中，我们知道如果编译器没有决定将乘法和加法融合为单条指令，此循环的性能将提高两倍。只有当我们检查了循环依赖并比较了 FMA 和 FADD 指令的延迟后，这一点才变得清晰。从 Clang 18 开始，你可以使用 `#pragma clang fp contract(off)` 在作用域内阻止生成 FMA 指令。[^4]

[^1]: Intel APX - [https://www.intel.com/content/www/us/en/developer/articles/technical/advanced-performance-extensions-apx.html](https://www.intel.com/content/www/us/en/developer/articles/technical/advanced-performance-extensions-apx.html)
[^2]: x86 指令延迟和吞吐量 - [https://uops.info/table.html](https://uops.info/table.html)
[^4]: LLVM 浮点标志扩展 - [https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags](https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags)
[^5]: 这个辩论也不有趣，因为经过 $\mu$ops 转换后，x86 变成了 RISC 风格的微架构。复杂指令被分解为更简单的指令。
[^6]: 此外，还有通过逆向工程实验收集的指令吞吐量和延迟数据，例如 [https://dougallj.github.io/applecpu/firestorm-simd.html](https://dougallj.github.io/applecpu/firestorm-simd.html)。由于这是非官方数据来源，你应该谨慎对待。
[^7]: 由于浮点值的不同舍入，两个版本将产生略微不同的结果。
