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

## C/C++模糊测试

参考实现：
- libFuzzer
- AFL++
- HonggFuzz
- libAFL

AFL++实现SUT执行的模式：
- 子进程。对于每个测试用例，都会创建一个子过程（execve）。例如，当SUTs通过命令行参数或标准输入接收输入时，就需要该函数。在这种情况下，每次调用都会启动一个新进程。可以通过设置环境变量AFL_NO_FORKSRV=1来启用该模式。由于性能较低，不建议使用该模式，但如果SUT在main函数前进行初始化，则可能需要使用该模式。
- fork server。为SUT启动子进程。程序一直执行到主函数。在启动实际的main函数之前，要fork进程。对于每次执行，流程都是从该执行点派生出来的。这节省并减少了execve系统调用的数量，从而提高了性能。这只在通过标准输入进行模糊测试时有效。
- 持久模式。模糊器在单个进程中运行测试用例。当测试用例崩溃时，父进程会重启模糊测试循环。当使用libfuzzer风格的LLVMFuzzerTestOneInput时，这是可用的。请注意，重要的是，测试在每次执行后都清楚地重置SUT的状态。否则，执行可能会相互影响。持久模式与__AFL_FUZZ_INIT（）和__AFL_FUZZ_TESTCASE_BUF宏有关。
- 共享内存模糊测试。只要启用了持久模式，测试用例仍然通过标准输入或从文件中读取。共享内存特性与__AFL_INIT（）和__AFL_LOOP(n)宏有关。通常，当在线遇到关于持久模式的例子时，也会启用共享内存模糊测试。参数模糊测试中的例子启用了持久模式和共享模式。如果模糊测试设置和操作系统允许，在上述所有情况下都会使用共享内存交换覆盖信息。这种共享内存的使用与afl++的“共享内存”特性没有关系。

基于LLVM的插桩：
LLVM项目提供SanitizerCoverage接口和工具来允许覆盖率数据收集。libFuzzer或afl++在进行模糊测试时的默认插桩会使用这个API来引导模糊器发现更多的代码。

SUT补丁：克服障碍
代码库通常对模糊测试不友好。例如，如果代码使用校验和或依赖于全局状态（如系统时间种子PRNG（即使用rand）），会导致代码对相同输入的行为不同，则可能发生这种情况。请参考实用线束规则来了解更多关于SUT中潜在问题的信息。如果你在SUT中遇到校验和或全局状态，可能需要应用专门针对模糊测试的补丁，改变程序在模糊测试期间的行为。

通常，C/ c++模糊器会定义宏FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION。至少对于libFuzzer、afl++、LibAFL和Hongfuzz来说是这样。如果定义了宏，那么程序将被编译以进行模糊测试。通过使用基于该宏的条件编译，可以克服代码中的障碍，例如散列检查，这些检查通常会阻碍模糊测试覆盖更深层的代码路径。下图展示了一个例子。
``` c
if (checksum != expected_hash) {
// Eliminate the need for guessing checksums by ignoring this error during fuzzing
#ifndef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION
  return -1;
#endif
}

// Continue program execution
```

注意，这意味着SUT在模糊测试和生产环境中的行为不同。在模糊测试中，不小心跳过检查可能会导致误报。例如，跳过配置文件的验证可能会导致SUT崩溃，因为代码期望配置值具有特定的格式。
该变量在实际应用中有一个具体示例，那就是在 OpenSSL 项目中。该项目利用此变量来改变加密算法的运作方式。

### 一、工作流程[^1]

2013年提出的AFL(American Fuzzing Lop)方案。

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

### 七、定制变异器

在定制变异器中，可以自由使用havoc对特定字段进行变异。
AFL++可以很容易地在新的模糊测试场景中进行扩展，适用于发现指定目标的漏洞。定制变异器允许模糊测试研究在AFL++之上构建新的调度、突变和修剪，而无需对AFL进行fork、patch，就像当前的许多工具一样。
对此的支持最初是在Holler fork的AFL中独立开发的，但后来得到了扩展，具有许多新功能。插件可以用C/C++编写，也可以用Python进行原型化。
以C/C++编写的变异器库为例，要在AFL++中单独使用该库，需要先设置环境变量：
``` c
export AFL_CUSTOM_MUTATOR_ONLY=1
export AFL_CUSTOM_MUTATOR_LIBRARY="full/path/to/libmutator.so"
```

## Rust模糊测试

参考的工具：
- cargo-fuzz

基于LLVM的插桩：
llvm-cov，参考libfuzzer_sys的实现

## Python模糊测试[^2]

参考的工具：
- Atheris 

两种方案：
- 对纯 Python 代码进行模糊测试，借助有效的 Atheris 环境，让我们对一些 Python 代码进行模糊测试。
``` python
import sys
import atheris

@atheris.instrument_func
def test_one_input(data: bytes):
    if len(data) == 4:
        if data[0] == 0x46:  # "F"
            if data[1] == 0x55:  # "U"
                if data[2] == 0x5A:  # "Z"
                    if data[3] == 0x5A:  # "Z"
                        raise RuntimeError("You caught me")

def main():
    atheris.Setup(sys.argv, test_one_input)
    atheris.Fuzz()

if __name__ == "__main__":
    main()
```
通常情况下，您会希望使用诸如 atheris.instrument_imports 或 atheris.instrument_all 这样的工具来对应用程序或库的更广泛部分进行模糊测试。
- 对 Python C 扩展进行模糊测试

## 定向模糊测试

参考工具：
- AFLgo

结合静态分析，叫定向模糊测试，源码中指定一个可能存在漏洞的函数，称为目标点，然后将边覆盖率反馈，变成距离这个目标点最近反馈。使得更快的覆盖到目标点。计算距离有函数级、基本块级两种。

## 快照式模糊测试

参考工具：
- https://github.com/0vercl0k/wtf（win，linux）
- https://github.com/Cisco-Talos/snap_wtf_macos（macos）
- https://github.com/nyx-fuzz

快照式模糊测试是这样一个过程：对目标程序或操作系统进行快照采集——包括内存状态、寄存器状态或其他恢复执行所需的信息——然后在模拟环境中继续执行，对内存中的数据进行变异，并在执行出现崩溃或达到指定位置时将程序重置回原始快照状态。

https://appsec.guide/docs/fuzzing/snapshot-fuzzing/

## 语料库

https://github.com/strongcourage/fuzzing-corpus - various file formats
https://github.com/dvyukov/go-fuzz-corpus - various file formats
https://github.com/randombit/crypto-corpus - asn1, cert, crl, oscp, pkcs1, pkcs8, spki, TLS client/server flows, x509_dn, x509_path
https://github.com/openssl/fuzz-corpora
https://github.com/wasmx/wasm-fuzzing-corpus.git - WASM corpus
https://github.com/bytecodealliance/wasmtime-libfuzzer-corpus
https://github.com/Cisco-Talos/clamav-fuzz-corpus
https://github.com/fumfel/yara-fuzzing-corpus
https://github.com/mozilla/nss-fuzzing-corpus
https://github.com/ethereum/solidity-fuzzing-corpus
https://github.com/FerretDB/fuzz-corpus - corpus for FerretDB
https://github.com/tmux/tmux-fuzzing-corpus - tmux
https://github.com/mozilla/pdf.js/tree/master/test/pdfs - PDFs

## AI驱动

### 一、Big Sleep 

https://projectzero.google/2024/10/from-naptime-to-big-sleep.html

原理：变体分析
"This was a previous bug; there is probably another similar one somewhere"

## 二、AI加持的OSS-Fuzz 

https://security.googleblog.com/2024/11/leveling-up-fuzzing-finding-more.html

原理：辅助编写harness
参考：
- https://github.com/google/oss-fuzz-gen 开源
- https://introspector.oss-fuzz.com/ 识别harness函数

编写harness是一个项目特定的手动过程，类似于编写单元测试。但对于项目维护者来说，编写一套全面的模糊目标是一项艰难的期望，他们通常是志愿者。

## 实例

### 一、二维码解析库模糊测试 

https://evilpan.com/2023/06/10/fuzzing-qrcode/

想法：
1. 微信二维码解码器 https://github.com/opencv/opencv_contrib ， 选择目标第一步：调研技术选型
2. 变异策略，采用结构化感知，针对性变异：
- 测试解析逻辑，则变异文件格式
- 测试解码逻辑，则保证格式正确，变异内部数据

### 二、Exploiting OPC-UA in Every Possible Way 

https://infocondb.org/con/def-con/def-con-31/exploiting-opc-ua-in-every-possible-way-practical-attacks-against-modern-opc-ua-architectures

大多数产品严重依赖于基础协议栈。
OPC-UA protocol stacks and products：

| Stack/Application Name      | Lang   | Complex Deep Nested Variants DoS | Worker Starvation DoS | Long Chunks DoS          | Unlimited Monitored Items DoS | UTF8 - UTF16 Conversions          |
|-----------------------------|--------|----------------------------------|------------------------|--------------------------|-------------------------------|-----------------------------------|
| open62541                   | C      | V                                | V                      | CVE-2022-25761           | V                             | V                                 |
| freeopcua (c++)             | C++    | V                                | V                      | V                        | CVE-2022-24298                | V                                 |
| ASNeG OpcUaStack            | C++    | V                                | V                      | CVE-2022-24381           | V                             | V                                 |
| Unified Automation          | C++    | V                                | V                      | V                        | Fixed, No CVE                 | V                                 |
| Softing OPC UA SDK          | C++    | V                                | V                      | V                        | V                             | V                                 |
| Kepware KEPServerEX         | C/C++  | V                                | V                      | V                        | V                             | CVE-2022-2848<br>CVE-2022-2825   |
| node-opcua                  | NodeJS | V                                | V                      | CVE-2022-21208           | CVE-2022-24375                | V
| python-opcua                | Python | V                                | V                      | CVE-2022-25304           | V                             | V                                 |
| opcua-asyncio               | Python | V                                | V                      | CVE-2022-25304           | V                             | V                                 |
| eclipse-milo                | Java   | V                                | V                      | V                        | CVE-2022-25897                | V                                 |
| locka99                     | Rust   | CVE-2022-25903                   | V                      | CVE-2022-25888           | V                             | V                                 |
| OPC Foundation .NET Stack   | C#     | CVE-2021-27432 (*)               | V                      | CVE-2022-29864           | V                             | V                                 |
| Prosys OPC UA               | Java   | V                                | V                      | CVE-2022-30551           | V                             | V                                 |
| OPC UA Legacy Java Stack    | Java   | V                                | V                      | V                        | V                             | V                                 |

| Name | Programming language | Is Open Source? |
|---|---|---|
| node-opcua | NodeJS | Yes |
| open62541 | C | Yes |
| freeopcua (C++) | C++ | Yes |
| python-opcua | Python | Yes |
| opcua-asyncio | Python | Yes |
| eclipse-milo | Java | Yes |
| ASNeG OpcUaStack | C++ | Yes |
| locka99 | Rust | Yes |
| Unified Automation | C++ | No |
| OPC Foundation .NET Stack | C# | Yes |
| Softing OPC UA SDK | C++ | No |
| Prosys OPC UA | Java | No |
| OPC UA Legacy Java Stack | Java | Yes |
| S2OPC | C | Yes |
| LibUA | C# | Yes |

We also researched popular products 
such as:
• Softing Secure Integration Server
• PTC Kepware KEPServerEx
• Triangle Microworks SCADA Data 
Gateway
• Honeywell Matrikon
• Inductive Automation Ignition

参考的模糊测试工具实现：
https://github.com/claroty/opcua_network_fuzzer
https://github.com/claroty/busybox-fuzzing

[^1]: afl under hood https://blog.ritsec.club/posts/afl-under-hood/
[^2]: Python Fuzzing https://appsec.guide/docs/fuzzing/python/
