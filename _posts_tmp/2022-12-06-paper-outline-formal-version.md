题目：面向大规模原生分布式数据库的视图选择算法
Title: View Selection for Large Scale Naive Distributed Database System


第1章 绪论

1.1 课题研究背景和意义

一串手写的数字、几个随机排列的字符、报纸上裁剪下来的一小块报道内容、几句口头转述的话语、一封泛黄的信笺，都包含着数据。但是这些数据是否有价值呢？这要取决于数据对应的信息对于接收和使用它们的人的意义。比如，这串数字是某个被遗忘在角落的账号的密码，这份报道是某人见义勇为事迹的嘉奖，这些转述的话诉说着对远方亲人的思念，这封信是第一次鼓起勇气递出情书后收到的回信，那么它们的意义将无比重大，它们的价值甚至无法用金钱来衡量。但是对大多数人而言，这样的数据所包含的信息，其实并没有太大价值。因为数据的价值是人们赋予它的；或者更准确的说，是由数据的使用者赋予的。对于无法从数据中解读出信息并加以利用的人来说，任何数据都只是一堆无意义符号组成的随机序列而已；但对于能够充分利用数据的人来说，数据的价值是难以估量的。

进入二十一世纪后，互联网的蓬勃发展使得全球无时无刻不在产生海量的数据，全世界产生的数据量在2012年首次超过1ZB。据IDC估计，2025年全世界产生的数据量将达到163ZB，这一趋势有力地推动了大数据处理和分析技术的发展：2003到2004年，谷歌研发的文件系统GFS[]和MapReduce编程模型[]，以及随后出现的开源产品Hadoop[]，掀起了“大数据时代”的热潮[]；而随后 2012 年 Hinton 等人在 ImageNet 图像分类任务中使用 RNN 取得重大突破[]，将机器学习重新带回人们的视野，让人类有机会尝试利用各种深度学习算法进一步从海量杂乱无章的数据中提取出更多的信息，创造更多的价值。

作为承载数据的软件容器，数据库系统从上世纪六十年代出现伊始发展至今，也从当初面向单个企业或组织、用于代替文件系统存储和管理数据的独立而功能单一的系统，发展为今天包括关系型数据库[]、包括键值数据库和图数据库等的NoSQL[]、分布式数据库[]、云原生数据库[]等在内的数据库大家族。数据库也不再是面向单个离散的企业或组织，而是通过网络与成千上万个节点或终端用户相连接，支撑着数千万、甚至上亿用户的应用服务。

随着互联网赋予数据的价值不断提高和业界对于数据的利用方式越来越丰富[]，通过数据仓库、Hadoop等大数据分析系统等进行的静态数据批量处理， 通过Strom/Spark/Flink等进行开源系统进行的流式数据处理，机器学习尤其是各类深度学习模型与大数据的基础上也能发掘出更多的数据价值。慢慢的，以往通过OLTP数据库系统进行事务处理、产生的数据经过ETL处理后导入OLAP系统进行数据分析这样的两套系统运行的方式，已经不足以满足如今数据分析任务实时性和用户体量迅速提升后的对系统扩展性、鲁棒性的要求了。

为了解决这一问题，业界开始整合OLTP和OLAP，寻求数据库的 HTAP 能力，即 Hybrid Transaction/Analysis Processing，在一套系统中同时提供事务处理和数据分析功能，在保证事务处理能力的同时，几乎无延迟的对最新数据进行分析[]。而这样的融合将会破坏原本解耦的事务处理和数据分析系统，也意味着传统的TP/AP技术必须针对新的应用场景作出调整。如TP系统的存储方式通常为行存，而AP系统更多采用列存的方式来获取更优的读取性能，因而HTAP系统要在两种存储方式之间权衡，做出取舍；物化视图也是传统数据仓库系统和主流OLTP系统中常见的一种通过持久化数据、减少重复计算开销的优化措施，尤其在数据仓库中，选择适合的视图集合进行物化，对整个数仓的性能有至关重要的影响。在新场景下，如何根据HTAP的工作负载特性，自动选择更高效的物化视图组合，也是该领域所面临的新挑战。

综合以上讨论，本文决定设计一种基于强化学习的分布式场景下视图选择方法，在分布式数据库Oceanbase开源版中实现算法原型，并与传统的启发式算法进行实验对比。


1.2 研究现状

1.2.1 生成和筛选候选视图
1.2.2 视图选择算法分类
视图选择算法可大致分为以下三个类别：

  1）静态选择算法
  静态选择算法需要预先获取完整的工作负载（即所有查询序列），大多数视图选择算法基本都归属于这一类别[2012 survey of view selection]。

  文献[1997 selection of a view]通过将视图和基础表构建成 AND/OR 图，转化为图的遍历搜索问题，并设计贪心算法选择每单位存储空间获得收益最大的视图，来实现多项式时间复杂度的视图选择，还将算法扩展到物化视图上存在索引的情况；但该工作只考虑了对视图空间占用的限制，对维护开销并没有做约束；

  文献[1998 maintenance cost]在[上一篇文献]的基础上，加入了视图的维护开销约束，同样通过贪心选择“得到增益与维护开销比值最高的视图”，来获取 OR view graph 中的近似最优解，同时还针对 AND-OR 设计了基于 A* 算法的启发算法，能够保证搜寻得到最优解；该工作的缺点也比较明显，首先 OR view graph 只是视图选择问题的一个子集，且最坏时间复杂度是指数级别，因而该贪心算法无法推广；另外，通过转化为 AND-OR 图之后虽然可以应用 A* 算法得到最优解，但是最坏情况下该算法可能退化为穷举算法，难以解决大规模视图的选择问题。

  2）动态选择算法
  动态视图选择算法可以工作在负载实时变动的场景下，因而算法需要根据当前负载变化的情况来对选择的视图进行调整，以便获取针对当前负载最优的视图组合。
  
  文献[1999 DynaMat]考虑到静态选择算法提供的视图选择可能会因工作负载的变化而失效，选择将适当的查询结果持久化以便重复利用，减少重复计算的开销。DynaMat通过额外的Fragment Locator来判断缓存的结果是否位于缓存池中；另外如果缓存池已满，DynaMat将使用LRU/LFU/SFF/SPF等缓存策略中的一种进行物化数据的替换。实验结果表明DynaMat能在视图的空间占用和维护时间窗口都受到限制的情况下进行动态视图选择，并且查询性能依然保持与静态选择算法相当的水平。

  文献[2007 dynamic materialized view]则针对工作负载访问数据存在偏斜的场景，提出有选择性的维护视图中一部分数据（通常选择访问频率较高的部分），加入Control Table来标记这些数据，并且根据负载的变化调整物化的部分。实验结果表明，在查询偏斜大于 90% 的情况下，对比物化完整视图，动态物化视图能够使用完整视图 5% 的存储空间达到相同的查询性能，并且维护开销远小于使用完整视图。

  3）基于机器学习的视图选择算法
  文献[2018 bigsub]

  文献[2020 automatic view generation]认为BigSub的迭代式选择过程为了保证能够收敛，需要禁止将已选择的子查询再转化为未选择，这可能会退化为贪心算法而得到较差的结果。因此他们使用马尔科夫决策和深度学习模型来进行视图选择，与传统的贪心算法和BigSub方法进行实验对比，在不同数据集上都获得了一定的性能提升。

  文献[2021 autoview]与BigSub类似，也将视图选择问题模型化为 ILP（Integer Linear Programming），但通过提取SQL对应物理计划，然后利用MVPP（Multi-View Processing Plan）来生成候选视图；在视图选择算法部分，他们采用了强化学习的DDQN模型，并且加入了Encoder-Reducer模块，在IMDB数据集上比BigSub和贪心算法表现更好。

以上文献的研究对象均为集中式的数据库和数据仓库，但随着信息量的增加、数据规模的增大，集中式系统开始无法满足应用的扩展性要求，研究人员开始探寻分布式场景下的视图选择工作。

文献[2003 view selection in distributed warehouse]，一种应用于分布式数据仓库的贪心视图选择算法

文献[2009 towards mv selection for distributed db]，提出一种面向物联网场景的分布式遗传算法来进行视图选择。

1.2.3 索引选择算法
数据库索引在作用、功能和结构上与物化视图有许多相似之处，在算法的设计上值得互相借鉴。

文献[2000 automated selection for mv & index]首次尝试将MV和Index看作同一类结构，同等的根据每单位存储空间带来的增益从物化视图或者索引中进行贪心选择。该工作还提出了一种筛选候选视图的方法，首先对查询语句进行句法分析来获取所有查询可能涉及的候选视图，然后使用贪心算法找出查询开销超过某个设定阈值的表的集合，称为 interesting table，再基于涉及 insteresting table 的查询来构造候选视图，来得到最终的候选视图集合。该工作的实验结果表明，将物化视图和索引共同进行选择，比起对视图和索引独立选择，能带来更好的性能。

文献[2022 AutoIndex]

不过综合以上工作，可以发现视图选择和索引选择在候选视图/索引的生成上还是有很大区别的。一些早期的工作因为数据量不大，采取的还是枚举式的生成方式；但这样产生的组合数量是指数级别的，启发式算法很难从中高效的选出解决方案。因此后来的研究开始寻找工作负载中查询的关联来生成候选索引/视图，有效的减少了算法需要搜索的空间，保证了算法性能。

1.2.4 代价估计模型

1）基于优化器的代价估计
2）基于深度学习方法的代价估计


1.3 本文主要内容

1）物化视图的设计与实现
2）候选视图生成和筛选方法
3）基于强化学习的视图选择算法设计与实现
4）实验和性能对比

----------------

第2章 相关技术

2.1 分布式数据库
2.2 共识算法
2.3 HTAP
2.4 蒙特卡洛树搜索

----------------

第3章 物化视图的设计与实现

3.1 物化视图

3.1.1 MJV 设计
3.1.2 MAV 设计 

3.2 查询重写模块设计与实现


----------------

第4章 基于强化学习的视图选择算法

4.1 生成候选视图

4.2 基于深度学习的代价估计模型
// 如果时间来不及就放弃这部分；直接使用优化器的代价估计

4.3 基于MCTS的视图选择

----------------

第5章 实验结果及分析

计划使用 IMDB 数据集和 Join Order Benchmark（JOB），对比 BigSub 算法和若干分布式贪心视图选择的算法；


对比使用深度学习的代价估计模型和使用优化器的代价估计模型，二者的准确率以及最终对算法性能的影响；

希望突出的创新点是：在性能上可以有一部分提升，并且将文献[]中的算法应用场景扩展到分布式环境中；

第6章 结论

-------
参考文献

[] 程学旗, 靳小龙, 王元卓, 等. 大数据系统和分析技术综述[J]. 软件学报, 2014, 25(9): 1889-1908.
[] Ghemawat S, Gobioff H, Leung S T. The Google file system[C]//Proceedings of the nineteenth ACM symposium on Operating systems principles. 2003: 29-43.
[] Dean J, Ghemawat S. MapReduce: simplified data processing on large clusters[J]. Communications of the ACM, 2008, 51(1): 107-113.
[] Shvachko K, Kuang H, Radia S, et al. The hadoop distributed file system[C]//2010 IEEE 26th symposium on mass storage systems and technologies (MSST). Ieee, 2010: 1-10.
[] Krizhevsky A, Sutskever I, Hinton G E. Imagenet classification with deep convolutional neural networks[J]. Communications of the ACM, 2017, 60(6): 84-90.
[] Huang D, Liu Q, Cui Q, et al. TiDB: a Raft-based HTAP database[J]. Proceedings of the VLDB Endowment, 2020, 13(12): 3072-3084.
[] Taft R, Sharif I, Matei A, et al. Cockroachdb: The resilient geo-distributed sql database[C]//Proceedings of the 2020 ACM SIGMOD International Conference on Management of Data. 2020: 1493-1509.
[] Yang Z, Yang C, Han F, et al. OceanBase: a 707 million tpmC distributed relational database system[J]. Proceedings of the VLDB Endowment, 2022, 15(12): 3385-3397.
[] Cao W, Li F, Huang G, et al. PolarDB-X: An Elastic Distributed Relational Database for Cloud-Native Applications[C]//2022 IEEE 38th International Conference on Data Engineering (ICDE). IEEE, 2022: 2859-2872.
[] Li G, Zhang C. HTAP Databases: What is New and What is Next[C]//Proceedings of the 2022 International Conference on Management of Data. 2022: 2483-2488.
[] Mami I, Bellahsene Z. A survey of view selection methods[J]. Acm Sigmod Record, 2012, 41(1): 20-29.
[] Baralis E, Paraboschi S, Teniente E. Materialized view selection in a multidimensional database[C]//VLDB. 1997, 97: 156-165.
[] Gupta H, Mumick I S. Selection of views to materialize under a maintenance cost constraint[C]//International Conference on Database Theory. Springer, Berlin, Heidelberg, 1999: 453-470.
[] Kotidis Y, Roussopoulos N. Dynamat: A dynamic view management system for data warehouses[J]. ACM Sigmod record, 1999, 28(2): 371-382.
[] Zhou J, Larson P A, Goldstein J, et al. Dynamic materialized views[C]//2007 IEEE 23rd International Conference on Data Engineering. IEEE, 2006: 526-535.
[] Jindal A, Karanasos K, Rao S, et al. Selecting subexpressions to materialize at datacenter scale[J]. Proceedings of the VLDB Endowment, 2018, 11(7): 800-812.
[] Yuan H, Li G, Feng L, et al. Automatic view generation with deep learning and reinforcement learning[C]//2020 IEEE 36th International Conference on Data Engineering (ICDE). IEEE, 2020: 1501-1512.
[] Han Y, Li G, Yuan H, et al. An autonomous materialized view management system with deep reinforcement learning[C]//2021 IEEE 37th International Conference on Data Engineering (ICDE). IEEE, 2021: 2159-2164.
[] Bauer A, Lehner W. On solving the view selection problem in distributed data warehouse architectures[C]//15th International Conference on Scientific and Statistical Database Management, 2003. IEEE, 2003: 43-51.
[] Chaves L W F, Buchmann E, Hueske F, et al. Towards materialized view selection for distributed databases[C]//Proceedings of the 12th international conference on extending database technology: advances in database technology. 2009: 1088-1099.
[] Agrawal S, Chaudhuri S, Narasayya V R. Automated selection of materialized views and indexes in SQL databases[C]//VLDB. 2000, 2000: 496-505.
[] Zhou X, Liu L, Li W, et al. Autoindex: An incremental index management system for dynamic workloads[C]. ICDE, 2022.
[] Bello R G, Dias K, Downing A, et al. Materialized views in Oracle[C]//VLDB. 1998, 98: 24-27.