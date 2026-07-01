## 函数分割

函数分割背后的思想是将热代码与冷代码分离。这种转换通常也称为*函数轮廓化*。这种优化对于具有复杂控制流图和热路径中大块冷代码的相对较大的函数是有益的。[@lst:FunctionSplitting1] 中显示了这种转换可能有益的代码示例。为了从热路径中移除冷基本块，我们剪切并将它们粘贴到一个新函数中，并创建对它的调用。

Listing: 函数分割：冷代码轮廓化到新函数。

~~~~ {#lst:FunctionSplitting1 .cpp}
void foo(bool cond1,                void foo(bool cond1,
         bool cond2) {                       bool cond2) {
  // 热路径                         // 热路径
  if (cond1) {                        if (cond1) {
    /* 冷代码 (1) */                 cold1(); 
  }                                   }
  // 热路径                         // 热路径
  if (cond2) {              =>        if (cond2) {
    /* 冷代码 (2) */                 cold2(); 
  }                                   }
}                                   }
                                    void cold1() __attribute__((noinline)) 
                                    { /* 冷代码 (1) */ }
                                    void cold2() __attribute__((noinline))
                                    { /* 冷代码 (2) */ }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

注意，我们通过使用 `noinline` 属性禁用了冷函数的内联。因为如果没有它，编译器可能会决定内联它，这将有效地撤销我们的转换。或者，我们可以在 `cond1` 和 `cond2` 分支上应用 `[[unlikely]]` 宏（参见 [@sec:secLIKELY]），以向编译器传达不希望内联 `cold1` 和 `cold2` 函数。

<div id="fig:FunctionSplitting">
![默认布局](../../img/cpu_fe_opts/FunctionSplitting_Default.png){#fig:FuncSplit_default width=50%}
![改进布局](../../img/cpu_fe_opts/FunctionSplitting_Improved.png){#fig:FuncSplit_better width=50%}

将冷代码分割到单独的函数中。
</div>

图 @fig:FunctionSplitting 给出了此转换的图形表示。在改进的布局中，我们在热路径中只保留了一个 `CALL` 指令，下一个热指令很可能与前一个位于同一缓存行中。这改善了 CPU 前端数据结构（如 I-cache 和 $\mu$op-cache）的利用率。

轮廓化函数应该创建在 `.text` 段之外，例如在 `.text.cold` 中。如果函数从未被调用，这可以改善内存占用，因为它不会在运行时加载到内存中。
