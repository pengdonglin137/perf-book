## 代码检测 {#sec:secInstrumentation}

可能发明的第一个性能分析方法是*代码检测*。这是一种将额外代码插入程序以收集特定运行时信息的技术。[@lst:CodeInstrumentation] 显示了在函数开头插入 `printf` 语句以指示该函数是否被调用的最简单示例。之后，你运行程序并计算在输出中看到"foo is called"的次数。也许世界上的每个程序员在他们职业生涯的某个时刻都至少做过一次。

清单：代码检测

~~~~ {#lst:CodeInstrumentation .cpp}
int foo(int x) {
+ printf("foo is called\n");
 // 函数体...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

行首的加号表示该行是添加的，不在原始代码中。通常，检测代码不应该被推送到代码库中；相反，它是用于收集所需数据，之后可以删除。

[@lst:CodeInstrumentationHistogram] 展示了一个更有趣的代码检测示例。在这个虚构的代码示例中，函数 `findObject` 在地图上搜索具有某些属性 `p` 的对象的坐标。所有对象都保证最终会被定位。函数 `getNewCoords` 返回在作为参数提供的更大区域内的新坐标。函数 `findObj` 返回使用当前坐标 `c` 定位正确对象的置信度。如果是精确匹配，我们停止搜索循环并返回坐标。如果置信度高于 `threshold`，我们调用 `zoomIn` 来找到对象的更精确位置。否则，我们获取 `searchArea` 内的新坐标以在下次尝试搜索。

检测代码由两个类组成：`histogram` 和 `incrementor`。前者跟踪我们感兴趣的任何变量值及其出现频率，然后在程序完成后打印直方图。后者只是一个帮助类，用于将值推送到 `histogram` 对象中。它很简单，可以快速调整以适应你的特定需求。[^3]

清单：代码检测

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
