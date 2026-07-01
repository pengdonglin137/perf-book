## 多线程程序中的性能扩展 {#sec:secAmdahl}

在处理单线程应用程序时，优化程序的一部分通常会产生积极的性能结果。但是，对于多线程应用程序，情况并非总是如此。可能存在一个应用程序，其中线程 `A` 执行长时间运行的操作，而线程 `B` 提前完成其任务并只是等待线程 `A` 完成。无论我们如何改进线程 `B`，应用程序延迟都不会降低，因为它将受更长时间运行的线程 `A` 的限制。

这种效果被广泛称为[阿姆达尔定律](https://en.wikipedia.org/wiki/Amdahl's_law)，[^6] 它认为并行程序的加速受其串行部分的限制。@fig:MT_AmdahlsLaw 说明了程序执行延迟作为执行它的处理器数量函数的理论加速。对于 75% 并行的程序，加速因子收敛到 4。

<div id="fig:AmdahlUSLLaws">
![根据阿姆达尔定律，理论加速限制作为处理器数量的函数。](../../img/mt-perf/AmdahlsLaw.png){#fig:MT_AmdahlsLaw width=45%}
![线性加速、阿姆达尔定律和通用可扩展性定律。](../../img/mt-perf/USL.png){#fig:MT_USL width=45%}

阿姆达尔定律和通用可扩展性定律。
</div>

实际上，进一步向系统添加计算节点可能会产生倒退加速。我们将在下一节中看到它的例子。这种效应被 Neil Gunther 解释为[通用可扩展性定律](http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1)[^8]（USL），它是阿姆达尔定律的扩展。USL 将计算节点（线程）之间的通信描述为限制性能的另一个因素。随着系统的扩展，开销开始抵消收益。超过一个临界点后，系统的能力开始下降（参见@fig:MT_USL）。USL 被广泛用于建模系统的容量和可扩展性。

USL 描述的减速由几个因素驱动。首先，随着计算节点数量的增加，它们开始争夺资源（争用）。这导致额外的时间花在同步这些访问上。另一个问题发生在许多工作者共享的资源上。我们需要在许多工作者之间保持共享资源的一致状态（一致性）。例如，当多个工作者频繁更改全局可见对象时，这些更改需要广播给所有使用该对象的节点。突然之间，由于维护一致性的额外需求，通常的操作开始花费更多时间。优化多线程应用程序不仅涉及本书到目前为止描述的所有技术，还涉及检测和缓解上述争用和一致性的影响。

[^6]: 阿姆达尔定律 - [https://en.wikipedia.org/wiki/Amdahl's_law](https://en.wikipedia.org/wiki/Amdahl's_law)。
[^8]: USL 定律 - [http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1](http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1)。
