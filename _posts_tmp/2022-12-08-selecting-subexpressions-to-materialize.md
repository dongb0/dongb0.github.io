---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - paper
---

abstract

在数据分析集群中，观察到许多任务之间存在重复计算的情况，这会对计算资源造成浪费，增加任务的执行时间；因而发现并处理这些重复计算的现象，是operator和user都关心的问题。但是现有方法并不支持（如此大规模）的workload。

本文将针对 large workload 中的 subexpression 选择问题，通过将任务的 common part 物化来提高任务的执行速度。文中将这一过程描述为 ILP 问题，并将其映射为二分图的标记问题。然后提出 BigSub算法，以顶点为中心的迭代式选择 subexpression 物化，并在微软的 SCOPE 中提供了分布式的实现，在实际系统中最多能节省40%的运行时间。

1 Introduction
现状是，分析型数据中心里，常常发现有大量（文中统计数据是45%）的任务存在相似性，（不过前提是数据中心规模比较大，数千用户每天会提交数十万个分析任务）；

因此考虑一系列优化措施：view selection、multi-query optimization、subexpression reusing；这些优化的相似之处在于：给定一组查询，寻找其中的相似部分，在给定的约束条件下实现最小化代价函数的目标；

但以往许多优化算法，应用场景（benchmark）中的查询数量为数十到数百个查询，这比 bigsub 要求的用户查询数量小至少3个数量级；

也有一些面向大规模集群的方法，如 nectar 
文献\[18] online caching 方面考虑实现 reusing（是否要去看一眼？
依靠 heuristic 来选择 subexpression 从而进行物化，虽然这样的方式不会受限于集群规模，但是依赖局部优化决策的方式也导致整体性能里全局最优解相去甚远；

因此需要全局优化的策略；这里提出 BigSub，主要关注的是 subexpression selection
周期性的对数千条查询构成的整体 workload 进行分析，选择其中可靠的 subexpression 物化；（因此可以算作是静态的

将 subexpression selection 映射为二分图标记问题；顶点表示查询和候选的 subexpression，边连接 query 和它的 subexression；

然后再划分为两个子问题：
1）标记 subexpresison vertex，表示它将被物化
2）标记 edge，表示物化的 subexpression 将被 query 使用；

同时为了将算法扩展到预期的规模，使用了一个 vertex-centric graph modle，迭代式的重复上述标记过程直到算法收敛

contribution：
*
*
*
* 提出减少 possible solution space 的方法，并且不会降低 solution 的质量；
* 在生产环境 workload 中实验表明，1）支持上万查询规模；2）优于启发式算法3个数量级；3）节省大约10～40%的运行时间

2 Problem Formulation

场景是：workload 中有些查询会周期性出现（monthly/daily）；考虑到 workload 相对 static，这里采用定期 offline 选择 subexpression 的方式；

2.1 Problem Statement

Subexpression Selection
目标与视图选择一样，都是从候选集合 S 中选出能够最小化 代价函数的 集合 s；并且要满足约束条件 C；

Definition 1（candidate subexpression）：假设有查询 q，称其对应的逻辑计划树结构为 t，t的任意子树 t‘ 都是 q 的候选 subexpression；

Candidate Subexpression Enumeration
选出所有可能的候选subexpression，用于后续的选择算法； complete subexpression strategy 会选择“所有”可能的subexpr，但这里只采用了优化器输出的逻辑计划；原因如下：1）更少intrusive，侵入性，也就是更容易实现；2）减少搜索空间；3）复用现有的plan signature 快速寻找 等价subexpr；4）利用之前的查询运行的统计数据

Utility of Subexpression
与视图的 utility 定义类似，同样是使用 subexpr 之后查询 q 减少的 cost 定义为 utility（要算上访问 subexpr 的开销 Sj，这里简单定为扫描 s 的开销）

Definition 2（Utility of Subexpression）：如上所述，此略

给定一个 subexpr S，查询 Q 可能存在多个重写查询 R 可以使用 S；因此还将 utility 规定为这些重写的 R 中能够带来最大开销缩减的值；这就要求考虑 rewriting 才能选择出适合的 subexpr 来物化；

Definition 3（Utility of Subexpreesion Set）：
单个查询 qi 在 S 上的 utility 就是
// 原文这里的公式我觉得应该是 MAX 才对啊？

最后 workload Q 的 utility 是所有查询 qi utility 之和；// 这个好理解

Subexpression interaction
要能够避免创建重复的（重叠的）subexpression；

Deifinition 4（Interacting Subexression）：
如果两个 subexpr s1，s2 的逻辑计划中，有一个为另一个的子树，那么我们说 s1，s2 是 interacting（有重叠的。

为了能够分析 interaction，这里定义了 interaction matrix，是一个m*m对称矩阵，m是candidate数量；如果第 i 个 subexpr 与第 k 个 subexpr 重叠则把 X_ij 设为 1； 

Subexpression Cost and constaints
文中场景的文件系统是 append only，不考虑更新，所以需要考虑的约束只有存储限制；

Utility-based subexpression selection：
反正就是用了后面定义的 utility 和 cost，重写一些代价函数的公式

Query rewriting using subexpr
选出 subepxr 之后，就可以将这些物化的 subexpr 提供给 optimizer 用于优化查询；优化器将根据需要对查询进行重写（可是怎么重写呢QAQ


2.2 Subexpression Selection as an ILP
接下来介绍如何模型化为 ILP 问题。设 Zj 为一个 0-1 变量，表示第 j 个 subexpr 是否被选择；考虑 budegt 为 B^max，则又可以重写代价函数为：（此略

还引入一个新的变量 Y_ij 使问题变为线性的；

但这个问题对于目前 STOA solver，在文中应用场景数百万数量级查询的情况下也难以解决，因此提出一种近似算法来使其能应用在大规模 workload 场景。

3 Scaling Subexpression Selection

3.1 Bipartite Graph Labeling Problem
可以将该 ILP 问题划分为多个子问题（子图？），根据前文给出的定义，现在可以把这个二分图标记问题表示为以下的 labeling problem：
1）在存储空间约束内，将0-1标签 z_j 赋值给每一个 Vertex // z_j 表示是否物化
2）将0-1标签 y_ij 赋给每个 edge e_ij （满足不重叠的约束和未物化不能被选择的约束，同时标记的 y_ij 要能最大化 utility） y_ij 表示查询 Qi是否使用 subexpr j

最后得到能使整个 workload utility 最大化的 z_j 的值，就是 ILP 问题的全局最优解；

算法的选择过程：先概率性翻转某个 z_i，然后根据翻转后的 z 集合更新对应的 edge label；
// 这里的概率如何得到？

好处是将寻找需要物化的视图 和 使用哪些视图回应查询 Q，分隔成两个问题；每个子问题的复杂度也降低到能够计算的程度，还可以并行化；

3.2 The BigSub Algorithm
这一节介绍近似算法 BigSub；

3.2.1 Overview
算法主要是 iterative approach，每次迭代包含两步：
1）将标签赋给 subexpr
2）根据步骤1）确定的标签，对edges进行标记；
重复这一过程直到所有V/E的标签不再改变（某种收敛）或者达到最大迭代次数

BigSub 的输入是 bipartitie graph G（包含n个query和m个候选subexpr），utility u_ij，interaction x_ij, storage footprint(大概是每个 subexpr 的存储开销) bj, 最大迭代次数 k

输出是将被物化的subexpr集合 Z_j；这里先介绍算法的基础版本，优化留到下一章节；

受限label初始化方式是：vertex随机赋值0或1，edge全是label 0；然后创建几个辅助变量这里先不提；

在决定label的部分采用 probolistic labeling 方法，这样可以避免依赖集中式协调者的决策；
// 坏了，那还怎么强化学习？
根据 current utility 和 used budget 来计算翻转 vertex label 的概率；//看看伪代码具体怎么算的？

3.2.2 Labeling Subexpression Vertices
目标：将utility最高的 subexpr 打上标签1（在存储约束下）；

这里采用概率性 flip 方式的主要目的也是为了能够分布式并行化的运行算法；
计算概率的方式为：
P_flip = P1 * P2;

P1 = 1 - min{B_cur, B_max} / max{B_cur, B_max};

P2 = 1 - Uj_cur / U_cur if z_j = 1
    else if ???///有些没搞懂这个的计算方式
    otherwise 0  

主要思路是：1）剩余存储的 budget 越多，翻转的概率越高；
2）当前已选择（标记为label-1）的subexpr对应 utility 越高，那么翻转它的概率越低；对于 label-0 的 subexpr 则相反（utility 越高翻转概率越高，公式中看不懂的第二项）；

这样设计概率翻转的考虑主要是为了平衡收敛速度和避免陷入局部最优解的概率；

3.2.3 Labeling Edges
这一步我们要用上一节选出的 vertex，来判断选择哪些subexpr（通过将相应的label标记为1表示被选中
添加几个显而易见的约束：我们不能用某个subexpr sj来优化某个获得utility为 0 的查询，或者不能选择没有被物化的 subexpr；

添加两个约束集合 
Mi={j:u_ij > 0}, utility 大于 0 的集合，是静态的已知的
M'i={j:z_j > 0}，选择的subexpr集合，在每次迭代过程第一步中确定
然后公式中需要添加在这俩约束考虑，并且维护 Mi 并 M‘i 的一个向量和一个矩阵

考虑这俩约束之后，问题的规模进一步减小，计算量小到可以在本地完成；

3.2.3 Analysis

Complexity
将一个全局的ILP问题拆分为 n 个 ILP 子问题，复杂度有所减小
最坏情况复杂度为 k*(m + n * 2^max(di)) di是查询节点的度数，比如 Mi 并 M‘i ；
m是查询的数量// 是这个吗？不太确定啊？；k是迭代的次数；
di的上界由查询的数量和每个查询允许使用的 subexpr 数量确定，所以可以说算法的复杂度基本还是与查询数量和subexpr的数量之和成线性关系的；
// 可以这么说吗？我艹我数学真的垃圾搞不懂；

Correctness
正确性的证明就懒得细细研究了，说结论：
首先关于存储限制的约束，在算法运行过程中因为 flip 的存在是可能超过限制的，但是随着迭代次数的增加，这种错误很快就减少到接近0的程度（大约10次，且下降得很快

第二项关于 subexpr interaction 的限制，// 不太看得懂，反正最后还通过构造 M‘ 来满足这一约束；

Convergence
因为前面提到一个加快收敛的机制：在迭代轮次达到某个数量之后（比如80%），开始进入stricter iteration，此后只将 subexpr 的 label 从0翻为1；（而不会反向flip）
这种情况下 U_cur 会收敛，所以只需要考虑另一种从 1 翻为 0 的收敛情况；


算法并不会在全局最优停下，因此最终也有可能结束于局部最优解；这与爬山算法存在的问题类似；

3.3 Distributed Execution
这里分布式执行的方式是某种 vertex-centric model，每个节点赋予自己或者邻边一个label（根据节点本身是query还是subexpr），然后把这个赋予的label封装在消息里发送给其他邻居；

以及这部分并没有提太多东西；（或者是说了但是我没看懂

4 Optimization
优化措施主要面向local ILP；

4.1 Branch-and-bound
算法中 line 24 的 localILP 函数，一种简单的实现思路是遍历 candidate subexpr 的所有组合，并从中选出 utility 最大的一个；显然这不太现实；

另一种方法是使用通用的 ILP solver；但文中认为通用的solver不够合适；因此采用标题的 branch-and-bound 方法，思想是：如果访问到某个 interacting 的 subexpr，那么不需要再访问它的超集（superset；

如 S1 interacts S2,那么访问了 subexpr {S1, S2} 之后，我们就找到了一个边界，不需要再访问 {S1, S2, S3}，这在interact subexp 多的情况下能带来很大的优化；

算法输入是 subexpr 的utility vector和interaction 矩阵 x_ij；其他细节暂时没看懂；

4.2 Exmploration Strategies
上面介绍的访问模式是 bottom-up，也可以使用 top-down 的方式；
如从 {s1,s2,s3}开始，逐个移除 subexpr，直到不存在interacting subexpr为止；在 subexpr 数量少的情况下比较快速；

另外，因为文中使用的 utility 函数是线性的，所以 top-down 可以在找到 utility 低于当前 subexpr 组合的情况下停止迭代；

文中的实现将根据情况选择这两种访问方式（如，根据interacting matrix X中 非零值 的数量；

4.3 Skipping Trivial States
不与其他 subexpr 重叠的 candidate 最后肯定会被选作 solution 的一部分（因为utilitiy 是递增的）；

提前选择这部分 subexpr 能有效减少 search space

// 一个问题是，有没有可能这样选出来的 subexp 会因为 storage 等约束而不在 solution 中？

最后，可以跳过那些 上次 vertex labeling 过程中 subexpr 没有变化的 ILP 部分；
// 是否可以说它们已经局部最优，或者说已经计算完了？

5 Evaluation

6 Related Work
View Selection
在 data warehouse 场景下提出了不少 techniques： AND/OR graph（最老的、模型化为 state optimization 文献\[46]、和 data cube lattice（1996 文献\[22]

Subexpression selection
这算是 view selection 的子问题，view selection 可以选择那些没有物理计划对应的逻辑计划；但同样的 view selection 的 search space 就会变得很大；而扩展性是本文的主要考虑因素，因此选择的是 subexpression。
文献 \[44] 考虑云上执行环境中，同一个任务脚本里的 common subexpression；

Multi-query Optimization
MQO 与 view selection 类似，区别在与 MQO 只是执行过程中暂时的物化查询视图； 文献\[42] 使用 AND/OR DAG 在火山模型的优化器中完成 MQO 任务，但是没有考虑存储限制。
文献\[30]提供一种 subexpression 数量的平方 级别的渐进算法
文献\[6] 采用遗传算法
但这些算法并不能并行化；
在 mapreduce 场景中也有对 MQP 的研究
文献\[8,38,49]，但query的数量都很小（tens of

Reusing intermediate result
该问题研究重新利用那些已经被物化的中间执行结果 文献\[23,37] 
这对于大数据平台而言非常有效，因为：1）包含大量重复计算；2）优化用时相较于执行时长来说非常短；3）获得的性能提升很大
有 Restore、Nectar（引用见原文）和文献\[39]

Shared Workload Optimization
为一组query构造common query plan，使得它们能够共享

Large-scale Workload Analytics
vertex-centric programming model 在大规模图分析中十分常见。【若干个系统】



7 Conclusion


我的总结：为了扩展性（算法能应用在数万查询体量的分布式场景），舍弃了一部分精度，没有追求最优解而是选择了更容易计算得到的近似解法（从 subexpr 代替 view selection，以及？可以看出）；但比起传统启发式算法已经得到了很大的优化（虽然可能有数据渲染夸大的成分在）

也就是个概率性的迭代算法啊，分类应该属于 randomized 里与模拟退火、遗传算法、登山算法的同一类；

但总体而言是非常符合需求的工程创新优化