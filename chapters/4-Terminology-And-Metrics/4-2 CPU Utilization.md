## CPU 利用率

CPU 利用率是核心在一段时间内处于繁忙状态的时间百分比。从技术上讲，当 CPU 没有运行内核 `idle` 线程时，就被认为是正在被利用。

$$
CPU~Utilization = \frac{CPU\_CLK\_UNHALTED.REF\_TSC}{TSC},
$$

其中 `CPU_CLK_UNHALTED.REF_TSC` 计算核心未处于停机状态时的参考周期数。`TSC` 代表时间戳计数器（在 [@sec:timers] 中讨论），它一直在计时。

如果 CPU 利用率低，通常意味着应用程序性能较差，因为部分时间被 CPU 浪费了。然而，高 CPU 利用率并不总是良好性能的指标。它只是系统正在做一些工作的标志，但没有说明它在做什么：即使 CPU 在等待内存访问时停顿，它也可能被高度利用。在多线程上下文中，线程在等待资源继续时也可能自旋。稍后，在 [@sec:secMT_metrics] 中，我们将讨论并行效率指标，特别是查看过滤自旋时间的"有效 CPU 利用率"。

Linux `perf` 自动计算系统中所有 CPU 的 CPU 利用率：

```bash
$ perf stat -- a.exe
  0.634874  task-clock (msec) #    0.773 CPUs utilized   
```
