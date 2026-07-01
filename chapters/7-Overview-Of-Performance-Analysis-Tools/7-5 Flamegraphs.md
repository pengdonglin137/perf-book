## 火焰图 {#sec:secFlameGraphs}

火焰图是可视化分析数据和程序中最频繁代码路径的流行方式。它使我们能够看到哪些函数调用占用了执行时间的最大部分。@fig:FlameGraph 显示了 [x264](https://openbenchmarking.org/test/pts/x264) 视频编码基准测试的火焰图示例，由 Brendan Gregg 开发的开源 [scripts](https://github.com/brendangregg/FlameGraph)[^1] 生成。如今，几乎所有的分析器都可以自动生成火焰图，只要在分析会话期间收集了调用栈。

![x264 基准测试的火焰图。](../../img/perf-tools/Flamegraph.jpg){#fig:FlameGraph width=100%}

在火焰图上，每个矩形（水平条）代表一个函数调用，矩形的宽度表示函数本身及其被调用者所花费的相对执行时间。函数调用从底部到顶部发生，因此我们可以看到程序中最热的路径是 `x264` → `threadpool_thread_internal` → `...` → `x264_8_macroblock_analyse`。函数 `threadpool_thread_internal` 及其被调用者占程序花费时间的 74%。但自身时间，即函数本身花费的时间相当小。类似地，我们可以对 `x264_8_macroblock_analyse` 进行同样的分析，它占运行时的 66%。这种可视化让你对最时间花费在哪里有很好的直觉。

火焰图是交互式的。你可以点击图像上的任何条，它将放大到该特定代码路径。你可以持续放大，直到找到一个不符合你期望的地方或到达叶子/尾部函数——现在你有了可以在分析中使用的可操作信息。另一种策略是找出程序中最热的函数（从这个火焰图中不立即清楚），然后自底向上通过火焰图，试图理解这个最热的函数是从哪里被调用的。

一些工具更喜欢使用*冰柱图*，它是火焰图的倒置版本（参见 [@sec:ContinuousProfiling] 中的示例）。

[^1]: Brendan Gregg 的火焰图 - [https://github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph)
[^2]: x264 视频编码基准测试 - [https://openbenchmarking.org/test/pts/x264](https://openbenchmarking.org/test/pts/x264)
