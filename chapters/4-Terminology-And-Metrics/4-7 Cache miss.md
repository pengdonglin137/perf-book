## 缓存未命中

如 [@sec:MemHierar] 中所讨论的，在特定缓存级别未命中的任何内存请求必须由更高级别的缓存或 DRAM 提供服务。这意味着此类内存访问的延迟显著增加。内存子系统组件的典型延迟如表 {@tbl:mem_latency} 所示。当内存请求在最后一级缓存（LLC）中未命中并一路下降到主内存时，性能会受到严重影响。[^3]

-------------------------------------------------
内存层次结构组件        延迟（周期/时间）

--------------------------   --------------------
L1 缓存                    4 周期（~1 纳秒）

L2 缓存                    10-25 周期（5-10 纳秒）

L3 缓存                    ~40 周期（20 纳秒）

主内存                     200+ 周期（100 纳秒）

-------------------------------------------------

表：x86 平台上内存子系统的典型延迟。{#tbl:mem_latency}

指令和数据获取都可能在缓存中未命中。根据 Top-down 微架构分析（参见 [@sec:TMA]），指令缓存（I-cache）未命中被描述为前端停顿，而数据缓存（D-cache）未命中被描述为后端停顿。指令缓存未命中发生在 CPU 流水线中非常早的指令获取阶段。数据缓存未命中发生在指令执行阶段的较晚时候。

Linux `perf` 用户可以通过运行以下命令收集 L1 缓存未命中数：

```bash
$ perf stat -e mem_load_retired.fb_hit,mem_load_retired.l1_miss,
  mem_load_retired.l1_hit,mem_inst_retired.all_loads -- a.exe
   29580  mem_load_retired.fb_hit
   19036  mem_load_retired.l1_miss
  497204  mem_load_retired.l1_hit
  546230  mem_inst_retired.all_loads
```

以上是 L1 数据缓存和填充缓冲区的所有加载的细分。加载可能命中已分配的填充缓冲区（`fb_hit`）、命中 L1 缓存（`l1_hit`），或者两者都未命中（`l1_miss`），因此 `all_loads = fb_hit + l1_hit + l1_miss`。[^2] 我们可以看到只有 3.5% 的所有加载在 L1 缓存中未命中，因此 *L1 命中率*为 96.5%。

我们可以通过运行以下命令进一步细分 L1 数据未命中并分析 L2 缓存行为：

```bash
$ perf stat -e mem_load_retired.l1_miss,
