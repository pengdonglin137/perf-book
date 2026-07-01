## 基本块放置 {#sec:secLIKELY}

假设我们有一个热路径，其中有一些错误处理代码（`coldFunc`）：

```cpp
// 热路径
if (cond)
  coldFunc();
// 热路径继续
```
@fig:BBLayout 显示了此代码片段的两种可能的物理布局。@fig:BB_default 是大多数编译器在没有提供提示时默认发出的布局。如果我们反转条件 `cond` 并将热代码作为直通放置，则可以实现@fig:BB_better 中显示的布局。

<div id="fig:BBLayout">
![默认布局](../../img/cpu_fe_opts/BBLayout_Default.png){#fig:BB_default width=50%}
![改进布局](../../img/cpu_fe_opts/BBLayout_Better.png){#fig:BB_better width=50%}

上述代码片段的两种机器代码布局版本。
</div>

哪种布局更好？好吧，这取决于 `cond` 通常为真还是假。如果 `cond` 通常为真，那么我们最好选择默认布局，否则我们将执行两次跳转而不是一次。此外，如果 `coldFunc` 是一个相对较小的函数，我们希望将其内联。但是，在这个特定示例中，我们知道 `coldFunc` 是一个错误处理函数，可能不经常执行。通过选择布局 @fig:BB_better，我们保持热代码之间的直通，并将采取的分支转换为未采取的分支。

@fig:BB_better 中显示的布局性能更好有几个原因。首先，@fig:BB_better 中的布局更好地利用了指令和 $\mu$op 缓存（DSB，参见 [@sec:uarchFE]）。所有热代码连续，没有缓存行碎片：L1 I-cache 中的所有缓存行都由热代码使用。$\mu$op 缓存也是如此，因为它也基于底层代码布局进行缓存。其次，采取的分支对获取单元来说也更昂贵。CPU 的前端获取连续对齐的字节块，通常为 16、32 或 64 字节，具体取决于架构。对于每个采取的分支，跳转指令之后和分支目标之前的获取块中的字节未被使用。这降低了最大有效获取吞吐量。最后，在某些架构上，未采取的分支从根本上比采取的分支更便宜。例如，Intel Skylake CPU 每个周期可以执行两个未采取的分支，但每两个周期只能执行一个采取的分支。[^2]

要建议编译器生成改进版本的机器代码布局，你可以使用 `[[likely]]` 和 `[[unlikely]]` 属性提供提示，这些属性自 C++20 以来可用。使用此提示的代码如下所示：

```cpp
// 热路径
if (cond) [[unlikely]] 
  coldFunc();
// 热路径继续
```

在上面的代码中，`[[unlikely]]` 提示将指示编译器 `cond` 不太可能为真，因此编译器应该相应地调整代码布局。在 C++20 之前，开发人员可以使用 [`__builtin_expect`](https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect)[^3] 构造。他们通常创建 `LIKELY` 包装器提示以使代码更可读。例如：

```cpp
#define LIKELY(EXPR)   __builtin_expect((bool)(EXPR), true)
#define UNLIKELY(EXPR) __builtin_expect((bool)(EXPR), false)
// 热路径
if (UNLIKELY(cond)) // NOT 
  coldFunc();
```

优化编译器在遇到 "likely/unlikely" 提示时不仅会改进代码布局。它们还会在其他地方利用此信息。例如，当应用 `[[unlikely]]` 属性时，编译器将阻止内联 `coldFunc`，因为它现在知道该函数不太可能经常执行，优化其大小更有利，即只留下一个 `CALL` 调用此函数。

在 switch 语句中也可以插入 `[[likely]]` 属性，如 [@lst:BuiltinSwitch] 所示。使用此提示，编译器将能够以稍有不同的方式重排代码，并优化热 switch 以更快地处理 `ADD` 指令。

Listing: switch 语句中使用 likely 属性

~~~~ {#lst:BuiltinSwitch .cpp}
for (;;) {
  switch (instruction) {
               case NOP: handleNOP(); break;
    [[likely]] case ADD: handleADD(); break;
               case RET: handleRET(); break;
    // handle other instructions
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[^2]: However, there is a special small loop optimization that allows very small loops to have one taken branch per cycle.
[^3]: More about builtin-expect here: [https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect](https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect).
[^10]: C++ standard `[[likely]]` attribute: [https://en.cppreference.com/w/cpp/language/attributes/likely](https://en.cppreference.com/w/cpp/language/attributes/likely).
