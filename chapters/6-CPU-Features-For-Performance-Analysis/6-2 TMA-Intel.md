### Intel 平台上的 TMA {#sec:secTMA_Intel}

TMA 方法由 Intel 于 2014 年首次提出，从 Sandy Bridge 系列处理器开始支持。Intel 的实现支持每个高级桶的嵌套类别，可以更好地理解程序中的 CPU 性能瓶颈（参见@fig:TMA）。

工作流程旨在"深入"到 TMA 层次结构的较低级别，直到我们获得性能瓶颈的非常具体的分类。首先，我们收集四个主要桶的指标：`前端绑定`、`后端绑定`、`退休`和`错误推测`。假设我们发现程序执行的很大一部分被内存访问阻塞（这是`后端绑定`桶，参见@fig:TMA）。下一步是再次运行工作负载，仅收集`内存绑定`桶的特定指标。重复此过程，直到我们确切知道根本原因，例如 `L3 绑定`。

![性能瓶颈的 TMA 层次结构。*© 图片由 Ahmad Yasin 提供。*](../../img/pmu-features/TMAM.png){#fig:TMA width=90%}

多次运行工作负载是可以的，每次深入并专注于特定指标。但通常，运行一次工作负载并收集所有 TMA 级别所需的所有指标就足够了。分析工具通过在单次运行中在不同性能事件之间进行多路复用（参见 [@sec:secMultiplex]）来实现这一点。此外，在实际应用程序中，性能可能受多个因素限制。例如，它可能同时经历大量的分支预测错误（`错误推测`）和缓存未命中（`后端绑定`）。在这种情况下，TMA 将同时深入多个桶，并识别每种类型的瓶颈对程序性能的影响。Intel VTune Profiler、AMD uProf 和 Linux `perf` 等分析工具可以通过单次基准测试运行计算所有 TMA 指标。但是，这仅在工作负载稳定时才可接受。否则，你最好回到原始策略，进行多次运行并在每次运行中深入研究。

TMA 指标的前两个级别表示为程序执行期间可用的所有流水线槽（参见 [@sec:PipelineSlot]）的百分比。这允许 TMA 在考虑处理器完整带宽的情况下，给出 CPU 微架构利用率的准确表示。到目前为止，一切应该很好地加起来达到 100%。但是，从第 3 级开始，桶可能以不同的计数域表示，例如时钟和停顿。因此，它们不一定与其他 TMA 桶直接可比。

一旦我们识别出性能瓶颈，我们需要知道它在代码中的确切位置。TMA 的第二步是将问题源定位到确切的代码行和相应的汇编指令。分析方法为每个类别的性能问题提供了你应该使用的性能事件。然后你可以对该事件进行采样，以找到源代码中导致第一阶段识别的性能瓶颈的行。如果这个过程对你来说听起来很复杂，请不要担心；一旦你通读案例研究，一切都会变得清楚。

### 案例研究：使用 TMA 减少缓存未命中数 {.unlisted .unnumbered}

作为本案例研究的示例，我采用了一个非常简单的基准测试，这样它易于理解和更改。它不能代表真实世界的应用程序，但它足以演示 TMA 的工作流程。我在本书的第二部分中有更多实际示例。

本书的大多数读者可能会将 TMA 应用于他们熟悉的应用程序。但即使你第一次看到该应用程序，TMA 也非常有效。因此，我不从展示基准测试的源代码开始。但这里有一个简短的描述：基准测试在堆上分配一个 200 MB 的数组，然后进入 1 亿次迭代的循环。在循环的每次迭代中，它生成一个随机索引到已分配的数组中，执行一些虚拟工作，然后从该索引读取值。

我在配备 Intel Core i5-8259U CPU（基于 Skylake）和 16GB DRAM（DDR4 2400 MT/s）的机器上运行了实验，运行 64 位 Ubuntu 20.04（内核版本 5.13.0-27）。

### 第 1 步：识别瓶颈 {.unlisted .unnumbered}

作为第一步，我们运行微基准测试并收集一组有限的事件，这些事件将帮助我们计算第 1 级指标。在这里，我们尝试通过将高级性能瓶颈归因于四个 L1 桶来识别应用程序的高级性能瓶颈：`前端绑定`、`后端绑定`、`退休`和`错误推测`。可以使用 Linux `perf` 工具收集第 1 级指标。`perf stat` 命令有一个专用的 `--topdown` 选项。在更新的版本中，它将默认输出这些指标。以下是我们基准测试的细分。本节中所有命令的输出都经过修剪以节省空间。

```bash
$ perf stat -- ./benchmark.exe
...
  TopdownL1 (cpu_core)  #  53.4 %  tma_backend_bound    <==
                        #   0.2 %  tma_bad_speculation
                        #  13.8 %  tma_frontend_bound
                        #  32.5 %  tma_retiring
...
```

通过查看输出，我们可以判断应用程序的性能受 CPU 后端限制。让我们深入一层。为了获得 TMA 指标第 2 级、第 3 级和更高级别，我将使用 `toplev` 工具，它是 [pmu-tools](https://github.com/andikleen/pmu-tools)[^7] 的一部分，由 Andi Kleen 编写。它在 Python 中实现，并在底层使用 Linux `perf`。必须启用特定的 Linux 内核设置才能使用 `toplev`；有关更多详细信息，请检查文档。

```bash
$ ~/pmu-tools/toplev.py --core S0-C0 -l2 -v --no-desc taskset -c 0 ./benchmark.exe
...
# Level 1
S0-C0  Frontend_Bound:                13.92 % Slots
S0-C0  Bad_Speculation:                0.23 % Slots
S0-C0  Backend_Bound:                 53.39 % Slots
S0-C0  Retiring:                      32.49 % Slots
# Level 2
S0-C0  Frontend_Bound.FE_Latency:     12.11 % Slots
S0-C0  Frontend_Bound.FE_Bandwidth:    1.84 % Slots
S0-C0  Bad_Speculation.Branch_Mispred: 0.22 % Slots
S0-C0  Bad_Speculation.Machine_Clears: 0.01 % Slots
S0-C0  Backend_Bound.Memory_Bound:    44.59 % Slots <==
S0-C0  Backend_Bound.Core_Bound:       8.80 % Slots
S0-C0  Retiring.Base:                 24.83 % Slots
S0-C0  Retiring.Microcode_Sequencer:   7.65 % Slots
```

在此命令中，我们将进程固定到 CPU0（使用 `taskset -c 0`）并将 `toplev` 的输出限制到此核心（`--core S0-C0`）。选项 `-l2` 告诉工具收集第 2 级指标。选项 `--no-desc` 禁用每个指标的描述。

我们可以看到应用程序的性能受内存访问限制（`Backend_Bound.Memory_Bound`）。几乎一半的 CPU 执行资源在等待内存请求完成时被浪费了。现在让我们再深入一层：[^17]

```bash
$ ~/pmu-tools/toplev.py --core S0-C0 -l3 -v --no-desc taskset -c 0 ./benchmark.exe
...
# Level 1
S0-C0    Frontend_Bound:                 13.91 % Slots
S0-C0    Bad_Speculation:                 0.24 % Slots
S0-C0    Backend_Bound:                  53.36 % Slots
S0-C0    Retiring:                       32.41 % Slots
# Level 2
S0-C0    FE_Bound.FE_Latency:            12.10 % Slots
S0-C0    FE_Bound.FE_Bandwidth:           1.85 % Slots
S0-C0    BE_Bound.Memory_Bound:          44.58 % Slots
S0-C0    BE_Bound.Core_Bound:             8.78 % Slots
# Level 3
S0-C0-T0 BE_Bound.Mem_Bound.L1_Bound:     4.39 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L2_Bound:     2.42 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L3_Bound:     5.75 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.DRAM_Bound:  47.11 % Stalls <==
S0-C0-T0 BE_Bound.Mem_Bound.Store_Bound:  0.69 % Stalls
S0-C0-T0 BE_Bound.Core_Bound.Divider:     8.56 % Clocks
S0-C0-T0 BE_Bound.Core_Bound.Ports_Util: 11.31 % Clocks
```

我们发现瓶颈在 `DRAM_Bound`。这告诉我们许多内存访问在所有级别的缓存中都未命中，一路到达主内存。如果我们收集程序的 L3 缓存未命中绝对数量，也可以确认这一点。对于 Skylake 架构，`DRAM_Bound` 指标使用 `CYCLE_ACTIVITY.STALLS_L3_MISS` 性能事件计算。让我们手动收集它：

```bash
$ perf stat -e cycles,cycle_activity.stalls_l3_miss -- ./benchmark.exe
  32226253316  cycles
  19764641315  cycle_activity.stalls_l3_miss
```

`CYCLE_ACTIVITY.STALLS_L3_MISS` 事件计算执行停顿的周期，同时 L3 缓存未命中需求加载处于未完成状态。我们可以看到大约 60% 的周期是这样的，这非常糟糕。

### 第 2 步：定位代码中的位置 {.unlisted .unnumbered}

TMA 过程的第二步是定位在代码中已识别的性能事件发生最频繁的位置。为此，你应该使用在第 1 步中识别的瓶颈类型对应的事件对工作负载进行采样。

找到此类事件的推荐方法是运行带有 `--show-sample` 选项的 `toplev` 工具，该选项将建议可用于定位问题的 `perf record` 命令行。为了理解 TMA 的机制，我们还展示了手动查找与特定性能瓶颈关联的事件的方法。性能瓶颈与用于确定源代码中瓶颈位置的性能事件之间的对应关系可以在 [TMA metrics](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx)[^2] 表中找到。`Locate-with` 列表示用于定位问题发生的确切代码位置的性能事件。在我们的案例中，要找到导致 `DRAM_Bound` 指标如此高值（L3 缓存未命中）的内存访问，我们应该对 `MEM_LOAD_RETIRED.L3_MISS_PS` 精确事件进行采样。以下是示例命令：

```bash
$ perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp -- ./benchmark.exe
$ perf report -n --stdio
...
# Samples: 33K of event 'MEM_LOAD_RETIRED.L3_MISS'
# Event count (approx.): 71363893
# Overhead   Samples  Shared Object   Symbol
# ........  ......... ..............  .................
#
    99.95%    33811   benchmark.exe   [.] foo
     0.03%       52   [kernel]        [k] get_page_from_freelist
     0.01%        3   [kernel]        [k] free_pages_prepare
     0.00%        1   [kernel]        [k] free_pcppages_bulk
```

几乎所有的 L3 未命中都是由可执行文件 `benchmark.exe` 中函数 `foo` 中的内存访问引起的。现在是时候看看基准测试的源代码了，可以在 [GitHub](https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM) 上找到。[^8]

为了避免编译器优化，函数 `foo` 用汇编语言实现，如 [@lst:TMA_asm] 所示。基准测试的"驱动"部分在 `main` 函数中实现，如 [@lst:TMA_cpp] 所示。我们分配一个足够大的数组 `a`，使其无法放入 6MB 的 L3 缓存。基准测试生成数组 `a` 的随机索引，并将此索引与数组 `a` 的地址一起传递给 `foo` 函数。随后 `foo` 函数读取此随机内存位置。[^11]

Listing: 函数 foo 的汇编代码。

~~~~ {#lst:TMA_asm .bash}
$ perf annotate --stdio -M intel foo
Percent |  Disassembly of benchmark.exe for MEM_LOAD_RETIRED.L3_MISS
------------------------------------------------------------
        :  Disassembly of section .text:
        :
        :  0000000000400a00 <foo>:
        :  foo():
   0.00 :    400a00:  nop  DWORD PTR [rax+rax*1+0x0]
   0.00 :    400a08:  nop  DWORD PTR [rax+rax*1+0x0]
                 ...  # more NOPs
 100.00 :    400e07:  mov  rax,QWORD PTR [rdi+rsi*1] <==
                 ...
   0.00 :    400e13:  xor  rax,rax
   0.00 :    400e16:  ret
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Listing: 函数 main 的源代码。

~~~~ {#lst:TMA_cpp .cpp}
extern "C" { void foo(char* a, int n); }
const int _200MB = 1024*1024*200;
int main() {
  char* a = new char[_200MB]; // 200 MB buffer
  ...
  for (int i = 0; i < 100000000; i++) {
    int random_int = distribution(generator);
    foo(a, random_int);
  }
  ...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通过查看 [@lst:TMA_asm]，我们可以看到函数 `foo` 中所有的 L3 缓存未命中都标记到了一条指令上。现在我们知道是哪条指令导致了如此多的 L3 未命中，让我们来修复它。

### 第 3 步：修复问题 {.unlisted .unnumbered}

在 `foo` 函数的开头有 NOP 模拟的虚拟工作。这在我们获取下一个将要访问的地址和实际加载指令之间创建了一个时间窗口。时间窗口的存在使我们能够在虚拟工作的同时并行预取内存位置。[@lst:TMA_prefetch] 展示了这个思想的实际应用。有关显式内存预取技术的更多信息可以在 [@sec:memPrefetch] 中找到。

Listing: 在 main 中插入内存预取。

~~~~ {#lst:TMA_prefetch .cpp}
  for (int i = 0; i < 100000000; i++) {
    int random_int = distribution(generator);
+   __builtin_prefetch ( a + random_int, 0, 1);
    foo(a, random_int);
  }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这个显式内存预取提示将执行时间从 8.5 秒减少到 6.5 秒。同时，`CYCLE_ACTIVITY.STALLS_L3_MISS` 事件的数量减少了近十倍：从 190 亿下降到 20 亿。

TMA 是一个迭代过程，因此一旦我们修复了一个问题，我们需要从第 1 步开始重复该过程。很可能它会将瓶颈转移到另一个桶中，在这种情况下是 `Retiring`。这是一个演示 TMA 方法工作流程的简单示例。分析真实世界的应用程序不太可能那么容易。本书第二部分的章节组织得方便与 TMA 过程一起使用。特别是，第 8 章涵盖了`内存绑定`类别，第 9 章涵盖了`核心绑定`，第 10 章涵盖了`错误推测`，第 11 章涵盖了`前端绑定`。这样的结构旨在形成一个清单，当你遇到某个性能瓶颈时可以用来驱动代码更改。

[^2]: TMA metrics - [https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx).
[^7]: PMU tools - [https://github.com/andikleen/pmu-tools](https://github.com/andikleen/pmu-tools).
[^8]: Case study example - [https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM](https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM).
[^11]: 根据 x86 Linux 调用约定（[https://en.wikipedia.org/wiki/X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)），前 2 个参数分别放在 `rdi` 和 `rsi` 寄存器中。
[^17]: 或者，我们可以使用 `-l2 --nodes L1_Bound,L2_Bound,L3_Bound,DRAM_Bound,Store_Bound` 选项代替 `-l3` 来限制收集，因为我们知道应用程序受内存限制。
