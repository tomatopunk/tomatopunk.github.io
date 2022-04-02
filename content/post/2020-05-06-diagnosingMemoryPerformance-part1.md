---
title: 诊断性能问题的工作流程(1)
date: 2020-05-06 14:27:26
update: 2020-11-13 14:27:26
categories: [Index]
tags:
    - C#
    - Maoni
    - PerfView
---
原文为Maoni发布在Microfost Blog中的, 诊断性能问题的工作流程系列. 
目前共更新三章. 在本章中, Maoni介绍了一些在进行性能诊断时的操作与分析. 
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

在此篇文章中, 我会讨论一些关于怎样对PerfView做出贡献的内容, 然后继续分析GCStats. 您可以直接[跳到分析部分](https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-1/#continuing-the-analysis). 

对于分析工具, 有一点令我沮丧, 市面上有很多的内存性能工具, 但很少有针对可通用类型以及与我所服务的客户的. 所有的工具都很基础, 很少有工具能进行中级和高级分析. 

我知道有很多人在抱怨PerfView的可用性 - 我确实赞同一些抱怨. 但尽管如此, 我还是喜欢PerfView, 因为这是它往往是我唯一一个能用来完成工作的工具. 

我希望大家能够理解

1) 我能放在PerfView上的精力非常有限. 我没有类似Visual Studio组织完整的工作团队;我只有部分时间, 来自于少部分成员的兼职, 所以很难满足所有用户的要求. 
2) 在进行高级分析时, 情况会变得非常复杂, 这意味着实现可用性没有那么简单-当有很多需要关注的细节时, permutation很快会变得十分庞大. 

对类似PerfView之类的项目做出贡献, 是对`. NET Core`做出贡献的绝佳方式, 它没有运行时本身那么陡峭的学习曲线, 但是您的贡献可能会帮助人们节省大量时间.  

您可以从克隆[Perfview repo](https://github.com/microsoft/perfview/)并编译它开始. 然后您可以通过单步执行代码来学习了 – IMO 单步执行代码,  这往往是最好的了解新鲜事物的方法. 

我在此讨论内容的代码大部分都位于2个文件中
    
    - src\TraceEvent\Computers\TraceManagedProcess. cs 
    - src\PerfView\GCStats. cs.  

如果您搜索诸如Clr. EventName(例如Clr. GCStart与Clr. GCStop), 这里就是进行事件分析的地方(您不需要关心对于跟踪的解析 – 那在其他地方处理的). 

对于GC的分析就是这个文件中的[GLAD](https://devblogs.microsoft.com/dotnet/glad-part-2/) (GC Latency Analysis and Diagnostics)库. GCStats. cs使用它来显示您在GCStats视图中看到的东西, 它是一个HTML文件. 如果您想在自己的工具上展示GC的相关信息, GCStats. cs是一个很好的使用GLAD的例子. 

### 继续分析


该篇包含两部分内容. 怎样对PerfView进行贡献, 然后我会继续分析 GCStats. 您也可以直接跳转到 [分析部分](#)(https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-1/#continuing-the-analysis).

虽然有许许多多的分析内存性能的工具, 能让我和我的客户完成大多数工作的的却很少. 每个工具都很基础, 无法进行中级和高级的分析.

我知道有很多人认为 PerfView 难以使用 - 我赞同其中的一些观点. 但尽管如此, 我还是喜欢PerfView. 往往我只能使用它来完成工作. 

希望大家能理解: 1) 我能放在 PerfView 项目上的精力非常有限. 我们没有类似 Visual Studio 这样完整的团队; 只有少部分成员的兼职, 所以很难满足所有用户的要求. 2) 在进行高级分析时, 情况往往非常复杂, 当有如此多的可能性, 列表会快速地变大.

对类似PerfView的项目进行贡献, 也是对 `.Net Core` 做出贡献的绝佳方式, 该项目没有运行时陡峭的学习曲线, 但是您的贡献会帮助人们节省大量时间. 您可以从克隆[Perfview repo](#)(https://github.com/microsoft/perfview/)并编译它开始. 然后您可以通过单步执行代码进行学习.

> 在我看来, 单步执行代码, 这总是理解新事物的最好方法.
> 
我在此处讨论内容的相关代码大部分都位于2个文件中. srcTraceEventComputersTraceManagedProcess.cs 和 srcPerfViewGCStats.cs. 

如果您搜索Clr.EventName (例如Clr.GCStart与Clr.GCStop) . 此处就是进行事件分析的地方 (您不需要关心对于跟踪的解析, 那是在其他地方处理的). 对于GC的分析就是这个文件中的[GLAD](#)(https://devblogs.microsoft.com/dotnet/glad-part-2/) (GC Latency Analysis and Diagnostics)库. GCStats.cs 使用它来显示您在 GCStats 视图, 这是一个 HTML 文件. 如果您想在自己的工具上展示 GC 的相关信息, GCStats.cs 是一个很好的使用 GLAD 的例子. 


### 继续分析

上一篇文章中, 我们谈到了收集 GCCollectOnly 跟踪和检查 PerfView 中的GCStats视图, 该视图由收集的 GC 事件启用. 您也可以在 Linux 上用 dotnet-trace 做这件事. 从它的[文档](https://github.com/dotnet/diagnostics/blob/master/documentation/dotnet-trace-instructions.md)中可以看出: 它提供的一个内置配置文件相当于 PerfView 的收集命令中的 `/GCCollectOnly` 参数。

``` Shell
 --profile

   [omitted]

   gc-collect   以极低的性能开销仅跟踪收集GC
```

您可以使用dotnet-trace命令, 在Linux上收集跟踪信息. 

** dotnet trace collect -p \<pid\> -o \<outputpath\> --profile gc-collect**

并在 Windows 上用 PerfView 查看. 从使用者的角度看, 当您查看 GCStats 视图时, 两者唯一的区别是, 在 Windows 上收集的跟踪, 您会看到所有的托管线程, 而在 Linux 上收集的跟踪只有您指定的pid的进程. 

在这篇博文中, 我将重点介绍您在 GCStats 中看到的表格. 我在这里展示一个例子. 一个进程的第一个表是 "GC Rollup By Generation "表 --

![WorkFlow-GCRollupByGeneration](#)(/posts/images/WorkFlow-GCRollupByGeneration.png)

我省略了 `Alloc MB/MSec GC` 与 `Survived MB/MSec GC` 这两列 – 他们在我开始研究PerfView之前就已经存在了, 如果能把它们修复得更有意义就好了, 但我一直没有处理.

现在, 如果您在做一个日常性分析, 也就是说您没有一个直接的目标, 只是想看看是否有什么需要改进的地方, 您可以从这个滚动表开始. 


如果我们查看上面的表格, 就会发现 gen2 的平均中断时间比 gen0/1 的 GCs 大很多. 我会猜测 gen2 可能没有经历过中断, 因为`Max Peak MB` 大约是13GB，如果我们要遍历所有这些内存, 大概要花费超过167ms. 所以这些很可能是BGCs (Background GC), 这一点在滚动表下面的 "Gen 2 for pid: process_name "表中得到了证实（我从表中删除了一些列，这样就不会太宽了） -_

![WorkFlow-GCRollupByGeneration2](#)(/posts/images/WorkFlow-GCRollupByGeneration2.png)

2B表示在后台执行的二代堆GC. 如果您想知道还有哪些组合, 只需将鼠标悬停在"Gen"的列标题上, 将显示以下文字:

** N=NonConcurrent/非并发式GC, B=Background/后台GC, F=Foreground/前台GC (一直以后台GC运行) I=Induced/触发式GC i=InducedNotForced/触发非前台 **

- N=NonConcurrent/非并发式GC
- B=Background/后台GC
- F=Foreground/前台GC (一直以后台GC运行)
- I=Induced/触发式GC
- i=InducedNotForced/触发非前台

所以对于`二代堆GC`, 您可能会看到2N, 2NI, 2Ni or 2Bi. 如果您使用GC. Collect来触发GC, 它有两个采用此参数的重载 –

``` cs
(bool blocking)
```

除非您将该参数指定为False, 它意味着将始终以阻塞的方式触发GC. 这就是为什么没有2BI的原因

在rollup表中, 始终有一列`Induced`显示为0 , 但如果这不是0, 特别是当与GC的总数相比时是一个相当大的数字时, 找出是谁在触发这些GC是一个非常好的主意. 这在这篇[博客](#)(https://devblogs.microsoft.com/dotnet/gc-etw-events-2/)中做了详细的讲解. 

所以, 我知道了这些GC总数全部来源于BGC, 但是, 对于BGC来说, 这些中断的时间太长了! 

请注意, 虽然我将两次中断显示成了一次, 但它实际是由两次中断组成的. 在[这张来自GC MSDN的图片](#)(https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/media/fundamentals/background-workstation-garbage-collection.png)中, 显示了一次BGC中的两次中断(蓝色的列所指的位置). 

但是, 您在GCStats中看到的中断时间是这两次中断的总和. 原因是最早地中断通常都非常短(图片中的蓝色列仅用于举例-它们并不代表实际上真正用了多长时间).  在这种情况下, 我想看每个单独地中断花费了多长时间 - 我正在考虑在GLAD中提供各个BGC的中断信息, 但在这之前, 您可以自己弄清楚. 

在[这篇博客中](#)(https://devblogs.microsoft.com/dotnet/gc-etw-events-3/), 我描述了BGC在触发事件时的顺序. 所以我需要找到两次`SuspendEE/RestartEE`事件. 

要做到这一点, 您可以在PerfView中打开`“Events”`视图, 然后从`“Pause Start”`开始. 

让我以GC#9217为例, 它的首次中断在789, 274. 32 您可以在“Start”输入框中输入它. 然后在“Process Filter”中输入“gc/”, 仅过滤GC事件, 然后选择SuspendEE/RestartEE/GCStart/GCStop事件, 摁下回车. 

下面是此时您将会看到的示例图片(出于隐私原因, 我删除了进程名字) - 
![WorkFlow1-0](#)(/posts/images/WorkFlow1-0.jpg)

这就是首次发生中断的地方, 如果您选择首次SuspendEEStart和首次RestartEEStop的时间戳, 我可以看到在这个视图的状态栏上显示了两个时间戳的差异是75. 902. 这已经非常长了 -通常来说, 首次中断时间每组都应当不超过几毫秒. 对于这种情况, 您基本上可以将其交给我, 因为在我的设计中, 不应该出现这种情况. 

但是, 如果您有兴趣自己继续诊断, 下一步是捕获更多的事件跟踪, 来向我展示挂起期间发生了什么. 通常, 我捕获的跟踪是CPU事件样本+GC的事件跟踪. CPU样本清楚的向我展示了真正的罪魁祸首. 其实并不是在GC中, 而是运行时中的其他东西. 后来我已经修复了, 这个性能问题只有在您的程序中有多个模块时才会发生(在这个特殊的场景下, 客户拥有数千个模块). 

第二次的BGC中断从SuspendEEStart事件, 原因是“SuspendForGCPrep”, 与第一次的SuspendEESrart事件不同的是, 此次原因是“SuspendForGC”. 

由GC为目的引发的停顿, 仅有两个可能, 而“SuspendForGCPrep”仅在初次中断的BGC期间可用. 

通常来说, 一个BGC仅会有两次中断, 但如果您启用了`GCHeapSurvivalAndMovementKeyword`事件, 您将在BGC期间添加第三个中断, 因为要触发这些事件, 托管线程必须处于中断状态. 如果是这种情况, 第三次暂停也会有“SuspendForGCPerp”原因, 并且通常比其他两个中断要长的多. 因为堆如果很大, 触发事件将花费很长的时间. 

我见过很多这种情况, 当大家根本不需要这些事件时, 却看到BGC的中断时间被人为拉长. 原因正是这个. 

您可能会问, 既然不需要这些事件, 为什么还会不小心的收集到这些事件. 这是因为您在收集运行时事件时, 它们已经包含在默认值中(您可以在srcTraceEventParsersClrTraceEventParser.cs 中看到默认值中包含的关键字, 搜索default. 您会看到许多关键字被包含在了默认值中). 

一般来说, 我认为PerfView的理念是, 默认情况下应当收集足够的事件提供给您以便进行调查. 在一般情况下, 这都是一个很好的策略, 因为您可能无法对问题进行复现. 但是您需要通过收集事件本身来判断, 什么是由于收集事件引发的, 什么是由于产品引发的. 

当然, 这是在建立在您有能力收集这么多事件的基础上. 有时绝对不是这种情况. 这就是为什么我通常要求人们从轻量级跟踪开始, 来向我表明这里是否存在问题, 以及如果存在问题, 我还需要收集哪些事件. 

我从gen2表注意到了的另一件事, 所有的BGC都是由AllocLarge触发的. 可能被触发的原因定义为GCReason:

- [srcTraceEventParsersClrTraceEventParser.cs](#)(https://github.com/microsoft/perfview/blob/master/src/TraceEvent/Parsers/ClrTraceEventParser.cs)

``` cs
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

最常见的原因是`AllocSmall`, 意味着您在`SOH(Small Object Heap)`的分配触发了GC. 而`AllocLarge`意味着`LOH(Long Object Heap)`的分配触发了GC.  


在这种特殊下团队已经意识到, 他们正在进行大量的LOH分配 – 但他们可能不知道会经常导致BGC. 

如果您查看“Gen2 Survival Rate %”列, 您会注意到二代的存活率非常高(97%), 但是“LOH Survival Rate %(LOH存活率 %)”却非常低-29%

这告诉了我, 有许多的LOH分配存活的相当短. 我会根据gen2的预算调整LOH的预算(分配量阈值), 因此在这种情况下, 我不会过多的触发gen2的GC. 

如果我想提高LOH的存活率, 我需要比这更加频繁地触发BGC. 如果您很清楚您的LOH的分配通常是临时的, 那么通过GCLOHThreshold的配置增大LOH的阈值就是一个不错的办法. 

这就是今天的全部内容了. 下次我将讨论GCStats视图中更多表.