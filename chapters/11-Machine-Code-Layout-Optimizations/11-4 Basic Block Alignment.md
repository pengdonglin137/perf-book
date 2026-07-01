## 基本块对齐

有时性能可能会根据指令在内存中放置的偏移量而发生显著变化。考虑 [@lst:LoopAlignment] 中显示的一个简单函数，以及使用 `-O3 -march=core-avx2 -fno-unroll-loops`（禁用循环展开以说明该思想）编译时相应的机器代码。

Listing: 基本块对齐

~~~~ {#lst:LoopAlignment .cpp}
void benchmark_func(int* a) {    │ 00000000004046a0 <_Z14benchmark_funcPi>:
  for (int i = 0; i < 32; ++i)   │ 4046a0: mov rax,0xffffffffffffff80
    a[i] += 1;                   │ 4046a7: vpcmpeqd ymm0,ymm0,ymm0
}                                │ 4046ab: nop DWORD [rax+rax+0x0]
                                 │ 4046b0: vmovdqu ymm1,[rdi+rax+0x80] # 循环开始
                                 │ 4046b9: vpsubd ymm1,ymm1,ymm0
                                 │ 4046bd: vmovdqu [rdi+rax+0x80],ymm1
                                 │ 4046c6: add rax,0x20
                                 │ 4046ca: jne 4046b0                  # 循环结束
                                 │ 4046cc: vzeroupper 
                                 │ 4046cf: ret 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

代码本身是合理的，但其布局并不完美（参见@fig:Loop_default）。对应于循环的指令以黄色高亮显示。粗框表示缓存行边界。缓存行长 64 字节。

<div id="fig:LoopLayout">
![默认布局](../../img/cpu_fe_opts/LoopAlignment_Default.png){#fig:Loop_default width=100%}

![改进布局](../../img/cpu_fe_opts/LoopAlignment_Better.png){#fig:Loop_better width=100%}

[@lst:LoopAlignment] 中循环的两种不同代码布局。
</div>

注意循环跨越多个缓存行：它开始于缓存行 `0x80-0xBF`，结束于缓存行 `0xC0-0xFF`。为了获取循环中执行的指令，处理器需要读取两个缓存行。这种情况有时会导致 CPU 前端出现性能问题，特别是对于像 [@lst:LoopAlignment] 中所示的小循环。

为了解决这个问题，我们可以使用单个 NOP 指令将循环指令向前移动 16 字节，使整个循环驻留在一个缓存行中。@fig:Loop_better 显示了使用 NOP 指令（以蓝色高亮显示）执行此操作的效果。

有趣的是，即使你只在微基准测试中运行这个热循环，性能影响也是可见的。这有点令人费解，因为代码量很小，在任何现代 CPU 上都不应该饱和 L1 I-cache 大小。@fig:Loop_better 中布局性能更好的原因并不简单，将涉及相当多的微架构细节，我们不在本书中讨论。感兴趣的读者可以在 Easyperf 博客的相关文章中找到更多信息。[^1]

默认情况下，LLVM 编译器识别循环并在 16B 边界处对齐它们，正如我们在@fig:Loop_default 中看到的。要达到我们示例中所需代码放置（如@fig:Loop_better 所示），你可以使用 `-mllvm -align-all-blocks=5` 选项，该选项将在对象文件中将每个基本块对齐到 32 字节边界。但是，我不建议使用此选项和类似选项，因为它们会影响翻译单元中所有函数的代码布局。还有其他侵入性较小的选项。
