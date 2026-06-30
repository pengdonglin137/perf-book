## Intel VTune Profiler {#sec:IntelVtuneOverview}

VTune Profiler（以前称为 VTune Amplifier）是一个用于 x86 机器的性能分析工具，具有丰富的图形界面。它可以在 Linux 或 Windows 操作系统上运行。我们跳过讨论 VTune 对 MacOS 的支持，因为它在 Apple 的芯片（如 M1 和 M2）上不起作用，而且基于 Intel 的 Macbook 正在迅速过时。

VTune 可以在 Intel 和 AMD 系统上使用。但是，高级基于硬件的采样需要 Intel 制造的 CPU。例如，你将无法在 AMD 系统上使用 Intel VTune 收集硬件性能计数器。

截至 2023 年初，VTune 作为独立工具或作为 Intel oneAPI Base Toolkit 的一部分免费提供。[^1]

### 如何配置 {.unlisted .unnumbered}

在 Linux 上，VTune 可以使用两种数据收集器：Linux perf 和 VTune 自己的驱动程序 SEP。第一种类型用于用户模式采样，但如果你想执行高级分析，你需要构建并安装 SEP 驱动程序，这并不太难。

```bash
# 转到 vtune 安装目录中的 sepdk 文件夹
$ cd ~/intel/oneapi/vtune/latest/sepdk/src
# 构建驱动程序
$ ./build-driver
# 添加 vtune 组并将你的用户添加到该组
# 创建新的 shell，或重启系统
$ sudo groupadd vtune
$ sudo usermod -a -G vtune `whoami`
# 安装 sep 驱动程序
$ sudo ./insmod-sep -r -g vtune
```

完成上述步骤后，你应该能够使用高级分析类型，如微架构探索和内存访问。

Windows 在安装 VTune 后不需要任何额外配置。收集硬件性能事件需要管理员权限。

### 你能用它做什么： {.unlisted .unnumbered}

- 查找热点：函数、循环、语句。
- 监控各种 CPU 特定的性能事件，例如分支预测错误和 L3 缓存未命中。
- 定位发生这些事件的代码行。
- 使用 TMA 方法表征 CPU 性能瓶颈。
- 过滤特定函数、进程、时间段或逻辑核心的数据。
- 观察工作负载随时间的行为（包括 CPU 频率、内存带宽利用率等）。

VTune 可以提供关于正在运行的进程的非常丰富的信息。如果你想要提高应用程序的整体性能，它是正确的工具。VTune 总是提供一段时间内的聚合数据，因此它可以用于查找"平均情况"的优化机会。
