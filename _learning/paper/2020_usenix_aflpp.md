---
title: "AFL++: Combining Incremental Steps of Fuzzing Research"
excerpt: 'USENIX WOOT 2020 , 模糊测试架构设计'

collection: learning
category: paper
permalink: /learning/2020-usenix-aflpp
tags: 
  - usenix
  - fuzz

layout: single
read_time: true
author_profile: false
comments: true
share: true
related: true
---

![](../images/learning/2020_usenix_aflpp/cover.png)

## 挑战与解决方案

AFL是当前模糊测试研究的基线，它在模糊测试架构上有很好的设计：
- 覆盖率引导的反馈，粗粒度地分桶统计边覆盖率。
- 变异，三种变异策略：确定性、破坏性和拼接。
- ForkServer，把通过进程间通信机制IPC控制的forkserver插桩到目标中，避免execve()的开销。
- 持久化模式，进一步减少ForkServer的开销，将一个loop patch到测试目标中，每次fork()得到的进程，会对一批而非单个测试用例进行处理。

AFL面对的一些挑战，业内通常fork原项目在其上修改。而AFL++采用集成SOTA研究的方式来解决相应的挑战。

### 一、解决 "调度算法单一" 的挑战

一个覆盖率引导的模糊测试工具，应有不同的优先级算法来调度模糊测试流水线中的测试用例。调度器的目标是：通过智能的测试用例选择来提高总体覆盖率和bug检测效率。

#### 1.集成AFLFast

Marcel Böhme等人提出的AFLFAST：相比于向高频路径倾斜过多的AFL，作了几项改进，转而强调低频路径。低频的路径是指，在模糊测试中测试用例很难或者是很少到达的路径。以探索更多分支进而发现更多bug。

AFL++集成了AFLFAST的所有调度方案：fast, coe, explore, quad, lin, exploit。这些调度方案取决于以下变量：
1. 当从队列中选出那颗种子的时候；
2. 与种子具有相同覆盖度的生成输入的数量；
3. 具有相同覆盖率的生成测试用例的平均数量；

除此之外，AFL++ 还添加了 mmopt 和 rare 两种调度方式。mmopt会提高最新种子的得分，以帮助更深入地探索新发现的路径。rare则不考虑种子的运行时间，并且特别关注那些与其他种子很少覆盖边的种子。

### 二、解决 "障碍绕过困难" 的挑战

很多时候，覆盖率引导的模糊测试会遇到障碍，无法探索障碍背后的路径。典型的障碍是大规模的比较，如字符串比较和校验和检查。为了解决这个问题，进行了一系列的研究。

#### 1.集成LAF-Intel

LAF-INTEL是一项旨在绕过多字节困难比较的工作，它将多字节比较拆分为多个单字节比较。这样，这些比较可以逐字节传递，覆盖率引导的模糊测试器便可以接收每个部分的反馈。详细地来说，LAF Intel作了以下工作：

1. 将>=(<=)运算符简化为 >(<) 和==比较的链。
2. 将有符号整数比较更改为仅比较符号和无符号比较的链。
3. 将所有位宽为64、32或16位的无符号整数比较拆分为8位的多重比较链。

LAF-intel列出了针对多字节困难比较、字符串比较两种情况的拆分方法。

**针对多字节困难比较：**
对于左图这样的代码，只有当变量input的值等于0xabad1dea时，才会执行bug代码。每当AFL生成一个不符合的输入，都不会使覆盖率产生变化，即使输入是几乎正确的0xabad1deb，也不会产生有作用的反馈。

![](../images/learning/2020_usenix_aflpp/laf_bytes.png)

如果像右图这样将其以单字节拆分，相比之前的四字节比较容易产生新的边，便可以引导AFL生成正确的输入，进而探索到bug代码。

**同理，针对字符串比较：**
调用了strcmp函数的左图，可修改为单字符比较的右图。

![](../images/learning/2020_usenix_aflpp/laf_string.png)

AFL++在LLVM插桩模式下集成了  LAF-INTEL pass。

#### 2.集成RedQueen

REDQUEEN在KAFL的基础上，探索了绕过困难比较和(嵌套)校验和检查的方案，但不使用诸如污点跟踪或符号执行之类的昂贵技术。该模糊测试方案专注于Input-to-State（I2S）的比较，这是一种与至少一个操作数中的输入直接相关的比较类型。作者表明，许多障碍都是这种类型的，并开发了一种定位和绕过它们的技术。
I2S关联的例子如下图所示。对cmp指令进行hook，运行指令时可以观察到eax的值为VALU，0x44434241则为ABCD(均为小端序)。

![](../images/learning/2020_usenix_aflpp/redqueen.png)

input中同样有VALU出现，由此推断：如果将输入中的VALU替换为ABCD，就有较大可能绕过这个障碍。

AFL++将RedQueen集成到变异器中。

### 三、解决 "边覆盖计数溢出" 的挑战

#### 1.NeverZero

AFL++对AFL的命中计数机制进行了优化。在位图中用一个字节保存命中计数的问题是，边执行的计数可能会溢出。如果边计数是256的倍数时，相应的字节将溢出到0。我们试图用两种解决方案来解决这个问题：NeverZero和饱和计数器。

NeverZero方案：为了避免溢出到0，将进位标志添加到位图项，因此，如果边至少执行一次，则该项永远不会为0。
饱和计数器方案：在边计数达到255时冻结计数器。

在所有项都饱和(255)之后的情况下，NeverZero比饱和计数器更好，因为NeverZero可以以非常低的成本区分具有不同计数的不同输入。
总之，AFL++选择将NeverZero作为大多数插桩后端的默认值。饱和计数器方案仍然存在于AFL++源码的一个分支中，用于进一步研究或复制。

### 四、解决 "变异结构化输入" 的挑战

除了集成并优化智能调度、绕过障碍方面的研究成果之外，AFL++相比于AFL还有更多有用的功能。

#### 1.设计定制变异器API

AFL++可以很容易地在新的模糊测试场景中进行扩展，适用于发现指定目标的漏洞。定制变异器允许模糊测试研究在AFL++之上构建新的调度、突变和修剪，而无需对AFL进行fork、patch，就像当前的许多工具一样。

对此的支持最初是在Holler fork的AFL中独立开发的，但后来得到了扩展，具有许多新功能。插件可以用C/C++编写，也可以用Python进行原型化。

以C/C++编写的变异器库为例，要在AFL++中单独使用该库，需要先设置环境变量：
``` shell
export AFL_CUSTOM_MUTATOR_ONLY=1
export AFL_CUSTOM_MUTATOR_LIBRARY="/path/to/libmutator.so"
```

### 四、解决 "插桩冲突，插桩后端单一" 的挑战

#### 1.插桩后端

AFL++支持以下五种插桩后端：LLVM，GCC、QEMU、Unicorn和QBDI。

**LLVM后端：**
LLVM是编译器后端，常搭配clang使用，若想使用LLVM模式，需要使用afl-clang-fast来编译目标文件。在LLVM模式下，除了支持AFL方案外，AFL++还支持一系列覆盖率度量：

1. Ctx coverage(上下文敏感的边覆盖率)：与AFL统计边覆盖率不同，Ctx方案把栈状态也加入了边覆盖率的映射运算。因为上下文改变之后，程序到达同一个分支可能触发新的栈状态，这种方案在代码覆盖率方面似乎是有效的，但会带来更多的碰撞和更低的速度。

2. Ngram：在记录边时不止异或当前块和目的块，还异或当前块的前N-1个块(其中N是2到16之间的数字)这样虽然增加了异或操作，但是能提高性能。

**GCC编译器：**
除了旧的afl-gcc封装外，afl++还提供了一个gcc插件。它包括对延迟初始化和持久模式的支持，如AFL LLVM模式。支持的功能与LLVM不一样，但AFL++正在计划提供其他功能，目标是实现功能的均等。

**QEMU模式：**
在AFL++中，AFL-QEMU补丁得到了升级。与其他二进制插桩相比(如基于RetroWrite的插桩)，QEMU模式在模拟时动态插桩，好处是可以在程序运行的过程中修改、插入代码，但这也使得模糊测试的效率极大降低。最近，QAsan等扩展了AFL++的QEMU模式，以支持对堆冲突的清理，并结合了ASan的基于动态二进制翻译的实现。
1. CompareCoverage：为了缩小源码级和二进制级模糊测试之间的功能差距，AFL++ QEMU模式可以使用CompareCoverage以类似于LAF INTEL的方式拆分比较。与LLVM pass不同，该代码不会被修改，但是会hook所有比较，并会比较每个操作数的每个字节，如果相等，则在位图上分别进行记录。
2. Persistent模式：与旧的AFL-QEMU模式不同，AFL ++的QEMU模式支持持久模式。此模式最高可以达到10倍加速，因此AFL++建议尽可能使用此模式。

有两种主要方法可以实现QEMU模式的Persistent模式：
1. 循环一个函数：用户指定一个函数的地址，模糊器会自动在while(__AFL_LOOP(NUM))循环中使用该函数，修改其返回地址。该地址也可以不是函数的第一条指令，但是在这种配置下，用户必须提供栈上的偏移量，以正确定位要修改的返回地址。
2. 指定入口点和出口点：用户可以指定循环的第一条指令和最后一条指令的地址，QEMU将在运行时修改代码以在这些地址之间生成循环；

![](../images/learning/2020_usenix_aflpp/modes.png)

## 工程实现

### 一、整体架构

整体架构设计如下图所示，由DeepWiki生成。

![](../images/learning/2020_usenix_aflpp/infra.png)

核心组件：
1. 编译器包装器，插桩主逻辑在 src/afl-cc.c
2. 模糊测试引擎，主流程逻辑在 src/afl-fuzz.c
3. 二进制模糊测试模式，封装在 frida_mode/ 、 unicorn_mode/ 和 qemu_mode/ 等目录下
4. 定制变异器扩展，以动态库的形式加载。
5. 其他关键组件，种子池管理组件，覆盖率统计反馈组件等。

### 二、编译器包装器

包装器通过下面的函数决定编译器模式：
- compiler_mode_by_callname()，检查调用名
- compiler_mode_by_environ()，检查环境变量
- compiler_mode_by_cmdline()，检查命令行参数

各种编译模式（非二进制）统一管理，如下图所示。

![](../images/learning/2020_usenix_aflpp/afl-cc.png)

具体实现在 instrumentation/ 文件夹中。

### 三、模糊测试引擎

通过下面的源码文件实现核心的模糊测试流程：
- 主流程，在 src/afl-fuzz.c 的 main()
- 单输入模糊测试，在 src/afl-fuzz-one.c，关键函数：fuzz_one_original(), fuzz_one()
- 变异，在 src/afl-fuzz-mutators.c，关键函数：Havoc, deterministic, splicing
- 队列管理，在 src/afl-fuzz-queue.c，关键函数：add_to_queue(), queue culling
- 状态管理，在 src/afl-fuzz-state.c	，关键函数：afl_state_init(), global state
- 统计，在 src/afl-fuzz-stats.c	，关键函数：show_stats(), performance metrics
- 高级扩展RedQueen/CMPLOG，在 src/afl-fuzz-redqueen.c	，关键函数：input_to_state_stage()

模糊测试器在 afl_state_t 中维护各种状态：
- queue_top, queue_cur: Queue management pointers
- virgin_bits, virgin_crash, virgin_tmout: Coverage tracking
- fsrv: Fork server state (afl_forkserver_t)
- Mutation engine parameters and statistics

#### 1.ForkServer和执行

forkserver是迭代测试的关键：

![](../images/learning/2020_usenix_aflpp/forkserver.png)

关键函数：
- afl_fsrv_start() - Initialize fork server ，在 src/afl-forkserver.c
- afl_fsrv_run_target() - Execute one test case
- __afl_forkserver_main() - Fork server loop in target，在插桩逻辑中 instrumentation/afl-compiler-rt.o.c

#### 2.代码覆盖率统计

The coverage mechanism:
- Each edge is assigned a unique cur_location value at compile time
- Runtime: map[cur_location ^ (__afl_prev_loc >> 1)]++
- The fuzzer reads trace_bits (copy of __afl_area_ptr)
- classify_counts() bucketizes hit counts (1, 2, 3, 4-7, 8-15, 16-31, ...)
- Comparison with virgin_bits detects new coverage

Key Variables:
- __afl_area_ptr - Coverage map (shared memory) ，instrumentation/afl-compiler-rt.o.c
- __afl_prev_loc - Previous edge location ，instrumentation/afl-compiler-rt.o.c
- trace_bits - Fuzzer's copy of coverage ，src/afl-fuzz.c
- virgin_bits - Tracks discovered edges ，src/afl-fuzz-state.c

#### 3.执行和反馈

Write mutated input to .cur_input file or shared memory
Execute via afl_fsrv_run_target()
Read coverage from trace_bits
If new coverage: add_to_queue(), save to queue/ directory
If crash: save to crashes/ directory
If timeout: save to hangs/ directory (if AFL_KEEP_TIMEOUTS)

### 四、高级特性

CMPLOG/RedQueen 实现在 src/afl-fuzz-redqueen.c
LAF-Intel 实现在 instrumentation/split-compares-pass.so.cc
