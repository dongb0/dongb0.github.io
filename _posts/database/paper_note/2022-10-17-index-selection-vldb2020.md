---
layout: post
title: "[db/thesis] 1 - index selection survey(vldb2020)"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - database
  - paper reading
  - index selection
---

> 这系列计划整理一下写毕业论文期间读文献记录的笔记，也算是对研究生生涯的一个交代。别看我挖坑的时间早，但是这些事其实是开始看论文的时间（而不是博客的实际写作时间。
> 
> 方向是基于强化学习的物化视图选择算法，其实相当于把一些RL/DRL方法套到数据库场景里，主要写的代码也就是py调强化学习算法调参，跟db内核的关联不是那么紧密。

这是我看的关于索引/视图选择问题第一篇文章，算是噩梦的开始吧。刚开始的时候文中很多术语（有RL的，也有该问题沿用历史文献的）指代的方法、出处搞不懂，按字面意思翻译出来也无法理解整句话的含义，非常影响对整体思路的把握。只能从头到尾一句句读，看看会不会在文中某个地方藏着我没注意的解释，最后干脆每段话都逐句翻译或归纳成一小段中文注释，所以读得很慢QAQ。

<!-- > 比如文献里常出现 what-if call, configuration,  -->

好在后来读了2000年前后的几篇引用比较多的论文之后，发现后面很多工作都沿用这里面术语和定义，对整个问题的理解也就慢慢清楚了一点。话虽如此，但对于什么是“动态负载”还是花了很长一段时间来消化，可能之后整理到相关的文献笔记时再做个归纳总结。

-------------

> KOSSMANN J, HALFPAP S, JANKRIFT M, et al. Magic mirror in my hand, which is the best in the land? an experimental evaluation of index selection algorithms[J]. Proceedings of the VLDB Endowment, 2020, 13(12): 2382-2395

在开始读论文之前首先明确目的。作为我看的第一篇该领域的文章，先得了解这一方向是要解决什么问题，一般会用什么方法。虽然实际看完的时候我还是一知半解，但是可以贴上我现在的理解。

> 索引选择问题是指，从某个候选索引集合中选出一个子集，该子集对应的索引创建后，能够使 Workload（即一组查询序列）的总开销尽可能小。
> 候选索引是根据 Workload 产生的，方法也简单，将所有输出的单个Column和过滤条件中出现的单个Column、以及这些Column构成的 size=2,3,4,...,n 个的新组合，就构成了我们需要的候选索引。可以想象这些组合的数量是非常庞大的（对于TPC-DS，只考虑单个索引候选的话数量大概在数百个，考虑到n=4的话则增加到大约130万），所以通常没法选择全部索引，只能选取某个子集，而我们知道一个size=n的集合，其子集有 $2^n$ 个，数量更加爆炸（想象一下如果要从size=130万的集合中找某个最优子集）。所以这也是索引/视图选择问题最大的难点。

> 而具体的应用场景，是作为DBA的辅助工具，实现自动创建和维护索引。目前有一些商业工具如微软的 AutoAdmin, IBM 的 DB2 等（既是工具名也是算法名），这些也包含在本文对比的算法中。

其次，这篇文章大致包含什么内容。

这是一篇关于索引选择传统算法的综述，主要对比了8种传统算法的性能，并开源了他们搭建的实验平台和对应算法实现，后续一些工作确实可以复用他们的框架来减少工作量。我通过他们的代码了解了一个python实现的索引选择算法如何结合到数据库的：索引选择算法不需要在数据库内核中实现，index candidate的生成、选择，都通过 python 实现；与数据库交互的部分只包括通过python的API连接数据库，用postgres的 `EXPLAIN` 命令来获取执行计划，以及postgres安装 `hypopg` 插件后，通过对应 `CREATE` 命令创建虚拟索引或实体索引，然后继续用 `EXPLAIN` 获取创建索引新的查询开销。

当然后续看文献了解到，有些索引/视图选择工作是将算法内嵌到优化器中的，这需要对内核代码进行修改，实现比较复杂，所以大部分工作还是用前面的方法。

> 另外打岔提一句，这篇论文的题目很有趣，*Magic mirror in my hand, which is the best in the land?An Experimental Evaluation of Index Selection Algorithms*，致敬白雪公主里的经典台词么这不是：*magic mirror on the wall, who is the fairest one of all*。

以上是看论文前的一些废话，下面正式开始阅读论文。

<!-- -------------
Q1：benefit-per-storage ratio 大概是怎么计算的？

Q2：图里的 relative workload cost 含义？标点的地方是意味着算法找到 best solution 了吗，如何验证它是 best 的呢？

Q3：如何对这些算法的具体步骤有更详细直观的认识？看原论文还是看代码？// 难道都看？

Q4：优化器如何估计查询执行的代价/开销？文章说精确计算索引带来 benefit 可以通过创建索引并实际执行的方法，但是开销大，那么实际通过什么方法来估计呢？是通过数据规模来估算使用索引后减少的IO次数吗？

索引选择要解决的问题：
在没有DBA指定索引的情况下，数据库通过索引选择算法，针对某些工作负载，自动创建索引，提高查询的执行效率；
------------- -->

Abstract 和 Introduction 的内容前面基本概括完了，我们直接看 Intro 最后的 Contribution 部分，看一眼文章的架构确定要读哪些部分：

- sec 2 formalize index selection problem
- sec 3 describe & analyze these algorithm
- sec 4 methodology
- sec 5 evaluation platform
- sec 6 experimental results
- sec 7 findings and insights obtained from experiments
- sec 8 conclusion

> 省流：我们下面主要归纳 sec 4，简单提一嘴 sec 6 里能用上的部分。

sec 2 问题定义其实前面也写了，sec 3 是对这8种算法的简要介绍，这里我们没打算看哪个具体的算法所以也跳过（因为发现看不懂，我在考虑想真正理解这些算法，要么去看原文，或者直接看代码 &nbsp; PS：虽然现在我也没理解这些个算法）。

直接来看 sec 4 吧。这一节的标题为 *METHODOLOGY*，汇总下面的小节标题和具体内容，感觉是在介绍实验设置、用什么测量方式、各个算法的constrain之类的，挺杂的。我们还是直接概括这一节里对我们有帮助的信息。

首先从sec 4.1我们知道本文用了 TPC-H, TPC-DS, JOB 三个 benchmark，前两个可以去 [TPC 网站][1]上下载工具来获取 sql 以及生成数据，JOB 则可以在 [github][2] 上找到 sql 文件，但是readme里导入数据的方法不好使了，导入一直失败（imdbpy这个包改版了），这里贴一份我找到的 [dump 文件][3]，用 `pg_restore` 导入数据库之后大概占11GB大小。

从sec 4.2我们知道衡量算法的效果（指获取query costs，包括带索引和不带索引的）可以通过 executing queries 或者获取优化器提供的 cost estimation（`EXPLAIN`命令提供的那个）。可以想见前者的运行时间会很长，比如我实验的时候，JOB 中许多SQL在2核8BG的云数据库中执行时间为1～5秒，少数需要10s～30s，而获取每条SQL的 cost estimation 通常只需要不到 500ms，体现到算法运行时间里可能就是前者需要20小时或者更多，后者需要5小时。

此外，创建索引的时间也很长，所以文中用了 postgres 的 hypopg 插件，来创建 hypothetical indexes 虚拟索引，我实际测下来单个虚拟索引的创建时间也是毫秒级。

> TODO: hypothetical index 的实现方法，应该是有对应论文的？ // 好吧没有，那么看官方博客还是源码？想知道具体原理 

sec 6 具体的实验结果我们就不看了，因为我们并不关心具体哪个算法比较快，只是想看看实验里有什么能借鉴的地方用到视图选择里去。来看 sec 6.5，这一小节提到算法运行的主要开销并不在于算法本身的计算逻辑，而在于向优化器请求 cost estimation 这一步骤。文中表格展示的数据现实，大部分算法运行过程中可能要发起 20～300 万个 cost request。由于 benchmark 和数据并不会改变，所以对于相同的 configuration，优化器评估得到的 cost 总是相同，因此可以用 configuration + sql id 做索引，来缓存 request。而缓存比例也能达到70～90%，能够极大加快算法运行速度。

// TODO：这里缓存具体的实现方式是什么？查一下代码，改一下

> 也就是说这些算法面向的 workload 还是静态的。

<!-- 7 Learning

General Insights
1）算法在不同工作负载和限制下，表现不同（根本废话）2）算法选择应根据用户对不同条件的要求 3）基于查询（query-based）的算法，会评估所有潜在索引；基于组合的算法（combination-based）倾向比较较小的索引集合，相对来说后者能更多的评估索引之间交互的影响；但前者更快 4）what-if calls 的开销并不固定，取决于查询和索引组合；
5）solution的粒度？
6）停止条件很关键（这种算什么finding？
7）根据 benefit per space 来对候选索引排序比直接根据 benefit 排序更高效
8）Reduction-based 方法在可用空间配额更大的场景下运行更快，但在实际应用场景他们会慢 -->

尽管当时翻译啊、自己的疑问啊之类的东西记了一大堆，但是现在回头看有用的其实也就上面这么点。

总结一下，通过这篇文章，我们了解了什么是索引选择问题，能大概搞清楚这个问题工程实现上需要我们做什么工作，然后能看到实验的设置，包括 benchmark、一些优化的技巧比如缓存what-if call、使用hypothetical index代替创建实体索引，甚至可以去github看源码。但是当时我不知道的是，虽然pg有插件支持 hypotheical index，但是并没有任何方式可以在视图选择中使用 hypothetical view...

那么如何解决这个问题，请看 // TODO：新博客连接


---------

The End

[1]: www.tpc.org
[2]: https://github.com/gregrahn/join-order-benchmark
[3]: https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/2QYZBT