---
title: 诊断性能问题的工作流程(1)
date: 2020-05-06 14:27:26
update: 2020-11-13 14:27:26
categories: C#
tags:
    - C#
    - 性能诊断
    - Maoni
    - PerfView
---
原文为Maoni发布在Microfost Blog中的,诊断性能问题的工作流程系列.
目前共更新三章.在本章中,Maoni介绍了一些在进行性能诊断时的操作与分析.
<!-- more -->

## 原文信息

- [@Maoni Stephens-Twitter](https://twitter.com/maoni0)
- [@Maoni Stephens-Github](https://github.com/Maoni0)

![authorize](/posts/images/1587561145552-0d8a560c-3b7d-443a-badc-a98ddbb6e7bf.png)

如果这篇文章可以帮到您，那么这将是我最大的荣幸，希望您点进原文，在文章下方留下善意的回复，您的支持将是这些可敬的社区磐石保持创作激情中最大的一部分:)

[原文](https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-1)

**中文版本将不会以任何形式收费，版权属与原作者**

---

## 正文

在此篇文章中,我会讨论一些关于怎样对PerfView做出贡献的内容,然后继续分析GCStats.您可以直接[跳到分析部分](https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-1/#continuing-the-analysis).

对于分析工具,有一点令我沮丧,市面上有很多的内存性能工具,但很少有针对可通用类型以及与我所服务的客户的.所有的工具都很基础,很少有工具能进行中级和高级分析.

我知道有很多人在抱怨PerfView的可用性 - 我确实赞同一些抱怨.但尽管如此,我还是喜欢PerfView,因为这是它往往是我唯一一个能用来完成工作的工具.

我希望大家能够理解

1) 我能放在PerfView上的精力非常有限.我没有类似Visual Studio组织完整的工作团队;我只有部分时间,来自于少部分成员的兼职,所以很难满足所有用户的要求.
2) 在进行高级分析时,情况会变得非常复杂,这意味着实现可用性没有那么简单-当有很多需要关注的细节时,permutation很快会变得十分庞大.

对类似PerfView之类的项目做出贡献,是对`.NET Core`做出贡献的绝佳方式,它没有运行时本身那么陡峭的学习曲线,但是您的贡献可能会帮助人们节省大量时间. 

您可以从克隆[Perfview repo](https://github.com/microsoft/perfview/)并编译它开始.然后您可以通过单步执行代码来学习了 – IMO 单步执行代码, 这往往是最好的了解新鲜事物的方法.

我在此讨论内容的代码大部分都位于2个文件中
    
    - src\TraceEvent\Computers\TraceManagedProcess.cs 
    - src\PerfView\GCStats.cs. 

如果您搜索诸如Clr.EventName(例如Clr.GCStart与Clr.GCStop),这里就是进行事件分析的地方(您不需要关心对于跟踪的解析 – 那在其他地方处理的).

对于GC的分析就是这个文件中的[GLAD](https://devblogs.microsoft.com/dotnet/glad-part-2/) (GC Latency Analysis and Diagnostics)库.GCStats.cs使用它来显示您在GCStats视图中看到的东西,它是一个HTML文件.如果您想在自己的工具上展示GC的相关信息,GCStats.cs是一个很好的使用GLAD的例子.

### 继续分析

在上一篇文章,我讨论了关于收集GCCollectOnly的跟踪,并在PerfView中启用GC事件集合以及检查GCStats视图.

我应该提醒您,在Linux上可以使用dotnet-trace来做这件事. 根据[dotnet-trace的文档](https://github.com/dotnet/diagnostics/blob/master/documentation/dotnet-trace-instructions.md):它提供了一个内置配置,与`/GCCollectOnly`参数等效的收集命令.

``` shell
 --profile

   [omitted]

   gc-collect   以极低的性能开销仅跟踪收集GC
```

您可以使用dotnet-trace命令,在Linux上收集跟踪信息.

**` dotnet trace collect -p <pid> -o <outputpath> --profile gc-collect`**

然后在Windows上用PerfView进行展示.当您查看GCStats视图时,从使用者的角度来看,与使用PerfView收集唯一的不同是:在Windows进行收集跟踪,您可以看到所有的托管线程.而在Linux收集跟踪则只会包含您指定PID的线程.

在这篇文章中,我将重点介绍您看到的GCStats表格.我会展示一个例子.流程的第一张表:“GC Rollup By Generation” –

![WorkFlow-GCRollupByGeneration](/posts/images/WorkFlow-GCRollupByGeneration.png)

我忽略了`Alloc MB/MSec GC`与`Survived MB/MSec GC`这两列 – 他们在我开始使用PerfView之前就已经存在了,如果能把他们修复一下,让它们更有意义就更好了,但我一直没有处理.

如果您想进行一个例行的分析,通常意味着您没有一个直接的目标,您只想看看是否有可以改进的地方,可以从`Rollup`表开始.

如果我查看上一个表,我马上就会注意到gen2的平均中断时间比gen0/1的GC大得多.我可以猜测这些`二代堆GC`可能没有经历过中断,因为`Max Peak MB(最大峰值MB)`大约为13GB,如果我要遍历所有的内存,大概要花费167ms.所以,这些可能是后台GC(Background GC),这一点在`Rollup`表下方的 “Gen 2 for pid: process_name” 表中得到了确认 (我删除了一些列,这样就不会太宽了) –

![WorkFlow-GCRollupByGeneration2](/posts/images/WorkFlow-GCRollupByGeneration2.png)

2B表示在后台执行的二代堆GC.如果您想知道还有哪些组合,只需将鼠标悬停在"Gen"的列标题上,将显示以下文字:

**` N=NonConcurrent/非并发式GC, B=Background/后台GC, F=Foreground/前台GC (一直以后台GC运行) I=Induced/触发式GC i=InducedNotForced/触发非前台 `**

- N=NonConcurrent/非并发式GC
- B=Background/后台GC
- F=Foreground/前台GC (一直以后台GC运行)
- I=Induced/触发式GC
- i=InducedNotForced/触发非前台

所以对于`二代堆GC`,您可能会看到2N, 2NI, 2Ni or 2Bi.如果您使用GC.Collect来触发GC,它有两个采用此参数的重载 –

``` java
bool blocking
```

除非您将该参数指定为False,它意味着将始终以阻塞的方式触发GC.这就是为什么没有2BI的原因

在rollup表中,始终有一列`Induced`显示为0 ,但如果这不是0,特别是当与GC的总数相比时是一个相当大的数字时,找出是谁在触发这些GC是一个非常好的主意.这在这篇[博客](https://devblogs.microsoft.com/dotnet/gc-etw-events-2/)中做了详细的讲解.

所以,我知道了这些GC总数全部来源于BGC,但是,对于BGC来说,这些中断的时间太长了! 

请注意,虽然我将两次中断显示成了一次,但它实际是由两次中断组成的.在[这张来自于GC MSDN的图片](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/media/fundamentals/background-workstation-garbage-collection.png)中,显示了一次BGC中的两次中断(蓝色的列所指的位置).

但是,您在GCStats中看到的中断时间是这两次中断的总和.原因是最初的中断通常都非常短(图片中的蓝色列仅用于举例-它们并不代表实际上真正用了多长时间). 在这种情况下,我想看每个单独的中断花费了多长时间 - 我正在考虑在GLAD中提供各个BGC的中断信息,但在这之前,您可以自己弄清楚.

在[这篇博客中](https://devblogs.microsoft.com/dotnet/gc-etw-events-3/),我描述了BGC在触发事件时的顺序.所以我需要找到两次`SuspendEE/RestartEE`事件.

要做到这一点,您可以在Perfview中打开`“Events”`视图,然后从`“Pause Start”`开始.

让我以GC#9217为例,它的首次中断在789,274.32 您可以在“Start”输入框中输入它.然后在“Process Filter”中输入“gc/”,仅过滤GC事件,然后选择SuspendEE/RestartEE/GCStart/GCStop事件,摁下回车.

下面是此时您将会看到的示例图片(处于隐私原因,我删除了进程名字) - 
![WorkFlow1-0](/posts/images/WorkFlow1-0.jpg)

这就是首次发生中断的地方,如果您选择首次SuspendEEStart和首次RestartEEStop的时间戳,我可以看到在这个视图的状态栏上显示了两个时间戳的差异是75.902.这已经非常长了 -通常来说,首次中断时间每组都应当不超过几毫秒.对于这种情况,您基本上可以将其交给我,因为在我的设计中,不应该出现这种情况.

但是,如果您有兴趣自己继续诊断,下一步是捕获更多的事件跟踪,来向我展示挂起期间发生了什么.通常,我捕获的跟踪是CPU事件样本+GC的事件跟踪.CPU样本清楚的向我展示了真正的罪魁祸首.其实并不是在GC中,而是运行时中的其他东西.后来我已经修复了,这个性能问题只有在您的程序中有多个模块时才会发生(在这个特殊的场景下,客户拥有数千个模块).

第二次的BGC中断从SuspendEEStart事件,原因是“SuspendForGCPrep”,与第一次的SuspendEESrart事件不同的是,此次原因是“SuspendForGC”.

由GC为目的引发的停顿,仅有两个可能,而“SuspendForGCPrep”仅在初次中断的BGC期间可用.

通常来说,一个BGC仅会有两次中断,但如果您启用了`GCHeapSurvivalAndMovementKeyword`事件,您将在BGC期间添加第三个中断,因为要触发这些事件,托管线程必须处于中断状态.如果是这种情况,第三次暂停也会有“SuspendForGCPerp”原因,并且通常比其他两个中断要长的多.因为堆如果很大,触发事件将花费很长的时间.

我见过很多这种情况,当大家根本不需要这些事件时,却看到BGC的中断时间被人为的拉长.原因正是这个.

您可能会问,既然不需要这些事件,为什么还会不小心的收集到这些事件.这是因为您在收集运行时事件时,它们已经包含在默认值中(您可以在src\TraceEvent\Parsers\ClrTraceEventParser.cs中看到默认值中包含的关键字,搜索default.您会看到许多关键字被包含在了默认值中).

一般来说,我认为PerfView的理念是,默认情况下应当收集足够的事件提供给您以便进行调查.在一般情况下,这都是一个很好的策略,因为您可能无法对问题进行复现.但是您需要通过收集事件本身来判断,什么是由于收集事件引发的,什么是由于产品引发的.

当然,这是在建立在您有能力收集这么多事件的基础上.有时绝对不是这种情况.这就是为什么我通常要求人们从轻量级跟踪开始,来向我表明这里是否存在问题,以及如果存在问题,我还需要收集哪些事件.

我从gen2表注意到了的另一件事,所有的BGC都是由AllocLarge触发的.可能被触发的原因定义为GCReason:

- [src\TraceEvent\Parsers\ClrTraceEventParser.cs](https://github.com/microsoft/perfview/blob/master/src/TraceEvent/Parsers/ClrTraceEventParser.cs)

``` java
public enum GCReason
{
    AllocSmall = 0x0,
    Induced = 0x1,
    LowMemory = 0x2,
    Empty = 0x3,
    AllocLarge = 0x4,
    OutOfSpaceSOH = 0x5,
    OutOfSpaceLOH = 0x6,
    InducedNotForced = 0x7,
    Internal = 0x8,
    InducedLowMemory = 0x9,
    InducedCompacting = 0xa,
    LowMemoryHost = 0xb,
    PMFullGC = 0xc,
    LowMemoryHostBlocking = 0xd
}
```

最常见的原因是`AllocSmall`,意味着您在`SOH(Small Object Heap)`的分配触发了GC.而`AllocLarge`意味着`LOH(Long Object Heap)`的分配触发了GC. 

在这种特殊下团队已经意识到,他们正在进行大量的LOH分配 – 但他们可能不知道会经常导致BGC.

如果您查看“Gen2 Survival Rate %”列,您会注意到二代的存活率非常高(97%),但是“LOH Survival Rate %(LOH存活率 %)”却非常低-29%

这告诉了我,有许多的LOH分配存活的相当短.我会根据gen2的预算调整LOH的预算(分配量阈值),因此在这种情况下,我不会过多的触发gen2的GC.

如果我想提高LOH的存活率,我需要比这更加频繁的触发BGC.如果您很清楚您的LOH的分配通常是临时的,那么通过GCLOHThreshold的配置增大LOH的阈值就是一个不错的办法.

这就是今天的全部内容了.下次我将讨论GCStats视图中更多表.