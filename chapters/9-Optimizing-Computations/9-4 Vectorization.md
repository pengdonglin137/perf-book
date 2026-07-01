## 向量化 {#sec:Vectorization}

在现代处理器上，使用 SIMD 指令可以比常规未向量化（标量）代码获得巨大的加速。在进行性能分析时，软件工程师的首要任务之一是确保代码的热部分被向量化。本节指导工程师发现向量化机会。有关现代 CPU 的 SIMD 功能的回顾，读者可以查看 [@sec:SIMD]。

向量化通常在没有任何用户干预的情况下自动发生；这称为编译器*自动向量化*。在这种情况下，编译器自动识别从源代码生成 SIMD 机器代码的机会。

自动向量化非常方便，因为现代编译器可以自动为各种程序生成快速的 SIMD 代码。然而，在某些情况下，没有软件工程师的干预，自动向量化不会成功。现代编译器具有扩展功能，允许高级用户控制自动向量化过程，并确保代码的某些部分被高效地向量化。我们将提供几个使用编译器自动向量化提示的示例。

在本节中，我们将讨论如何利用编译器自动向量化，特别是内循环向量化，因为它是自动向量化最常见的类型。另外两种类型（外循环向量化和超级字级并行向量化）不在本书中讨论。

### 编译器自动向量化

多个障碍可能阻止自动向量化，其中一些是编程语言语义固有的。例如，编译器必须假设循环索引可能溢出，这可能阻止某些循环转换。另一个例子是 C 语言的假设：程序中的指针可能指向重叠的内存区域，这会使程序分析变得非常困难。

另一个主要障碍是处理器本身的设计。在某些情况下，处理器没有针对某些操作的有效向量指令。例如，谓词（位掩码控制）加载和存储操作在大多数处理器上不可用。尽管存在所有这些挑战，你可以解决其中的许多问题并启用自动向量化。在本节后面，我们提供关于如何与编译器协作并确保热代码被编译器向量化的指导。

向量化器通常分三个阶段构建：合法性检查、盈利性检查和转换本身：

* **合法性检查**：在此阶段，编译器检查是否可以合法地将循环（或另一种类型的代码区域）转换为使用向量。合法性阶段收集需要发生的一系列要求，以使循环的向量化合法。循环向量化器检查循环的迭代是否连续，这意味着循环线性推进。向量化器还确保循环中的所有内存和算术操作都可以展宽为连续操作。循环的控制流在所有 lane 上是统一的，内存访问模式是统一的。编译器必须检查或确保生成的代码不会触及它不应该触及的内存，并且操作顺序将被保留。编译器需要分析指针的可能范围，如果它缺少某些信息，它必须假设转换是非法的。

* **盈利性检查**：接下来，向量化器检查转换是否盈利。它比较不同的向量化宽度，并找出哪个执行最快。向量化器使用成本模型来预测不同操作的成本，例如标量加法或向量加载。它需要考虑将数据混洗到寄存器中添加的指令，预测寄存器压力，并估计确保允许向量化的前提条件的循环保护的成本。检查盈利性的算法很简单：1）将代码中所有操作的成本相加，2）比较每个版本代码的成本，3）将成本除以预期执行次数。例如，如果标量代码成本为 8 个周期，而向量化代码成本为 12 个周期但一次执行 4 次循环迭代，那么循环的向量化版本可能更快。

* **转换**：最后，在向量化器确定转换是合法且盈利的之后，它转换代码。此过程还包括插入启用向量化的保护。例如，大多数循环使用未知的迭代次数，因此编译器必须生成循环的标量版本（余数），以及循环的向量化版本，以处理最后几次迭代。编译器还必须检查指针是否不重叠等。所有这些转换都使用在合法性检查阶段收集的信息完成。

### 发现向量化机会。{#sec:DiscoverVectOpptnt}

发现改进向量化的机会应该从分析程序中的热循环开始，并检查编译器执行了哪些优化。检查编译器向量化报告（参见 [@sec:compilerOptReports]）是了解这一点的最简单方法。现代编译器可以报告某个循环是否被向量化，并提供其他详细信息，例如向量化因子（VF）。当编译器无法向量化循环时，它也能够说明失败的原因。

使用编译器优化报告的另一种方法是检查汇编输出。最好分析来自分析工具的输出，该工具显示给定循环的源代码和生成的汇编指令之间的对应关系。这样，你只关注重要的代码，即热代码。然而，理解汇编语言比像 C++ 这样的高级语言要困难得多。可能需要一些时间来弄清楚编译器生成的指令的语义。然而，这项技能非常有价值，通常提供有价值的见解。

有经验的开发人员可以通过查看指令助记符和这些指令使用的寄存器名称来快速判断代码是否已被向量化。例如，在 x86 ISA 中，向量指令对打包数据进行操作（因此名称中有 `P`）并使用 `XMM`、`YMM` 或 `ZMM` 寄存器，例如，`VMULPS XMM1, XMM2, XMM3` 将 `XMM2` 和 `XMM3` 中的四个单精度浮点数相乘，并将结果保存在 `XMM1` 中。但要小心，人们通常从看到 `XMM` 寄存器被使用就得出结论它是向量代码——不一定。例如，`VMULSS XMM1, XMM2, XMM3` 指令将只乘以一个单精度浮点值，而不是四个。

潜在向量化机会的另一个指标是高 `Retiring` 指标（高于 80%）。在 [@sec:TMA] 中，我们说过 `Retiring` 指标是性能良好代码的良好指标。其背后的原理是执行没有停顿，CPU 以高速率退休指令。然而，有时它可能隐藏真正的性能问题，即低效计算。也许工作负载执行大量可以被向量指令替换的简单指令。在这种情况下，高 `Retiring` 指标不会转化为高性能。

开发人员在尝试加速可向量化代码时经常遇到几种常见情况。下面我们介绍四种典型场景，并就每种情况给出一般性指导。

#### 向量化是非法的。

在某些情况下，遍历数组元素的代码根本不可向量化。优化报告非常有效地解释了出了什么问题以及为什么编译器无法向量化代码。[@lst:VectDep] 显示了阻止向量化的循环内依赖的示例。[^31]

Listing: 向量化：写后读依赖。

~~~~ {#lst:VectDep .cpp}
void vectorDependence(int *A, int n) {
  for (int i = 1; i < n; i++)
    A[i] = A[i-1] * 2;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

虽然某些循环由于硬性限制（如写后读依赖）无法向量化，但当某些约束被放宽时，其他循环可以被向量化。例如，[@lst:VectIllegal] 中的代码无法被编译器自动向量化，因为它会改变浮点运算的顺序并可能导致不同的舍入和略微不同的结果。浮点加法是可交换的，这意味着你可以交换左边和右边而不改变结果：`(a + b == b + a)`。然而，它不是可结合的，因为舍入在不同时间发生：`((a + b) + c) != (a + (b + c))`。

Listing: 向量化：浮点算术。

~~~~ {#lst:VectIllegal .cpp .numberLines}
// a.cpp
float calcSum(float* a, unsigned N) {
  float sum = 0.0f;
  for (unsigned i = 0; i < N; i++) {
    sum += a[i];
  }
  return sum;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你告诉编译器你可以容忍最终结果的微小变化，它将为你自动向量化代码。Clang 和 GCC 编译器有一个标志 `-ffast-math`，[^29] 允许这种转换，即使结果程序可能给出略微不同的结果：

```bash
$ clang++ -c a.cpp -O3 -march=core-avx2 -Rpass-analysis=.*
...
a.cpp:5:9: remark: loop not vectorized: cannot prove it is safe to reorder floating-point operations; allow reordering by specifying '#pragma clang loop vectorize(enable)' before the loop or by providing the compiler option '-ffast-math'. [-Rpass-analysis=loop-vectorize]
...
$ clang++ -c a.cpp -O3 -march=core-avx2 -ffast-math -Rpass=.*
...
a.cpp:4:3: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
...
```

不幸的是，此标志涉及微妙且可能危险的行为更改，包括对于非数字（NaN）、有符号零、无穷大和非规格化数。因为第三方代码可能没有为这些影响做好准备，所以不应在没有仔细验证结果（包括边缘情况）的情况下在大段代码上启用此标志。从 Clang 18 开始，你可以使用专用 pragma 限制转换范围，例如 `#pragma clang fp reassociate(on)`。[^4]

让我们看另一种典型情况，编译器可能需要开发人员的支持来执行向量化。当编译器无法证明循环在非重叠内存区域的数组上操作时，它们通常选择安全的一边。给定 [@lst:OverlappingMemRefions] 中的代码，编译器应该考虑数组 `a`、`b` 和 `c` 的内存区域重叠的情况。

Listing: a.c

~~~~ {#lst:OverlappingMemRefions .cpp .numberLines}
void foo(float* a, float* b, float* c, unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下是 GCC 10.2 提供的优化报告（使用 `-fopt-info` 启用）：

```bash
$ gcc -O3 -march=core-avx2 -fopt-info
a.cpp:2:26: optimized: loop vectorized using 32-byte vectors
a.cpp:2:26: optimized:  loop versioned for vectorization because of possible aliasing
```

GCC 已识别内存区域之间的潜在重叠，并创建了循环的多个版本。编译器插入了运行时检查[^36]来检测内存区域是否重叠。基于这些检查，它在向量化和标量版本之间调度。在这种情况下，向量化伴随着插入可能昂贵的运行时检查的代价。如果开发人员知道数组 `a`、`b` 和 `c` 的内存区域不重叠，可以在循环前插入 `#pragma GCC ivdep`[^37] 或使用 `__restrict__` 关键字，如 [@sec:compilerOptReports] 所示。此类编译器提示将消除 GCC 编译器插入上述运行时检查的需要。

一些动态工具，如 Intel Advisor，可以检测循环中是否出现跨迭代依赖或访问具有重叠内存区域的数组等问题。但请注意，此类工具仅提供建议。随意插入编译器提示可能会导致真正的问题。

#### 向量化无益。

在某些情况下，编译器可以向量化循环，但认为这样做不划算。在 [@lst:VectNotProfit] 中展示的代码中，编译器可以向量化对数组 `A` 的内存访问，但需要将对数组 `B` 的访问拆分为多个标量加载。scatter/gather 模式相对昂贵，能够模拟操作成本的编译器通常决定避免向量化具有此类模式的代码。

Listing: 向量化：无益。

~~~~ {#lst:VectNotProfit .cpp .numberLines}
// a.cpp
void stridedLoads(int *A, int *B, int n) {
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下是 [@lst:VectNotProfit] 中代码的编译器优化报告：

```bash
$ clang -c -O3 -march=core-avx2 a.cpp -Rpass-missed=loop-vectorize
a.cpp:3:3: remark: the cost-model indicates that vectorization is not beneficial [-Rpass-missed=loop-vectorize]
  for (int i = 0; i < n; i++)
  ^
```

用户可以使用 `#pragma` 提示强制 Clang 编译器向量化循环，如 [@lst:VectNotProfitOverriden] 所示。但请记住，向量化是否有利在很大程度上取决于运行时数据，例如循环的迭代次数。编译器没有这些可用信息，[^1] 因此它们往往倾向于保守。不过你可以将此类提示用于性能实验。

Listing: 向量化：无益。

~~~~ {#lst:VectNotProfitOverriden .cpp .numberLines}
// a.cpp
void stridedLoads(int *A, int *B, int n) {
#pragma clang loop vectorize(enable)
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

开发人员应该意识到使用向量化代码的隐藏成本。使用 AVX 尤其是 AVX-512 向量指令可能导致频率降频或启动开销，在某些 CPU 上这也可能影响后续代码数微秒。代码的向量化部分应该足够热以证明使用 AVX-512 是合理的。[^38] 例如，已发现排序 80 KiB 足以分摊此开销并使向量化值得。[^39]

#### 循环已向量化但使用了标量版本。

在某些场景中，编译器成功向量化了代码，但在分析器中没有显示为正在执行。检查循环的相应汇编时，通常很容易找到循环体的向量化版本，因为它使用了程序其他部分不常用的向量寄存器。

如果向量代码没有执行，一个可能的原因是生成的代码假设的循环迭代次数高于程序实际使用的。例如，编译器可能决定以每次迭代处理 64 个元素的方式向量化和展开循环。输入数组可能没有足够的元素甚至无法执行循环的一次迭代。在这种情况下，将使用循环的标量版本（余数）。检测这些情况很容易，因为标量循环会在分析器中亮起来，而向量化代码将保持冷态。

此问题的解决方案是强制向量化器使用更低的向量化因子或展开计数，以减少循环处理的元素数量。你可以使用 `#pragma` 提示来实现。对于 Clang 编译器，你可以使用 `#pragma clang loop vectorize_width(N)`，如 Easyperf 博客文章所示。[^30]

#### 循环以次优方式向量化。

当你看到循环被自动向量化并在运行时执行时，程序的这部分很可能已经表现良好。然而，也有例外。有些情况下，循环的标量未向量化版本比向量化版本性能更好。这可能是由于昂贵的向量操作，如 `gather/scatter` 加载、掩码、`inserting/extracting` 元素、数据混洗等，如果编译器被要求使用它们来使向量化发生。性能工程师也可以尝试以不同方式禁用向量化。对于 Clang 编译器，可以通过编译器选项 `-fno-vectorize` 和 `-fno-slp-vectorize` 完成，或使用特定于特定循环的提示，例如 `#pragma clang loop vectorize(disable)`。

需要注意的是，有一系列问题是 SIMD 重要的，但自动向量化不起作用且在不久的将来也不太可能起作用。一个例子可以在 [@Mula_Lemire_2019] 中找到。另一个例子是外循环自动向量化，编译器目前没有尝试。向量化浮点代码是有问题的，因为重新排序算术浮点运算会导致不同的舍入和略微不同的值。

自动向量化还有一个微妙的问题。随着编译器的发展，它们所做的优化也在变化。在前一个编译器版本中成功自动向量化的代码可能在下一个版本中停止工作，反之亦然。此外，在代码维护或重构期间，代码结构可能发生变化，导致自动向量化突然开始失败。这可能在原始软件编写后很长时间才发生，因此此时修复或重新实现的成本会更高。

#### 具有显式向量化的语言。{#sec:ISPC}

向量化也可以通过将程序的部分重写为专用并行计算的编程语言来实现。这些语言使用特殊构造和程序数据的知识来将代码高效地编译为并行程序。最初，此类语言主要用于将工作卸载到特定处理单元，如图形处理单元（GPU）、数字信号处理器（DSP）或现场可编程门阵列（FPGA）。然而，其中一些编程模型也可以针对你的 CPU（如 OpenCL 和 OpenMP）。

其中一种并行语言是 Intel 隐式 SPMD 程序编译器 [(ISPC)](https://ispc.github.io/)，[^33] 我将在本节中简要介绍。ISPC 语言基于 C 编程语言，使用 LLVM 编译器基础设施为许多不同的架构生成优化代码。ISPC 的关键特性是"接近底层"的编程模型和跨 SIMD 架构的性能可移植性。它需要从传统的编写程序思维转变，但给程序员更多对 CPU 资源利用的控制。

ISPC 的另一个优势是其互操作性和易用性。ISPC 编译器生成标准目标文件，可以与传统 C/C++ 编译器生成的代码链接。ISPC 代码可以轻松插入任何原生项目，因为用 ISPC 编写的函数可以像 C 代码一样被调用。

[@lst:ISPC_code] 展示了我之前在 [@lst:VectIllegal] 中展示的函数的 ISPC 版本。ISPC 考虑程序将基于目标指令集在并行实例中运行。例如，当使用 SSE 处理 `float` 时，它可以并行计算 4 个操作。每个程序实例将操作 `i` 的向量值为 `(0,1,2,3)`，然后是 `(4,5,6,7)`，依此类推，有效地一次计算 4 个求和。如你所见，使用了几个对 C 和 C++ 不典型的关键字：

* `export` 关键字表示该函数可以从 C 兼容语言调用。

* `uniform` 关键字表示变量在程序实例之间共享。

* `varying` 关键字表示每个程序实例有自己的变量本地副本。

* `foreach` 与经典的 `for` 循环相同，只是它将在不同的程序实例之间分配工作。

Listing: ISPC 版本的数组元素求和。

~~~~ {#lst:ISPC_code .cpp}
export uniform float calcSum(const uniform float array[], 
                             uniform ptrdiff_t count)
{
    varying float sum = 0;
    foreach (i = 0 ... count)
        sum += array[i];
    return reduce_add(sum);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于函数 `calcSum` 必须返回单个值（`uniform` 变量），而我们的 `sum` 变量是 `varying` 的，我们需要使用 `reduce_add` 函数*收集*每个程序实例的值。ISPC 还负责根据需要生成剥离和余数循环，以考虑未正确对齐或不是向量宽度倍数的数据。

**"接近底层"的编程模型**：传统 C 和 C++ 语言的问题之一是编译器并不总是向量化代码的关键部分。ISPC 通过假设每个操作默认是 SIMD 来帮助解决此问题。例如，ISPC 语句 `sum += array[i]` 被隐式视为并行执行多次加法的 SIMD 操作。ISPC 不是自动向量化编译器，它不会自动发现向量化机会。由于 ISPC 语言与 C 和 C++ 非常相似，它比内置函数（参见 [@sec:secIntrinsics]）更具可读性，因为它允许你专注于算法而不是低级指令。此外，据报道它已匹配 [@ISPC_Paper] 或超越[^34]手写内置函数代码的性能。

**性能可移植性**：ISPC 可以自动检测 CPU 的功能以充分利用所有可用资源。程序员可以编写一次 ISPC 代码并编译到多种向量指令集，如 SSE4、AVX2 和 ARM NEON。

[^1]: 除了 Profile Guided Optimizations（参见 [@sec:secPGO]）。
[^2]: 例如，编译器优化报告，参见 [@sec:compilerOptReports]。
[^29]: 编译器标志 `-Ofast` 启用 `-ffast-math` 以及 `-O3` 编译模式。
[^30]: 使用 Clang 的优化 pragma - [https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts](https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts)
[^31]: 一旦你展开循环的几次迭代，就很容易发现写后读依赖。参见 [@sec:compilerOptReports] 中的示例。
[^33]: ISPC 编译器：[https://ispc.github.io/](https://ispc.github.io/)。
[^34]: Unreal Engine 中使用 SIMD 内置函数的部分已使用 ISPC 重写，这带来了加速：[https://software.intel.com/content/www/us/en/develop/articles/unreal-engines-new-chaos-physics-system-screams-with-in-depth-intel-cpu-optimizations.html](https://software.intel.com/content/www/us/en/develop/articles/unreal-engines-new-chaos-physics-system-screams-with-in-depth-intel-cpu-optimizations.html)。
[^36]: 参见 Easyperf 博客上的示例：[https://easyperf.net/blog/2017/11/03/Multiversioning_by_DD](https://easyperf.net/blog/2017/11/03/Multiversioning_by_DD)。
[^37]: 这是 GCC 特定的 pragma。对于其他编译器，请检查相应手册。
[^38]: 更多详情请阅读此博客文章：[https://travisdowns.github.io/blog/2020/01/17/avxfreq1.html](https://travisdowns.github.io/blog/2020/01/17/avxfreq1.html)。
[^39]: AVX-512 降频研究：见 [VQSort readme](https://github.com/google/highway/blob/master/hwy/contrib/sort/README.md#study-of-avx-512-downclocking)
[^4]: LLVM 扩展以指定浮点标志 - [https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags](https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags)
