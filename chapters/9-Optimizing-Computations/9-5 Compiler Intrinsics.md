## 编译器内置函数 {#sec:secIntrinsics}

有些类型的应用程序具有值得大力调优的热点。但是，编译器并不总是能够在这些热点中生成我们想要的代码。有时人类专家可以想出比编译器生成的代码性能更好的代码。它通常涉及一些棘手或专门的算法，这些算法对于编译器来说可能非常困难甚至不可能找出。在这些情况下，可能无法使用 C 和 C++ 语言的标准构造让编译器生成所需的汇编代码。

当绝对有必要生成特定汇编指令时，你不应该依赖编译器自动向量化。在这种情况下，代码可以使用*编译器内置函数*编写。在大多数情况下，编译器内置函数提供与汇编指令的 1 到 1 映射。[@lst:Intrinsics] 中的示例展示了如何使用编译器内置函数编写数组元素的水平求和（参见 [@lst:VectIllegal]）的 C++ 版本。

Listing: 使用编译器内置函数对数组元素求和。
		
~~~~ {#lst:Intrinsics .cpp .numberLines}
#include <immintrin.h>

float calcSum(float* a, unsigned N) {  
  __m128 sum = _mm_setzero_ps();      // 用零初始化求和
  unsigned i = 0;
  for (; i + 3 < N; i += 4) {
    __m128 vec = _mm_loadu_ps(a + i); // 从数组加载 4 个浮点数
    sum = _mm_add_ps(sum, vec);       // 将 vec 累加到 sum
  }

  // 128 位向量的水平求和
  __m128 shuf = _mm_movehdup_ps(sum); // 将元素 3,1 广播到 2,0
  sum = _mm_add_ps(sum, shuf);        // 部分和 [0+1] 和 [2+3]
  shuf = _mm_movehl_ps(shuf, sum);    // 高半部分 -> 低半部分
  sum = _mm_add_ss(sum, shuf);        // 结果在低元素中
  float result = _mm_cvtss_f32(sum);  // 空操作（编译器消除它）

  // 处理剩余元素
  for (; i < N; i++)
      result += a[i];
  return result;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当为 SSE 目标编译 [@lst:VectIllegal] 时，编译器将生成与 [@lst:Intrinsics] 基本相同的汇编代码。我展示这个例子只是为了说明目的。显然，如果编译器可以生成相同的机器代码，就没有必要使用内置函数。你应该仅在编译器无法生成所需代码时使用内置函数。

当你利用编译器自动向量化时，它将插入所有必要的运行时检查。例如，它将确保有足够的元素来馈送向量执行单元（参见 [@lst:Intrinsics]，第 6 行）。此外，编译器将生成循环的标量版本来处理余数（第 19 行）。当你使用内置函数时，你必须自己处理安全方面。

内置函数比内联汇编更好，因为编译器执行类型检查，负责寄存器分配，并进行进一步的优化，例如窥孔转换和指令调度。然而，它们通常仍然冗长且难以阅读。

当你使用不可移植的平台特定内置函数编写代码时，你还应该为其他架构提供后备选项。Intel 平台上所有可用内置函数的列表可以在这个[参考](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)[^11]中找到。对于 ARM，你可以在 Arm 的网站上找到这样的列表。[^14]

### 内置函数的包装库 {#sec:secIntrinsicLibraries}

在低工作量但不可预测的自动向量化，和冗长/不可读但可预测的内置函数之间，有一条中间道路，你可以使用内置函数的包装库。这些库往往更具可读性，提供可移植性，同时仍然给开发人员对生成代码的控制。存在许多这样的库，它们在对最新或"特殊"操作的覆盖范围以及支持的平台数量方面有所不同。

ISPC 的一次编写、多目标模型很有吸引力。然而，你可能希望与 C++ 程序更紧密地集成。例如，与模板的互操作性，或避免单独的构建步骤并使用相同的编译器。相反，内置函数提供更多控制，但开发成本更高。

包装库结合了两者的优点并避免了缺点，使用所谓的嵌入式领域特定语言，其中向量操作表示为普通的 C++ 函数。你可以将这些函数视为"可移植的内置函数"。甚至将代码编译多次，每个指令集一次，可以在普通的 C++ 库中完成，通过使用预处理器用不同的编译器设置"重复"你的代码，但在唯一的命名空间中。此类库的一个例子是 Highway，[^12] 它只需要 C++11 标准。

[@lst:HWY_code] 展示了 Highway 版本的数组元素求和。`ScalableTag<float> d` 是一个类型描述符，表示"可扩展"类型，意味着它可以调整到目标硬件上的可用向量宽度（例如 AVX2 或 NEON）。`Zero(d)` 将 `sum` 初始化为填充零的向量。此变量将在函数遍历 `array` 时存储累加的和。for 循环一次处理 `Lanes(d)` 个元素，其中 `Lanes(d)` 表示可以加载到单个 SIMD 向量中的浮点数数量。`LoadU` 操作从 `array` 加载 `Lanes(d)` 个连续元素。`Add` 操作执行加载的值与当前 `sum` 的逐元素加法，将结果累加到 `sum` 中。

Listing: Highway 版本的数组元素求和。

~~~~ {#lst:HWY_code .cpp}
#include <hwy/highway.h>

float calcSum(const float* HWY_RESTRICT array, size_t count) {
  const ScalableTag<float> d;  // 类型描述符；没有实际数据
  auto sum = Zero(d);
  size_t i = 0;
  for (; i + Lanes(d) <= count; i += Lanes(d)) {
    sum = Add(sum, LoadU(d, array + i));
  }
  sum = Add(sum, MaskedLoad(FirstN(d, count - i), d, array + i));
  return ReduceSum(d, sum);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

注意循环处理向量大小 `Lanes(d)` 的倍数后对余数的显式处理。虽然这更冗长，但它使实际发生的事情可见，并允许优化，如重叠最后一个向量而不是依赖 `MaskedLoad`，甚至在已知 `count` 是向量大小的倍数时完全跳过余数。最后，`ReduceSum` 操作通过将向量 `sum` 中的所有元素相加来将其归约为单个标量值。

与 ISPC 一样，Highway 也支持检测最佳可用指令集，分组为"集群"，在 x86 上对应于 Intel Core（S-SSE3）、Nehalem（SSE4.2）、Haswell（AVX2）、Skylake（AVX-512）或 Icelake/Zen4（带扩展的 AVX-512）。然后它从相应的命名空间调用你的代码。与内置函数不同，代码保持可读性（没有每个函数上的前缀/后缀）和可移植性。

当你使用内置函数或包装库时，仍然建议使用 C++ 编写初始实现。这允许快速原型设计和正确性验证，通过将原始代码的结果与新的向量化实现进行比较。

Highway 支持 200 多种操作，可以分为以下几类：

\begin{multicols}{2}
\begin{itemize}
\tightlist
\item 初始化
\item 获取/设置 lane
\item 获取/设置块
\item 打印
\item 元组
\item 算术
\item 逻辑
\item 掩码
\item 比较
\item 内存
\item 缓存控制
\item 类型转换
\item 合并
\item 混洗/排列
\item 128 位块内的混洗
\item 归约
\item 加密
\end{itemize}
\end{multicols}

完整操作列表请参见其文档。[^13] Highway 不是此类的唯一库。其他库包括 nsimd、SIMDe、VCL 和 xsimd。注意，从 Vc 库开始的 C++ 标准化工作产生了 std::experimental::simd，然而，它提供了一组非常有限的操作，截至撰写本文时并非所有主要编译器都支持。

[^11]: Intel 内置函数指南 - [https://software.intel.com/sites/landingpage/IntrinsicsGuide/](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)。
[^12]: Highway 库：[https://github.com/google/highway](https://github.com/google/highway)
[^13]: Highway 快速参考 - [https://github.com/google/highway/blob/master/g3doc/quick_reference.md](https://github.com/google/highway/blob/master/g3doc/quick_reference.md)
[^14]: ARM 内置函数指南 - [https://developer.arm.com/architectures/instruction-sets/intrinsics/](https://developer.arm.com/architectures/instruction-sets/intrinsics/)
