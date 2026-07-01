## 收集性能监控事件 {#sec:counting}

性能监控计数器（PMCs）是低级性能分析的重要工具。它们可以提供关于程序执行的独特信息。PMCs 通常以两种模式使用："计数"或"采样"。计数模式主要用于计算我们在 [@sec:PerfMetrics] 中讨论的各种性能指标。采样模式用于查找热点，我们稍后将讨论。

计数背后的思想非常简单：我们想在程序运行时计算某些性能监控事件的总数。PMCs 在 Top-down 微架构分析（TMA）方法中被大量使用，我们将在 [@sec:TMA] 中详细讨论。@fig:Counting 说明了从程序开始到结束计算性能事件的过程。

![计算性能事件。](../../img/perf-analysis/CountingFlow.png){#fig:Counting width=80%}

@fig:Counting 中概述的步骤大致代表了典型分析工具计算性能事件的过程。`perf stat` 工具中实现了类似的过程，该工具可用于计算各种硬件事件，如指令数、周期数、缓存未命中数等。以下是 `perf stat` 输出的示例：

```bash
$ perf stat -- ./my_program.exe
 10580290629  cycles         #    3,677 GHz
  8067576938  instructions   #    0,76  insn per cycle
  3005772086  branches       # 1044,472 M/sec
   239298395  branch-misses  #    7,96% of all branches 
```

这些数据可能变得非常有用。首先，它使我们能够快速发现一些异常，例如高分支预测错误率或低 IPC。此外，当你进行了代码更改并想验证更改是否提高了性能时，它可能派上用场。查看相关事件可能有助于你证明或拒绝代码更改。`perf stat` 工具可用作轻量级基准测试包装器。它可以作为性能调查的第一步。有时可以立即发现异常，这可以为你节省一些分析时间。

可用事件名称的完整列表可以通过 `perf list` 查看：

```bash
$ perf list
  cycles            [Hardware event]
  ref-cycles        [Hardware event]
  instructions      [Hardware event]
  branches          [Hardware event]
  branch-misses     [Hardware event]
  ...
cache:
  mem_load_retired.l1_hit
  mem_load_retired.l1_miss
  ...
```

现代 CPU 有数百个可观察的性能事件。很难记住所有这些事件及其含义。理解何时使用特定事件更加困难。这就是为什么通常我不建议手动收集特定事件，除非你真的知道自己在做什么。相反，我建议使用 Intel VTune Profiler 等工具，它们可以自动收集所需事件来计算各种指标。

性能事件并非在每个环境中都可用，因为访问 PMCs 需要 root 权限，而虚拟化环境中运行的应用程序通常没有这个权限。对于在公共云中执行的程序，如果虚拟机（VM）管理器没有向客户机正确暴露 PMU 编程接口，直接在客户机容器中运行基于 PMU 的分析器不会产生有用的输出。因此，基于 CPU 性能监控计数器的分析器在虚拟化和云环境中工作得不太好 [@PMC_virtual]，尽管情况正在改善。VMware® 是最早启用[^4]虚拟性能监控计数器（vPMC）的 VM 管理器之一。AWS EC2 云也为专用主机启用了[^5] PMCs。

[^4]: VMware vPMC - [https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.vm_admin.doc/GUID-2E04FD07-8382-4D4E-B7B2-99F1F5AE7F37.html](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.vm_admin.doc/GUID-2E04FD07-8382-4D4E-B7B2-99F1F5AE7F37.html)

[^5]: AWS EC2 PMCs - [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-performance-counters.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-performance-counters.html)
