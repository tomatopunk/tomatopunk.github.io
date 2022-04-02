---
title: 诊断性能问题的工作流程(2)
date: 2020-05-07 14:27:26
update: 2020-11-13 14:27:26
categories: [Index]
tags:
    - C#
    - 性能诊断
    - Maoni
    - PerfView
---
原文为Maoni发布在Microfost Blog中的, 诊断性能问题的工作流程系列. 
目前共更新三章. 在本章中, Maoni继续介绍了如何进行诊断性能问题, 以及一个略微棘手的问题. 
<!-- more -->

## 原文信息

[@Maoni Stephens-Twitter](https://twitter.com/maoni0)
[@Maoni Stephens-Github](https://github.com/Maoni0)

![authorize](/posts/images/1587561145552-0d8a560c-3b7d-443a-badc-a98ddbb6e7bf.png)

如果这篇文章可以帮到您，那么这将是我最大的荣幸，希望您点进原文，在文章下方留下善意的回复，您的支持将是这些可敬的社区磐石保持创作激情中最大的一部分:)

[原文](https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-2)

**中文版本将不会以任何形式收费，版权属与原作者**

---

## 正文

在这一章中, 我将讨论一下您应该将精力花费在哪里, 然后继续我的分析. 原本我打算深入研究GCStats视图, 但我刚刚调试了一个长时间挂起的问题. 我想分享给大家, 当您在分析问题时, 其中的一些通用的思路. 当然, 您也可以直接[跳到分析部分](https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-2/#continuing-the-analysis). 

### 请明智的花费精力

PerfView不仅仅是用于收集痕迹的, 更重要的是, 它是用于分析痕迹的. 我遇到过很多人, 他们仅仅使用Perfview进行收集痕迹. 我真的强烈鼓励您将其作为一个分析工具来使用. 

PerfView中内置了大量的帮助, 显然, 我写这个系列的博客主要原因也是为了帮助您, 只是更侧重内存方面. 我这样做的最终目的-是帮助您
获得一些搞清楚性能问题的思路跟方法, 而不是详细的列出您可能碰到的所有问题, 这完全不现实. 

我的理念是您应该明智的花费自己的时间, 我都非常忙, 有做不完的任务. 我也明白, 我中的许多人的性格都很独立, 喜欢自己想办法. 

所以, 在请别人帮忙之前, 您会自己完成多少工作呢?这是我遵循的几个规则 -

- 慷慨的花时间去学习以后会经常用到的知识与技能, 如果我正在研究一个以后不大可能再研究的领域中的问题, 我倾向于尽早的寻求帮助, 因为我知道在这里获得的知识可能只会用到一次. 但如果我知道我需要再次解决这个领域的问题, 我会花费尽可能多的时间来了解它. 

- 如果我有一个紧急的问题, 而我认识的人多半已经知道答案, 我会尽早的向他们寻求帮助. 而如果是我认为这是我需要知道的事情, 我会先把问题处理好, 然后花费时间去了解更多的细节(其他团队可能正在等待解决方案). 当我真的向对方请求帮助时, 我会提供问题的详细描述与调试的信息给对方, 以节省对方询问的时间. 

### 继续分析

在[Part0](https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-0/)我提到了我会从两个跟踪来开始调查. 第二个跟踪是获取CPU采样和一些其他的通用事件, 比如磁盘/网络IO :

``` shell
PerfView /nogui /accepteula /KernelEvents=default+Memory+VirtualAlloc /ClrEvents:GC+Stack /MaxCollectSec:600 /BufferSize:3000 /CircularMB:3000 collect
```

第一个跟踪, 也就是GCCollectOnly跟踪, 是为了准确的了解GC的性能--您想在最小的干扰下进行收集, 而命令行的参数GCCollectOnly正是做这个的. 

第二个跟踪可以让您了解机器上的运行情况. GC生活在一个进程中的线程中, 其他进程可能会影响GC的运行. 请注意, 目前在dotnet-trace中还没有等效的功能 - 您需要在Linux上使用[perfcollect脚本](https://github.com/dotnet/coreclr/blob/master/Documentation/project-docs/linux-performance-tracing. md#preparing-your-machine)收集跟踪, 例如perf+Lttng, 不幸的是它不能提供完全等同的功能(Lttng没有栈)但是在其他方面, 它确实能提供机器范围的视图, 而不是像dotnet-trace只提供一个进程的事件. 

请注意, 我也指定了 **`/BufferSize:3000 /CircularMB:3000`** 参数, 我现在收集的事件比较多, 默认值可能不够用, 对于GCCollectOnly, 我知道它收集的事件不多, 所以默认值就足够了. 一般来说, 我发现3000MB对于这个跟踪的两个参数都是足够的. 如果这些大小有问题, PerfView会给出非常有参考价值的信息, 所以请注意它弹出的对话框! 这是由HandleLostEvents方法触发的:

``` cs
private void HandleLostEvents(Window parentWindow,bool truncated,int numberOfLostEvents,int eventCountAtTrucation,StatusBar worker)
{
    string warning;
    if (!truncated)
    {
        // TODO see if we can get the buffer size out of the ETL file to give a good number in the message.  
        warning = "WARNING: There were " + numberOfLostEvents + " lost events in the trace. \r\n" +
            "Some analysis might be invalid. \r\n" +
            "Use /InMemoryCircularBuffer or /BufferSize:1024 to avoid this in future traces. ";
    }
    else
    {
        warning = "WARNING: The ETLX file was truncated at " + eventCountAtTrucation + " events. \r\n" +
            "This is to keep the ETLX file size under 4GB,  however all rundown events are processed. \r\n" +
            "Use /SkipMSec:XXX after clearing the cache (File->Clear Temp Files) to see the later parts of the file. \r\n" +
            "See log for more details. ";
    }
}
```
**`If`** 的情况是告诉您有事件丢失了, 您同时记录了太多的事件, 而缓冲区不够大. 所以它告诉您, 应该通过 **`/BuffSize`** 指定一个更大的值. 我一般觉得 **`/InMemoryCircularBuffer`** 参数不可靠, 所以我一般不用它. 

**`Else`** 的情况是告诉您, 您使用的 **`/CircularMB`** 参数太大了, 导致PerfView生成的. etlx文件太大, 不能一次解析完, 所以当您使用PerfView查看跟踪时, 它将只显示可以容纳在4GB. etlx中第一部分的信息. 这并不意味着您需要减少这个参数的大小, 这只是意味着您需要采取额外的步骤才能看到所有信息. 要看到后面的部分, 您需要跳过第一部分, 这正是对话框中告诉您需要做的. 通常您会看到这个对话框与第二个跟踪. 

我的做法是, 查看GCStats找出哪个时间段是我感兴趣的, 然后跳过之前的部分. 请注意, 如果您看我在[这篇博客](https://devblogs.microsoft.com/dotnet/gc-perf-infrastructure-part-1/)提到的GC性能基础结构的跟踪, 您就不会有这个问题, 因为基础结构会查看. etl文件, 不会经过. etlx的步骤. 这就解释了为什么当您使用GC Perf infra时, 您可以看到比您在PerfView中更多的GC

指定 **`/ClrEvents:GC+Stack`** 参数是很重要的, 运行时的默认会收集大量的关键词: 

- Default = GC
- Type
- GCHeapSurvivalAndMovement
- Binder
- Loader
- JIt
- NGen
- SupressNGen
- StopEnumeration
- Security
- AppDomainResourceManagement
- Exception
- Threading
- Contention
- Stack
- JittedMethodILToNativeMap
- ThreadTransfer
- GCHeapAndTypeNames
- Codesymbols
- Compilation

其中一些会人为的增大很多GC的暂停时间. 比如 **`GCHeapSurvivalAndMovement`**, 它实际上向BGC添加了另一个STW暂停, 可能会使BGC的实际STW暂停时间增加十倍以上. 

当我知道自己要专注于GC的性能时, 我会选择不收集rundown事件, 即在命令行中添加 **`/NoV2Rundown /NoNGENRundown /NoRundown`** 这意味着我不会得到一些托管的调用帧(ie,  moduleA!? instead of something like **`moduleX!methodFoo(argType)`** ). 但如果您作为一个使用GC的客户, rundown事件是非常有用的, 您可以得到来自于您代码的托管调用帧, 来验证是否可以通过修改代码来帮助改善性能. 

性能问题的类别之一是偶尔的长时间GC(您可以很容易的在GCCollectOnly的跟踪中发现他们), 您可以使用这个命令行, 让PerfView在观察到长GC时立刻停止跟踪:

``` shell
PerfView.exe /nogui /accepteula /StopOnGCOverMSec:100 /Process:MyProcess /DelayAfterTriggerSec:0 /CollectMultiple:3 /KernelEvents=default+Memory+VirtualAlloc /ClrEvents:GC+Stack /BufferSize:3000 /CircularMB:3000 collect
```

用您的进程名字替换掉MyProcess(如果您的进程名字是a. exe, 这里的参数不含. exe, 应该是/Process:A)

用一个恰当的数字替换掉100(如果您想捕获一个500ms的GC, 就用500替换). 

我在[这篇博客](https://devblogs.microsoft.com/dotnet/you-should-never-see-this-callstack-in-production/)中解释了很多这样的参数, 所以在这里就不在赘述了. 

这个停止触发器(在本例中是 **`/StopOnGCOverMSec`**)是我在PerfView中最喜欢的功能之一. dotnet-trace暂时还没有这个功能(没有不提供的理由, 这是需要处理的工作之一). 我通常会从这里开始, 尤其是当长GC已经相当可观时. 他的开销确实比/GCCollectOnly的大得多, 但不会太大(通常是个位数的百分比). 所以我预测产品仍然可以正常的运行, 并且一直重现同样的问题. 我见过人们追逐由`Heisenberg effect(海森堡效应)`引发的不同性能问题. 

还有其他的停止触发器. 要获得有关于它们的帮助, 点击Help/Command Line Helper, 然后在帮助页面上搜索StopOn. 你会看到一堆与各种触发条件有关的停止跟踪开关. 与GC相关的有:

- [-StopOnPerfCounter:STRING,…]
- [-StopOnEtwEvent:STRING,…]
- [-StopOnGCOverMsec:0]
- [-StopOnGCSuspendOverMSec:0]
- [-StopOnBGCFinalPauseOverMsec:0]

前两个是通用的, 所以可以使用在任何的性能计数器/ETW事件上. 我自己从没有手动的使用过-StopOnEtwEvent触发器. 我觉得这个参数很有潜力, 只是我还没有时间去实验它. 

最后三个不需要我解释.  **`StopOnGCOverMSec`** 是最常用的一个. 请注意, **`StopOnGCOverMSec`** 指的是GC/Start与GC/Stop的间隔时间, 如果您指定了/StopOnGCMSec:500, 意味着一旦检测到GC/Start与GC/Stop的间隔超过了500ms, 跟踪就会停止. 如果您正在观察长时间的挂起, 您需要使用 **`StopOnGCSuspendOverMSec`** 触发器, 它实际上在内部与StopOnEtwEvent一起实现, 比较 **`SuspendEEStart`** 和 **`SuspendEEStop`** 时间的间隔触发 - 

``` text
etwStopEvents. Add("E13C0D23-CCBC-4E12-931B-D9CC2EEE27E4/GC/SuspendEEStart;StopEvent=GC/SuspendEEStop;StartStopID=ThreadID;Keywords=0x1;TriggerMSec=" + StopOnGCSuspendOverMSec);
```

#### 长时间挂起

现在来说说我刚调试的客户问题-从来自GCCollectOnly的跟踪中, 我看到一些GC花费了很长的时间在挂起上(3/4秒!). 我还让他们收集了第二个跟踪, 并查看发生长时间挂起期间的CPU采样. 请注意, 出现这种情况时, 只有挂起的时间很长, GC部分是正常的. 还有一种特殊情况是, 挂起与GC的时间都是随机长的, 或其中一个很长. 通常这种情况是有什么东西阻碍了GC线程的运行. 最常见的原因是机器上有一个优先级很高的线程(通常是意外的)在运行时会出现这种情况. 

在这种情况下, 我已经验证了这个问题总是在挂起时才会表现出来. 而从第二个跟踪中, 我可以看到一些IO正在进行, 我知道这可能就是阻止挂起的原因. 然而, 这只是基于我所掌握的知识, 如果我不知道, 我可以做些什么来验证这个理论?另外, 我想展示给我的客户, 到底是如何阻止挂起的. 这个时候就需要一个更加重量级的跟踪, 即 **`ThreadTime`** 跟踪. 如果您有一个线程没有完成它的工作, 要么这个工作就是需要很长时间, 要么就是有什么东西导致它长时间的阻塞. 如果它被阻塞了, 在某些时候它会被唤醒, 我想知道是谁唤醒了它. 收集Context Switch(上下文切换)与ReadyThread(准备线程)事件正是做这个的--看看一个线程为什么会被切换, 以及谁来启用并再次执行它. 在PerfView中叫做 **`ThreadTime`** 跟踪. 它仅收集默认的kernel事件加上这两个事件. 所以您可以把 **`/KernelEvents=default+Memory+VirtualAlloc`** 替换成 **`/KernelEvents=ThreadTime+Memory+VirtualAlloc`** . 这些Context Switch和Ready Thread事件会非常多, 所以有时候原来的问题长时间都不能复现, 在这种情况下, 问题会复现的. 

请查看PerfView的帮助来获得更多使用 **ThreadTime**跟踪的说明. 

当您有Ready Thread事件时, PerfView中的"Advanced Group(高级组)"下方会额外多出一个视图, 叫做"ThreadTime(with ReadyThread) stacks - 线程时间(含准备线程))栈".  我通常要做的就是打开这个视图, 搜索阻塞的时间段内被阻塞的调用, 并查看它的"READIED_BY"调用栈, 导致哪些线程被唤醒. 所以我做的是384, 026. 930 到 387, 822. 395的这个时间段, 也就是GCStats标识的挂起事件, 复制到这个视图中的Start和End框中, 然后在Find框中搜索suspend, 并点击Callees. 这就是我所看到的- 

![WorkFlow-suspension-readythread-view](/posts/images/WorkFlow-suspension-readythread-view. jpg)

我希望看到一个READIED_BY的栈, 它唤醒并调用SuspendEE线程, 确实有一个, 但是没有用, 因为它展示的调用栈不够深, 引发问题的我的代码或者客户代码的一些东西. 它仍然在ntoskrnl中(Windows系统内核). 

所以对于这种情况, 我可以请PerfView的owner来看看(当时已经是深夜了;即便他当时真的响应了, 也未必有时间马上来看, 而且这个看起来也不是什么可以马上解决的小事. 我真的很想把这个问题搞清楚😀), 或者我可以自己想一些其他的办法来取得更多的进展. 当针对一类问题设计的视图不能使用时, 总会有一个视图可以拯救您, 那就是事件视图. 当然, 可能还有其他工具可以解决它们, 比如WPA. 但我上一次认真使用WPA已经是好几年的前的事了, UI和我熟悉的UI已经有很大的不同. 


事件视图是事件的原始版本, 包含在. etl文件中(这并不完全准确, PerfView仍然对一些事件进行了处理;但在多数情况下, 它是非常原始的--您可以得到事件的名称和每个事件的字段). 而我最感兴趣的ReadyThread事件, 他告诉我哪个线程是由其他的哪个线程唤醒的. 所以我做的第一件事就是过滤到我想看到的事件. 否则会有太多的事件. 我打开"Event"视图, 再次输入开始和结束的时间戳, 就像我在其他视图中做的那样, 然后在Process Filter中输入感兴趣的过程(为了保护隐私, 我仅仅使用X来说明). 为了只过滤挂起和ReadyThread事件, 我在filter框中输入sus|ready. 这将只包含"sus"或"ready"的事件 --

![WorkFlow-suspension0](/posts/images/WorkFlow-suspension0. jpg)

现在, 选中三个事件并输入回车, 现在我只会看到这三个-

![WorkFlow-suspension1](/posts/images/WorkFlow-suspension1. jpg)

这里有很多事件, 但我只对那些唤醒我的线程感兴趣, 也就是本例中的GC线程`ThreadID 7736`(在服务器GC中, SuspendEE总是由Heap0的GC线程调用), 手动检查这些线程是很困难的, 所以我希望只过滤那些有趣的线程. 做到这一点的方法是让事件视图展示一列单独的字段, 这样我就可以进行排序--在默认情况下, 它只在一列中显示所有字段. 我可以通过点击Cols按钮("Columns To Display"旁边), 选择我想要显示的字段. 我选择了三个字段, 分别是--

![WorkFlow-suspension2](/posts/images/WorkFlow-suspension2. jpg)

现在, 它显示了这三个字段, 然后我按AwakenedThreadID进行排序, 并寻找我的线程7736. 

![WorkFlow-suspension3](/posts/images/WorkFlow-suspension3. jpg)

果然, 有一个有意思的线程-线程33108. 如果我点击其中一个时间戳, 然后按Alt+S(意味着打开与这个时间戳相关的任意调用栈;您也可以通过上下文的菜单进入), 我看到了这个调用栈--

![WorkFlow-suspension4](/posts/images/WorkFlow-suspension4. jpg)

底部写着"Readied Thread 7736". 而其他的时间戳几乎都有相同的调用栈. 我只显示了运行时里面的部分, 但是对于客户代码里面的部分, 总是同一个dll抛出的异常. 原来这是我挂起代码的一个BUG-它在调用 **`GetFileVersionInfoSize`** 系统API之前, 应该先切换到抢占模式. 我猜测这是一个处理异常的代码地址(最上面的异常处理代码叫做 **`coreclr!DwGetFileVersionInfo`** ), 它没有被广泛使用, 所以我直到现在才注意到. 对客户来说, 一个变通的方法是避免让他们的dll抛出这些异常, 这样会使它不调用这个运行时的代码地址. 

这就是今天的全部内容了, 如果您有任何反馈, 请一如既往的告诉我. 