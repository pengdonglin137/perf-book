## 显式内存预取 {#sec:memPrefetch}

到现在，你应该知道未从缓存解析的内存访问通常非常昂贵。现代 CPU 通过预测程序将来会访问哪些内存位置并提前预取它们来努力降低缓存未命中的惩罚。如果请求的内存位置在程序请求时不在缓存中，那么我们将遭受缓存未命中惩罚，因为我们必须去 DRAM 获取数据。但是，如果 CPU 能够及时将该内存位置带入缓存，或者如果请求被预测且数据正在传输中，那么缓存未命中的惩罚将低得多。

现代 CPU 有两种自动机制来解决这个问题：硬件预取和 OOO 执行。硬件预取器通过在重复内存访问模式上发起预取请求来帮助隐藏内存访问延迟。OOO 引擎向前查看 `N` 条指令并提前发出加载，以实现未来将需要此数据的指令的平滑执行。

当数据访问模式太复杂而无法预测时，硬件预取器会失败。软件开发人员对此无能为力，因为我们无法控制这个单元的行为。另一方面，OOO 引擎不像硬件预取那样尝试预测将来需要的内存位置。因此，它的成功衡量标准是它通过提前调度加载能够隐藏多少延迟。

考虑 [@lst:MemPrefetch1] 中的一小段代码，其中 `arr` 是一个包含一百万个整数的数组。索引 `idx` 被赋值为一个随机值，立即用于访问 `arr` 中的一个位置，这几乎肯定会因为它是随机的而在缓存中未命中。硬件预取器无法预测它，因为每次加载都到达内存中一个全新的位置。从知道内存位置的地址（从函数 `random_distribution` 返回）到请求该内存位置的值（调用 `doSomeExtensiveComputation`）的时间间隔称为*预取窗口*。在这个示例中，OOO 引擎没有机会提前发出加载，因为预取窗口非常小。这导致内存访问 `arr[idx]` 的延迟在执行循环时处于关键路径上，如@fig:SWmemprefetch1 所示。程序等待值返回（图中带阴影的填充矩形），而没有向前推进。

你可能会想："但循环的下一次迭代应该开始并行推测执行"。这是正确的，事实上，它在@fig:SWmemprefetch1 中有所反映。`doSomeExtensiveComputation` 函数需要大量工作，当执行接近第一次迭代结束时，CPU 推测地开始执行下一次迭代的指令。这在迭代之间创建了正的执行重叠。事实上，我们提出了一个乐观场景，其中处理器能够生成下一个随机数并与循环的上一次迭代并行发出加载。但是，CPU 无法完全隐藏加载的延迟，因为它无法向前查看当前迭代那么远来提前发出加载。也许未来的处理器将具有更强大的 OOO 引擎，但目前，存在需要程序员干预的情况。

Listing: 随机数是后续加载的索引。

~~~~ {#lst:MemPrefetch1 .cpp}
for (int i = 0; i < N; ++i) {
  size_t idx = random_distribution(generator);
  int x = arr[idx]; // 缓存未命中
  doSomeExtensiveComputation(x);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![显示加载延迟处于关键路径上的执行时间线。](../../img/memory-access-opts/SWmemprefetch1.png){#fig:SWmemprefetch1 width=80%}

幸运的是，这不是死胡同，因为有一种方法可以通过将加载与 `doSomeExtensiveComputation` 的执行完全重叠来加速此代码，这将隐藏缓存未命中的延迟。我们可以通过称为*软件流水线*和*显式内存预取*的技术来实现这一点。[@lst:MemPrefetch2] 中展示了此思想的实现。我们流水线化随机数的生成，并在与 `doSomeExtensiveComputation` 并行时开始为下一次迭代预取内存位置。

Listing: 利用显式软件内存预取提示。

~~~~ {#lst:MemPrefetch2 .cpp}
size_t idx = random_distribution(generator);
for (int i = 0; i < N; ++i) {
  int x = arr[idx]; 
  idx = random_distribution(generator);
  // 为下一次迭代预取元素
  __builtin_prefetch(&arr[idx]);
  doSomeExtensiveComputation(x);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这种转换的图形说明如@fig:SWmemprefetch2 所示。我们利用软件流水线为下一次迭代生成随机数。换句话说，在迭代 `M` 上，我们生成一个将在迭代 `M+1` 上使用的随机数。这使我们能够提前发出内存请求，因为我们已经知道数组中的下一个索引。这种转换使我们的预取窗口大得多，并完全隐藏了缓存未命中的延迟。在迭代 `M+1` 上，实际加载有很大的机会命中缓存，因为它在迭代 `M` 上被预取了。

![通过与其他执行重叠来隐藏缓存未命中延迟。](../../img/memory-access-opts/SWmemprefetch2.png){#fig:SWmemprefetch2 width=80%}

注意 [`__builtin_prefetch`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)[^4] 的使用，开发者可以用这个特殊提示显式请求 CPU 预取某个内存位置。另一个选项是使用编译器内建函数。在 x86 平台上有 `_mm_prefetch` 内建函数，在 ARM 平台上有 `__pld` 内建函数。编译器将为 x86 生成 `PREFETCH` 指令，为 ARM 生成 `pld` 指令。

有些情况下软件内存预取是不可能的。例如，当遍历链表时，预取窗口非常小，无法隐藏指针追踪的延迟。

在 [@lst:MemPrefetch2] 中我们看到了为下一次迭代预取的示例，但你也可能经常遇到需要为 2、4、8 次甚至更多迭代预取的情况。[@lst:MemPrefetch3] 中的代码就是其中一个可能有益的场景。它展示了一个用边填充图的典型代码。如果图非常稀疏且有大量顶点，访问 `this->out_neighbors` 和 `this->in_neighbors` 向量很可能经常在缓存中未命中。这是因为每条边都可能连接当前不在缓存中的新顶点。

此代码与前一个示例不同，因为每次迭代没有大量的计算，所以缓存未命中的惩罚很可能主导每次迭代的延迟。但我们可以利用我们知道将来要访问的所有元素这一事实。向量 `edges` 的元素是顺序访问的，因此很可能被硬件预取器及时带到 L1 缓存中。我们这里的目标是将缓存未命中的延迟与执行足够的迭代重叠，以完全隐藏它。

一般规则是，为了使预取提示有效，它必须提前很早插入，以便在加载的值用于其他计算时，它已经在缓存中。但是，它也不应该插入得太早，因为可能会用很长时间不使用的数据污染缓存。注意，在 [@lst:MemPrefetch3] 中，`lookAhead` 是一个模板参数，使程序员可以尝试不同的值并查看哪个给出最佳性能。更高级的用户可以尝试使用 [@sec:timed_lbr] 中描述的方法来估计预取窗口；使用此方法的示例可以在 Easyperf 博客上找到。[^5]

Listing: 为接下来 8 次迭代进行软件预取的示例。

~~~~ {#lst:MemPrefetch3 .cpp}
template <int lookAhead = 8>
void Graph::update(const std::vector<Edge>& edges) {
  for(int i = 0; i + lookAhead < edges.size(); i++) {
    VertexID v = edges[i].from;
    VertexID u = edges[i].to;
    this->out_neighbors[u].push_back(v);
    this->in_neighbors[v].push_back(u);

    // prefetch elements for future iterations
    VertexID v_next = edges[i + lookAhead].from;
    VertexID u_next = edges[i + lookAhead].to;
    __builtin_prefetch(this->out_neighbors.data() + v_next);
    __builtin_prefetch(this->in_neighbors.data()  + u_next);
  }
  // process the remainder of the vector `edges` ...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

显式内存预取最常用于循环中，但你也可以将这些提示插入到父函数中；这完全取决于可用的预取窗口。

这项技术是一个强大的武器，但应极其谨慎地使用，因为它不容易做好。首先，显式内存预取不具有可移植性，这意味着如果它在一个平台上获得性能提升，并不保证在另一个平台上有类似的加速。它非常依赖于具体实现，平台不必遵守这些提示。在这种情况下，它很可能会降低性能。我的建议是验证影响在所有可用工具上都是积极的。不仅要检查性能数据，还要确保缓存未命中数量（特别是 L3）有所下降。一旦更改提交到代码库中，在你运行应用程序的所有平台上监控性能，因为它可能对周围代码的更改非常敏感。如果收益不能抵消潜在的维护负担，请考虑放弃这个想法。

对于一些复杂的场景，确保代码预取正确的内存位置。当循环的当前迭代依赖于前一次迭代时，这可能变得棘手，例如存在 `continue` 语句或要处理的下一个元素的更改由 `if` 条件保护。在这种情况下，我的建议是插装代码以测试预取提示的准确性。因为使用不当时，它可能通过驱逐其他有用数据来降低缓存的性能。

最后，显式预取增加代码大小并给 CPU 前端增加压力。预取提示只是一个进入内存子系统的假加载，没有目标寄存器。就像任何其他指令一样，它消耗 CPU 资源。极其谨慎地使用它，因为使用错误时，它可能会使程序性能变差。

[^4]: GCC builtins - [https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html).
[^5]: "Precise timing of machine code with Linux perf" - [https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf#application-estimating-prefetch-window](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf#application-estimating-prefetch-window).
