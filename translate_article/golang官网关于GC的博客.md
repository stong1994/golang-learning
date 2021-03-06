本人能力有限，有错误的地方望众大佬多多提点！
> 本文主要讲述了Go的Gc的优化思路与成果，并展望了摩尔定律会对GC优化产生的影响。


*本篇只翻译重要部分*

**原文**：https://blog.golang.org/ismmkeynote

# 关于go：一个关于go的GC的旅程
> 12 July 2018

![image](https://blog.golang.org/ismmkeynote/image63.png)
这篇博客实际上是作者在ISMM上的发言记录

![](https://blog.golang.org/ismmkeynote/image38.png)
在开始讨论之前，我们需要展示对Go的GC的看法。

![](https://blog.golang.org/ismmkeynote/image32.png)
首先，go程序有成千上万的栈。它们由Go的调度器管理并且总是在安全点（safepoints）被抢占。Go调度器将Goroutine多路复用到OS线程上，希望每个 HW 线程运行一个 OS 线程。我们通过复制栈和更新栈中的指针来管理栈和它们的大小。这是一个本地操作，所以它表现的相当不错。

![](https://blog.golang.org/ismmkeynote/image22.png)
下一件重要的事情是，Go是一种**值传导**的语言，像传统的 c类语言一样，而不是引用传导语言。举个例子，这展示了 tar 包中的类型是如何在内存中布局的。 所有字段都直接嵌入到 Reader 中。 这使程序员在需要时能够更好地控制内存布局。 通过将相关的字段放在一起，能够提高局部缓存（可以理解为将字段类型按照占用内存大小的顺序进行排列设置，能够减小整体的内存占用）。
> 相关资料：https://www.flysnow.org/2017/07/02/go-in-action-unsafe-memory-layout.html

值传导还有助于使用外部函数接口。 显然，谷歌有大量可用的工具，但它们都是用 c + + 编写的。 我们像要在 Go 中重新实现所有这些功能，所以 Go 必须通过外部函数接口访问这些系统。

这个设计决定导致在运行时产生了令人惊奇的事情。 这可能是Go语言区别于其他 GCed 语言最重要的地方。

![](https://blog.golang.org/ismmkeynote/image60.png)
Go可以使用指针，甚至内部指针。这样的指针能够使整个值保持活动状态并且是很常见的做法。

![](https://blog.golang.org/ismmkeynote/image29.png)
我们还有一个提前编译系统，所以二进制文件包含整个运行系统。
没有JIT 重新编译。 这有利也有弊。 首先，程序执行的可重复性要容易得多，这使得编译器的改进更快。
另一方面，我们没有机会像使用 JITed 系统那样进行反馈优化。

![](https://blog.golang.org/ismmkeynote/image13.png)
Go 有两个控制 GC 的旋钮。 第一个是 **GCPercent**。 这是一个旋钮，你可以用来调整使用多少 CPU和多少内存。 默认值为100，这意味着一半的堆用于活动内存，一半的堆用于分配。 你可以随意修改。

**Maxheap** 尚未发布，但正在内部使用和计算，它使得程序员可以设置堆的最大大小。 内存不足，OOMs 对于Go来说是很困难的; 内存使用的暂时峰值应该通过增加 CPU 成本来解决，而不是通过终止。 基本上，如果 GC 看到内存压力，它会通知应用程序应该减少负载。 一旦一切恢复正常，GC 就通知应用程序可以返回其常规加载。 Maxheap 还提供了更大的调度灵活性。 运行时不必总是担心有多少内存可用，而是可以计算堆的大小，直到 MaxHeap。

Gc是非常重要的。

![](https://blog.golang.org/ismmkeynote/image3.png)
我们来聊一聊Go的运行。

![](https://blog.golang.org/ismmkeynote/image59.png)
在2014年，如果 Go 不能以某种方式解决 GC 延迟问题，那么 Go 就不会成功。

其他新语言也面临着同样的问题。 像 Rust 这样的语言走了一条不同的道路，但是我们将要讨论Go走过的道路。

为什么延迟如此重要？

![](https://blog.golang.org/ismmkeynote/image7.png)
一个99% 独立的 GC **延迟服务级别目标(SLO)** ，例如，在99%的时间里GC周期时间小于10ms，就是不能进行伸缩。重要的是整个会话的延迟，或者在一天时间内多次使用同一个app的延迟。假设在一个会话中浏览几个网页产生了100个服务请求，或者它产生了20个请求而且你在一天内有5个会话。在这种情况下，只有37%的用户在整个会话中一直能有低于10毫秒延迟的体验。（0.99^100=0.37）

所以我们需要数学公式来明确延迟在实际项目中的影响。

在2014年，Jeff Dean 刚刚发表了他的论文 'The Tail at Scale'，这篇论文对上述情况进行了深入研究。这篇论文在谷歌备受欢迎，并且对谷歌的未来发展和规模产生了严重影响。

我们把这个问题称为9秒暴政。

![](https://blog.golang.org/ismmkeynote/image36.png)
那么，你如何对抗9秒暴政呢？

2014年我们做了很多事情。

如果你想要得到10个答案，那就多问几个，然后取前10个，这些就是放在搜索页面上的答案。如果请求超过50％，则重新发出请求或将请求转发给另一台服务器。**如果 GC 即将运行，请拒绝新的请求或将请求转发到另一台服务器，直到完成 GC**。 诸如此类。

所有这些变通方法都来自于非常聪明的人，他们解决了非常实际的问题，但是他们没有解决 GC 延迟的根本问题。 在谷歌，我们必须解决根本问题。 为什么？

![](https://blog.golang.org/ismmkeynote/image48.png)
冗余无法扩大规模，冗余成本很高。 这需要新的服务器群的成本。

我们希望我们能够解决这个问题，并把它看作一个改善服务器生态系统的机会，在这个过程中拯救一些濒临灭绝的玉米地，让一些玉米粒有机会在7月4日达到膝盖高度，并充分发挥其潜力。

![](https://blog.golang.org/ismmkeynote/image56.png)
这是2014年的 SLO。 是的，这是真的，我是沙袋，我是团队的新人，这是一个新的过程，我不想过度承诺。

此外，其他语言中关于 GC 延迟的演示非常吓人

![](https://blog.golang.org/ismmkeynote/image67.png)
最初的计划是做一个**无读屏障的并发复制GC**。 这是一个长期的计划。 读屏障有很多不确定因素，所以 Go 想要避免它们。

但短期来看，2014年我们必须齐心协力。 我们必须将所有的运行时和编译器转换为 Go。 他们当时是用 c 写的。 不再有 c 语言，也不再有长长的 bug，因为 c 语言编码员不理解 GC，但是对如何复制字符串有一个很酷的想法。 我们还需要一些快速的、专注于延迟的东西，但是性能损失必须小于编译器提供的加速比。 所以我们是有限的。 我们基本上有一年的编译器性能改进，我们可以通过使 GC 并发消耗掉这些性能改进。 但仅此而已。 我们不能放慢Go程序。 这在2014年是站不住脚的。

![](https://blog.golang.org/ismmkeynote/image28.png)
所以我们后退了一点。 我们不打算做复制部分。
我们的决定是做一**个三色并发算法**。 在我职业生涯的早期，Eliot Moss和我证明过 Dijkstra 的算法适用于多个应用程序线程。 我们还展示了我们可以解决 STW 问题，并且我们有证据证明这是可以做到的。

我们还关心编译器的速度，即编译器生成的代码的速度。 如果我们大部分时间关闭写屏障，编译器优化的影响将最小，编译器团队可以快速前进。 在2015年，Go 也迫切需要短期的成功。

![](https://blog.golang.org/ismmkeynote/image55.png)
让我们来看看我们做过的一些事情。
我们采用**了尺寸分离跨度**(size segregated span)。 内部指针是一个问题。

垃圾收集器需要有效地找到对象的起始点。 如果它知道一个区域中对象的大小，那么它就会向下舍入到这个大小，这将是对象的开始。

当然，尺寸分离跨度还有其他一些优点。

**低碎片化**: 由于有 c 的经验，除了谷歌的 TCMalloc 和 Holder，我还密切参与了英特尔的 Scalable Malloc，这项工作让我们相信碎片化不会成为不动分配器（non-moving allocators）的问题。

**内部结构**: 我们充分理解它们，并对它们有经验。 我们知道如何做尺寸分离跨度，我们知道如何降低甚至零竞争分配路径。

**速度**: 非拷贝不关我们的事，分配固然可能比较慢，但还是按 c 的顺序。 它可能没有凹凸指针那么快，但是还可以。

我们还有这个**外部函数接口**的问题。如果你有一个移动的收集器，你试图固定对象并在C和你正在使用的Go对象之间放置间接级别的东西时可能会遇到一连串的错误，如果我们不移动对象，那么我们就没有必要处理这些错误。

![](https://blog.golang.org/ismmkeynote/image5.png)
下一个设计,我们的选择是处理**将对象的元数据放在哪里**。 我们需要一些关于对象的信息，因为我们没有头部。 标记位保留在边上，用于标记和分配。 每个单词都有两个相关的位，用来告诉你这个单词里面是一个标量还是一个指针。 它还对对象中是否有更多的指针进行编码，这样我们就可以尽快停止扫描物体。 我们还有一个额外的位编码，可以用作额外的标记位或做其他调试工作。 这对于让这些东西运行和发现 bug 是非常有价值的。

![](https://blog.golang.org/ismmkeynote/image19.png)
那么**写屏障**是什么呢？ 只有在 GC 期间，写屏障是打开的。 在其他时候，编译后的代码加载一个全局变量并查看它。 因为 GC 通常是关闭的，所以硬件能绕过写屏障进入分支。 当我们在 GC 时，变量是不同的，**写屏障负责确保在三色操作期间不会丢失可访问的对象**。

![](https://blog.golang.org/ismmkeynote/image50.png)
代码的另一部分是 GC **Pacer**。 这是Austin 所做的一些伟大的工作。 它基本上是基于一个反馈循环来决定何时最好地启动 GC 循环。 如果系统处于稳定状态并且没有相位变化，标记将在内存耗尽的时候结束。

但事实可能并非如此，因此**Pacer还必须监控标记进度，确保分配不超过并发标记**。

如果需要，Pacer会减慢分配速度，同时加快标记速度。在高水平上，Pacer会 停止做了大量分配的 Goroutine，并把它们用来做标记。 工作量与 Goroutine 的分配成比例。 这会加快垃圾收集器的速度，同时减慢辅助GC（mutator）的速度。

当所有这些都完成后，Pacer将从这个 GC 周期以及之前的 GC 周期和项目中学到什么时候开始下一个 GC。

它所做的远不止这些，但这些是最基本的方法。

数学绝对是令人着迷的，能够给我提供设计文档。 如果你正在做一个并发 GC，那么你真的应该看看这个算法，看看它是否与你的算法相同。

*[Go 1.5 concurrent garbage collector pacing](https://golang.org/s/go15gcpacing) and [Proposal: Separate soft and hard heap size goal](https://github.com/golang/proposal/blob/master/design/14951-soft-heap-limit.md)

![](https://blog.golang.org/ismmkeynote/image40.png)
是的，所以我们取得了很多成功。 如果是年轻一点的疯子，Rick 可能会拿出一些这样的图表，并把它们纹在我的肩膀上，我真为它们感到骄傲。

![](https://blog.golang.org/ismmkeynote/image20.png)
这是一系列为 Twitter 生产服务器绘制的图表。 当然，我们与生产服务器无关。  Brian Hatfield做了这些测量，并奇怪地在推特上发布了相关信息。

在 y 轴上，GC 延迟时间以毫秒为单位。 x 轴上是时间。 每个要点都是该 GC 期间的停止世界暂停时间(stw)。

在我们2015年8月的第一个版本中，我们看到**从300-400毫秒下降到30或40毫秒**。 这很好，数量级很好。

我们将从根本上将 y 轴从0毫秒到400毫秒，减少到0到50毫秒。

![](https://blog.golang.org/ismmkeynote/image54.png)
这是6个月后的情况。 这种改进很大程度上是由于系统地消除了我们在STW所做的所有 o (堆)事情。 这是我们从**40毫秒降低到4或5毫秒**的第二个数量级改进。

![](https://blog.golang.org/ismmkeynote/image1.png)
我们将再次改变 y 轴，这次减少到0到5毫秒。

![](https://blog.golang.org/ismmkeynote/image68.png)
在2016年8月，第一次发布的一年后。 我们再次不断地敲掉这些 o (堆大小)停止世界进程。 我们在这里谈论的是一个18g 的堆。 我们有更大的堆，当我们去掉这些 o (堆大小) stop the world 暂停时，堆的大小显然可以增长相当大，而不会影响延迟。 所以这对1.7有点帮助。

![](https://blog.golang.org/ismmkeynote/image58.png)
下一次发布是在2017年3月。 我们的最后一次大延迟下降是因为我们想出了如何避免在 GC 周期结束时避免对stw 栈扫描。 这使我们进入了亚毫秒级的范围。 同样的，y 轴将会改变为**1.5毫秒**，我们可以看到第三个数量级的改善。

![](https://blog.golang.org/ismmkeynote/image45.png)
2017年8月发布的版本几乎没有改进。 我们知道是什么导致了剩下的停顿。 这里的 SLO whisper number大约是100-200微秒，我们将向这个方向推进。 如果你看到超过几百微秒的东西，那么我们真的很想和你谈谈，看看它是否适合我们已知的东西，或者是否是我们没有研究过的新东西。 在任何情况下，似乎都不需要更低的延迟。 值得注意的是，这些延迟水平可能出于各种各样的非 gc 原因，正如俗话所说:"你不需要比熊快，你只需要比你旁边的家伙快就行了。"

2月18日1.10发布的版本中没有实质性的变化，只是一些清理和追踪的案件。

![](https://blog.golang.org/ismmkeynote/image6.png)
这里可以看到我们2018年的SLO。

我们把CPU的总数降低到GC时的CPU总数。
堆仍然是两倍。

我们现在的目标是每个 GC 周期停止世界暂停**500微秒**。 也许这里有点沙袋（sandbagging ）。

分配将继续与 GC 协助（GC assists）成比例。

Pacer 已经变得更好了，所以我们希望看到最小的 GC协助在一个稳定的状态。

我们对此很满意。 同样，这也不是 SLA，而是一个 SLO，因此它是一个目标，而不是一个协议，因为我们不能控制操作系统这样的东西。

![](https://blog.golang.org/ismmkeynote/image64.png)
这是好东西。 让我们转移话题，开始谈论我们的失败。 这些是我们的伤疤; 它们有点像纹身，每个人都有。 无论如何，他们有更好的故事，所以让我们讲一些这样的故事。

![](https://blog.golang.org/ismmkeynote/image46.png)
我们的第一个尝试是做一个叫做**面向请求的收集器**(request oriented collector)或者 **ROC**。 这里可以看到这个假设。

![](https://blog.golang.org/ismmkeynote/image34.png)
那么这意味着什么呢？

Goroutine 是看起来像 Gophers 的轻量级线程，所以这里我们有两个 Goroutines。 它们共享一些东西，比如中间的两个蓝色物体。 他们有自己的私人堆栈和自己的私人对象。 假设左边的家伙想要分享这个绿色对象。

![](https://blog.golang.org/ismmkeynote/image9.png)
左边的goroutine 把它放在共享区域，这样其他 Goroutine 就可以进入。 它们可以将其挂接到共享堆中的某个内容，或者将其分配给一个全局变量，其他 Goroutine 就可以看到它。

![](https://blog.golang.org/ismmkeynote/image26.png)
最后，左边的 Goroutine 走向了死亡的边缘，它走向了死亡,悲伤。

![](https://blog.golang.org/ismmkeynote/image14.png)
正如你所知道的，当你死的时候，你不能带走你的物品。 你也不能拿走你的那一叠。 此时堆栈实际上是空的，对象是不可访问的，因此您可以简单地回收它们。

![](https://blog.golang.org/ismmkeynote/image2.png)
这里重要的是，**所有操作都是本地的，不需要任何全局同步**。 这与我们 GC 的方法有着根本的不同，**我们希望从不需要同步中得到的伸缩性足以让我们取得胜利**。

![](https://blog.golang.org/ismmkeynote/image27.png)
这个系统的另一个问题是，写屏障总是处于打开状态。 无论何时发生写操作，我们都必须查看它是否将指向私有对象的指针写入公共对象。 如果是这样的话，我们必须将引用对象设置为 public，然后对可访问的对象进行传递遍历，以确保它们也是 public 的。 这是一个相当昂贵的写屏障，可能会导致许多缓存失败。

![](https://blog.golang.org/ismmkeynote/image30.png)
也就是说，我们取得了一些不错的成功。

这是一个端到端 RPC 基准测试。 标记错误的 y 轴从0毫秒到5毫秒(越小越好) ，反正就是这样。 X 轴基本上是镇流器(ballast )或核心数据库是多大。

正如你可以看到的，如果你有 ROC，并且没有很多的分享，事实上，规模相当不错。 如果没有 ROC 的话就没那么好了。

![](https://blog.golang.org/ismmkeynote/image35.png)
但这还不够好，我们还必须确保 ROC 不会放慢该系统其他部分的速度。 在这一点上有很多关于我们的编译器的问题，我们不能放慢我们的编译器。 不幸的是，编译器正是 ROC 做得不好的程序。 我们看到了30% 、40% 、50% 以及更多的速度放缓，这是不可接受的。 对其编译器的速度感到自豪，所以我们不能降低编译器的速度，当然不能降低这么多。

![](https://blog.golang.org/ismmkeynote/image61.png)
然后我们去看了一些其他的程序。 这些是我们的性能基准。 我们有一个包含200或300个基准测试的语料库，编译人员认为这些基准测试对他们的工作和改进非常重要。 这些根本不是 GC 人员选择的。 这些数字都很糟糕，ROC 不会成为赢家。

![](https://blog.golang.org/ismmkeynote/image44.png)
这是真的，我们规模，但我们只有4到12个硬件线程系统，所以我们不能克服写障碍带来的弊端。 也许在未来，当我们拥有128个核心系统并且Go正在利用它们的时候，ROC 的缩放特性可能会是一个胜利。 当这种情况发生时，我们可能会回过头来重新考虑这个问题，但就目前而言，ROC 是一个失败的命题。

![](https://blog.golang.org/ismmkeynote/image66.png)
那么我们接下来要做什么呢？ 让我们试试**分代 GC**(generational GC.)。 这是一首老歌，但也是一首好歌。 ROC 没有成功，所以让我们回到我们有更多经验的东西。

![](https://blog.golang.org/ismmkeynote/image41.png)
我们不会放弃我们的延迟，我们不会放弃我们不动的事实。 所以我们需要一个不移动的分代 GC。

![](https://blog.golang.org/ismmkeynote/image27.png)
那么，我们可以这样做吗？ 是的，但是对于一个年代久远的 GC，写入的障碍总是存在的。 当 GC 循环运行时，我们使用今天使用的写屏障，但当 GC 关闭时，我们使用一个快速的 GC 写屏障，缓冲指针，然后当指针溢出时将缓冲区刷新到一个卡片标记表。

![](https://blog.golang.org/ismmkeynote/image4.png)
那么在没有移动的情况下，这是如何工作的呢？ 这是标记 / 分配图。 基本上你保持一个当前的指针。 当你分配的时候，你寻找下一个零，当你找到那个零的时候，你在那个空间中分配了一个对象。

![](https://blog.golang.org/ismmkeynote/image51.png)
然后将当前指针更新为下一个0。

![](https://blog.golang.org/ismmkeynote/image17.png)
您继续执行，直到某个时候需要执行生成 GC。 您将注意到，如果在标记 / 分配向量中有一个，那么该对象在上一个 GC 中是活的，因此它是成熟的。 如果它是零，你达到它，那么你知道它是年轻的。

![](https://blog.golang.org/ismmkeynote/image53.png)
那么，你们是如何推广。 如果你发现标有1的东西指向标有0的东西，那么你可以简单地将0设置为1来提升参照物（referent）。

![](https://blog.golang.org/ismmkeynote/image49.png)
您必须进行传递性遍历，以确保所有可访问的对象都得到了提升。

![](https://blog.golang.org/ismmkeynote/image69.png)
当提升了所有可访问的对象时，次要的 GC 将终止。

![](https://blog.golang.org/ismmkeynote/image62.png)
最后，要完成分代 GC 循环，只需将当前指针设置回向量的开始位置，就可以继续了。 在 GC 循环期间没有达到所有的零，因此是空闲的，并且可以重用。 正如你们中的许多人所知道的，这被称为粘性位('sticky bits')，是由Hans Boehm和他的同事们发明的。

![](https://blog.golang.org/ismmkeynote/image21.png)
那么性能是什么样的呢？ 对于那些大的堆来说还不错。 这些是 GC 应该做好的基准测试。 这一切都很好。

![](https://blog.golang.org/ismmkeynote/image65.png)
然后我们在我们的性能基准上运行了它，结果事情并不顺利。 那么，到底发生了什么？

![](https://blog.golang.org/ismmkeynote/image43.png)
写屏障很快，但就是不够快。 此外，它很难优化。 例如，如**果在分配对象和下一个安全点之间存在初始化写操作，则可能发生写屏障省略**。 但是我们不得不转移到一个系统，在每条指令上都有一个 GC 安全点，所以真的没有任何可以省略的写屏障。

![](https://blog.golang.org/ismmkeynote/image47.png)
我们还进行了逃跑分析，结果越来越好。 还记得我们谈论的值传导的东西吗？ 我们传递的不是一个指向函数的指针，而是实际的值。 因为我们传递的是一个值，所以逸出分析只需要进行过程内逸出分析，而不需要进行过程间分析。

当然，在指向本地对象的指针逃逸的情况下，对象将被分配为堆。

**这并不是说Go的分代假设不正确，只是年轻的对象在堆栈上生存和死亡都很年轻。 其结果是，分代采集的效率比其他托管运行时语言低得多**。

![](https://blog.golang.org/ismmkeynote/image10.png)
所以这些抵抗写屏障的力量开始聚集。 今天，我们的编译器比2014年好多了。 逃逸 分析提取了大量这些对象并把它们放到了分代收集器可能会帮助的堆栈对象上。我们开始创建工具来帮助我们的用户找到逃逸的对象，如果它是小的，他们可以对代码进行更改，并帮助编译器在堆栈上分配。

用户越来越聪明地接受值传导的方法，指针的数量也在减少。 **数组和映射保存值，而不是指向结构体的指针**（在写屏障的角度来看？TODO）。 一切都很好。

这并不是为什么Go中的写屏障在未来会有一场艰难的战斗的主要原因。

![](https://blog.golang.org/ismmkeynote/image8.png)
让我们看看这个图表。 这只是一张标记成本的分析图。 每一行代表一个可能有标记开销的不同应用程序。 比如说你的标记成本是20% ，这是相当高的，但这是有可能的。 红线是10% ，仍然很高。 较低的线是5% ，这大概是现在一个写屏障的成本。 那么如果堆大小增加一倍会发生什么呢？ 这就是右边的那个点。 **由于 GC 循环不那么频繁，标记阶段的累计成本大大降低。 写屏障成本是恒定的，因此增加堆大小的成本将使标记成本低于写屏障成本。**

![](https://blog.golang.org/ismmkeynote/image39.png)
这里有一个更常见的写屏障成本，为4% ，我们看到，即使这样，我们可以通过简单地增加堆大小，将标记屏障的成本降低到低于写屏障的成本。
**分代 GC 的真正价值在于，在查看 GC 时间时，写屏障成本被忽略**，因为他们被涂抹在辅助GC（mutator）上。 这是分代 GC 的巨大优势，它大大减少了完整 GC 周期的较长的 STW 时间，但不一定提高吞吐量。Go没有这个STW的问题，所以它必须更仔细地研究吞吐量问题，这就是我们所做的。

![](https://blog.golang.org/ismmkeynote/image23.png)
这是一个很大的失败，这样的失败带来了食物和午餐(什么梗？？？)。 我像往常一样发牢骚:"如果没有写障碍，这不是很棒吗?"

与此同时，Austin刚刚花了一个小时与谷歌的 HW GC 人员交谈，他说我们应该与他们交谈，并试图找出如何获得 HW GC 的支持，这可能会有所帮助。 然后我开始讲述关于零填充缓存线路、可重新启动的原子序列以及其他一些在我为一家大型硬件公司工作时无法实现的事情。 当然，我们把一些东西放进了一种叫做 Itanium 的芯片中，但是我们不能把它们放进今天更流行的芯片中。 所以这个故事的寓意很简单，就是用我们现有的HW。

不管怎样，这让我们开始交谈，那些疯狂的事情呢？

![](https://blog.golang.org/ismmkeynote/image25.png)
没有写屏障的卡片标记怎么样？ 原来Austin 有这些文件，他把所有他的疯狂想法都写进了这些文件，不知道为什么他没有告诉我。 我认为这是某种治疗的东西。 我曾经对Eliot做过同样的事情。 新的想法很容易被粉碎，在你让它们走向世界之前，你需要保护它们并使它们变得更加强大。 不管怎样，他提出了这个想法。

这个想法是，在每张卡片中维护一个成熟指针的哈希。 如果指针被写入卡片中，哈希值将发生变化，卡片将被认为是标记的。 这将把写屏障的成本换成哈希的成本。

![](https://blog.golang.org/ismmkeynote/image31.png)
但更重要的是，它与硬件结合。

今天的现代体系结构有 AES (高级加密标准)指令。 其中一条指令可以执行加密级别的哈希操作，如果我们也遵循标准的加密策略，那么使用加密级别的哈希操作，我们就不必担心冲突。 所以哈希不会花费我们太多，但是我们必须加载我们要哈希的内容。 幸运的是，我们依次遍历了内存，因此获得了非常好的内存和缓存性能。 如果你有一个 DIMM，你按顺序地址，那么这是一个胜利，因为他们将比按随机地址更快。 硬件预取程序将发挥作用，这也将有所帮助。 无论如何，我们有50年，60年的时间设计硬件来运行 Fortran，运行 c，以及运行 SPECint 基准测试。 毫不奇怪，结果就是硬件能够快速运行这种东西。

![](https://blog.golang.org/ismmkeynote/image12.png)
我们进行了测量。 结果是相当好的。 这个基准测试测试的是较大的堆，结果应该是好的。

![](https://blog.golang.org/ismmkeynote/image18.png)
然后我们说，性能基准看起来像什么？ 不太好，有几个异常值。 但是现在我们已经将写屏障从 mutator 中始终处于打开状态移动到作为 GC 循环的一部分运行。 现在，关于是否要进行分代 GC 的决定被推迟到 GC 周期的开始。 我们有更多的控制，因为我们已经对卡片工作进行了本地化。 现在我们已经有了工具，我们可以把它移交给 Pacer，它可以很好地动态地切除掉那些偏向右边但不受益于分代 GC 的程序。 但是，这种情况会在未来取得成功吗？ 我们必须知道或者至少考虑一下未来的硬件将会是什么样子。
![](https://blog.golang.org/ismmkeynote/image52.png)
未来的内存是什么样的

![](https://blog.golang.org/ismmkeynote/image11.png)
让我们看看这个图表。 这是经典的摩尔定律图表。 在 y 轴上有一个对数刻度，显示单个芯片中晶体管的数量。 X 轴是1971年至2016年之间的年份。 我要指出的是，在这些年里，有人预料摩尔定律已经失效。

Dennard 标度在大约十年前就结束了频率的提高。 新的流程需要更长的时间来适应。 因此，他们现在不是2年，而是4年或更长。 所以很明显，我们正在进入一个**摩尔定律变慢**的时代。

让我们看看红圈里的芯片。 这些芯片是最能支持摩尔定律的。

这些芯片的逻辑越来越简单，而且被复制了很多次。 许多相同的内核、多个内存控制器和缓存、 gpu、 tpu 等等。

随着我们继续简化和增加复制，我们最终得到了几根电线、一个晶体管和一个电容器。 换句话说，DRAM 存储单元。

换句话说，我们认为双倍内存比双倍芯片更有价值。

[一个关于摩尔定律未来的图](http://www.kurzweilai.net/ask-ray-the-future-of-moores-law)

![](https://blog.golang.org/ismmkeynote/image57.png)
让我们看看另一张关注 DRAM 的图表。 这些数字来自卡内基梅隆大学最近的一篇博士论文。 如果我们看这个，我们可以看到摩尔定律是蓝线。 红线是capacity ，它似乎遵循摩尔定律。 奇怪的是，我看到一个图表，可以一直追溯到1939年，当时我们正在使用drum 内存，capacity 和摩尔定律同时发生，所以这个图表已经持续了很长时间，肯定比这个房间里的任何人在世的时间都要长。

如果我们把这个图和 CPU 频率或者各种摩尔定律已死的图进行比较，我们得出的结论是，内存，或者至少芯片容量，将遵循摩尔定律的时间比 CPU 更长。 带宽，也就是黄线，不仅与存储器的频率有关，而且还与芯片上的引脚数量有关，所以它跟不上芯片的速度，但也不算太差。

延迟，绿色线，做得非常糟糕，虽然我会注意到，延迟为顺序访问比随机访问更好。

2017年5月，美国宾夕法尼亚州匹兹堡卡内基梅隆大学，电子与计算机工程学博士 Kevin k. Chang m.s. ，电子与计算机工程卡内基梅隆大学，电子与计算机工程卡内基梅隆大学。 参见 Kevin k. Chang 的论文。 在前言中的原始图表不是我可以很容易地在上面画一条摩尔定律线的形式，所以我改变了 x 轴，使它更加统一。)

![](https://blog.golang.org/ismmkeynote/image15.png)
让我们去那个橡皮碰上路的地方吧。 这是 DRAM 的实际定价，从2005年到2016年，它的价格普遍下降。 我选择2005年，因为那是Dennard 缩放结束的时候，伴随着它的是频率的提高。

如果你看看这个红色的圆圈，基本上就是我们为减少 Go 的 GC 延迟所做的工作，我们可以看到前几年的价格表现不错。 最近，情况就不那么好了，因为需求已经超过了供给，导致了过去两年的价格上涨。 当然，晶体管还没有变得更大，在某些情况下芯片的容量增加了，所以这是由市场力量驱动的。 Rambus 和其他芯片制造商表示，我们将看到我们的下一个进程缩小，在未来的2019-2020年。

除了指出定价是循环的，长期供应有满足需求的趋势之外，我不会去猜测存储行业的全球市场力量。

从长远来看，我们相信内存的价格会以比 CPU 更快的速度下降。

![](https://blog.golang.org/ismmkeynote/image37.png)
让我们看看另一条线。 要是我们在这条线上就好了。 这是固态硬盘线路。 它在保持低价方面做得更好。 这些芯片的物理原理要比 DRAM 复杂得多。 逻辑更加复杂，不是每个单元一个晶体管，而是半打左右。

展望未来，DRAM 和固态硬盘之间有一条线，在那里 NVRAM，如英特尔的3D XPoint 和相变存储器(PCM)将生存。 在接下来的十年里，增加这种类型的内存的可用性可能会变得更加主流，这只会强化这样一种观点，即增加内存是为我们的服务器增加价值的廉价方式。

更重要的是，我们可以期待看到 DRAM 的其他竞争对手。 我不会假装知道哪一个会在五年或十年内受到青睐，但竞争将是激烈的，堆内存将更接近这里突出显示的蓝色 SSD 线。

所有这些加强了我们的决定，避免永远在线的屏障，而是增加内存。

![](https://blog.golang.org/ismmkeynote/image16.png)
那么这一切对于Go的发展意味着什么呢？

![](https://blog.golang.org/ismmkeynote/image42.png)

我们打算使运行时更灵活和健壮，因为我们看到来自我们的用户的角落情况。 我们的希望是缩小调度器，获得更好的确定性和公平性，但是我们不想牺牲任何性能。

我们也不打算增加 GC API 表面。 我们现在已经有将近十年了，我们有两个旋钮，感觉很对。 没有一个应用程序重要到足以让我们添加一个新标志。

我们还将研究如何改进我们已经相当不错的逃逸分析和优化Go的值传导的编程。 不仅在编程方面，而且在我们为用户提供的工具方面。

在算法上，我们将关注设计空间中最小化屏障使用的部分，特别是那些一直打开的障碍。

最后，也是最重要的，我们希望在接下来的5年里，甚至在接下来的10年里，能够利用摩尔定律中内存优于 CPU 的趋势。

So that's it. Thank you