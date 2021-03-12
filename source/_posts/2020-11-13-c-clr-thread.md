---
title: 大话线程(CLR视角)
date: 2020-11-10 14:27:26
update: 2020-11-18
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

## Translating!!!
- [x] Managed Thread vs Native Threads
- [x] Data Structures
- [x] Thread Lifetimes
- [x] Suspension
- [x] Entering Cooperative Model
- [x] Suspending the EE
- [x] Hijacking
- [ ] Thread Abort / AppDomain-Unload
- [ ] Synchronization:Managed
- [ ] Synchronization:Native
- [ ] GC Mode
- [ ] Crst
- [ ] Special Threads
- [ ] Finalizer Thread
- [ ] GC Threads
- [ ] Debugger Thread
- [ ] Appdomain-Unload Thread
- [ ] ThreadPool Threads

### 托管线程(Managed Thread) vs 原生线程(Native Threads)

托管代码在“托管线程”上执行,“托管线程”与操作系统提供的原生线程不同,原生线程是在物理机上执行原生代码的线程.托管代码是在CLR虚拟机执 行的虚拟线程.

就像JIT编译器将“虚拟”IL指令映射到原生指令在物理机上执行一样,CLR的线程**infrastructure**将“虚拟”的托管线程映射到操作系统的原生线程上.

在任何时候,一个托管线程也许会,也可能不会分配一个原生线程来执行.例如一个已创建(通过“new System.Threading.Thread”)但是未启动(通过System.Threading.Thread.Start)的托管线程没有被分配任何的原生线程.同样,一个托管线程也可能在执行过程中在多个的原生线程之间移动,尽管实际上CLR目前不支持此操作.

托管代码可使用公共线程接口有意隐藏了关于底层中原生线程的细节,因为

- 托管线程不一定会映射到单个原生线程(也可能根本没有映射到原生线程上)
- 不同到操作系统提供了不同的原生线程的抽象.
- 原则上,托管线程是“虚拟的”.

CLR提供了托管线程的抽象.例如,未公开操作系统的线程本地存储(TLS)机制.而是提供了托管的“静态线程(thread-static)”变量.同样没有暴露原生线程的“线程ID”,而是提供了独立于OS的托管“线程ID”.出于诊断的目的,可以通过System.Diagnostics命名空间下的类型获取原生线程的一些底层细节.

托管线程需要一些原生线程通常不用的功能.首先,托管线程需将GC引用保存在栈上,以便CLR能够在每次发生GC时枚举(或修改)这些引用.为此,CLR需要“挂起(suspend)”所有托管线程(暂停在可以找到所有GC引用的点).第二,当AppDomain卸载时.CLR必须确保没有线程在该AppDomain中执行代码.这要求CLR能够让线程强制从该AppDomain中回退出来.CLR通过向这些线程注入ThreadAbortException来实现.

### 数据结构(Data Structures)

每一个托管线程都有关联的Thread对象,定义在[threads.h](https://github.com/dotnet/coreclr/blob/master/src/vm/threads.h).该对象跟踪虚拟机需要的关于托管线程的一切信息,如当前GC模式与Frame chain的必要信息,以及处于性能原因分配到每个线程中的信息(例如arena-style allocators[1]).

所有的线程对象都存储在ThreadStore(同样定义在[threads.h]),包含了所有已知线程对象的简单列表.要枚举所有的托管线程,必须先获取ThreadStoreLock,然后使用ThreadStore::GetAllThreadList来枚举全部的线程对象.该列表可能包含未分配原生线程的托管线程(它们可能没有启动或原生线程已经退出).

所有被分配了托管线程的原生线程,都可以通过访问线程局部存储(TLS)来获取绑定的托管线程.这允许执行在原生线程的代码获取相应的线程对象,例如GetThread().

**Additionally, many managed threads have a managed Thread object (System.Threading.Thread) which is distinct from the native Thread object. The managed Thread object provides methods for managed code to interact with the thread, and is mostly a wrapper around functionality offered by the native Thread object. The current managed Thread object is reachable (from managed code) via Thread.CurrentThread.**/此外,许多托管线程都有一个托管的线程对象(System.Threading.Thread),它与原生的线程对象不同.托管的线程对象提供了托管代码与线程进行交互的方法.主要是对原生线程对象提供的功能进行了封装.当前的托管线程对象可以通过(来自托管代码)Thread.CurrentThread拿到.

debugger时,可以使用SOS的拓展命令"!Thread"枚举出ThreadStore中的所有线程对象.

### 线程生命周期(Thread Lifetimes)

一个托管线程将在以下场景被创建
    1. 托管代码通过System.Threading.Thread,明确要求CLR进行创建新线程.
    2. CLR直接创建托管线程(参见下面的[特殊线程][https://github.com/dotnet/coreclr/blob/master/Documentation/botr/threading.md#special-threads])
    3. 在原生线程上,由原生代码调用托管代码,而原生线程还没有与托管线程关联(通过"reverse p/invoke"或COM互操作).
    4. 托管进程启动(在进程的主线程上调用Main函数)

在#1,#2情况下,CLR负责创建一个原生线程来支持托管线程.这只会在线程被实际启动后完成.在这种情况下,CLR"持有"这个原生线程;CLR负责这个原生线程的生命周期.在这种情况下,**~the CLR is aware of the existence of the thread by virtue of the fact that the CLR created it in the first place.**

在#3,#4的情况下,原生线程在托管线程被创建前就已经存在,并且由CLR的外部代码"".CLR不负责原生线程的生命周期.CLR在这些线程首次尝试调用托管代码时,才会意识到这些线程的存在.

当原生线程死亡时,CLR会通过DllMain方法接收到通知.这发生在OS的"loader lock"内部,在处理此通知时,几乎无法(安全)完成.因此与其销毁托管线程相关联的数据结构,不如将线程简单地标记成"dead"并向finalizer线程发送运行信号(signals).finalizer线程将遍历ThreadStore中的线程,并通过托管代码销毁已死且不可达的线程.

### 挂起(Suspension)

CLR必须能够找到所有的托管对象的引用以便执行GC.托管代码不断访问GC堆,操作存储在栈与寄存器中的引用.CLR必须确保所有的托管线程都已经停止(所以他们不会修改堆),以便安全可靠的找到所有的托管对象.它将仅在安全点暂停,此时可以检查寄存器与栈空间的实时引用.

针对GC堆的另外一种说法:GC堆以及每个线程的栈和寄存器状态都是"共享状态(shared state)",可以被多个线程访问.与大多数共享状态一样,需要某种锁来保护它.托管代码在访问堆时必须持有此锁,并且只能在安全点释放该锁.

CLR将此"锁"称为线程的"GC模式",对于"Cooperatibe mode(合作模式)"的线程将持有此锁;它必须与GC合作(通过释放此锁)才能进行GC.处于"Preemptive(抢占模式)"的线程不需要持有锁-GC可以抢占的执行因为已知线程不会访问GC堆.

只有当所有的托管线程都处于"抢占模式(不持有锁)"GC才可以进行.将所有的托管线程移动至抢占模式的过程,称为"GC suspension-GC挂起"或"suspending the Execution Engine(EE)-挂起执行引擎".

一个关于"锁"的天真方案,每个托管线程在访问GC堆前后都获取并释放锁,然后GC将简单的尝试获取每个线程的锁;一旦持有了全部线程的锁时,就可以安全的执行GC

然而,上述的方案存在两个缺陷,首先,托管代码会在持有跟释放锁上花费大量时间(或最小检查GC是否正在尝试持有锁.也叫GC轮询).其次,将要求JIT生成大量的"GC info",描述每一行代码经过JIT后栈和寄存器的布局.

我们改进了上述方案,将JIT后的托管代码分成两类:"partially interruptible(部分可中断)"和"(fully interruptible)完全可中断".在部分可中断的代码中,唯一安全点是调用其他方法,且JIT明确生成"GC轮询"点,以便检查是否有GC正在等待.JTI只需要在这些地方生成GC信息.在完全可中断的代码中,每条指令都是安全点,且JIT会为每条指令生成GC信息-但不会生成GC轮询的代码.与之相反,完全可中断的代码可能会通过"hijacking(劫持)"线程的方式来中断(这个过程将放到本文后面讨论).JIT基于启发式方法选择是否生成完全/部分可中断的代码.在代码质量,GC信息的大小与GC的挂起延迟之间找到最佳的权衡点.

综上所述,我们定义了三个阶段:进入抢占模式,离开抢占模式和挂起执行引擎(Execution Engine)

### 进入合作模式(Entering Cooperative Mode)

当需要发生GC时,第一步是调用GCHeap::SuspendEE,挂起EE,具体操作如下:
    1. 设置一个全局flag(g_fTrapReturningThreads)标识GC正在进行中.任何尝试进入合作模式的线程都会阻塞,直到GC完成.
    2. 查找当前正处于合作模式的线程.尝试劫持线程,以迫使它离开合作模式.
    3. 重复上述步骤,直到没有现成处于合作模式.

### 劫持(Hijacking)

在GC的挂起过程中,劫持由Thread::SysSuspendForGC完成.该方法通过强制将当前正处于合作模式下的托管线程,在"安全点"离开合作模式.并通过枚举所有托管线程(通过访问ThreadStore),对当前正处于合作模式下的进行一下操作:
    1. 通过调用Win32 SuspendThread API,挂起底层的原生线程.强制线程从运行状态停止在任意状态(不一定是安全点).
    2. 通过 GetThreadContext获取当前线程上下文.这是OS的概念,上下文包含了线程当前的寄存器状态.以便我们检查它的指令指针寄存器(IP/IAR),从而确定当前正在执行的代码类型.
    3. 再次检查线程是否处于合作模式,因为它在被挂起前就已经离开了合作模式.If so,代码就正处于危险区域:线程也许正在执行任何的原生代码,必须立刻恢复以避免死锁.
    4. 检查线程是否在运行托管代码.可能正处与合作模式中执行原生VM代码(详见下文的Synchronization章节),这种情况也需如上一步,立即恢复线程.
    5. 现在,该线程已经在托管代码中被挂起.根据代码处与完全/部分可中断,执行以下操作之一:

- If fully interruptable, it is safe to perform a GC at any point, since the thread is, by definition, at a safe point. It is reasonable to leave the thread suspended at this point (because it's safe) but various historical OS bugs prevent this from working, because the CONTEXT retrieved earlier may be corrupt). Instead, the thread's instruction pointer is overwritten, redirecting it to a stub that will capture a more complete CONTEXT, leave cooperative mode, wait for the GC to complete, reenter cooperative mode, and restore the thread to its previous state.

- If partially-interruptable, the thread is, by definition, not at a safe point. However, the caller will be at a safe point (method transition). Using that knowledge, the CLR "hijacks" the top-most stack frame's return address (physically overwrite that location on the stack) with a stub similar to the one used for fully-interruptable code. When the method returns, it will no longer return to its actual caller, but rather to the stub (the method may also perform a GC poll, inserted by the JIT, before that point, which will cause it to leave cooperative mode and undo the hijack).