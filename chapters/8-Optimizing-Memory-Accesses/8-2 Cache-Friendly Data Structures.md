## 缓存友好数据结构 {#sec:secCacheFriendly}

编写缓存友好的算法和数据结构是高性能应用程序的关键要素之一。缓存友好代码的关键支柱是我们在 [@sec:MemHierar] 中介绍的时间和空间局部性原则。这里的目标是具有可预测的内存访问模式并高效地存储数据。

缓存行是可以在缓存和主内存之间传输的最小数据单元。在设计缓存友好代码时，不仅要考虑单个变量及其在内存中的位置，还要考虑缓存行，这很有帮助。

接下来，我们将讨论使数据结构更缓存友好的几种技术。

### 顺序访问数据

利用缓存空间局部性的最佳方式是进行顺序内存访问。通过这样做，我们使硬件预取机制（参见 [@sec:HwPrefetch]）能够识别内存访问模式并提前获取下一块数据。[@lst:CacheFriend] 中显示了行主序与列主序遍历的示例。注意，代码中只有一个微小的变化（交换了 `col` 和 `row` 下标），但它对性能有很大的影响。

左边的代码不是缓存友好的，因为它在内循环的每次迭代中跳过 `NCOLS` 个元素。这导致缓存的使用非常低效：我们没有在缓存行被驱逐之前充分利用整个预取的缓存行。相反，右边的代码按照矩阵在内存中布局的顺序访问元素。这保证了缓存行在被驱逐之前会被完全使用。行主序遍历利用空间局部性，是缓存友好的。@fig:ColRowMajor 说明了两种遍历模式之间的区别。

Listing: 缓存友好的内存访问。

~~~~ {#lst:CacheFriend .cpp}
// 列主序顺序                              // 行主序顺序
for (row = 0; row < NROWS; row++)                  for (row = 0; row < NROWS; row++)
  for (col = 0; col < NCOLS; col++)                  for (col = 0; col < NCOLS; col++)
    matrix[col][row] = row + col;          =>          matrix[row][col] = row + col;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![列主序与行主序遍历。](../../img/memory-access-opts/ColumnRowMajor.png){#fig:ColRowMajor width=60%}

上面展示的例子是经典的，但通常，现实世界的应用程序比这复杂得多。有时你需要额外努力来编写缓存友好的代码。如果数据在内存中的布局不是算法最优的，可能需要先重新排列数据。

考虑在大型排序数组中实现二分查找的标准方法，在每次迭代中，你访问中间元素，将其与你正在搜索的值进行比较，然后向左或向右。该算法不利用空间局部性，因为它测试位于不同位置的元素，这些元素彼此相距很远，并且不共享相同的缓存行。解决此问题的最著名方法是使用 Eytzinger 布局 [@EytzingerArray] 存储数组元素。其思想是维护一个隐式二叉树搜索树，使用类似 BFS 的布局（通常在二叉堆中看到）打包到数组中。如果代码在数组中执行大量二分查找，将其转换为 Eytzinger 布局可能是有益的。

### 使用适当的容器。

几乎任何语言中都有各种各样的即用型容器。但了解它们的底层存储和性能影响很重要。记住数据将如何被访问和操作。你应该考虑的不仅是数据结构操作的时间和空间复杂性，还有与之相关的硬件影响。

默认情况下，远离依赖指针的数据结构，例如链表或树。遍历元素时，它们需要额外的内存访问来跟随指针。如果最大元素数量相对较小且在编译时已知，C++ `std::array` 可能比 `std::vector` 更好。如果你需要关联容器但不需要按排序顺序存储元素，`std::unordered_map` 应该比 `std::map` 更快。选择适当 C++ 容器的逐步指南可以在 [@fogOptimizeCpp, Section 9.7 Data structures, and container classes] 中找到。

有时，存储指向包含对象的指针而不是对象本身更有效。考虑需要在数组中存储许多对象的情况，同时每个对象的大小很大。此外，对象经常被洗牌、删除和插入。在数组中存储对象将要求每次对象顺序更改时移动大块内存，这是昂贵的。在这种情况下，最好在数组中存储指向对象的指针。这样，只移动指针，这便宜得多。然而，这种方法有其缺点。它需要指针的额外内存，并引入了额外的间接级别。

### 数据打包

数据缓存的利用率也可以通过使数据更紧凑来提高。打包数据有许多方法。经典的例子之一是使用位字段。[@lst:DataPacking] 中显示了数据打包可能有益的代码示例。如果我们知道 `a`、`b` 和 `c` 表示需要一定位数来编码的枚举值，我们可以减少结构体 `S` 的存储。

Listing: 数据打包

~~~~ {#lst:DataPacking .cpp}
// S is 3 bytes                         // S is 1 byte
struct S {                              struct S {
  unsigned char a;                        unsigned char a:4;
  unsigned char b;                =>      unsigned char b:2;
  unsigned char c;                        unsigned char c:2;
};                                      };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

注意打包版本的 `S` 所需空间减少了三倍。这大大减少了来回传输的内存量并节省了缓存空间。然而，使用位字段有额外的成本。[^15] 由于 `a`、`b` 和 `c` 的位被打包到单个字节中，编译器需要执行额外的位操作来提取和插入它们。例如，要加载 `b`，你需要将字节值右移（`>>`）2 位并与 `0x3` 进行逻辑与（`&`）。类似地，需要左移（`<<`）和逻辑或（`|`）操作来将更新后的值存回打包格式。在额外计算比低效内存传输造成的延迟更便宜的地方，数据打包是有益的。

此外，程序员可以通过重新排列结构体或类中的字段来减少内存使用，避免编译器添加的填充。插入未使用的内存字节（填充）可以有效地存储和获取结构体的各个成员。在 [@lst:AvoidPadding] 的示例中，如果按大小递减的顺序声明成员，`S` 的大小可以减小。@fig:AvoidPadding 说明了重新排列结构体 `S` 中字段的效果。

Listing: 避免编译器填充。

~~~~ {#lst:AvoidPadding .cpp}
// S is `sizeof(int) * 3` bytes          // S is `sizeof(int) * 2` bytes
struct S {                               struct S {
  bool b;                                  int i;
  int i;                         =>        short s;
  short s;                                 bool b;
};                                       };

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![通过重新排列字段避免编译器填充。空白单元格代表编译器填充。](../../img/memory-access-opts/AvoidPadding.png){#fig:AvoidPadding width=90%}

### 字段重排序

重新排列数据结构中的字段也可能因另一个原因而有益。考虑 [@lst:FieldReordering] 中的示例。假设 `Soldier` 结构体用于跟踪游戏中战场上数千个单位中的每一个。游戏有三个阶段：战斗、移动和交易。在战斗阶段，使用 `attack`、`defense` 和 `health` 字段。在移动阶段，使用 `coords` 和 `speed` 字段。在交易阶段，仅使用 `money` 字段。

左侧代码中 `Soldier` 结构体的组织问题是字段没有按照游戏的阶段进行分组。例如，在战斗阶段，程序需要访问两个不同的缓存行来获取所需的字段。`attack` 和 `defense` 字段很可能驻留在同一个缓存行上，但 `health` 字段总是被推到下一个缓存行上。移动阶段也是如此（`speed` 和 `coords` 字段）。

我们可以通过按照 [@lst:FieldReordering] 右侧所示重新排列字段，使 `Soldier` 结构体更缓存友好。通过此更改，一起访问的字段被分组在一起。

Listing: 字段重排序。

~~~~ {#lst:FieldReordering .cpp}
struct Soldier {                                 struct Soldier {
  2DCoords coords;   /*  8 bytes */                unsigned attack;  // 1. battle
  unsigned attack;                                 unsigned defense; // 1. battle
  unsigned defense;                     =>         unsigned health;  // 1. battle
  /* other fields */ /* 64 bytes */                2DCoords coords;  // 2. move
  unsigned speed;                                  unsigned speed;   // 2. move
  unsigned money;                                  // other fields
  unsigned health;                                 unsigned money;   // 3. trade
};                                                };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

自 Linux 内核 6.8 起，`perf` 工具中有新功能，允许你找到数据结构重排序的机会。`perf mem record` 命令现在可用于分析数据结构访问模式。`perf annotate --data-type` 命令将显示数据结构布局以及归因于数据结构每个字段的分析样本。使用此信息，你可以识别一起访问的字段。[^5]

数据类型分析在发现提高缓存利用率的机会方面非常有效。最近的 Linux 内核历史包含许多提交，这些提交重新排列结构体、[^1] 填充字段、[^3] 或打包[^2] 它们以提高性能。

### 其他数据结构重组技术

为了结束缓存友好数据结构的主题，我们将简要提及其他两种可用于提高缓存利用率的技术：*结构体拆分*和*指针内联*。

**结构体拆分**。将大型结构体拆分为较小的结构体可以提高缓存利用率。例如，如果你有一个包含大量字段的结构体，但其中只有少数字段一起访问，你可以将结构体拆分为两个或更多较小的结构体。这样，你可以避免将不必要的数据加载到缓存中。[@lst:StructureSplitting] 中展示了结构体拆分的示例。通过将 `Point` 结构体拆分为 `PointCoords` 和 `PointInfo`，当只需要 `PointCoords` 时，我们可以避免将 `PointInfo` 数据加载到缓存中。这样，我们可以在单个缓存行上容纳更多的点。

Listing: 结构体拆分。

~~~~ {#lst:StructureSplitting .cpp}
struct Point {                                struct PointCoords {
  int X;                                        int X;
  int Y;                                        int Y;
  int Z;                                        int Z;
  /*many other fields*/            =>         };
};                                            struct PointInfo {
std::vector<Point> points;                      /*many other fields*/
                                              };
                                              std::vector<PointCoords> pointCoords;
                                              std::vector<PointInfo> pointInfos;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**指针内联**。将指针内联到结构体中可以提高缓存利用率。例如，如果你有一个包含指向另一个结构体的指针的结构体，你可以将指针内联到第一个结构体中。这样，你可以避免额外的内存访问来获取第二个结构体。[@lst:PointerInlining] 中展示了指针内联的示例。`weight` 参数在许多图算法中使用，因此经常被访问。然而，在左侧的原始版本中，获取边权重需要额外的内存访问，这可能导致缓存未命中。通过将 `weight` 参数移入 `GraphEdge` 结构体，我们避免了此类问题。

Listing: 将 `weight` 参数移入父结构体。

~~~~ {#lst:PointerInlining .cpp}
struct GraphEdge {                            struct GraphEdge {
  unsigned int from;                            unsigned int from;
  unsigned int to;                              unsigned int to;
  GraphEdgeProperties* prop;                    float weight;
};                                 =>           GraphEdgeProperties* prop;
struct GraphEdgeProperties {                  };
  float weight;                               struct GraphEdgeProperties {
  std::string label;                            std::string label;
  // ...                                        // ...
};                                            };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[^1]: Linux commit [54ff8ad69c6e93c0767451ae170b41c000e565dd](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=54ff8ad69c6e93c0767451ae170b41c000e565dd)
[^2]: Linux commit [e5598d6ae62626d261b046a2f19347c38681ff51](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e5598d6ae62626d261b046a2f19347c38681ff51)
[^3]: Linux commit [aee79d4e5271cee4ffa89ed830189929a6272eb8](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aee79d4e5271cee4ffa89ed830189929a6272eb8)

[^5]: Linux `perf` 数据类型分析 - [https://lwn.net/Articles/955709/](https://lwn.net/Articles/955709/)

[^12]: aligned_alloc - [https://en.cppreference.com/w/c/memory/aligned_alloc](https://en.cppreference.com/w/c/memory/aligned_alloc)
[^13]: Linux 手册页 `memalign` - [https://linux.die.net/man/3/memalign](https://linux.die.net/man/3/memalign)
[^14]: 生成对齐内存 - [https://embeddedartistry.com/blog/2017/02/22/generating-aligned-memory/](https://embeddedartistry.com/blog/2017/02/22/generating-aligned-memory/)
[^15]: 此外，你不能获取位字段的地址。
