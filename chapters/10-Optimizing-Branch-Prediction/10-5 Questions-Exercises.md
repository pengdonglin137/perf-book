## 问题和练习 {.unlisted .unnumbered}

\markright{问题和练习}

1. 重新审视 [@lst:LookupBranches] 中右边显示的代码示例。假设我们开始频繁获得 `[0-50)` 范围之外的数字。这将为保护对 `buckets` 数组越界访问的分支引入许多新的预测错误。你将如何更改代码来消除这些新引入的预测错误？
2. 使用我们在本章中讨论的技术解决以下实验室作业：
- `perf-ninja::branches_to_cmov_1`
- `perf-ninja::lookup_tables_1`
- `perf-ninja::virtual_call_mispredict`
- `perf-ninja::conditional_store_1`
3. 运行你日常使用的应用程序。收集 TMA 分解并检查 `BadSpeculation` 指标。查看被归因于最多分支预测错误的代码。是否有使用我们在本章中讨论的技术来避免分支的方法？

**编码练习**：编写一个微基准测试，该测试将经历 50% 的预测错误率或尽可能接近。你的目标是编写一半分支指令被预测错误的代码。这不像你想的那么简单。一些提示和想法：

* 分支预测错误率衡量为 `BR_MISP_RETIRED.ALL_BRANCHES / BR_INST_RETIRED.ALL_BRANCHES`。
* 如果你用 C++ 编码，你可以使用 1) 类似于 perf-ninja 的 Google benchmark 库，2) 编写一个普通的控制台程序并使用 Linux `perf` 收集 CPU 计数器，或者 3) 将 libpfm 库集成到微基准测试中（参见 [@sec:MarkerAPI]）。
* 没有必要发明一些复杂的算法。一个简单的方法是生成 `[0;100)` 范围内的伪随机数，并检查它是否小于 50。随机数可以提前预生成。
* 请记住，现代 CPU 可以记住长（但仍然有限）的分支结果序列。
