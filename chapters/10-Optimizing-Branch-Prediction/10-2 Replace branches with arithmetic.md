## 用算术替换分支

在某些场景中，可以用算术替换分支。[@lst:LookupBranches] 中的代码也可以使用简单的算术公式重写，如 [@lst:ArithmeticBranches] 所示。注意，对于此代码，Clang-17 编译器将昂贵的除法替换为更便宜的乘法和右移操作。

清单：用算术替换分支。

~~~~ {#lst:ArithmeticBranches .cpp}
int8_t mapToBucket(unsigned v) {             │    mov al, -1
  constexpr unsigned BucketRangeMax = 50;    │    cmp edi, 49
  if (v < BucketRangeMax)                    │    ja .exit
    return v / 10;                           │    movzx eax, dil
  return -1;                                 │    imul eax, eax, 205
}                                            │    shr eax, 11
                                             │  .exit:
                                             │    ret
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

截至 2024 年，编译器通常无法自己找到这些捷径，因此由程序员手动完成。如果你能找到一种方法用算术替换经常预测错误的分支，你很可能会看到性能改进。
