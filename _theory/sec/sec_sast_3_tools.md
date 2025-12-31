---
title: "静态漏洞挖掘：工具 （Part 3/4）"
excerpt: 'Joern，CodeQL'

collection: theory
category: sec
permalink: /theory/sec/sast-tools
tags: 
  - code audit
  - sast

layout: single
read_time: true
author_profile: false
comments: true
share: true
related: true
---

![](../../images/theory/sec/sast_tools/joern.png)

## joern

joern是针对源代码、字节码和二进制码进行健壮性分析的平台。核心数据结构是代码属性图（CPG），作为跨语言代码分析的抽象表示，CPG存储于图数据库中，使用特定的Scala查询语句能够挖掘代码中的漏洞。

https://elmanto.github.io/posts/sast_derby_joern_vs_codeql

### 一、适配的语言

joern除了能够分析源代码，还能够分析编译后的字节码、二进制码。

**极度**适配的语言：
- C/C++，基于Eclipse CDT
- Java，基于JavaParser

**高度**适配的语言：
- JavaScript，基于GraalVM
- Python，基于JavaCC
- x86/x64，基于Ghidra

**中度**适配的语言：
- JVM Bytecode，基于Soot
- Kotlin，基于IntelliJ PSI
- PHP，基于PHP-Parser
- Go，基于go.parser

### 二、核心功能

joern有以下5个核心功能：
1. 健壮的解析，对于不能提供工作构建环境或部分代码丢失，也能够导入joern进行分析。
2. CPG，创建具有语义的代码属性图，存储在基于内存存储的图数据库中。
3. 污点分析，提供静态污点分析引擎，跟踪恶意数据的传播。
4. 查询语句，基于Scala实现了查询语言，用于手动制定漏洞的查询规则，也可以使用机器学习技术自动推断漏洞。
5. 使用CPG pass进行扩展，CPG是多层的，在不同的抽象层次上都提供有关代码的信息，joern允许通过CPG pass添加额外信息，扩展查询语句。

### 三、查询语言

查询基于Scala，以step的形式从CPG查询可能存在的各种漏洞模式。最顶层的step是cpg，从cpg出发能够实现复杂的查询规则。下面是常用的step。

**1.call step**

作为顶层step时，表示代码中所有的调用点，通常根据其子step：name或者filter进行匹配。
| 子step | 使用示例 |
| :-: | :-: |
|Code string	|cpg.call.code.l
|Call name	|cpg.call.name.l
|Location	|cpg.call.name("foo").location.l
|Calling method	|cpg.call.name("foo").method.l
|Argument	|cpg.call.name("foo").argument.code.l
|Filter	|cpg.call.filter(_.argument.code("42")).name.l

作为子step时，用来列出调用和被调用的方法。

| step | 使用示例 |
| :-: | :-: |
| .callOut | cpg.method.name("main").callOut.name.l
| .callIn | cpg.method.name("exit").callIn.code.l

**2.method step**


**3.taint step**

reachableBy 和 reachableByFlows作为污点step的实现。

如果source到sink是可到达的，reachableBy会返回具体的source，reachableByFlows会返回source到sink的路径。

**4.filt step**

除了step自身的模式匹配，filter step用来提供复杂的过滤机制。除此之外，还有where和wherenot。

下面是一个查询示例：

``` scala
val src =
  cpg.method(".*malloc$").callIn.where(_.argument(1).arithmetic)

cpg.method("(?i)memcpy").callIn.filter { memcpyCall =>
  memcpyCall
    .argument(1)
    .reachableBy(src)
    .where(_.inAssignment.target.codeExact(memcpyCall.argument(1).code))
    .whereNot(_.argument(1).codeExact(memcpyCall.argument(3).code))
    .hasNext
}
```

在c语言中，堆溢出有很多情况引发，上面的查询示例针对的是其中一种情况：
1. 使用malloc系列函数

**基础用法**

**1.方法**

列出所有匹配的方法
cpg.method.name(".\*parse.*").name.l

创建匹配的方法的键值对 <地点，代码>
cpg.method.name(".\*parse_sig.*").map( m=> (m.location.filename, m.start.dump)).l

查找一个方法定义的局部变量
cpg.method.name("parse_public_key_packet").local.name.l

查找在哪个文件，哪一行
cpg.method.name("parse_public_key_packet").location.map( x=> (x.lineNumber.get, x.filename)).l

查找方法中定义的第一个局部变量的类型
cpg.method.name("parse_public_key_packet").local.typ.name.l.head

查找方法中的所有调用（call-site）
cpg.method.name("parse_public_key_packet").call.name.l

查找调用给定方法的方法
cpg.method.name("parse_public_key_packet").caller.name.l

**2.图的重复遍历**

找出从给定方法开始向上的调用序列
cpg.method.name("parse_public_key_packet").repeat(_.caller)(_.emit).name.l

查找方法的调用，直到遇到给定的方法
cpg.method.name("download_key").repeat(_.callee)(_.emit.until(_.isCallTo("parse_public_key_packet"))).name.l

**3.类型、变量和过滤器**

列出所有类型为`vlc_.*`的局部变量
cpg.types.name("vlc_.*").localOfType.name.l

查找结构体的成员变量

cpg.types.name("vlc_log_t").map(x=> (x.name, x.start.member.name.l)).l

找到局部变量并根据其类型过滤它们

cpg.local.where(_.typ.name(“vlc_log_t”)).name.l

它们在哪个方法中使用？

joern > cpg.local.where(_.typ.name(“vlc_log_t”)).method.dump

获取这些方法所在的文件名

cpg.local.where(_.typ.name(“vlc_log_t”)).method.file.name.l

**4.代码复杂度**

找出参数超过4个的函数

cpg.method.filter(_.parameter.size > 4).name.l

用> 4控制结构标识函数（圈复杂度）

cpg.method.filter(_.controlStructure.size > 4).name.l

找出代码超过500行的函数

cpg.method.filter(_.numberOfLines >= 500).name.l

使用多个return语句标识函数

cpg.method.filter(_.ast.isReturn.l.size > 1).name.l

识别超过4个循环的函数

cpg.method.filter(_.controlStructure.controlStructureType("FOR|DO|WHILE").size > 4).name.l

识别嵌套深度大于3的函数

cpg.method.filter(_.depth(_.isControlStructure) > 3).name.l

**5.调用库**

程序使用的所有外部方法的名称

cpg.method.external.name.l.distinct.sorted

所有对strcpy的调用

cpg.call("str.*").code.l

所有调用strcpy的方法

cpg.call("str.*").method.name.l

查看参数：sprintf的第二个参数不是字面量
joern> cpg.call("sprintf").argument(2).whereNot(_.isLiteral).code.l

**内存分配错误-零分配/溢出**

malloc进行算术运算的位置

cpg.call("malloc").where(_.argument(1).isCallTo(Operators.multiplication)).code.l

识别是否有某个方法调用了这些奇怪的mallocs
def source = cpg.method.name(".*alloc.*").parameter
def sink = cpg.call("malloc").where(_.argument(1).isCallTo(Operators.multiplication)).argument
sink.reachableByFlows(source).p

编写一个查询来查找一个double free示例代码（提示：从分配到自由的数据流）

def source = cpg.call(".*alloc.*")
def sink = cpg.call("free").argument
sink.reachableByFlows(source).p

**VLC漏洞**

查找具有加法操作的malloc调用的返回值到达memcpy（）的第一个参数的memcpy（）调用

def src = cpg.call("malloc").where(_.argument(1).isCallTo(Operators.addition)).l
cpg.call("memcpy").where { call => 
call.argument(1)
 .reachableBy(src)
}.code.l

从这些有趣的数据源和sink中查找数据流

def source = cpg.call("malloc").where(_.argument(1).isCallTo(Operators.addition))
def sink = cpg.call("memcpy").argument
sink.reachableByFlows(source).p

### 四、集成

使用sc脚本和python

https://blog.convisoappsec.com/en/finding-classes-to-exploit-insecure-unchecked-vulnerabilities-in-java-with-joern/
2021-NSEC

**joern脚本**

将可能的缓冲区溢出查询包装在一个函数中并使用它

保存到 /home/$USER/bin/joern/mytools.sc 中：
 def buffer_overflows(cpg : io.shiftleft.codepropertygraph.Cpg) = {
 def src = cpg.call("malloc").where(_.argument(1).isCallTo(Operators.addition)).l
           cpg.call("memcpy").where { call => 
call.argument(1)
 .reachableBy(src)
           }.code.l
  }

运行脚本
joern> import $file.mytools // import your script
joern> mytools.buffer_overflows(cpg) // run the script from within Joern Shell!

或者外部脚本的形式：

保存到 /home/$USER/bin/joern/buffer_overflows.sc 中：
@main def execute(graph: String) = {
  open(graph)
  println("Finding possible buffer overflows")
  def src = cpg.call("malloc").where(_.argument(1).isCallTo(Operators.addition)).l
  cpg.call("memcpy").where { call =>
  call.argument(1)
    .reachableBy(src)
  }.code.l
}

您可以使用importCode（）等其他命令替换open（图）来处理新代码。你也可以生成json，创建报告等等。

在外部运行：
$ joern --script buffer_overflows.sc --params graph=vlc-3.0.12
