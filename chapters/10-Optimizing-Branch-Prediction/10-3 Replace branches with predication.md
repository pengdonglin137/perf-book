## 用选择替换分支 {#sec:BranchlessSelection}

某些分支可以通过执行分支的两个部分然后选择正确的结果来有效消除。[@lst:ReplaceBranchesWithSelection] 中显示了这种转换可能有益的代码示例。如果 TMA 建议 `if (cond)` 分支具有非常多的预测错误，你可以尝试通过右边所示的转换来消除分支。

清单：用选择替换分支。

~~~~ {#lst:ReplaceBranchesWithSelection .cpp}
int a;                                             int x = computeX();
if (cond) { /* 经常预测错误 */   =>     int y = computeY();
  a = computeX();                                  int a = cond ? x : y;
} else {                                           foo(a);
  a = computeY();
}
foo(a);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于右边的代码，编译器可以替换来自三元运算符的分支，并生成 `CMOV` x86 指令。`CMOVcc` 指令检查 `EFLAGS` 寄存器中一个或多个状态标志（`CF`、`OF`、`PF`、`SF` 和 `ZF`）的状态，并在标志处于指定状态或条件时执行移动操作。类似的转换可以使用 `FCMOVcc` 和 `VMAXSS/VMINSS` 指令对浮点数完成。在 ARM ISA 中，有 `CSEL`（条件选择）指令，但也有 `CSINC`（选择并递增）、`CSNEG`（选择并取反）和其他一些条件指令。

清单：用选择替换分支 - x86 汇编代码。

~~~~ {#lst:ReplaceBranchesWithSelectionAsm .bash}
# 原始版本              # 无分支版本
400504: test ebx,ebx            400537: mov eax,0x0
400506: je 400514               40053c: call <computeX> # 计算 x; a = x
400508: mov eax,0x0             400541: mov ebp,eax     # ebp = x
40050d: call <computeX>    =>   400543: mov eax,0x0
400512: jmp 40051e              400548: call <computeY> # 计算 y; a = y
400514: mov eax,0x0             40054d: test ebx,ebx    # 测试 cond
400519: call <computeY>         40054f: cmovne eax,ebp  # 如果需要，用 x 覆盖 a
40051e: mov edi,eax             400552: mov edi,eax
400521: call <foo>              400554: call <foo>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[@lst:ReplaceBranchesWithSelectionAsm] 显示了原始版本和无分支版本的汇编列表。与原始版本相比，无分支版本没有跳转指令。但是，无分支版本独立计算 `x` 和 `y`，然后选择一个值并丢弃另一个。虽然这种转换消除了分支预测错误的惩罚，但它比原始代码做了更多的工作。

我们已经知道原始版本左边的分支很难预测。这就是我们首先尝试无分支版本的动机。在这个例子中，此更改的性能增益取决于 `computeX` 和 `computeY` 函数的特性。如果函数很小[^1] 并且编译器可以内联它们，那么选择可能会带来显著的性能优势。如果函数很大[^2]，承担分支预测错误的成本可能比执行 `computeX` 和 `computeY` 函数更便宜。最终，性能测量总是决定哪个版本更好。

再看看 [@lst:ReplaceBranchesWithSelectionAsm]。在左边，处理器可以预测，例如，`je 400514` 分支将被采取，推测地调用 `computeY`，并开始从函数 `foo` 运行代码。记住，分支预测发生在我们知道分支实际结果之前的许多周期。到我们开始解析分支时，我们可能已经完成了 `foo` 函数的一半，尽管它仍然是推测性的。如果我们正确，我们节省了很多周期。如果我们错误，我们必须承担惩罚并从正确路径重新开始。在后一种情况下，我们不会从已经完成 `foo` 的一部分这一事实中获得任何收益，它都必须被丢弃。如果预测错误发生得太频繁，恢复惩罚就会超过推测执行的收益。

使用条件选择，情况就不同了。没有分支，所以处理器不必推测。它可以并行执行 `computeX` 和 `computeY` 函数。但是，在计算 `CMOVNE` 指令的结果之前，它无法开始从 `foo` 运行代码，因为 `foo` 将其用作参数（数据依赖）。当你使用条件选择指令时，你将控制流依赖转换为数据流依赖。
