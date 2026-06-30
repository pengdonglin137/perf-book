\phantomsection
# 附录 D. Windows 事件跟踪分析 {.unnumbered}

\markboth{附录 D}{附录 D}

我们在 [@sec:ETW] 中介绍了 ETW。在本节中，我们将继续之前的内容，探索用于记录和分析 ETW 的工具。为了展示这些工具的外观，我们介绍了一个调试程序启动缓慢的案例研究。

### 记录 ETW 跟踪的工具 {.unlisted .unnumbered}

以下是你可用于捕获 ETW 跟踪的工具列表：

- `WPR.exe`：命令行记录工具，Windows 10 和 Windows Performance Toolkit 的一部分。
- `WPRUI.exe`：用于记录 ETW 数据的简单 UI，Windows Performance Toolkit 的一部分
- `xperf`：wpr 的命令行前身，Windows Performance Toolkit 的一部分。
- `PerfView`：[^3] 以 .NET 应用程序为主要焦点的图形记录和分析工具。这是 Microsoft 开发的开源应用程序。
- `Performance HUD`：[^7] 一个鲜为人知但非常强大的 GUI 工具，用于跟踪 UI 延迟以及通过所有未平衡资源分配的实时 ETW 记录的用户/句柄泄漏，以及泄漏/阻塞调用栈跟踪的实时显示。
- `ETWController`：[^4] 一个记录工具，能够记录键盘输入和屏幕截图以及 ETW 数据。这个由 Alois Kraus 开发的开源应用程序还支持在两台机器上同时进行分布式分析。
- `UIForETW`：[^6] 这个由 Bruce Dawson 开发的开源应用程序是 `xperf` 的包装器，具有用于记录 Google Chrome 问题的特殊选项。它还可以记录键盘和鼠标输入。

### 查看和分析 ETW 跟踪的工具 {.unlisted .unnumbered}

- `Windows Performance Analyzer`（WPA）：查看 ETW 数据的最强大的 UI。WPA 可以可视化和叠加磁盘、CPU、GPU、网络、内存、进程等多种数据源，以全面了解系统的行为和正在执行的操作。虽然 UI 非常强大，但对于初学者来说也可能相当复杂。WPA 支持插件来处理来自其他来源的数据，而不仅仅是 ETW 跟踪。可以导入由 Linux perf、LTTNG、Perfetto 等工具生成的 Linux/Android[^8] 分析数据，以及多种日志文件格式：dmesg、Cloud-Init、WaLinuxAgent 和 AndroidLogcat。
- `ETWAnalyzer`：[^5] 读取 ETW 数据并生成聚合摘要 JSON 文件，可以在命令行中查询、过滤和排序，或导出到 CSV 文件。
- `PerfView`：主要用于排查 .NET 应用程序。为垃圾回收和 JIT 编译触发的 ETW 事件被解析并易于作为报告或 CSV 数据访问。

### 案例研究 - 程序启动缓慢 {.unlisted .unnumbered}

现在我们将看一个使用 ETWController 捕获 ETW 跟踪和 WPA 进行可视化的示例。

**问题陈述**：在 Windows 资源管理器中双击下载的可执行文件时，启动有明显的延迟。似乎有什么东西延迟了进程启动。可能的原因是什么？磁盘慢？

#### 设置 {.unlisted .unnumbered}

- 下载 ETWController 以记录 ETW 数据和屏幕截图。
- 下载最新的 Windows 11 Performance Toolkit[^1]，以便能够使用 WPA 查看数据。确保较新的 Win 11 `WPR.exe` 在你的路径中排在第一位，方法是在系统环境对话框中将 WPT 的安装文件夹移到 `C:\\Windows\\system32` 之前。它应该是这样的：

```
C> where wpr 
C:\Program Files (x86)\Windows Kits\10\Windows Performance Toolkit\WPR.exe
C:\Windows\System32\WPR.exe
```
