---
title: "RobotAgent漏洞：补丁（Part 2/4）"
excerpt: '漏洞的修复方案'

collection: theory
category: sec
permalink: /theory/sec/vuln-patch
tags: 
  - patch
  - diff

layout: single
read_time: true
author_profile: false
comments: true
share: true
related: true
---

![](../../images/theory/sec/vuln_patch/patch.png)

## 综述

## 补丁模式

### 一、代码漏洞补丁

通过对CVEs的漏洞修复源码比对，可以找出其修补漏洞的共通模式，三种通用的修复模式如下[^1]：
1. 条件终止(ABORT)
2. 条件执行(EXEC)
3. 条件分配(ASSIGN)

**1.条件终止**

特点是增加一个if语句，对程序涉及的变量进行检查，如果不满足条件，则中断控制流(例如，从函数返回)。如下图的CVE-2017-8395漏洞修复：

![](../../images/theory/sec/vuln_patch/CVE-2017-8395.png)

**2.条件执行**

特点是在现有条件语句(if、while和for)中添加一个连接布尔表达式，对程序涉及的变量进行检查。如下图CVE-2017-8392的漏洞修复。

![](../../images/theory/sec/vuln_patch/CVE-2017-8392.png)

**3.条件分配**

特点是增加一个新的if语句，对程序涉及的变量进行检查，满足条件则进行赋值。如下图CVE-2013-0211的漏洞修复。

![](../../images/theory/sec/vuln_patch/CVE-2013-0211.png)

针对RobotAgent安全提出方案

### 修复竞态条件漏洞

修复竞态条件中的TOCTTOU：
修复此漏洞的最简单方法之一是将check和use作为单独的原子操作。在下图(左)调用access的例子里，操作系统在打开文件时已经检查了文件的权限。为了让它与真实ID进行对比，暂时删除有效ID来匹配真实ID就足够了，下图(右)。

TOCTTOU漏洞(与其他类型的软件缺陷一样)通常应该以尽可能简单的方式修复，因为更复杂的修复可能只是移动漏洞，或者引入一个新的漏洞。

[^1]: FixReverter https://www.usenix.org/conference/usenixsecurity22/presentation/zhang-zenong