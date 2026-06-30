## 系统调优 {#sec:SysTune}

在完成所有调优应用程序以利用 CPU 微架构的所有复杂功能的艰苦工作之后，我们最不希望的是系统固件、操作系统或内核破坏我们所有的努力。如果一个高度调优的应用程序不时被停止整个系统的系统中断打断，那么它将意义不大。这样的中断可能一次运行数十到数百毫秒。

开发人员通常对应用程序执行的环境几乎没有控制权。当我们发布产品时，调整客户可能拥有的每个设置是不现实的。通常，足够大的组织有单独的运维团队（Ops），他们处理此类问题。我们有责任为他们提供建议，以设置系统从我们的应用程序中获得最佳性能。

现代系统中有很多需要调整的东西，避免基于系统的干扰并非易事。x86 服务器部署的性能调优手册的一个例子是 Red Hat [指南](https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf)[^5]。在那里，你将找到消除或显着最小化来自系统 BIOS、Linux 内核和设备驱动程序等来源的缓存干扰中断的提示，以及许多其他应用程序干扰源。这些指南应作为所有新服务器构建的基准映像，然后再将任何应用程序部署到生产环境中。

大多数开箱即用的平台都配置为在尽可能节省功耗的同时实现最佳吞吐量。但有些行业有实时需求，它们更关心低延迟而不是其他一切。这种行业的一个例子是在汽车装配线中运行的机器人。这些机器人执行的动作由外部事件触发，并且通常有预定的时间预算来完成，因为下一个中断很快就会到来（通常称为"控制循环"）。满足此类平台的实时目标可能需要牺牲机器的整体吞吐量或允许其消耗更多能量。该领域的一种流行技术是禁用处理器睡眠状态[^7]，以保持其准备好立即响应。另一个有趣的技术称为缓存锁定，[^8] 其中 CPU 缓存的一部分被保留用于特定的数据集。它有助于简化应用程序内的内存延迟。

超频 CPU 是提高性能的更极端尝试。超频是将 CPU 以高于其设计频率运行的过程。这是一种有风险的操作，因为它可能使保修无效并可能损坏 CPU。超频不适合生产环境，通常由愿意为性能承担风险的爱好者执行。要超频 CPU，你需要拥有合适的部件，主要是支持超频的主板、具有解锁时钟频率的 CPU，以及可以处理增加的热输出的冷却系统。2024 年初，超频专家在广泛可用的 CPU 上突破了 9 GHz 大关。[^9]

了解应用程序中的性能瓶颈有助于调整正确的设置。可扩展性研究可以帮助你确定你的应用程序对各种系统设置的敏感性。例如，你可能会发现你的应用程序不随核心数量扩展（参见 [@sec:ThreadCountScalingStudy]），或者它受内存延迟限制。使用此信息，你可以就调整系统设置或为计算系统购买新硬件组件做出明智的决定。在下一个案例研究中，我们将展示如何确定应用程序是否对最后一级缓存（LLC）的大小敏感。

[^5]: Red Hat 低延迟调优指南 - [https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf](https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf)
[^7]: 电源管理状态：P 状态、C 状态 - [https://software.intel.com/content/www/us/en/develop/articles/power-management-states-p-states-c-states-and-package-c-states.html](https://software.intel.com/content/www/us/en/develop/articles/power-management-states-p-states-c-states-and-package-c-states.html)
[^8]: 缓存锁定。缓存锁定技术调查 [@CacheLocking]。伪锁定缓存部分的示例，然后在 Linux 文件系统中作为字符设备暴露并可用于 `mmap`：[https://events19.linuxfoundation.org/wp-content/uploads/2017/11/Introducing-Cache-Pseudo-Locking-to-Reduce-Memory-Access-Latency-Reinette-Chatre-Intel.pdf](https://events19.linuxfoundation.org/wp-content/uploads/2017/11/Introducing-Cache-Pseudo-Locking-to-Reduce-Memory-Access-Latency-Reinette-Chatre-Intel.pdf)
[^9]: CPU 超频记录 - [https://press.asus.com/news/press-releases/rog-maximus-z790-apex-encore-sets-3-overclocking-world-records/](https://press.asus.com/news/press-releases/rog-maximus-z790-apex-encore-sets-3-overclocking-world-records/)
