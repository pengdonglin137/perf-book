## 性能指标 {#sec:PerfMetrics}

能够收集各种性能事件对性能分析非常有帮助。但是，有一个注意事项。假设你运行了一个程序并收集了 `MEM_LOAD_RETIRED.L3_MISS` 事件，该事件计算 LLC 未命中数，它显示的值为十亿。当然，这听起来很多，所以你决定调查这些缓存未命中的来源。错了！你确定这是一个问题吗？如果一个程序只进行二十亿次加载，那么是的，这是一个问题，因为一半的加载在 LLC 中未命中。相反，如果一个程序进行一万亿次加载，那么只有一千分之一的加载导致 L3 缓存未命中。

这就是为什么除了硬件性能事件之外，性能工程师还经常使用建立在原始事件之上的指标。表 {@tbl:perf_metrics} 显示了 Intel 第 12 代 Golden Cove 架构的指标列表，以及描述和公式。该列表并不详尽，但它显示了最重要的指标。Intel CPU 的完整指标列表及其公式可以在 [TMA_metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx) 中找到。[^1] [@sec:PerfMetricsCaseStudy] 展示了如何在实践中使用性能指标。

\small

--------------------------------------------------------------------------
指标名称   描述                        公式
------- -------------------------- ---------------------------------------
L1MPKI  L1 缓存真实未命中          1000 * MEM_LOAD_RETIRED.L1_MISS_PS /
        每千条退休需求加载的      INST_RETIRED.ANY
        未命中数。

L2MPKI  L2 缓存真实未命中          1000 * MEM_LOAD_RETIRED.L2_MISS_PS /
        每千条退休需求加载的      INST_RETIRED.ANY
        未命中数。

L3MPKI  L3 缓存真实未命中          1000 * MEM_LOAD_RETIRED.L3_MISS_PS /
        每千条退休需求加载的      INST_RETIRED.ANY
        未命中数。

分支    所有分支中预测错误的      BR_MISP_RETIRED.ALL_BRANCHES /
预测    比率                      BR_INST_RETIRED.ALL_BRANCHES
错误率

代码    STLB（二级 TLB）代码       1000 * ITLB_MISSES.WALK_COMPLETED
STLB    推测未命中每千条          / INST_RETIRED.ANY
MPKI    指令数（完成页表遍历的
        任何页面大小的未命中数）

加载    STLB 数据加载              1000 * DTLB_LD_MISSES.WALK_COMPLETED
STLB    推测未命中每千条          / INST_RETIRED.ANY
MPKI    指令数

存储    STLB 数据存储              1000 * DTLB_ST_MISSES.WALK_COMPLETED
STLB    推测未命中每千条          / INST_RETIRED.ANY
MPKI    指令数

--------------------------------------------------------------------------
表：Intel 第 12 代 Golden Cove 架构的性能指标。{#tbl:perf_metrics}

\normalsize
