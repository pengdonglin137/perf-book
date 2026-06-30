## 用查找替换分支

避免频繁预测错误分支的一种方法是使用查找表。[@lst:LookupBranches] 中显示了这种转换可能有益的代码示例。与往常一样，原始版本在左边，改进版本在右边。函数 `mapToBucket` 将 `[0-50)` 范围内的值映射到相应的五个桶，并为超出此范围的值返回 `-1`。对于均匀分布的 `v` 值，`v` 落入任何桶的概率相等。在原始版本的生成汇编中，我们可能会看到许多分支，这些分支可能具有高预测错误率。希望可以使用单个数组查找来重写函数 `mapToBucket`，如右边所示。

清单：用查找表替换分支。

~~~~ {#lst:LookupBranches .cpp}
int8_t mapToBucket(unsigned v) {       int8_t buckets[50] = {
  if      (v < 10) return 0;             0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
  else if (v < 20) return 1;             1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
  else if (v < 30) return 2;      =>     2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
  else if (v < 40) return 3;             3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
  else if (v < 50) return 4;             4, 4, 4, 4, 4, 4, 4, 4, 4, 4 };
  return -1;
}                                      int8_t mapToBucket(unsigned v) {
                                         if (v < (sizeof(buckets) / sizeof(int8_t)))
                                           return buckets[v];
                                         return -1;
                                       }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于右边改进版本的 `mapToBucket`，编译器可能会生成单个分支指令来保护对 `buckets` 数组的越界访问。通过此函数的典型热路径将执行未采取的分支和一个加载指令。该分支将被 CPU 分支预测器很好地预测，因为我们期望大多数输入值落入 `buckets` 数组覆盖的范围内。查找也将很快，因为 `buckets` 数组很小，并且可能在 L1 D-cache 中。

如果我们需要映射更大的值范围，例如 `[0-1M)`，分配一个非常大的数组是不切实际的。在这种情况下，我们可以使用区间映射数据结构，使用更少的内存但对数查找复杂度来实现该目标。读者可以在 [Boost](https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html)[^2] 和 [LLVM](https://llvm.org/doxygen/IntervalMap_8h_source.html)[^3] 中找到区间映射容器的现有实现。

[^2]: C++ Boost `interval_map` - [https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html](https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html)
[^3]: LLVM 的 `IntervalMap` - [https://llvm.org/doxygen/IntervalMap_8h_source.html](https://llvm.org/doxygen/IntervalMap_8h_source.html)
