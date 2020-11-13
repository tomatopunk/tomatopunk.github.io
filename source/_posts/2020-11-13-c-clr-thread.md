---
title: CLR线程概述
date: 2020-11-10 14:27:26
update: 2020-11-13 14:27:26
categories: C#
tags:
    - C#
    - 性能诊断
    - 线程
    - 概要设计
    - CLR Thread
    - Thread
---

### 原文信息

**原文来自于CLR的概要设计系列中,线程篇**
**本文将会介绍一些关于`托管线程` `原生线程`,`线程生命周期`,`线程数据结构`等设计**
[原文:dotnet/coreclr](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/threading.md)
<!-- more -->

### 托管线程(Managed Thread) vs 原生线程(Native Threads)

托管代码执行在“托管线程上”,“托管线程”与操作系统提供的原生线程不同,原生线程是在物理机上执行原生代码的线程.托管代码是在CLR虚拟机执行的虚拟线程.

就像JIT编译器将“虚拟”IL指令映射到原生指令在物理机上执行一样,CLR的线程**infrastructure**将“虚拟”的托管线程映射到操作系统的原生线程上.

在任何时候,一个托管线程可能会分配给一个原生线程来执行,也可能不会.例如一个已经创建(通过“new System.Threading.Thread”)但是尚未启动(通过System.Threading.Thread.Start)的托管线程没有分配任何的原生线程.同样,一个托管线程也可能在执行过程中在多个的原生线程之间移动,尽管实际上CLR目前不支持此操作.

托管代码可使用公共线程接口有意隐藏了底层原生线程的细节,因为

- 托管线程不一定会映射到单个原生线程(也可能根本没有映射到原生线程上)
- 不同到操作系统提供了不同的原生线程的抽象.
- 原则上,托管线程是“虚拟的”.

CLR提供了同样的托管线程的抽象,由CLR本身自己实现.例如,它没有公开操作系统的线程本地存储(TLS)机制.而是提供了托管的“静态线程(thread-static)”变量.同样,它没有暴露原生线程的“线程ID”,而是提供了独立于OS的“托管线程ID”.然而,出于诊断的目的,可以通过System.Diagnostics命名空间下的类型获取原生线程的一些底层细节.

托管线程需要一些原生线程不需要的拓展功能.首先,托管线程将GC的引用保存在栈上,所以CLR必须能够每次发生GC时枚举(或是修改)这些引用.要做到这个,CLR必须“挂起(suspend)”每一个托管线程(在可以找到其所有的GC引用的地方停止它).其次,当AppDomain卸载时.CLR必须确保没有线程在该AppDomain中执行代码.这就需要能够让线程强制从该AppDomain中回退出来.CLR通过向这些线程注入ThreadAbortException来实现它.

### 数据结构(Data Structures)

每一个托管线程都有关联的Thread对象,在threads.h中定义.该对象跟踪虚拟机需要知道的关于托管线程的一切信息.这包括了一些必要的东西,当前的GC模式和栈帧(Frame chain(帧链,调用链?)),以及许多出于性能原因而按线程分配的东西( **~比如速度最快的竞技场分配风格/ fast arena-style allocators**).

所有的线程对象都存储在ThreadStore(在threads.h中定义),它是所有已知线程对象的简单列表.要枚举所有的托管线程,必须先获取ThreadStoreLock,然后使用ThreadStore::GetAllThreadList来枚举全部的线程对象.这个列表可能没有包括未分配原生线程的托管线程.(例如它们可能还没有启动或者原生线程已经退出).

每个托管线程都会分配一个原生线程,这些托管线程都可以通过原生线程来访问到原生的"thread-local storage(TLS)"slot.这将允许该原生线程上执行的代码通过GetThread()来获取对应的Thread对象.

此外,许多托管线程都有一个托管的线程对象(System.Threading.Thread),它与原生的线程对象不同.托管的线程对象提供了托管代码与线程进行交互的方法.主要是对原生线程对象提供的功能进行了封装.当前的托管线程对象可以通过(来自托管代码)Thread.CurrentThread拿到.

在debugger时,可以使用SOS的拓展命令"!Thread"来枚举出ThreadStore中的所有线程对象.

### 线程的生命周期(Thread Lifetimes)

一个托管线程将在以下场景被创建
    1. 托管代码通过System.Threading.Thread,明确要求CLR进行创建新线程.
    2. CLR直接创建托管线程(参见下面的"特殊线程")
    3. 机器码在原生线程上调用托管代码,而原生线程还没有与托管线程关联(通过"reverse p/invoke" or COM互操作).
    4. 一个托管进程的启动(在进程的主线程上调用Main函数)

在#1,#2情况下,CLR会负责创建一个原生线程来支持托管线程.在线程被实际启动之前不会完成.在这种情况下,CLR"拥有"这个原生线程;
CLR负责这个原生线程的生命周期.在这种情况下,**~the CLR is aware of the existence of the thread by virtue of the fact that the CLR created it in the first place.**

在#3,#4的情况下,原生线程在托管线程被创建前就已经存在,并且由CLR的外部代码"拥有".CLR不负责原生线程的生命周期.CLR在这些线程第一次尝试调用托管代码时,就会意识到这些线程的存在.

当原生线程死亡时,CLR会通过DllMain方法接收到通知.这发生在OS的"loader lock"内部,在处理此通知时,几乎无法(安全)完成.因此与其破坏掉托管线程相关联的数据结构,不如将线程简单地标记成"dead"并向finalizer线程发送运行信号(signals).finalizer线程将遍历ThreadStore中的线程,并通过托管代码销毁已死并且不可访问的线程.

### 挂起(Suspension)

CLR必须能够找到所有的托管对象的引用,以便执行GC.托管代码不断访问GC堆,并且操作存储在栈与寄存器中的引用.CLR必须确保所有的托管线程都已经停止(所以他们不会修改堆),以便安全可靠的找到所有的托管对象.当可以检查到寄存器和stack locations的实时引用时,它将仅在安全点中断.

另外一种说法是GC堆,以及每个线程的栈空间和寄存器状态都是"共享状态(shared state)",可以被多个线程访问.与大多数共享状态一样,需要某种锁来保护它.托管代码在访问堆时必须持有此锁,并且只能在安全点释放该锁.

CLR将此"锁"称为线程的"GC模式",对于"Cooperatibe mode(合作模式)"的线程将持有此锁;它必须与GC合作(通过释放此锁)才能进行GC.处于"Preemptive(抢占模式)"的线程不需要持有锁-GC可以抢占的执行因为已知线程不会访问GC堆.

只有当所有的托管线程都处于"抢占模式(不持有锁)"GC才可以进行.将所有的托管线程移动至抢占模式的过程,称为"GC suspension-GC挂起"或"suspending the Execution Engine(EE)-挂起执行引擎".

A naïve implementation of this "lock" would be for each managed thread to actually acquire and release a real lock around each access to the GC heap. Then the GC would simply attempt to acquire the lock on each thread; once it had acquired all threads' locks, it would be safe to perform the GC.

However, this naïve approach is unsatisfactory for two reasons. First, it would require managed code to spend a lot of time acquiring and releasing the lock (or at least checking whether the GC was attempting to acquire the lock – known as "GC polling.") Second, it would require the JIT to emit "GC info" describing the layout of the stack and registers for every point in JIT'd code; this information would consume large amounts of memory.

We refined this naïve approach by separating JIT'd managed code into "partially interruptible" and "fully interruptible" code. In partially interruptible code, the only safe points are calls to other methods, and explicit "GC poll" locations where the JIT emits code to check whether a GC is pending. GC info need only be emitted for these locations. In fully interruptible code, every instruction is a safe point, and the JIT emits GC info for every instruction – but it does not emit GC polls. Instead, fully interruptible code may be "interrupted" by hijacking the thread (a process which is discussed later in this document). The JIT chooses whether to emit fully- or partially-interruptible code based on heuristics to find the best tradeoff between code quality, size of the GC info, and GC suspension latency.

Given the above, there are three fundamental operations to define: entering cooperative mode, leaving cooperative mode, and suspending the EE.