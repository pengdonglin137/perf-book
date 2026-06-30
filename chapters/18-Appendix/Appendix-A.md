\phantomsection
# 附录 A. 减少测量噪声 {.unnumbered}

\markboth{附录 A}{附录 A}

以下是一些可能导致性能测量中非确定性增加的功能示例，以及一些减少噪声的技术。我在 [@sec:secFairExperiments] 中介绍了该主题。

本节主要特定于 Linux 操作系统。鼓励读者在网上搜索有关如何配置其他操作系统的说明。

## 动态频率缩放 {.unlisted .unnumbered}

[动态频率缩放](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling)[^11]（DFS）是一种通过在系统运行要求高的任务时自动提高 CPU 运行频率来提高系统性能的技术。作为 DFS 实现的一个示例，Intel CPU 具有称为 Turbo Boost 的功能，AMD CPU 使用 Turbo Core 功能。

以下是 Turbo Boost 对在 Intel® Core™ i5-8259U 上运行的单线程工作负载的影响示例：

```bash
# TurboBoost 已启用
$ cat /sys/devices/system/cpu/intel_pstate/no_turbo
0
$ perf stat -e task-clock,cycles -- ./a.exe 
    11984.691958  task-clock (msec) #    1.000 CPUs utilized
  32,427,294,227  cycles            #    2.706 GHz
      11.989164338 seconds time elapsed

# TurboBoost 已禁用
$ echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
1
$ perf stat -e task-clock,cycles -- ./a.exe 
    13055.200832  task-clock (msec) #    0.993 CPUs utilized
  29,946,969,255  cycles            #    2.294 GHz
      13.142983989 seconds time elapsed
```

启用 Turbo Boost 时平均频率更高（2.7 GHz 对 2.3 GHz）。

DFS 可以在 BIOS 中永久禁用。要在 Linux 系统上以编程方式禁用 DFS 功能，你需要 root 访问权限。以下是如何实现这一点：

```bash
# Intel
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
```
