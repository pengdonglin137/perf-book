## Windows 事件跟踪 {#sec:ETW}

Microsoft 开发了一个名为 Windows 事件跟踪（ETW）的系统级跟踪设施。它最初旨在帮助设备驱动程序开发人员，但后来也用于分析通用应用程序。ETW 在所有受支持的 Windows 平台（x86 和 ARM）上可用，具有相应的平台相关安装包。ETW 在用户和内核代码中记录结构化事件，具有完整的调用栈跟踪支持，使你能够观察运行系统中的软件动态并解决许多具有挑战性的性能问题。

### 如何配置 {.unlisted .unnumbered}

从 Windows 10 开始，使用 `WPR.exe` 无需额外下载即可录制 ETW 数据。但要启用系统级分析，你必须是管理员并启用 `SeSystemProfilePrivilege`。Windows 性能记录器工具支持一组内置录制配置文件，适用于常见的性能问题。你可以通过编写具有 `.wprp` 扩展名的自定义性能记录器配置文件 XML 文件来定制你的录制需求。

如果你想不仅录制而且查看录制的 ETW 数据，你需要安装 Windows 性能工具包（WPT）。你可以从 Windows SDK[^1] 或 ADK[^2] 下载页面下载它。Windows SDK 很大；你不一定需要它的所有部分。在我们的案例中，我们只启用了 Windows 性能工具包的复选框。你被允许将 WPT 作为你自己应用程序的一部分重新分发。

### 你能用它做什么： {.unlisted .unnumbered}

- 使用可配置的 CPU 采样率（从 125 微秒到 10 秒）识别热点。默认值为 1 毫秒，成本约为 5-10% 的运行时开销。
- 确定什么阻塞了某个线程以及多长时间（例如，延迟的事件信号、不必要的线程休眠等）。
- 检查磁盘提供读/写请求的速度，并发现什么启动了该工作。
- 检查文件访问性能和模式（包括导致无磁盘 IO 的缓存读/写）。
- 跟踪 TCP/IP 堆栈以查看数据包如何在网络接口和计算机之间流动。

上面列出的所有项目都在系统范围内为所有进程录制，具有可配置的调用栈跟踪（内核和用户模式调用栈被组合）。也可以添加你自己的 ETW 提供程序，将系统级跟踪与你的应用程序行为关联起来。你可以通过检测代码来扩展收集的数据量。例如，你可以在函数中注入进入/离开 ETW 跟踪钩子到你的源代码中，以测量某个函数被执行的频率。

### 你不能用它做什么： {.unlisted .unnumbered}

ETW 跟踪不适用于检查 CPU 微架构瓶颈。为此，请使用供应商特定的工具，如 Intel VTune、AMD uProf、Apple Instruments 等。

ETW 跟踪捕获系统级别所有进程的动态，但它可能会生成大量数据。例如，捕获线程上下文切换数据以观察各种等待和延迟，每分钟很容易生成 1-2 GB。这就是为什么在不覆盖先前存储的跟踪的情况下记录高容量事件数小时是不切实际的。

如果你想了解更多关于 ETW 的信息，附录 D 中有更详细的讨论。我们探讨了录制和分析 ETW 的工具，并介绍了调试程序启动缓慢的案例研究。

[^1]: Windows SDK 下载 - [https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/](https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/)
[^2]: Windows ADK 下载 - [https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)
