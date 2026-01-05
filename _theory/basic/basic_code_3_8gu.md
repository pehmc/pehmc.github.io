---
title: "编程：八股文（Part 3/3）"
excerpt: 'Python，C++，Golang'

collection: theory
category: basic
permalink: /theory/basic/code-8gu
tags: 
  - 8gu
  - cpp
  - python
  - golang

layout: single
read_time: true
author_profile: false
comments: true
share: true
related: true
---
![](../../images/theory/8gu.png)

## Python

**Python的优点：**
- 脚本语言
- 动态类型语言
- 丰富的第三方库
- 虚拟环境

### 一、Python包管理

**1.介绍一下模块和包？**
- 模块，是一个py文件，作用是代码复用、项目逻辑划分。
- 包，是一个包含多个模块的目录，使用__init__.py声明。

**2.包管理工具有哪些？**
- pipenv，官方认可，依赖管理文件Pipfile，Pipfile.lock。
- conda，负责依赖管理，依赖管理文件environment.yml。
- uv，速度极快，依赖管理文件pyproject.toml，uv.lock。

**3.为什么要引入虚拟环境？**
- 解决第三方包的依赖冲突，避免不兼容的函数接口。
- 虚拟环境：独立的 Python 解释器副本或链接、独立的 pip 工具、独立的 site-packages 目录。

### 二、Python中xx和xx的区别

**1.list、dict、tuple、set的区别？**
- 类型，可变性，有序性，元素要求，索引方式，场景。
- list，可变，有序，任意，下标访问，存储有序数据。
- dict，可变，有序，键必须可哈希，键访问，存储关联数据。
- tuple，不可变，有序，任意，下标访问，函数返回值。
- set，元素不可变，无序，元素可哈希，不可索引，去重、求交集。

**2.list存储tuple、tuple存储list的区别？**
- list存储tuple，可以增删tuple，tuple的元素不可变。
- tuple存储list，不可以增删list，list的元素可变。

**3.赋值、浅拷贝、深拷贝的区别？**
- 赋值，不发生拷贝，是创建对象的引用。
- 浅拷贝，仅拷贝顶层对象，是创建新对象，但将其子对象的引用插入新对象。
- 深拷贝，拷贝顶层对象及其所有嵌套层次的对象。

**4.生成器、迭代器、装饰器的区别？**
- 生成器，是函数，通过yield实现特殊的迭代器。
- 迭代器，是对象，实现了__iter__() 和__next__()方法。
- 装饰器，是函数，接受原函数作为输入，返回一个函数作为输出，比如添加函数运行计时。

**5.参数传递时，传值、传址的区别？**
- 不可变对象是传值，比如int，str，tuple。
- 可变对象是传址，比如list，dict，set，自定义对象。

**6.==和is的区别？**
- == 比较变量的值，底层是，调用可重写的__eq__()方法。
- is 比较两个变量是否指向同一个对象，无法重写。

**7.\*args 和 \*\*kwargs的区别？**
- *args，收集任意数量的位置参数，并将其放入一个tuple中。
- **kwargs，收集任意数量的关键字参数，并将其放入一个dict中。
- *args 必须位于 **kwargs 之前。

**8.单下划线、双下划线、前后双下划线的区别？**
- 单下划线_name，约定仅供内部使用。
- 双下划线__name，触发名称修饰，目的是避免子类中的命名冲突。
- 前后双下划线__name__，特殊方法的命名方式。

**9.__new__和__init__的区别？**
- __init__用来初始化，在对象创建后设置其初始状态。
- __new__用来实现单例模式、或者继承内置不可变类型。
- __new__成功返回一个正确类的实例后，__init__才有机会被执行。

### 三、Python异常处理

**1.如何处理异常，如何捕获多个异常？**
- try，执行可能引发异常的代码
- except (ValueError, TypeError)，使用tuple捕获多个异常
- else，在无异常时执行
- finally，总是执行。

### 四、Python并发

**1.全局解释锁GIL是什么？**
- GIL是一个互斥锁，确保只有一个线程执行字节码，因此在多核CPU上无法实现多线程并行。
- 引入GIL是保证垃圾回收机制（引用计数）的线程安全。

**2.Python多线程和多进程的优劣？**
- 多线程，对于I/O密集型任务，线程阻塞时主动释放GIL，影响较小，对于CPU密集型任务，影响极大。
- 多进程，每个进程有自己的 GIL，可以利用多核CPU实现多进程并行。

**3.Python多协程的实现？**
- 调用asyncio包，使用其async/await机制实现多协程。

### 五、Python垃圾回收

**1.Python的垃圾回收机制？**
- 引用计数机制，引用计数变为 0 时，对象立即被回收，内存被释放。
- 分代回收 + 标记清除机制，设计用来处理循环引用的问题。

**2.产生内存泄露的原因？**
- 全局变量。
- 未关闭的资源，即没有使用with as管理。
- 第三方库问题。

### 六、Python高级特性

**1.对于匿名函数、闭包的理解？**
- 匿名函数使用场景：没有代码复用的临时函数、作为回调函数。
- 匿名函数使用lambda关键字创建，只包含2个部分：参数、表达式。
```python
# 函数赋值给变量
add = lambda x, y: x + y
```
- 闭包是一个函数，在内部定义嵌套函数，将嵌套函数返回。

## C++

C++的优点：
- 高性能矩阵计算
- 硬件级别优化

## Golang

**Golang的优点：**
- 有GC
- 编译、测试方便
- 原生并发
- 包管理

### 一、Golang包管理

**1.有哪些包管理方式？**
- GOPATH，已被废弃，没有版本控制、存在依赖冲突。
- Go Modules，通过核心文件`go.mod`实现版本管理、依赖隔离。

**2.go mod的管理方式？**
- `go mod init`初始化目录为模块。
- `go mod tidy`自动更新和整理依赖关系，解决依赖冲突。
- `go get`自动将添加该依赖到go.mod，通过在包名附加@version实现特定版本依赖。
- `go.sum`文件记录了依赖模块的加密哈希值，校验依赖的一致性。

**2.如何解决包的循环依赖？**
- 合并包，使用场景：两个包职责高度相关。
- 提取包，使用场景：有被多个包共享的公共类型或逻辑
- 依赖注入，使用场景：包之间有行为依赖，特别是双向调用。

**3.init函数的执行顺序？**
- 包内的顺序：首先按照`文件名`排序，同一文件中按照`代码声明顺序`执行。
- 跨包依赖的顺序：直接、间接依赖包，按导入顺序执行。 

### 二、Golang中xx和xx的区别

**1.值类型和引用类型的区别？**
- 场景。值类型有int、float32、bool、array和struct；引用类型有string、slice、map、chan、interface和func，通过各自类型的结构体实现，结构体使用指针来引用底层数据。
- 赋值。值类型赋值时，进行值拷贝；引用类型赋值时，仅拷贝类型的结构体，共享底层数据。
- 函数传参。值类型传参时，产生完整副本，不影响实参；引用类型传参时，仅复制类型的结构体，共享底层数据，影响实参。

**2.string、[]byte的区别？**
- 不可修改性。string是不可修改的，任何修改操作会创建新的string；[]byte本质是slice，是字节切片，通过索引直接修改。
- 内存分配。虽然string和[]byte都使用指针来访问底层数据，string的底层数据存储在只读区、堆上；[]byte的底层数据存储在栈、堆上。
- 类型转换。string和[]byte的互相转换都会发生底层数据的复制。

**3.make和new的区别？**
- 返回的类型。make用于创建并初始化slice、map和chan等引用类型，返回非nil的引用类型；new用于分配array、struct等值类型的内存，并返回该值类型的指针。
- 分配的地点。make和new都可以分配在栈、堆上。make时，map和chan永远分配到堆上；new时，指针逃逸、大对象和闭包捕获分配到堆上。

**4.数组和切片的区别？**
- 即值类型和引用类型的区别。

**5.传递参数时，传值类型和传引用类型的区别？**
- 值类型传递时，函数获得一个完整、独立的副本。
- 引用类型传递时，函数获得该引用结构体中指针的副本。

### 三、Golang中xx的底层实现

**1.string的底层实现？**
```go
type stringStruct struct {
    str unsafe.Pointer      //字符串首地址，指向底层字节数组的指针
    len int                 //字符串长度
}
```
- string本身是结构体，存储在栈、堆上（逃逸分析-gcflags="-m"决定），它其中的指针指向底层数据（字节数组），底层数据存储在只读区、堆上。
- string底层数据可能在只读区，因此string不可修改。

**2.slice的底层实现？**
```go
type slice struct {
   array unsafe.Pointer // 指向底层数组的指针
   len   int            // 切片的当前长度
   cap   int            // 切片的容量
}
```
- slice也是使用指针来访问底层数据，增加了cap管理容量，cap是实际的内存占用。
- append后len大于cap，slice进行扩容，双倍扩容，此时1.超过了堆区大小，通过mmap分配；2.栈上的小切片，可能扩容后分配在堆上。

**3.map的底层实现？**
```go
type hmap struct {
    count     int    // map 中元素的个数，即 len(m)
    flags     uint8
    B         uint8  // buckets 的对数，buckets 数量 = 2^B
    noverflow uint16 // 溢出桶的大致数量
    hash0     uint32 // 哈希种子
 
    buckets    unsafe.Pointer // 指向 bucket 数组的指针，大小为 2^B
    oldbuckets unsafe.Pointer // 在扩容时，指向旧的 bucket 数组
    nevacuate  uintptr        // 扩容进度计数器
    // ...
}
```
- map基于hmap实现，通过一个hash函数计算key的hash值，根据低b位映射到存储桶（bucket），减少buckets数量，提高索引效率。每个存储桶有8个槽位，用于处理hash冲突。
- map存储的kv是无序的，因为hmap的存储桶是无序的。
- map是并发不安全的，需要使用sync.Map替代。
- 装载因子过大或溢出桶过多时，map进行扩容，双倍扩容或等量整理。
- map的key必须是可比较的类型，因此slice不可以作为key，而eface可以。

**4.空接口、方法接口的底层实现？**
```go
// 方法接口 (iface)
type iface struct {
    tab  *itab          // 类型信息+方法表
    data unsafe.Pointer  // 实际值指针
}

// 空接口 (eface)
type eface struct {
    _type *_type         // 动态类型信息
    data  unsafe.Pointer // 实际值指针
}
```
- 两个空接口变量可以比较，相等的情况有两种：1）都是nil；2）类型相同，且值也相同。

**5.channel的底层实现？**
```go
type hchan struct {
    qcount   uint           // 当前队列中剩余的元素个数
    dataqsiz uint           // 环形队列的大小，即可以存放的元素个数（make(chan T, N) 中的 N）
    buf      unsafe.Pointer // 指向环形队列的指针（有缓冲 channel 才非 nil）
    elemsize uint16         // 每个元素的大小
    closed   uint32         // channel 是否已关闭的标志
    elemtype *_type         // 元素的类型（用于在运行时进行类型检查）

    sendx    uint           // 发送索引（send index），指向环形队列中下一个发送的位置
    recvx    uint           // 接收索引（receive index），指向环形队列中下一个接收的位置

    recvq    waitq          // 等待接收的 goroutine 队列（list of recv waiters）
    sendq    waitq          // 等待发送的 goroutine 队列（list of send waiters）

    lock     mutex          // 互斥锁，保护 hchan 中的所有字段，以及此 channel 上被阻塞的 goroutines
}
```
- 同步机制，在发送数据`ch <- value`和接收数据`<-ch`时，都要先获取chn的互斥锁。
- 阻塞机制，goroutine 被放入等待队列（sendq 或 recvq）并被调度器挂起（gopark）。
- 对已关闭 Channel 的操作：1）二次关闭，依次唤醒 recvq 队列中的所有等待者（它们会收到该类型的零值），再唤醒 sendq 队列中的所有等待者（它们会引发 panic）；2）向已关闭的 channel 发送数据会引发 panic；3）从已关闭的 channel 接收数据，如果缓冲区有数据，则正常取数据；如果缓冲区没数据，则直接返回类型的零值。
- select 语句中的非阻塞操作（default 分支）在获取锁之后，会先检查操作是否能立即完成（例如，缓冲区是否有空间或数据）。如果不能，则直接执行 default 分支并释放锁，而不会进入等待队列。。

### 四、Golang异常处理

**1.介绍一下defer？**
- 用于延迟函数的执行，将推迟到函数执行之后，通常⽤于资源释放、锁的释放、⽇志的记录等。比如，在异常或正常时关闭redis。
```go
type _defer struct {
    siz     int32    //参数和结果的内存大小
    started bool
    heap    bool   //是否是堆上分配
    openDefer bool  // 是否经过开放编码的优化
    sp        uintptr   //栈指针
    pc        uintptr   // 调用方的程序计数器
    fn        *funcval  // 传入的函数
    _panic    *_panic   
    link      *_defer  //defer链表
    fd   unsafe.Pointer  
    varp uintptr        
    framepc uintptr
}
```
- defer将函数参数、传入的函数存储在结构体中，按照后进先出（LIFO）的顺序执行，以链表存储。

**2.介绍一下panic和recover？**
- panic用来引发运行时错误，首先执行每个函数中的 defer 延迟函数，然后终⽌程序
- recover用来恢复程序执行，只能在defer中使用，首先返回panic的值，然后从panic的地方继续执行。
```go
func example() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    panic("Something went wrong!")
}
```

**3.父 Goroutine可以捕获子Goroutine 的 panic吗？**
- 父 Goroutine可以通过chan捕获子 Goroutine 的 panic。

### 五、Golang并发

**1.介绍一下GMP模型？**
- go协程的调度，采用了GMP模型用来实现高并发的服务，指的是三个关键组件：
1. G，go轻量级线程。
2. M，负责将 goroutine 映射到真正的操作系统线程上。
3. P，调度器，负责调度goroutine，维护goroutine队列。
- M和P一对一绑定，M从P获取G。


**2.为什么sync.map并发安全？**
- 将数据分成两部分：“只读数据” 和 “脏数据”。
- 读操作 优先无锁地访问只读数据，极大提升了读性能。
- 写操作 只在必要时才加锁操作脏数据，并且锁的粒度很小。
```go
type Map struct {
    mu sync.Mutex       // 锁，用于保护 dirty map

    // read 是一个原子指针，指向 readOnly 结构。
    // 读操作首先会原子性地读取这个指针，因此读操作通常无需加锁。
    read atomic.Pointer[readOnly]

    // dirty 是一个原生的 Go map，它包含了一部分“当前”的数据。
    // 访问 dirty 需要持有 mu 锁。
    dirty map[any]*entry

    // misses 是一个计数器，记录从 read 中找不到key，不得不加锁去 dirty 中查找的次数。
    misses int
}
```

**3.介绍一下goroutine？**
- 理念，共享数据通过通信⽽不是通过共享来实现
- 通道，chan实现同步，数据的传递，比如chan来处理ctl信号。
- 多路复用，select处理阻塞的问题，default解决所有阻塞。

### 七、Golang垃圾回收

**1.GC的原理是什么？**
- 三⾊标记法+混合写屏障
- 三色标记法，通过并发标记，解决了程序长时间停顿的问题。白色是需要回收的。
- 混合写屏障，在指针修改操作时进行标记，解决了并发标记阶段，被引用对象被错误回收的问题。

**2.什么是内存逃逸？**
- 逃逸逃逸分析标志`-gcflags="-m"`，在编译时即执行分析。
- 如果局部变量逃逸出作用域，决定从栈分配到堆上。
- 因此，函数返回局部变量的指针是完全安全的。

**3.pprof如何调优？**
- 检测内存泄露
1. 获取堆快照`go tool pprof http://localhost:6060/debug/pprof/heap`，两个快照作对比。
2. 常见原因：全局变量无限增长、被 Goroutine 阻塞持有的 channel 或 slice 等引用、未正确关闭的资源。
- 检测goroutine泄漏
1. 获取 Goroutine快照`go tool pprof -http=:8080 http://localhost:6060/debug/pprof/goroutine`，查看火焰图，寻找热点。
2. 常见原因：chan 操作不当/等待网络IO导致阻塞、死锁、无法从 ctx.Done() 接收到退出信号。

### 八、Golang高级特性

**1.对于匿名函数、闭包的理解？**
- 匿名函数，常与go关键字合用，立即并发执行；或者作为回调函数。
- 闭包是一个函数，在内部定义嵌套函数，将嵌套函数返回。

