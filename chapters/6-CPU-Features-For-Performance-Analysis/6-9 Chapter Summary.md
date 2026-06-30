## 章节总结 {.unlisted .unnumbered}

\markright{总结}

* 现代处理器提供增强性能分析的功能。使用这些功能大大简化了查找底层优化的机会。
* Top-down 微架构分析（TMA）方法是识别程序对 CPU 微架构低效使用的技术，即使对于经验不足的开发人员也易于使用。TMA 是一个迭代过程，包括多个步骤，包括表征工作负载和定位源代码中发生瓶颈的确切位置。我们建议 TMA 应该是每次底层调优工作的起点之一。
* 分支记录机制，如 Intel 的 LBR、AMD 的 LBR 和 ARM 的 BRBE，在执行程序的同时持续记录最近的分支结果，造成最小的减速。这些设施的主要用途之一是收集调用栈。此外，它们有助于识别热分支和预测错误率，并能够精确计时机器代码。
* 现代处理器通常提供基于硬件的采样功能用于高级分析。这些功能通过将多个样本存储在专用缓冲区中而不使用软件中断来降低采样开销。它们还引入了"精确事件"，能够精确定位导致特定性能事件的确切指令。此外，还有其他一些不太重要的用例。此类基于硬件的采样功能的示例实现包括 Intel 的 PEBS、AMD 的 IBS 和 ARM 的 SPE。
* Intel 处理器跟踪（PT）是一项 CPU 功能，通过以高度压缩的二进制格式编码数据包来记录程序执行，可用于重建每条指令带有时间戳的执行流程。PT 具有广泛的覆盖范围和相对较小的开销。其主要用途是事后分析和查找性能故障的根本原因。Intel PT 功能在附录 C 中介绍。基于 ARM 架构的处理器也具有称为 Arm [CoreSight](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace) 的跟踪功能，[^2] 但它主要用于调试而不是性能分析。

[^2]: Arm CoreSight - [https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace)

\sectionbreak
