## 代码检测 {#sec:secInstrumentation}

可能发明的第一个性能分析方法是*代码检测*。这是一种将额外代码插入程序以收集特定运行时信息的技术。[@lst:CodeInstrumentation] 显示了在函数开头插入 `printf` 语句以指示该函数是否被调用的最简单示例。之后，你运行程序并计算在输出中看到"foo is called"的次数。也许世界上的每个程序员在他们职业生涯的某个时刻都至少做过一次。

Listing: 代码检测

~~~~ {#lst:CodeInstrumentation .cpp}
int foo(int x) {
+ printf("foo is called\n");
 // 函数体...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

行首的加号表示该行是添加的，不在原始代码中。通常，检测代码不应该被推送到代码库中；相反，它是用于收集所需数据，之后可以删除。

[@lst:CodeInstrumentationHistogram] 展示了一个更有趣的代码检测示例。在这个虚构的代码示例中，函数 `findObject` 在地图上搜索具有某些属性 `p` 的对象的坐标。所有对象都保证最终会被定位。函数 `getNewCoords` 返回在作为参数提供的更大区域内的新坐标。函数 `findObj` 返回使用当前坐标 `c` 定位正确对象的置信度。如果是精确匹配，我们停止搜索循环并返回坐标。如果置信度高于 `threshold`，我们调用 `zoomIn` 来找到对象的更精确位置。否则，我们获取 `searchArea` 内的新坐标以在下次尝试搜索。

检测代码由两个类组成：`histogram` 和 `incrementor`。前者跟踪我们感兴趣的任何变量值及其出现频率，然后在程序完成后打印直方图。后者只是一个帮助类，用于将值推送到 `histogram` 对象中。它很简单，可以快速调整以适应你的特定需求。[^3]

Listing: 代码检测

~~~~ {#lst:CodeInstrumentationHistogram .cpp}
+ struct histogram {
+   std::map<uint32_t, std::map<uint32_t, uint64_t>> hist;
+   ~histogram() {
+     for (auto& tripCount : hist)
+       for (auto& zoomCount : tripCount.second)
+         std::cout << "[" << tripCount.first << "]["
+                   << zoomCount.first << "] :  "
+                   << zoomCount.second << "\n";
+   }
+ };
+ histogram h;

+ struct incrementor {
+   uint32_t tripCount = 0;
+   uint32_t zoomCount = 0;
+   ~incrementor() {
+ 	   h.hist[tripCount][zoomCount]++;
+   }
+ };

Coords findObject(const ObjParams& p, Coords searchArea) {
+ incrementor inc;
  Coords c = getNewCoords(searchArea);
  while (true) {
+   inc.tripCount++;
    float match = findObj(p, c);
    if (exactMatch(match))
      return c;
    if (match > threshold) {
      searchArea = zoomIn(searchArea, c);
+     inc.zoomCount++;
    } else {
      c = getNewCoords(searchArea);
    }
  }
  return c;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在这个假设场景中，我们添加了检测代码来了解在找到对象之前 `zoomIn` 的频率。变量 `inc.tripCount` 计算循环在退出前运行的迭代次数，变量 `inc.zoomCount` 计算我们在同一次循环中减少搜索区域（调用 `zoomIn`）的次数。我们总是期望 `inc.zoomCount` 小于或等于 `inc.tripCount`。

`findObject` 函数使用各种输入被多次调用。以下是运行检测程序后我们可能观察到的输出：

```
// [tripCount][zoomCount]: 出现次数
[7][6]:  2
[7][5]:  6
[7][4]:  20
[7][3]:  156
[7][2]:  967
[7][1]:  3685
[7][0]:  251004
[6][5]:  2
[6][4]:  7
[6][3]:  39
[6][2]:  300
[6][1]:  1235
[6][0]:  91731
[5][4]:  9
[5][3]:  32
[5][2]:  160
[5][1]:  764
[5][0]:  34142
...
```

方括号中的第一个数字是循环的迭代次数，第二个是我们在同一次循环中调用 `zoomIn` 的次数。冒号后的数字是该特定组合出现的次数。例如，有两次我们观察到 7 次循环迭代和 6 次 `zoomIn`。251004 次循环运行了 7 次迭代且没有 `zoomIn`，依此类推。你也可以绘制数据以进行更好的可视化，或采用其他统计方法，但我们可以得出的主要结论是 `zoomIn` 并不频繁。

对 `findObject` 的总调用次数约为 40 万次；我们可以通过对直方图中所有桶求和来计算。如果我们对所有 `zoomCount` 非零的桶求和，得到约 1 万次；这是 `zoomIn` 函数被调用的次数。因此，每次 `zoomIn` 调用，我们大约调用 40 次 `findObject` 函数。

本书后续章节包含许多如何利用此类信息进行优化的示例。在我们的案例中，我们得出结论：`findObj` 经常找不到对象。这意味着循环的下一次迭代将尝试使用新坐标但在同一搜索区域内查找对象。了解这一点后，我们可以尝试一些优化：1）并行运行多个搜索，如果任何一个成功则同步；2）为当前搜索区域预计算某些内容，从而消除 `findObj` 内部的重复工作；3）编写软件流水线，调用 `getNewCoords` 生成下一组所需坐标，并从内存中预取相应的映射位置。本书的第二部分更深入地探讨了其中一些技术。

代码检测在你需要关于程序执行的特定知识时提供了非常详细的信息。它允许我们跟踪程序中每个变量的任何信息。使用这种方法在优化大段代码时通常能获得最佳洞察，因为你可以使用自顶向下的方法（检测主函数然后深入到其调用者）来更好地理解应用程序的行为。代码检测使开发者能够观察应用程序的架构和流程。对于处理不熟悉代码库的人来说，这种技术特别有帮助。

代码检测技术在实时场景（如视频游戏和嵌入式开发）的性能分析中被大量使用。一些分析器将检测与其他技术（如跟踪或采样）相结合。我们将在 [@sec:Tracy] 中介绍一种这样的混合分析器 Tracy。

虽然代码检测在许多情况下很强大，但它不提供关于代码如何从操作系统或 CPU 角度执行的任何信息。例如，它不能告诉你进程被调度进出执行的频率（由操作系统知道）或发生了多少次分支误预测（由 CPU 知道）。检测代码是应用程序的一部分，与应用程序本身具有相同的权限。它在用户空间中运行，无法访问内核。

这种技术的一个更重要的缺点是，每次需要检测新的内容（比如另一个变量）时，都需要重新编译。这可能成为负担并增加分析时间。不幸的是，还有其他缺点。由于你通常关心应用程序中的热路径，你正在检测位于代码性能关键部分的内容。在热路径中注入检测代码很容易导致整体基准测试速度降低 2 倍。记住不要对检测过的程序进行基准测试。通过检测代码，你改变了程序的行为，因此你可能看不到之前观察到的相同效果。

以上所有因素都增加了实验之间的间隔时间并消耗了更多开发时间，这就是为什么工程师现在不太手动检测代码的原因。然而，自动化代码检测仍被编译器广泛使用。编译器能够自动检测整个程序（第三方库除外）以收集关于执行的有趣统计信息。自动化检测最广为人知的用例是代码覆盖率分析和配置文件引导优化（参见 [@sec:secPGO]）。

在讨论检测时，重要的是提到*二进制检测*技术。二进制检测背后的思想类似，但它是在已经构建的可执行文件上完成的，而不是在源代码上。有两种类型的二进制检测：静态（提前完成）和动态（检测代码在程序执行时按需插入）。动态二进制检测的主要优点是它不需要程序重新编译和重新链接。此外，使用动态检测，可以将检测量限制在仅感兴趣的代码区域，而不是检测整个程序。

二进制检测在性能分析和调试中非常有用。最流行的二进制检测工具之一是 Intel Pin[^1] 工具。Pin 在发生有趣事件时拦截程序的执行，并从程序中的该点开始生成新的检测代码。这使得能够收集各种运行时信息。基于 Pin 构建的最流行工具之一是 Intel SDE（Software Development Emulator）。[^2] 另一个知名的二进制检测工具叫做 DynamoRIO。[^4] 以下是使用二进制检测工具可以收集的一些内容：

* 指令计数和函数调用计数
* 指令组合分析
* 拦截函数调用和应用程序中任何指令的执行
* 内存强度和占用（参见 [@sec:MemoryIntensityFootprint]）

与代码检测一样，二进制检测只检测用户级代码，并且可能非常慢。

[^1]: Pin - [https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool)
[^2]: Intel SDE - [https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html](https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html)
[^3]: 我有一个稍微高级一些的版本，通常会复制粘贴到我正在处理的任何项目中，之后再删除。
[^4]: DynamoRIO - [https://github.com/DynamoRIO/dynamorio](https://github.com/DynamoRIO/dynamorio)。它支持 Linux 和 Windows 操作系统，并在 x86 和 ARM 硬件上运行。
