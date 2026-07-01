## 数据依赖

当一个程序语句引用前一个语句的输出时，我们说这两个语句之间存在*数据依赖*。有时人们也使用术语_依赖链_或*数据流依赖*。我们最熟悉的例子是遍历链表（参见@fig:LinkedListChasing）。要访问节点 `N+1`，我们应该首先解引用指针 `N->next`。对于右边的循环，这是一个*循环*数据依赖，意味着它跨越循环的多次迭代。遍历链表是一个非常长的依赖链。

![遍历链表时的数据依赖。](../../img/computation-opts/LinkedListChasing.png){#fig:LinkedListChasing width=80%}

传统程序是假设顺序执行模型编写的。在这个模型中，指令一个接一个地执行，原子地且按照程序指定的顺序。然而，正如我们已经知道的，这不是现代 CPU 的构建方式。它们被设计为乱序、并行地执行指令，并以最大化可用执行单元利用率的方式执行。

当出现长数据依赖时，处理器被迫顺序执行代码，只利用其全部能力的一部分。长依赖链阻碍并行性，这破坏了现代超标量 CPU 的主要优势。例如，指针追逐不会从 OOO 执行中受益，因此将以顺序 CPU 的速度运行。正如我们将在本节中看到的，依赖链是性能瓶颈的主要来源。

你不能消除数据依赖；它们是程序的基本属性。任何程序都需要输入来计算某些东西。事实上，人们已经开发了技术来发现语句之间的数据依赖并构建数据流图。这称为*依赖分析*，更适合编译器开发人员，而不是性能工程师。我们对为整个程序构建数据流图不感兴趣。相反，我们想要在热代码片段（如循环或函数）中找到关键的依赖链。

你可能想知道："如果你不能摆脱依赖链，你能做什么？"好吧，有时这将是性能的限制因素，不幸的是，你将不得不接受它。但有些情况下，你可以打破不必要的数据依赖链或重叠它们的执行。[@lst:DepChain] 中展示了一个这样的例子。与其他一些情况类似，我们在左边展示源代码，在右边展示相应的 ARM 汇编。此外，此代码示例包含在 Performance Ninja 在线课程的 `dep_chains_2` 实验室作业中，因此你可以自己尝试。[^2]

这个小程序模拟随机粒子运动。我们有 1000 个粒子在 2D 表面上移动，没有约束，这意味着它们可以离起始位置任意远。每个粒子由其在 2D 表面上的 x 和 y 坐标以及速度定义。初始 x 和 y 坐标在 [-1000,1000] 范围内，速度在 [0,1] 范围内，不会改变。程序为每个粒子模拟 1000 个移动步骤。对于每个步骤，我们使用随机数生成器（RNG）生成一个角度，该角度设置粒子的移动方向。然后我们相应地调整粒子的坐标。

鉴于手头的任务，你决定编写自己的 RNG、正弦和余弦函数，以牺牲一些精度并使其尽可能快。毕竟，这是*随机*移动，所以这是一个很好的权衡。你选择了一个中等质量的 `XorShift` RNG，因为它只有 3 次移位和 3 次 XOR。还有什么比这更简单的？此外，你搜索了网络并找到了使用多项式进行正弦和余弦近似的算法，这些算法足够精确且非常快。

Listing: 2D 表面上的随机粒子运动

~~~~ {#lst:DepChain .cpp .numberLines}
struct Particle {                                    │
  float x; float y; float velocity;                  │
};                                                   │
                                                     │
class XorShift32 {                                   │
  uint32_t val;                                      │
public:                                              │
  XorShift32 (uint32_t seed) : val(seed) {}          │
  uint32_t gen() {                                   │
    val ^= (val << 13);                              │
    val ^= (val >> 17);                              │
    val ^= (val << 5);                               │
    return val;                                      │ .loop:
  }                                                  │   eor    w0, w0, w0, lsl #13
};                                                   │   eor    w0, w0, w0, lsr #17
                                                     │   eor    w0, w0, w0, lsl #5
static float sine(float x) {                         │   ucvtf  s1, w0
  const float B = 4 / PI_F;                          │   fmov   s2, w9
  const float C = -4 / ( PI_F * PI_F);               │   fmul   s2, s1, s2
  return B * x + C * x * std::abs(x);                │   fmov   s3, w10
}                                                    │   fadd   s3, s2, s3
static float cosine(float x) {                       │   fmov   s4, w11
  return sine(x + (PI_F / 2));                       │   fmul   s5, s3, s3
}                                                    │   fmov   s6, w12
                                                     │   fmul   s5, s5, s6
/* Map degrees [0;UINT32_MAX) to radians [0;2*pi)*/  │   fmadd  s3, s3, s4, s5
float DEGREE_TO_RADIAN = (2 * PI_D) / UINT32_MAX;    │   ldp    s6, s4, [x1, #0x4]
                                                     │   ldr    s5, [x1]
void particleMotion(vector<Particle> &particles,     │   fmadd  s3, s3, s4, s5
                    uint32_t seed) {                 │   fmov   s5, w13
 XorShift32 rng(seed);                               │   fmul   s5, s1, s5
 for (int i = 0; i < STEPS; i++)                     │   fmul   s2, s5, s2
  for (auto &p : particles) {                        │   fmadd  s1, s1, s0, s2
   uint32_t angle = rng.gen();                       │   fmadd  s1, s1, s4, s6
   float angle_rad = angle * DEGREE_TO_RADIAN;       │   stp    s3, s1, [x1], #0xc
   p.x += cosine(angle_rad) * p.velocity;            │   cmp    x1, x16
   p.y += sine(angle_rad) * p.velocity;              │   b.ne   .loop
  }                                                  │
}                                                    │
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我使用 Clang-17 C++ 编译器编译了代码，并在 Mac mini（Apple M1，2020）上运行。让我们检查生成的 ARM 汇编代码：

* 前三条 `eor`（异或）指令结合 `lsl`（左移）或 `lsr`（右移）对应于 `XorShift32::gen` 函数。
* 接下来的 `ucvtf`（无符号整数转浮点数）和 `fmul`（浮点乘法）用于将角度从度转换为弧度（代码第 35 行）。
* 正弦和余弦函数都有两条 `fmul` 指令和一条 `fmadd`（浮点融合乘加）指令。余弦还有一条额外的 `fadd`（浮点加法）指令。
* 最后，我们还有一对 `fmadd` 指令分别计算 x 和 y，以及一条 `stp` 指令存储坐标对。

你期望这段代码"飞起来"，然而，有一个非常讨厌的性能问题拖慢了程序。不看后面的文本，你能在代码中找到一个循环依赖链吗？

如果你找到了，恭喜。`XorShift32::val` 上存在一个循环循环依赖。要生成下一个随机数，生成器必须先产生前一个数。方法 `XorShift32::gen` 的下一次调用将基于前一个数生成新数。图 @fig:DepChain 可视化了有问题的循环携带依赖。注意，计算粒子坐标的代码（将角度转换为弧度、正弦、余弦、将结果乘以速度）在对应的随机数准备好后才开始执行，但不会更早。

![[@lst:DepChain] 中依赖执行的可视化](../../img/computation-opts/DepChain.png){#fig:DepChain width=90%}

计算粒子 `N` 坐标的代码不依赖于粒子 `N-1`，因此将它们向左拉以进一步重叠它们的执行可能是有益的。你可能想问："但是那三条（或六条）指令怎么就能拖慢整个循环的性能呢？"确实，循环中还有许多其他"重量级"指令，如 `fmul` 和 `fmadd`。然而，它们不在关键路径上，因此可以与其他指令并行执行。而且由于现代 CPU 非常宽，它们将同时执行来自多个迭代的指令。这使得 OOO 引擎能够有效地在循环的不同迭代中找到并行性（独立指令）。

让我们做一些粗略的估算。[^1] 每条 `eor` 和 `lsl` 指令产生 2 个周期的延迟：一个周期用于移位，一个用于 XOR。我们有三个依赖的 `eor + lsl` 对，因此生成下一个随机数需要 6 个周期。这是我们这个循环的绝对最小值：我们无法以每次迭代少于 6 个周期的速度运行。后面的代码至少需要 20 个周期的延迟来完成所有 `fmul` 和 `fmadd` 指令。但这不重要，因为它们不在关键路径上。重要的是这些指令的吞吐量。一个有用的经验法则：如果指令在关键路径上，看它的延迟，否则看它的吞吐量。在每次循环迭代中，我们有 5 条 `fmul` 和 4 条 `fmadd` 指令，它们在同一组执行单元上运行。M1 处理器每周期可以运行 4 条此类指令，因此至少需要 `9/4 = 2.25` 个周期来发射所有 `fmul` 和 `fmadd` 指令。所以，我们有两个性能限制：第一个由软件施加（由于依赖链每次迭代 6 个周期），第二个由硬件施加（由于执行单元的吞吐量每次迭代 2.25 个周期）。现在我们受第一个限制的约束，但我们可以尝试打破依赖链以接近第二个限制。

解决这个问题的方法之一是使用额外的 RNG 对象，使其中一个对象为循环的偶数迭代提供数据，另一个为奇数迭代提供数据，如 [@lst:DepChainFixed] 所示。注意，我们还手动展开了循环。现在我们有两个独立的依赖链，可以并行执行。你可以争辩说这改变了程序的功能，但用户无法分辨差异，因为粒子的运动本来就是随机的。另一种解决方案是选择一个内部依赖链开销更小的不同的 RNG。

Listing: 2D 表面上的随机粒子运动

~~~~ {#lst:DepChainFixed .cpp}
void particleMotion(vector<Particle> &particles, 
                    uint32_t seed1, uint32_t seed2) {
  XorShift32 rng1(seed1);
  XorShift32 rng2(seed2);
  for (int i = 0; i < STEPS; i++) {
    for (int j = 0; j + 1 < particles.size(); j += 2) {
      uint32_t angle1 = rng1.gen();
      float angle_rad1 = angle1 * DEGREE_TO_RADIAN;
      particles[j].x += cosine(angle_rad1) * particles[j].velocity;
      particles[j].y += sine(angle_rad1)   * particles[j].velocity;
      uint32_t angle2 = rng2.gen();
      float angle_rad2 = angle2 * DEGREE_TO_RADIAN;
      particles[j+1].x += cosine(angle_rad2) * particles[j+1].velocity;
      particles[j+1].y += sine(angle_rad2)   * particles[j+1].velocity;
    }
    // 余数（未显示）
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一旦你做了这个转换，编译器就开始自动向量化循环体，即将两条链拼接在一起并使用 SIMD 指令并行处理它们。为了隔离打破依赖链的效果，我禁用了编译器向量化。

为了衡量更改的性能影响，我运行了"之前"和"之后"的版本，观察到运行时间从每次迭代 19ms 下降到每次迭代 10ms。这几乎是 2 倍的加速。`IPC` 也从 4.0 上升到 7.1。为了尽职调查，我还测量了其他指标，以确保性能没有因为其他原因意外提高。在原始代码中，`MPKI` 为 0.01，`BranchMispredRate` 为 0.2%，这意味着程序最初没有遭受缓存未命中或分支预测错误。这是另一个数据点：在 Intel 的 Alder Lake 系统上运行相同代码时，它显示 74% 的 Retiring 和 24% 的 Core Bound，这确认了性能受计算约束。

通过一些额外的更改，你可以将此解决方案推广为拥有任意多的依赖链。对于 M1 处理器，测量表明拥有 2 条依赖链足以非常接近硬件限制。拥有超过 2 条链带来的性能改进可以忽略不计。然而，有一种趋势是 CPU 正在变得更宽，即它们越来越能够并行运行多条依赖链。这意味着未来的处理器可能从拥有超过 2 条依赖链中受益。一如既往，你应该测量并找到代码将运行在的平台的最佳平衡点。

有时仅仅打破依赖链是不够的。想象一下，你不是有一个简单的 RNG，而是有一个非常复杂的加密算法，它有 `10,000` 条指令长。所以，我们现在在关键路径上有 `10,000` 条指令，而不是很短的 6 条指令依赖链。你立即做了上面相同的更改，期待一个不错的 2 倍加速，但只看到 5% 的性能提升。怎么回事？

这里的问题是 CPU 根本"看不到"第二条依赖链来开始执行它。回顾第 3 章，保留站（RS）容量不足以看到前面的 `10,000` 条指令，因为其条目数量要小得多。所以，CPU 将无法重叠两条依赖链的执行。为了解决这个问题，我们需要*交错*这两条依赖链。使用这种方法，你需要更改代码，使 RNG 对象同时生成两个数字，函数 `XorShift32::gen` 内的*每条*语句都被复制并交错。即使编译器内联了所有代码并能清楚地看到两条链，它也不会自动交错它们，所以你需要注意这一点。你可能遇到的另一个限制是寄存器压力。并行运行多条依赖链需要保持更多状态，因此需要更多寄存器。如果你耗尽了架构寄存器，编译器将开始将它们溢出到堆栈，这将减慢程序速度。

值得一提的是，数据依赖也可以通过内存产生。例如，如果你在循环迭代 `N` 写入内存位置 `M` 并在迭代 `N+1` 从该位置读取，那么实际上就存在一条依赖链。存储的值可以转发给加载，但这些指令不能被重新排序和并行执行。

作为结束语，我想强调找到那条关键依赖链的重要性。这并不总是容易的，但了解循环、函数或其他代码块中关键路径上是什么至关重要。否则，你可能会发现自己在修复几乎不会产生差异的次要问题。

[^1]: Apple 发布了指令延迟和吞吐量数据，见 [@AppleOptimizationGuide, Appendix A]。
[^2]: Performance Ninja: Dependency Chains 2 - [https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/dep_chains_2](https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/dep_chains_2)
