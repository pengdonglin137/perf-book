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

通过检查上面的优化报告，我们可以看到该循环没有被向量化，而是被展开了。对于开发人员来说，识别 [@lst:optReport] 中第 6 行循环中的循环携带依赖并不总是容易的。`c[i-1]` 加载的值依赖于前一次迭代的存储（参见图 @fig:VectorDep 中的操作 \circled{2} 和 \circled{3}）。可以通过手动展开循环的前几次迭代来揭示依赖关系：

```cpp
 
// 迭代 1
  a[1] = c[0];
