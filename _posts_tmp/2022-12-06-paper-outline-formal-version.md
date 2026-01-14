题目：面向大规模原生分布式数据库的视图选择算法
Title: View Selection for Large Scale Native Distributed Database System


第1章 绪论

1.1 课题研究背景和意义

一串手写的数字、几个随机排列的字符、报纸上裁剪下来的一小块报道内容、几句口头转述的话语、一封泛黄的信笺，都包含着数据。但是这些数据是否有价值呢？这要取决于数据对应的信息对于接收和使用它们的人的意义。比如，这串数字是某个被遗忘在角落的账号的密码，这份报道是某人见义勇为事迹的嘉奖，这些转述的话诉说着对远方亲人的思念，这封信是第一次鼓起勇气递出情书后收到的回信，那么它们的意义将无比重大，它们的价值甚至无法用金钱来衡量。但是对大多数人而言，这样的数据所包含的信息，其实并没有太大价值。因为数据的价值是人们赋予它的；或者更准确的说，是由数据的使用者赋予的。对于无法从数据中解读出信息并加以利用的人来说，任何数据都只是一堆无意义符号组成的随机序列而已；但对于能够充分利用数据的人来说，数据的价值是难以估量的。

进入二十一世纪后，互联网的蓬勃发展使得全球无时无刻不在产生海量的数据，全世界产生的数据量在2012年首次超过1ZB。据IDC估计，2025年全世界产生的数据量将达到163ZB，这一趋势有力地推动了大数据处理和分析技术的发展：2003到2004年，谷歌研发的文件系统GFS[]和MapReduce编程模型[]，以及随后出现的开源产品Hadoop[]，掀起了“大数据时代”的热潮[]；而随后 2012 年 Hinton 等人在 ImageNet 图像分类任务中使用 RNN 取得重大突破[]，将机器学习重新带回人们的视野，让人类有机会尝试利用各种深度学习算法进一步从海量杂乱无章的数据中提取出更多的信息，创造更多的价值。

作为承载数据的软件容器，数据库系统从上世纪六十年代出现伊始发展至今，也从当初面向单个企业或组织、用于代替文件系统存储和管理数据的独立而功能单一的系统，发展为今天包括关系型数据库[]、包括键值数据库和图数据库等的NoSQL[]、分布式数据库[]、云原生数据库[]等在内的数据库大家族。数据库也不再是面向单个离散的企业或组织，而是通过网络与成千上万个节点或终端用户相连接，支撑着数千万、甚至上亿用户的应用服务。

随着互联网赋予数据的价值不断提高和业界对于数据的利用方式越来越丰富[]，通过数据仓库、Hadoop等大数据分析系统等进行的静态数据批量处理， 通过Strom/Spark/Flink等进行开源系统进行的流式数据处理，机器学习尤其是各类深度学习模型与大数据的结合，使得人们能够发掘出更多以往杂乱无章的数据的潜在价值。慢慢的，以往通过OLTP数据库系统进行事务处理、产生的数据经过ETL处理后导入OLAP系统进行数据分析这样的两套系统运行的方式，已经不足以满足如今数据分析任务实时性和用户体量迅速提升后的对系统扩展性、鲁棒性的要求了。
// 这部分参考 HTAP 综述，有一些应用是要求实时性的，可以举一些应用场景

为了解决这一问题，业界开始整合OLTP和OLAP，寻求数据库的 HTAP 能力，即 Hybrid Transaction/Analysis Processing，在一套系统中同时提供事务处理和数据分析功能，在保证事务处理能力的同时，几乎无延迟的对最新数据进行分析[]。而这样的融合将会破坏原本解耦的事务处理和数据分析系统，也意味着传统的TP/AP技术必须针对新的应用场景作出调整。如TP系统的存储方式通常为行存，而AP系统更多采用列存的方式来获取更优的读取性能，因而HTAP系统要在两种存储方式之间权衡，做出取舍；物化视图也是传统数据仓库系统和主流OLTP系统中常见的一种通过持久化数据、减少重复计算开销的优化措施，尤其在数据仓库中，选择适合的视图集合进行物化，对整个数仓的性能有至关重要的影响。在新场景下，如何根据HTAP的工作负载特性，自动选择更高效的物化视图组合，也是该领域所面临的新挑战。

综合以上讨论，本文决定设计一种基于强化学习的分布式场景下视图选择方法，在分布式数据库Oceanbase开源版中进行了实验，并与传统算法进行实验对比。

1.2 研究现状

1.2.1 生成和筛选候选视图
根据工作负载中查询的特点，生成有效的候选视图作为视图选择算法的输入，是解决视图选择问题的第一步工作。现有的生成候选视图方法可大致分为三类：构建查询的有向无环图、对查询进行句法分析、基于查询重写的生成方法。[2012 survey]

1）构建查询有向无环图

构建有向无环图通过合并多个查询的执行计划来寻找查询之间的公共子表达式（common sub-expression），这背后的思想主要是这些公共子表达式物化比起单独物化每个查询对应的视图能够节省更多存储空间和维护开销。文献中使用这种方式来生成候选视图的有以下几类：

MVPP：
叶节点表示关系，中间节点表示OP,根节点为查询（或者说查询结果？//好像不能说是结果？；

// 要不要用 MVPP 的select 语句画个例子示意图，对比 AND/OR？

AND/OR view graph：
节点分为两类：Operation Node 和 Equivalent Node，OP node 是（关系代数表达式，如Select Project Join，叶节点是关系，根节点是查询结果

[1997 selection of a view]
[1998 under maintenance cost constraint]
属于这一类

Data Cube Lattice:
// 这部分可能就丢了不写了

2）查询句法分析
// 概括一下语义分析是什么
[2000 automated select mv&index]
[2009 toward ... in distributed database]
属于这一类，// 我咋没太看出来？

[2020 tsinghua View generation LSTM]

SWIRL 似乎也算？BOO 

3）查询重写
// 并没有看到
有的，survey 中的文献\[45,46,48] \[47]


// 视图怎么说？//总结一下这些看过的算法中生成候选视图的策略；

1.2.2 视图选择算法
视图选择算法根据算法对工作负载的要求可大致分为静态选择算法和动态选择算法，在此基础上又出现了基于机器学习的视图选择算法和面向分布式场景的视图选择算法。

  1）静态选择算法
  静态选择算法需要预先获取完整的工作负载（即所有查询的序列），大多数视图选择算法基本都归属于这一类别[2012 survey of view selection]。

  文献[1997 MVPP] （和后续的 framework 的文章）通过合并查询的执行计划构建出多视图处理计划（Multi View Processing Plan）来寻找其中的公共执行计划作为物化的候选视图，并且穷举搜索来选择所有对查询带来增益大于维护成本的公共执行计划节点作为最终物化视图集合；该工作应用的场景只是由几条查询语句构成的小规模负载，没有提供算法性能的相关实验，也没有考虑存储空间的限制。

  文献[1997 selection of a view]通过将视图和基础表构建成 AND/OR 图，转化为图的遍历搜索问题，并设计贪心算法选择每单位存储空间获得收益最大的视图，来实现多项式时间复杂度的视图选择，还将算法扩展到物化视图上存在索引的情况；但该工作只考虑了对视图空间占用的限制，对维护开销并没有做约束，同时也没有进行实验对算法性能进行评估； 
  // 没有对算法性能进行评估（evaluation），没有给出实验

  文献[1998 maintenance cost]在[上一篇文献]的基础上，加入视图的维护开销约束，同样通过贪心选择“得到增益与维护开销比值最高的视图”，来获取 OR view graph 中的近似最优解，同时还针对 AND-OR 设计了基于 A* 算法的枚举搜索算法，能够保证搜寻得到最优解；该工作的缺点也比较明显，首先 OR view graph 只是视图选择问题的一个子集，且最坏时间复杂度是指数级别，很难将该贪心算法推广到大规模查询的场景中；另外，通过转化为 AND-OR 图之后虽然可以应用 A* 算法得到最优解，但是最坏情况下该算法可能退化为穷举算法，同样难以解决大规模的视图选择问题。

  // 文献（一些遗传算法和模拟退火算法），大概两篇，速看；

  2）动态选择算法
  动态视图选择算法可以工作在负载实时变动的场景下，因而算法需要根据当前负载变化的情况来对选择的视图进行调整，以便获取针对当前负载最优的视图组合。
  
  文献[1999 DynaMat]考虑到静态选择算法提供的视图选择可能会因工作负载的变化而失效，选择将适当的查询结果持久化以便重复利用，减少重复计算的开销。DynaMat引入缓存池来物化一部分查询的计算结果，通过额外的Fragment Locator来判断某个查询的结果是否位于缓存池中，若在则直接从缓存读取而不用重复计算生成；另外如果缓存池已满，DynaMat将使用LRU/LFU/SFF/SPF等缓存策略进行物化数据的替换。实验结果表明DynaMat能在视图的空间占用和维护时间窗口都受到限制的情况下进行动态视图选择，并且查询性能依然保持与静态选择算法相当的水平。但是文中将缓存池的更新限制为离线的，如果数据源发生更新后需要暂停服务来对缓存池中物化的数据进行同步，这对系统的性能有较大影响。

  文献[2007 dynamic materialized view]则针对工作负载访问数据存在偏斜的场景，提出有选择性的维护视图中一部分数据（通常选择访问频率较高的部分），加入Control Table来标记这些数据，并且根据负载的变化调整物化的部分；在缓存替换部分，他们对比了LRU/Clock/2Q等替换策略后，决定基于2Q策略设计了一种新的替换方法，但没有展开介绍该方法的细节。最后该工作的实验结果表明，在查询偏斜大于 90% 的情况下，对比物化完整视图，动态物化视图能够使用完整视图 5% 的存储空间达到相同的查询性能，并且维护开销远小于使用完整视图。
  

  3）基于机器学习的视图选择算法
  近十年来深度学习 领域 研究的蓬勃发展，使得
  新模型/ 在传统场景中的应用，
  能够带来新的驱动力。

  人们在寻找 AI 能够落地应用的方向。
  目光开始投向传统 // 行业？ 数据库好像也不是很传统

  能够自适应、自我调节的数据库一直以来都是数据库行业内普遍的追求，能够将DAB从繁琐的调优维护工作中解放出来，这方面也已经积累了许多研究「能不能来点引用？」，但许多研究的应用尚且存在局限，仍未出现一个一站式的解决方案。而 AI 的出现让 （人们）看到了这一追求实现的可能：有了AI的学习和调整能力，未来的数据库有希望成为一个不需要任何（或者少量）人工监管，依靠AI就能针对各种各样工作负载，将硬件性能发挥到最大，时刻为用户提供最高效服务的基础架构。

  虽然这一过程还有很长的路要走，但这光明的前景和 ？？ 挑战性的工作 吸引着众多研究者投身其中 // 哈哈太煽动性了不太像学术论文 // 好像跟前文不太搭，要删吗QAQ

  近年来，AI for DB 领域得到众多研究者的关注

  AI for DB 的应用包括 learning-based 数据库配置，如基于深度强化学习的查询重写、数据库调优，基于强化学习的自动化索引/物化视图推荐；另外还可应用于执行代价估计、优化器设计、索引结构、底层数据结构等各个模块。我们这里主要关注机器学习在视图选择方向上的研究。

  文献[2018 bigsub]进行视图选择的场景是希望能够支持大规模的工作负载（每天数十万个分析任务），该工作关注的重点是算法的扩展性，精度和性能可以略微牺牲。因此他们提出了一种类似爬山算法的随机算法BigSub，将视图选择模型化为二分图标记问题（bipartite graph labeling)，其中的顶点表示查询和候选视图，边连接一个查询顶点和它所有对应的候选视图顶点，通过概率性的翻转某个顶点（实际含义表示选择该视图，增益越高选择该视图的概率越高）来搜寻能够获得更高增益的视图集合。这一标记问题可以拆分为若干子问题并行完成，并且候选视图的生成上他们选择采用的是优化器提供的逻辑计划中的子表达式（subexpression）作为候选，来进一步缩小搜索范围，使得算法能扩展支持更大规模的查询负载。最终的实验结果表明该算法能支持微软中数千节点集群每日数十万分析任务的应用场景要求，比起贪心算法和其他启发式算法最多能达到3个数量级的性能提升。但存在的缺点是算法无法保证在全局最优解停止，最终可能收敛于局部最优解，因此性能上比起全局最优解略有欠缺，这一点与爬山算法类似。

  文献[2020 automatic view generation]认为BigSub的迭代式选择过程为了保证能够收敛，禁止将已选择的子查询再转化为未选择，这可能会导致退化为贪心算法而得到较差的结果。该工作首先从执行计划中提取子查询（包括连接、投影和聚合），使用EQUITAS方法来检测等价子查询[]，并将等价的子查询归类，从每一类中选出开销最低的子查询作为算法的候选输入；对数值特征和非数值特征分别使用了CNN和LSTM来提取信息，并训练该模型预测查询的开销以获取比优化器更精确的执行代价估计；最后将视图选择的 ILP 问题模型化为马尔科夫决策过程，使用DQN算法来进行视图选择。与传统的贪心算法和BigSub方法进行实验对比，在不同数据集上都获得了一定的性能提升。
  // TODO:这里改一改，描述一下EQUitas大概是什么方法
  // EQUITAS 引用，是个检测 equivalent subqueries 的方法，论文：
  // Automated veriﬁcation of query equivalence using satisﬁability modulo theories 

  文献[2021 autoview]与BigSub类似，也将视图选择问题模型化为 ILP（Integer Linear Programming）问题，但该工作通过提取SQL对应物理计划，然后通过构建MVPP（Multi-View Processing Plan）来生成候选视图；在查询代价估计方面，他们使用了GRU构成的Encoder-Reducer模块来评估视图带来的增益；在视图选择算法部分采用强化学习的DDQN模型，最后在Join Order Benchmark上获得了比BigSub和贪心算法更好的性能。不过相对于BigSub的应用规模而言，该工作能支持的查询规模有限，尤其是生成候选视图的MVPP方法是一种穷举式的方法，加上强化学习方法面对大搜索空间时极慢的收敛速度，使得我们难以直接将其扩展到大规模的查询工作负载中。


  这些基于机器学习的研究，比起传统算法能更好的学习和挖掘查询之间的关联，往往比起传统方法能获得更好的性能。但是我们也看到这些研究中大部分都不再考虑维护视图开销的约束，只考虑存储空间这一约束条件。这与存储设备单位价格不断下降、对于企业而言限制维护开销比限制存储空间更重要的实际需求有所出入。

4）分布式场景下的视图选择
以上文献的研究对象多为集中式的数据库或数据仓库，但随着信息量的增加、数据规模的增大，集中式系统开始无法满足应用的扩展性要求，研究人员也开始探寻分布式场景下的视图选择方法。

文献[2003 view selection in distributed warehouse]，一种应用于分布式数据仓库的贪心视图选择算法

文献[2009 towards mv selection for distributed db]，提出一种面向物联网场景的分布式遗传算法来进行视图选择。

1.2.3 索引选择算法
数据库索引在作用、功能和结构上与物化视图有许多相似之处，在算法的设计上值得互相借鉴，因此这里也对索引选择算法的研究现状进行了探讨。

文献[2000 automated selection for mv & index]首次尝试将MV和Index看作同一类结构，同等的根据每单位存储空间带来的增益从物化视图或者索引中进行贪心选择。该工作还提出了一种筛选候选视图的方法，首先对查询语句进行句法分析来获取所有查询可能涉及的候选视图，然后使用贪心算法找出查询开销超过某个设定阈值的表的集合，称为interesting table，再基于涉及insteresting table的查询来构造候选视图，来得到最终的候选视图集合。该工作的实验结果表明，将物化视图和索引共同进行选择，比起对视图和索引独立选择，能带来更好的性能。

// 要不要看看 Extend 啊？

文献[2022 SWIRL]通过强化学习的PPO（Proximal Policy Optimization）方法来进行索引选择，采取了一种新的表征方式来对工作负载进行表示：通过词袋模型来对Operator进行表征，文中称为BOO（Bag of Operator），使用潜在语义索引（Latent Semantic Indexing）降维后得到最终能够记录工作负载足够丰富信息的表征向量，用于后续的强化学习训练过程。另外，该工作采用了Invalid Action Masking的方式将超出存储限制、已经创建的索引等无效的行动从搜索空间中暂时去除，极大加快了算法收敛速度，对于最终推荐索引的质量提升也有一定帮助。值得注意的是文中强调过早对候选索引进行筛选可能会影响算法质量，因此他们生成候选索引的方式是选择所有语义相关的列对应的索引（包括它们的排列）。
// PPO 引用
// Latent Semantic Indexing 引用
// invalid action masking 是否也有引用？

文献[2022 AutoIndex]设计实现了一种面向动态工作负载的索引管理系统，能够根据工作负载的变化来自动增加新索引或删除低性价比的旧索引。该工作通过将相似的查询映射到一组模板中，按照模板来产生候选索引，缩小了算法的搜索空间；然后设计实现了一个使用深度回归模型的代价估计模块，可以实现的对索引更新开销的估计；最后的索引选择算法部分，他们通过维护一棵决策树，使用蒙特卡洛树搜索完成索引推荐。本质上来说该工作的算法其实仍然属于静态索引选择，他们的系统在新到来的查询到达某个阈值之后，将其与历史查询合并为一个新的工作负载，再次运行索引选择算法来推荐出新的索引集合，以此实现面向动态工作负载的自动索引管理。


// 这段内容需要更新，跟前文衔接一下
不过综合以上工作，可以发现视图选择和索引选择在候选视图/索引的生成上还是有很大区别。一些早期的工作因为数据集和工作负载规模较小，采取的还是枚举式的生成方式；但这样产生的组合数量是指数级别的，贪心算法和启发式算法很难从中选出高质量的解决方案。因此后来的研究开始寻找工作负载中查询的关联来生成候选索引/视图，有效的减少了算法的搜索空间，在不损失过多性能的情况下，大幅提高了运行速度。

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
  // OB的文档
  分布式中间件+单机数据库：TDSQL Polar-X
  共享存储（share storage）：TDSQL-C GaussDB for MySQL
  原生分布式（share nothing）：OceanBase TiDB

2.2 共识算法
2.3 HTAP
  HTAP（Hybrid Transaction/Analysis Processing），即混合事务与分析处理，是研究机构Gartner于2014年首次提出的概念「软件学报的综述」，定义HTAP系统为一个能在内存中并发执行分析任务和事务处理的数据存储系统；2017年进一步扩展了定义，不再限定于“在内存中执行的计算任务”[2017 gartner][2022 HTAP]。

  挑战包括：
  数据组织。HTAP数据库的数据组织方式可以分为：1）行存+内存中列式存储。（介绍概念，典型系统）；2）行存+分布式列存；3）列存+增量行存。典型有SAP-HANA

  MemSQL 内存中行存，转磁盘中列存

  数据同步。不管HTAP数据库使用何种数据组织方式，（为了避免查询时处理和计算增量数据的开销），都需要在？之间进行数据同步。1）memory delta merge。（解释；2）disk-based delta merge。（解释；3）rebuild from primary row store「居然还有用这种方式同步的数据库」

  查询优化、资源调度 // 清华的tutorial也有提及； 2017年 SIGMOD 的也是tutorial

  HTAP系统目前广泛应用于？、？、？等领域。
  银行交易分析系统，大型电商交易分析平台，大型物联网系统，金融反诈系统

  // 综合一下 一些 HTAP 和 传统架构的 架构对比图；画一个；以前画图用什么来着？虽然我觉得drawio和exclidraw画出来的都挺好看的  // 图现在比较喜欢 橘黑灰白 这一系列的配色

2.4 蒙特卡洛树搜索

2.X
PPO
DQN
MCTS等等一堆都可以写（当然如果用到的话
// 但是现在连候选视图都还没生成我怎么选？

----------------

第3章 物化视图的设计与实现

3.1 物化视图

3.1.1 MJV 设计
3.1.2 MAV 设计 

3.2 查询重写模块设计与实现


----------------

第4章 基于强化学习的视图选择算法

tmp1：问题/术语定义
// 搞一堆公式，描述问题

// 这部分内容怎么这么少？，加上公式能有半页纸吗？

view 视图
materialized view 物化视图
view candidate 候选视图
benefit 物化视图的增益
common subquery 公共子查询

view selection 视图选择（问题）
从候选视图集合中选出一个视图集合子集，使目标函数（代价函数）最小，（且


！代价函数

// TODO：存储和维护的限制如何搞？

4.1 生成候选视图
//方法描述

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

// 传统关系型数据库，MySQL、PG、Oracle等的引用
[] Douglas K, Douglas S. PostgreSQL: a comprehensive guide to building, programming, and administering PostgresSQL databases[M]. SAMS publishing, 2003.
[] Ports D R K, Grittner K. Serializable snapshot isolation in PostgreSQL[J]. arXiv preprint arXiv:1208.4179, 2012.


// NoSQL数据库代表的引用，如FDB？图数据库来一点，其他的就算了
[] Han J, Haihong E, Le G, et al. Survey on NoSQL database[C]//2011 6th international conference on pervasive computing and applications. IEEE, 2011: 363-366.

[Aurora Cloud-Native] Verbitski A, Gupta A, Saha D, et al. Amazon aurora: Design considerations for high throughput cloud-native relational databases[C]//Proceedings of the 2017 ACM International Conference on Management of Data. 2017: 1041-1052.
[alibaba cloud native] Li F. Cloud-native database systems at Alibaba: Opportunities and challenges[J]. Proceedings of the VLDB Endowment, 2019, 12(12): 2263-2272.

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

// HTAP survey 1～2
[] 张超, 李国良, 冯建华, 等. HTAP 数据库关键技术综述[J]. 软件学报, 2022: 0-0.
[] Özcan F, Tian Y, Tözün P. Hybrid transactional/analytical processing: A survey[C]//Proceedings of the 2017 ACM International Conference on Management of Data. 2017: 1771-1775.

//PPO algorithm
[] Schulman J, Wolski F, Dhariwal P, et al. Proximal policy optimization algorithms[J]. arXiv preprint arXiv:1707.06347, 2017.

// AI4DB 
[] Li G, Zhou X, Cao L. AI meets database: AI4DB and DB4AI[C]//Proceedings of the 2021 International Conference on Management of Data. 2021: 2859-2866.
[] Zhou X, Chai C, Li G, et al. Database meets artificial intelligence: A survey[J]. IEEE Transactions on Knowledge and Data Engineering, 2020.
