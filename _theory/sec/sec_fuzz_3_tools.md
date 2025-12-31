---
title: "动态漏洞挖掘：工具 （Part 3/4）"
excerpt: 'AFL(++)，Atheris，AFLgo'

collection: theory
category: sec
permalink: /theory/sec/fuzz-tools
tags: 
  - fuzz

layout: single
read_time: true
author_profile: false
comments: true
share: true
related: true
---

![](../../images/theory/sec/fuzz_tools/aflpp.png)

## AFL(++)

2013年提出的AFL(American Fuzzing Lop)方案。

### 一、工作流程[^1]

![](../../images/theory/sec/fuzz_tools/progress.png)

1. 确定模糊测试的对象，是命令行工具、动态库、虚拟机还是数据库等等。模糊测试的对象决定了构造的测试用例怎么输入，是从标准输入、编写harness还是通过驱动程序。
2. 源码插桩/静态重写/qemu插桩，以便跟踪边覆盖率。
3. 编写自定义的变异器，比如针对libxml2库，变异生成的测试用例需要符合xml结构，这就需要自己写变异函数。
(可选)构造字典，提高变异效率。
4. 创建语料库，选择初始种子库作为输入队列。
5. 通过afl-fuzz启动被测程序，循环执行模糊测试流程，检测是否有crash或者ASAN报错。

### 二、环境变量

**1.准备阶段变量**

__AFL_SHM_ID：共享内存shmid传递给被插桩程序，将覆盖率信息存储在其中。
AFL_MAP_SIZE：设置覆盖率统计图的大小。
AFL_SKIP_CPUFREQ：跳过对CPU扩展策略的检查，会有一些性能损失。
AFL_SKIP_BIN_CHECK：跳过插桩检查，需要目标程序自行控制模糊测试流程。

**2.测试阶段变量**

AFL_DEBUG：显示调试信息、目标程序的输出和错误信息。
AFL_DEBUG_CHILD：只显示目标程序的输出和错误信息。
AFL_DISABLE_TRIM：禁用测试用例裁剪。
AFL_FAST_CAL：快速校验，减少校验中的循环次数。

**3.Sanitizer变量**

AFL_USE_ASAN：使用ASAN，进行无冲突源码插桩和afl-fuzz判定bug。
AFL_USE_QASAN：使用QASAN，进行无冲突QEMU运行时插桩，afl-fuzz判定bug。
AFL_USE_KASAN：使用KASAN，进行内核模糊测试时，afl-fuzz判定bug。

**4.变异器变量**

AFL_CUSTOM_MUTATOR_LIBRARY：指定定制变异器。
AFL_CUSTOM_MUTATOR_ONLY：只使用指定的定制变异器。

**5.QEMU模式变量**

AFL_INST_LIBS：使用QEMU对动态链接库进行动态插桩。
AFL_QEMU_PERSISTENT_ADDR：通常是harness函数的偏移，QEMU通过非PIE形式加载目标程序，因此可以定位到harness。
AFL_QEMU_PERSISTENT_HOOK：指定定制的hook.so文件，hook.c需要根据harness参数传递时使用的寄存器来定制，因此需要搭配AFL_QEMU_PERSISTENT_GPR。
AFL_QEMU_PERSISTENT_GPR：持久模式，恢复寄存器。

### 三、插桩工具

编译时修改CC和CXX：
afl-gcc/afl-g++：AFL中使用，被AFL++淘汰。
afl-cc/afl-c++：使用clang编译插桩。
afl-clang/afl-clang++：指向afl-cc/afl-c++。
afl-clang-fast/afl-clang-fast++/：指向afl-cc/afl-c++。
afl-clang-lto/afl-clang-lto++：需要下载llvm和lld，设置LLVM_CONFIG=llvm-config-xx，再编译AFL++，指向afl-cc/afl-c++。

### 四、状态窗口

![](../../images/theory/sec/fuzz_tools/state.png)

Process timing：Fuzzer运行时长、以及距离最近发现的路径、崩溃和挂起经过了多长时间。
Overall results：Fuzzer当前状态的概述。
Cycle progress：我们输入队列的距离。
Map coverage：目标二进制文件中的插桩代码所观察到覆盖范围的细节。
Stage progress：Fuzzer现在正在执行的变异策略、执行次数和执行速度。
Findings in depth：有关我们找到的执行路径，异常和挂起数量的信息。
Fuzzing strategy yields：关于突变策略产生的最新行为和结果的详细信息。
Item geometry：有关Fuzzer找到的执行路径的信息。
CPU load：CPU利用率

何时终止fuzz：
1. 状态窗口中"cycles done"字段颜色,随着周期数不断增大，其颜色也会由洋红色，逐步变为黄色、蓝色、绿色。当其变为绿色时，继续Fuzzing下去也很难有新的发现了。
2. 距上一次发现新路径（或者崩溃）已经过去很长时间了。
3. 目标程序的代码几乎被测试用例完全覆盖，对于小型项目，可能存在该情况。

### 五、输出目录

queue目录：存放所有具有独特执行路径的测试用例。
crashes目录：导致目标接收致命signal而崩溃的独特测试用例。
hangs目录：导致目标超时的独特测试用例。
fuzz_bitmap文本：当前位图的内容。
fuzzer_stats文本：afl-fuzz的运行状态。
plot_data文本：用于afl-plot绘图。

### 六、处理crash

判断是正常退出还是由于漏洞、ASAN退出，是通过检查子进程状态，即被测进程的状态，进程信号来判断是normal还是abort。

afl-fuzz crash模式：
将crash的例子作为输入，afl-fuzz -C -i -o ./vuln，在已知崩溃的条件下，探索其他代码路径。

crashwalk：
如果你想得到更细致的crashes分类结果，以及导致crashes的具体原因，这个工具基于gdb的exploitable插件，crashwalk支持AFL和Manual两种模式。

基于ASAN/QASAN：
配合harness逻辑触发asan日志。

## 变异

### 二、定制变异器

在定制变异器中，可以自由使用havoc对特定字段进行变异。
AFL++可以很容易地在新的模糊测试场景中进行扩展，适用于发现指定目标的漏洞。定制变异器允许模糊测试研究在AFL++之上构建新的调度、突变和修剪，而无需对AFL进行fork、patch，就像当前的许多工具一样。
对此的支持最初是在Holler fork的AFL中独立开发的，但后来得到了扩展，具有许多新功能。插件可以用C/C++编写，也可以用Python进行原型化。
以C/C++编写的变异器库为例，要在AFL++中单独使用该库，需要先设置环境变量：
``` c
export AFL_CUSTOM_MUTATOR_ONLY=1
export AFL_CUSTOM_MUTATOR_LIBRARY="full/path/to/libmutator.so"
```

## Atheris

[^2]

## AFLgo

结合静态分析，叫定向模糊测试，源码中指定一个可能存在漏洞的函数，称为目标点，然后将边覆盖率反馈，变成距离这个目标点最近反馈。使得更快的覆盖到目标点。计算距离有函数级、基本块级两种。

[^1]: afl under hood https://blog.ritsec.club/posts/afl-under-hood/
[^2]: Python Fuzzing https://appsec.guide/docs/fuzzing/python/
