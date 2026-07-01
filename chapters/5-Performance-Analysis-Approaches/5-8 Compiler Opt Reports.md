## 编译器优化报告 {#sec:compilerOptReports}

如今，软件开发在很大程度上依赖编译器进行性能优化。编译器在加速软件方面起着关键作用。大多数开发人员将优化代码的工作留给编译器，只在看到改进编译器无法完成的机会时才进行干预。可以说，这是一个很好的默认策略。但当你寻求最佳性能时，它就不那么有效了。如果编译器未能执行关键优化（如向量化循环）怎么办？你如何知道这一点？幸运的是，所有主要编译器都提供优化报告，我们现在将讨论这些报告。

假设你想知道一个关键循环是否被展开。如果它被展开了，展开因子是多少？有一种困难的方法可以知道：研究生成的汇编指令。不幸的是，并非所有人都习惯阅读汇编语言。如果函数很大，它调用其他函数或有许多也被向量化的循环，或者编译器创建了同一循环的多个版本，这可能尤其困难。大多数编译器，包括 GCC、Clang、Intel 编译器和 MSVC[^9] 都提供优化报告，以检查特定代码完成了哪些优化。

让我们看看 [@lst:optReport]，它显示了一个未被 `clang 16.0` 向量化的循环示例。

Listing: a.c

~~~~ {#lst:optReport .cpp .numberLines}
void foo(float* __restrict__ a, 
         float* __restrict__ b, 
         float* __restrict__ c,
         unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    a[i] = c[i-1]; // 从前一次迭代传递的值
    c[i] = b[i];
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

要在 Clang 编译器中发出优化报告，你需要使用 [-Rpass*](https://llvm.org/docs/Vectorizers.html#diagnostics) 标志：

```bash
$ clang -O3 -Rpass-analysis=.* -Rpass=.* -Rpass-missed=.* a.c -c
a.c:5:3: remark: loop not vectorized [-Rpass-missed=loop-vectorize]
  for (unsigned i = 1; i < N; i++) {
  ^
a.c:5:3: remark: unrolled loop by a factor of 8 with run-time trip count [-Rpass=loop-unroll]
  for (unsigned i = 1; i < N; i++) {
  ^
```

通过检查上面的优化报告，我们可以看到该循环没有被向量化，而是被展开了。对于开发人员来说，识别 [@lst:optReport] 中第 6 行循环中的循环携带依赖并不总是容易的。`c[i-1]` 加载的值依赖于前一次迭代的存储（参见@fig:VectorDep 中的操作 \circled{2} 和 \circled{3}）。可以通过手动展开循环的前几次迭代来揭示依赖关系：

```cpp
 
// 迭代 1
  a[1] = c[0];
  c[1] = b[1];
// 迭代 2
  a[2] = c[1];
  c[2] = b[2];
...
```

![可视化 [@lst:optReport] 中的操作顺序。](../../img/perf-analysis/VectorDep.png){#fig:VectorDep width=40%}

如果我们对 [@lst:optReport] 中的代码进行向量化，将导致数组 `a` 中写入错误的值。假设 CPU SIMD 单元一次可以处理四个浮点数，我们将得到以下伪代码表示的代码：

```cpp
// 迭代 1
  a[1..4] = c[0..3]; // 糟糕！a[2..4] 得到错误的值
  c[1..4] = b[1..4]; 
...
```

[@lst:optReport] 中的代码不能被向量化，因为循环内操作的顺序很重要。此示例可以通过交换第 6 行和第 7 行来修复，如 [@lst:optReport2] 所示。这不会改变代码的语义，因此是完全合法的更改。或者，可以通过将循环拆分为两个单独的循环来改进代码。这样做会使循环开销翻倍，但这一缺点将被通过向量化获得的性能改进所抵消。

Listing: a.c

~~~~ {#lst:optReport2 .cpp .numberLines}
void foo(float* __restrict__ a, 
         float* __restrict__ b, 
         float* __restrict__ c,
         unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在优化报告中，我们现在可以看到循环被成功向量化了：

```bash
$ clang -O3 -Rpass-analysis=.* -Rpass=.* -Rpass-missed=.* a.c -c
a.cpp:5:3: remark: vectorized loop (vectorization width: 8, interleaved count: 4) [-Rpass=loop-vectorize]
  for (unsigned i = 1; i < N; i++) {
  ^
```

这只是使用优化报告的一个示例；我们将在 [@sec:DiscoverVectOpptnt] 中提供更多示例，讨论如何发现向量化机会。编译器优化报告可以帮助你找到错过的优化机会，并理解为什么这些机会被错过了。此外，编译器优化报告对于测试假设很有用。编译器通常根据其成本模型分析来决定某个转换是否有益。但编译器并不总是做出最优选择。一旦你在报告中检测到关键的缺失优化，你可以尝试通过更改源代码或以 `#pragma`、属性、编译器内建函数等形式向编译器提供提示来纠正它。与往常一样，通过在实际环境中测量来验证你的假设。

编译器报告可能相当大，并且为每个源代码文件生成单独的报告。有时，在输出文件中查找相关记录可能成为一项挑战。我们应该提到，这些报告最初明确设计为供编译器编写者用来改进优化过程。多年来，已经开发了几种工具来使优化报告对应用程序开发人员更易访问和可操作，最著名的是 [opt-viewer](https://github.com/llvm/llvm-project/tree/main/llvm/tools/opt-viewer)[^7] 和 [optview2](https://github.com/OfekShilon/optview2)。[^8] 此外，[Compiler Explorer](https://godbolt.org/) 网站为基于 LLVM 的编译器提供了 "Optimization Output" 工具，当鼠标悬停在相应源代码行上时报告执行的转换。所有这些工具都有助于可视化基于 LLVM 的编译器的成功和失败的代码转换。

在链接时优化（LTO）[^5] 模式下，一些优化在链接阶段进行。要从编译和链接阶段发出编译器报告，你应该向编译器和链接器都传递专用选项。有关更多信息，请参阅 LLVM "Remarks" [指南](https://llvm.org/docs/Remarks.html)[^6]。

Intel® [ISPC](https://ispc.github.io/ispc.html)[^3] 编译器（在 [@sec:ISPC] 中讨论）采用了略有不同的方式报告缺失的优化。它对编译为相对低效代码的代码构造发出警告。无论哪种方式，编译器优化报告都应该是你工具箱中的关键工具之一。它们是检查特定热点完成了哪些优化以及查看是否有重要优化失败的快速方法。我通过编译器优化报告发现了许多改进机会。

[^1]: 使用编译器优化 pragma - [https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts](https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts)
[^3]: ISPC - [https://ispc.github.io/ispc.html](https://ispc.github.io/ispc.html)
[^5]: 链接时优化，也称为过程间优化（IPO）。更多信息请阅读：[https://en.wikipedia.org/wiki/Interprocedural_optimization](https://en.wikipedia.org/wiki/Interprocedural_optimization)
[^6]: LLVM 编译器 remarks - [https://llvm.org/docs/Remarks.html](https://llvm.org/docs/Remarks.html)
[^7]: opt-viewer - [https://github.com/llvm/llvm-project/tree/main/llvm/tools/opt-viewer](https://github.com/llvm/llvm-project/tree/main/llvm/tools/opt-viewer)
[^8]: optview2 - [https://github.com/OfekShilon/optview2](https://github.com/OfekShilon/optview2)
[^9]: 截至撰写时（2024），MSVC 仅提供向量化报告。
