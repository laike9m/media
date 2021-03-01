如果你读任何 (C)Python 源码分析的书或者文章，里面都会讲 Python 的 [ `ceval.c` ](https://github.com/python/cpython/blob/master/Python/ceval.c) 中有一个大 `switch`，会根据不同的 opcode 跳到相应的 `case` 去执行字节码。甚至 [Anthony Shaw](https://realpython.com/courses/cpython-internals-book/#team) 的新书[《CPython Internals》](https://realpython.com/courses/cpython-internals-book/)也是这么讲的。

然而实际上，CPython （在大部分情况下）早就不这么做了。这篇文章会解释为什么。下文中 Python 均指 CPython。

## 分支预测

为了解释清楚整个过程，我们必须从底层讲起。[分支预测](https://en.wikipedia.org/wiki/Predication_(computer_architecture))（Branch Prediction）是 CPU 必备的功能。准确的分支预测能大大提高指令流水线的性能，反之则导致性能损耗。

Python 刚发明的时候，每一个 bytecode 的确需要进行一次 `switch` 跳转。而 C 语言的 `switch` 语句会导致分支预测不准，因为 CPU 无法预测程序会跳到哪个 `case`（下一个指令）。如果不进行优化，每执行一条字节码指令都将导致一次分支预测错误，可以想象对性能的损耗有多大。

## 优化技巧

问题的关键就在于 `switch...case` 的跳转是个一对多的情况。想象你面对一道有十个选项的选择题，你不会知道哪个是正确答案。然而如果有十道题各一个选项，答案就都确定了。Python 采用的优化业无非也就是这么回事。

Python 用了一种叫做 [computed goto](https://eli.thegreenplace.net/2012/07/12/computed-goto-for-efficient-dispatch-tables) 的技巧，依赖于编译器的 [label as value](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html) 功能。详细原理大家就看链接里的解释吧。总之效果就是，可以把解释器主循环中的 `switch...case` 替换成 while loop，在每个 Python 的指令处理完之后，显示地用 `goto` 跳转到下一条指令。大概长这样：

```c
int interp_cgoto(unsigned char* code, int initval) {
    /* The indices of labels in the dispatch_table are the relevant opcodes
    */
    static void* dispatch_table[] = {&&do_op1, &&do_op2, &&do_op3};
    #define DISPATCH() goto *dispatch_table[code[pc++]]

    int pc = 0;
    int val = initval;

    DISPATCH();
    while (1) {
        do_op1:
            return val;
        do_op2:
            val++;
            DISPATCH();
        do_op3:
            val--;
            DISPATCH();
    }
}
```

每个 `goto` 都是一次跳转，并且只有一个目标（下一条指令），这就让分支预测变得容易，从而性能也获得了极大提升（根据 Python 的[注释](https://github.com/python/cpython/blob/3.8/Python/ceval.c#L805)，提升可达 15% ~ 20%）。

## 实现

下面讲一下 Python 是如何实现上面提到的优化的。

### 编译中判断要不要开启 computed goto

这部分。。非常蛋疼。我花了不少时间试图弄清楚，最后还是放弃了。[这个回答](https://stackoverflow.com/a/62037189/2142577)里记录了我的一些发现。总之大概的意思就是，Python 允许用户通过 `--with-computed-gotos` 和 `--without-computed-gotos` 参数在编译的时候控制要不要开启 computed goto。默认是开启的。同时编译期间还会判断当前使用的编译器支不支持 computed goto（GCC, Clang 支持，MSVC 不支持）。如果两个条件都满足，在 ceval.c 中就会启用 computed goto。

### ceval.c 中的实现

首先，label 是在 [`opcode_targets.h`](https://github.com/python/cpython/blob/master/Python/opcode_targets.h) 中定义，并被 [include](https://github.com/python/cpython/blob/3.8/Python/ceval.c#L838) 到 ceval.c 的。

[这一段](https://github.com/python/cpython/blob/3.8/Python/ceval.c#L836-L878) 定义了 computed goto enabled 和 disabled 时的跳转行为，我把简化过的代码贴在下面：

```c
#if USE_COMPUTED_GOTOS
/* Import the static jump table */
#include "opcode_targets.h"

#define TARGET(op) op: TARGET_##op

#define DISPATCH() { \
    _Py_CODEUNIT word = *next_instr; \
    opcode = _Py_OPCODE(word); \
    oparg = _Py_OPARG(word); \
    next_instr++; \
    goto *opcode_targets[opcode]; \
}

#else

#define TARGET(op) op
#define DISPATCH() continue

#endif
```

嗯，简化了相当多的东西，但这能让我们更清晰地看出只有在 `USE_COMPUTED_GOTOS` 的情况下，`DISPATCH()` 才会用 `goto` 去跳到下一个 opcode 的 label。

下面是极简版的主循环代码：

```c
for (;;) {
  switch (opcode) {
      case TARGET(LOAD_CONST): {
        // 处理 LOAD_CONST
        DISPATCH();
      }
      // 其它 opcode 省略了
  }
}
```

先看 **computed goto 未启用**时，把代码中的宏展开（替换 `TARGET` 和 `DISPATCH`）的结果：

```c
for (;;) {
  switch (opcode) {
      case LOAD_CONST: {
        // 处理 LOAD_CONST
        continue;
      }
  }
}
```

嗯，就是这样了，很直接了当的 `switch...case`。

再看 **computed goto 启用**时，把代码中的宏展开的结果：

```c
for (;;) {
  switch (opcode) {
    case LOAD_CONST:
    TARGET_LOAD_CONST: { // 注意这里多了一个 label
      // 处理 LOAD_CONST
      _Py_CODEUNIT word = *next_instr;
      opcode = _Py_OPCODE(word);
      oparg = _Py_OPARG(word);
      next_instr++;
      goto *opcode_targets[opcode];
    }
  }
}
```

可以看到，在处理完 `LOAD_CONST` 之后，通过 `next_instr` 拿到下一个 opcode，然后执行`goto *opcode_targets[opcode];` ，跳到另一个由 opname 命名的 label（参见 [`opcode_targets.h`](https://github.com/python/cpython/blob/master/Python/opcode_targets.h) ），比如 `TARGET_STORE_NAME:`。这里的关键点在于，**代码直接跳转到了下一个指令对应的 label，并不经过 switch**。这也就回答了文章一开始的问题：

> Python 真的是靠一个 switch 来执行字节码的吗？

答案是：

**只要 Python 启用了 computed goto （比如在 Mac 和 Linux 上），字节码的执行就不依赖 switch。**

顺便一提，这个功能是在 [Python 3.2](https://docs.python.org/3/whatsnew/3.2.html#build-and-c-api-changes) 中变为默认开启的：

> Computed gotos are now enabled by default on supported compilers (which are detected by the configure script). They can still be disabled selectively by specifying `--without-computed-gotos`.