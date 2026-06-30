### 使用标记 API {#sec:MarkerAPI}

在某些场景中，我们可能对分析特定代码区域的性能感兴趣，而不是整个应用程序。当你正在开发一段新代码并希望只关注该代码时，就会出现这种情况。自然地，你希望跟踪优化进度并捕获额外的性能数据来帮助你。大多数性能分析工具提供特定的*标记 API*，让你可以做到这一点。以下是两个例子：

* Intel VTune 有 `__itt_task_begin / __itt_task_end` 函数。
* AMD uProf 有 `amdProfileResume / amdProfilePause` 函数。

这种混合方法结合了检测和性能事件计数的优点。标记 API 允许我们将性能统计归因于代码区域（循环、函数）或功能部分（远程过程调用（RPC）、输入事件等），而不是度量整个程序。你获得的数据质量很容易证明其价值。例如，在调查仅在特定类型 RPC 中发生的性能错误时，你可以仅为该类型 RPC 启用监控。

下面我们提供一个使用 [libpfm4](https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/) 的非常基本的示例，[^1] 这是用于收集性能监控事件的流行 Linux 库之一。它建立在 Linux `perf_events` 子系统之上，该子系统允许你直接访问性能事件计数器。`perf_events` 子系统相当底层，因此 `libpfm4` 包在这里很有用，因为它添加了一个发现工具来识别 CPU 上可用的事件，以及围绕原始 `perf_event_open` 系统调用的包装库。下面的代码列表显示了如何使用 `libpfm4` 来检测 [C-Ray](https://openbenchmarking.org/test/pts/c-ray)[^2] 基准测试的 `render` 函数。

```cpp
+#include <perfmon/pfmlib.h>
+#include <perfmon/pfmlib_perf_event.h>
...
/* 将 xsz/ysz 维度的一帧渲染到提供的帧缓冲区 */
void render(int xsz, int ysz, uint32_t *fb, int samples) {
   ...
+  pfm_initialize();
+  struct perf_event_attr perf_attr;
+  memset(&perf_attr, 0, sizeof(perf_attr));
+  perf_attr.size = sizeof(struct perf_event_attr);
+  perf_attr.read_format = PERF_FORMAT_TOTAL_TIME_ENABLED | 
+                          PERF_FORMAT_TOTAL_TIME_RUNNING | PERF_FORMAT_GROUP;
+   
+  pfm_perf_encode_arg_t arg;
+  memset(&arg, 0, sizeof(pfm_perf_encode_arg_t));
+  arg.size = sizeof(pfm_perf_encode_arg_t);
+  arg.attr = &perf_attr;
+   
+  pfm_get_os_event_encoding("instructions", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  int leader_fd = perf_event_open(&perf_attr, 0, -1, -1, 0);
+  pfm_get_os_event_encoding("cycles", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  int event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+  pfm_get_os_event_encoding("branches", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+  pfm_get_os_event_encoding("branch-misses", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+
+  struct read_format { uint64_t nr, time_enabled, time_running, values[4]; };
