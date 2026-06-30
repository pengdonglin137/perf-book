## 多个测试单个分支 {#sec:MultipleCmpSingleBranch}

我们在本章中讨论的最后一种技术旨在通过组合多个测试来最小化动态分支指令的数量。这里的主要思想是避免对大数组的每个元素执行分支。相反，目标是同时执行多个测试，这主要涉及使用 SIMD 指令。多个测试的结果是一个向量掩码，可以转换为字节掩码，通常可以用单个分支指令处理。这使我们能够消除许多分支指令，你很快就会看到。你可能会在各种算法（如 JSON/HTML 解析、媒体编解码器等）的 SIMD 实现中遇到这种技术。

[@lst:LongestLineNaive] 显示了一个通过逐个测试字符来查找输入字符串中最长行的函数。我们遍历输入字符串并搜索换行符（`eol`）（`\n`，ASCII 中的 0x0A）。对于每个找到的 `eol` 字符，我们检查当前行是否是最长的，并将当前行的长度重置为零。此代码将为每个字符执行一条分支指令。[^1]

清单：查找最长行（逐个字符）。

~~~~ {#lst:LongestLineNaive .cpp}
uint32_t longestLine(const std::string &str) {
  uint32_t maxLen = 0;
  uint32_t curLen = 0;
  for (auto s : str) {
    if (s == '\n') {
      maxLen = std::max(curLen, maxLen);
      curLen = 0;
    } else {
      curLen++;
    }
  }
  // 如果最后没有换行符
  maxLen = std::max(curLen, maxLen);
  return maxLen;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

考虑 [@lst:LongestLineSIMD] 中显示的替代实现，它一次测试八个字符。你通常会看到使用编译器内置函数（参见 [@sec:secIntrinsics]）实现的这个想法，但是，为了清晰起见，我决定展示标准 C++ 代码。这个确切的案例出现在 Performance Ninja 的一个实验室作业中，[^2] 所以你可以自己尝试编写 SIMD 代码。请记住，我展示的代码是不完整的，因为它缺少一些角落情况；我提供它只是为了说明这个想法。

清单：查找最长行（一次 8 个字符）。

~~~~ {#lst:LongestLineSIMD .cpp .numberLines}
uint32_t longestLine(const std::string &str) {
  uint32_t maxLen = 0;
  const uint64_t eol = 0x0a0a0a0a0a0a0a0a;
  auto *buf = str.data();
  uint32_t lineBeginPos = 0;
  for (uint32_t pos = 0; pos + 7 < str.size(); pos += 8) {
    // 加载输入字符串的 8 字节块。
    uint64_t vect = *((const uint64_t*)(buf + pos));
    // 检查此块中的所有字符。
