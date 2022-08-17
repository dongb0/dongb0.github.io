---
layout: post
title: "[tinyky] Debug日志 （1）"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - raft
  - golang
---

> 记录实现tinykv遇到的各类bug和解决方案

###### 1（project2b）client查询时发现一部分应用成功的put请求，没有查询到对应的value
  
  - 出错原因

     处理 PeerStorage的Get/Snap 请求，直接从leader的state machine中读取然后返回查询结果；如果是正常的leader这样确实能返回正确结果，但是被分割的leader在这种逻辑下也会向client返回过时的数据；另外，分区或其他原因导致leader切换时，新leader的applyIndex可能并没有赶上commitIndex，但是raft认为commit的日志所对应的请求已经处理完毕并且返回响应结果，所以也可能查询不到value。
  - 解决方法
    
    TODO：什么是线性一致
    
      1. 实现最简单但低效的是，将Get/Snap等读请求也写入日志，等待共识完成之后才返回查询结果。这样可以保证有且只有一个有效的leader能响应client的查询请求。
      2. 优化1：ReadIndex。参考[1]、[2]  
        读请求不需要写state machine，只是为了通过写日志这一方式进行集群共识来确认leader是否有效（即所在partition是否有超过半数的节点），但写日志这一操作其实可以免去。  
        ReadIndex实现在处理读请求时改用heartbeat的方式确认leader是否有效，并且用增加一个readIndex变量记录接到请求时的commitIndex，来保证向client返回结果时，applyIndex能赶上commitIndex。
        具体过程如下：  
          a.leader记录接到读请求时的commitIndex，称为readIndex  
          b.leader发送心跳，等待半数节点的响应  
          c.等待state machine应用日志index大于等于readIndex  
          d.返回读请求的查询结果给client  

      3.优化2：LeaseRead（TODO）

###### 2（project2aa）raft检查electionConflict的几率大于30%

  - 出错原因

    我最开始处理选举随机性的逻辑是，每次tick时生成一个随机的超时时间electionTimeout + rand.Intn(electionTimeout)，如果当前时间大于该时间则发起选举，但是得到的candidate选举冲突的概率总是在0.3～0.35（5个节点）。但是继续增加rand的随机数范围的话，其他的测试又可能因为在$2 \times electionTimeout $时间内没有成功发起选取而失败。后来我把这个过程抽象出一个概率问题，在同学的帮助下发现这样实现的选举方式，冲突期望确实为0.35，所以要想把冲突降到0.3以下需要采用其他的随机方式。
  
  - 解决方法

    暂时没有特别好的解决思路，所以决定增加随机数的数量，现在采用的做法是在每次重置electionTimeElapse时不是全部置为0,而是设为`rand.Intn(timeout/2)`；另外生成的超时时间也变为`timeout+rand.Intn(timeout/2)+rand.Intn(timeout/2)`。这样得到的冲突概率约为0.25。（这次的期望不会算了）

  - 附：问题及题解

    问题是这样的：某国特工在五角大楼五个角上各装了一枚定时炸弹，炸弹同时从0开始计时，每过1个时间周期T，计数器加1，并生成一个［1，10］的随机整数，当计数大于等于该随机数时炸弹引爆。两枚或以上的炸弹同时爆炸可以摧毁大楼，如果只有一枚炸弹先引爆则任务失败，问任务失败的期望是多少。

    题解：因为只有1枚炸弹单独爆炸任务才失败，所以计算每一秒只有1枚爆炸的概率。

    首先第1秒只有1枚炸弹爆炸的概率是 $ C_5^1 \times 0.1 \times 0.9^4 = 0.32805 $；
    第2秒时，先是第1秒没有炸弹爆炸的概率 $ 0.9^5 $，乘上4枚第2秒也不爆炸的概率 $ 0.8^4 $，以及1枚第2秒爆炸的 0.2（摇到1和2都会爆炸所以是 $0.2$），得到 $C_5^1 \times 0.9^5 \times 0.8^4 \times 0.2 = 0.24186$ ;
    第3秒时，是前2秒没有炸弹爆炸的 $ 0.9^5 \times 0.8^5 $, 因为每一秒都是生成随机数概率都是独立的，这里得用第1秒不爆炸的概率乘上第2秒的概率，然后乘4枚第3秒不炸的 $ 0.7^4 $和1枚第3秒爆炸的 0.3，得到 $C_5^1 \times 0.9^5 \times 0.8^5 \times 0.7^4 \times 0.3 = 0.069686$;

    然后以此类推得到第4秒的概率$C_5^1 \times 0.9^5 \times 0.8^5 \times 0.7^5 \times 0.6^4 \times 0.4 = 0.008429$，第5秒$C_5^1 \times 0.9^5 \times 0.8^5 \times 0.7^5 \times 0.6^5 \times 0.5^4 \times 0.5 = 0.003951$，第6秒的概率0.000006，再往后的概率都低于0.00001就不计算了，最后累加得到的结果是 0.648431

    这里任务失败的期望是无冲突的期望，所以任务成功的概率0.351569就是我想知道的冲突期望。

###### 3（project2b）日志输出显示从storage读取到的entries是正常的，但leader发送appendEntries包含的全部是最后一个entry

  - 出错原因
  
    使用以下代码片段方式来取entry的指针：
        entries := []Entry{}
        entPointers := []*Entry{}
        for _,v := range entries { 
          entPointers = append(entPointers, &v)
        }
    
    但是golang关于range的使用方式说得很清楚：
  
    > When ranging over a slice, two values are returned for each iteration. The first is the index, and the second is a copy of the element at that index.

    这里的`v`只是一个临时变量，range在遍历过程中会不断给这个变量赋值来，我这样取到的地址其实并不是entry真正的地址，只是这个临时变量的地址；而当遍历结束时临时变量`v`的值变成了数组的最后一个元素，所以指针切片里输出的元素全都是它。

  - 解决方法
    
    采用下标引用的方式取指针即可，`for i := range entries { entPointers = append(entPointers, &v) }`
    

###### 4（proj2c）TestOneSnapshot2C小概率出现panic: can't get value 76313030 for key 6b313030

  - 出错原因
  
    我修改过测试用例，TestOneSnapshot2C现在的测试逻辑是3个节点在partition时会固定将leader隔离，剩下两个节点重新选举。出现上述panic原因是cluster向leader发送请求时，非常非常恰巧**所有**请求都先发给了旧leader超时之后才重新向新leader请求，导致旧leader的日志长度跟新leader一致(但是日志的term不同）。分区恢复之后，我的实现是先通过heartbeat更新Prs，但是处理heartbeat的逻辑里忘记检查日志的term，导致旧leader此时会告诉新leader它已经持有相同长度的日志了，最后没正确获取最新日志从而在后续查询出现问题。

    在原本的逻辑里出现该错误概率非常小，目前跑了150次以上该测试只出现了这一次；但是比较好复现，只要集群向所有能检测到的leader都发送请求就会出现。
    
    解决方案：leader处理heartbeat response时，判断是否日志是否最新时，不单要看日志长度，还需要看对应日志的term是否相同。只有日志长度和term都相同，才能表明是相同的日志。这本应该是需要时刻牢记的点，在handleAppen时也做了，却在处理heartbeat时忘记了。

###### 5（proj2c）节点重启后读取持久化的日志时，某节点的日志会部分丢失

  - 出错原因

    似乎有好几处bug造成了这个错误。

    1. PeerMsgHandler会向raft发送AdminCmdType_CompactLog请求，等待raft将这个消息写入日志并提交之后，对PeerStoage中的日志进行截断压缩。问题出在我没有考虑一个Ready中包含Snapshot和AdminCmdType_CompactLog的时候，他们都需要对applyState进行修改，导致其中一个的修改被吞掉了。出错的地方往往是Snapshot已经删除了日志，但是CompactLog请求又把applyState.TruncatedIndex覆盖了Snapshot的修改，将其改为一个更小的值。比如原本日志为\[100, 200\],CompactLog请求要截断为\[150, 200\]，Snapshot应用时实际更新日志为\[160,220\]。我先应用snapshot然后再处理CompactLog请求，导致Peer读TruncatedIndex时认为150～160这部分日志还在，导致该错误。  
    解决方法：TruncatedIndex更新时取最大值；或者以Snapshot的更新为准，由Snapshot覆盖CompactLog对TruncatedIndex的修改的修改。

    2. Raft层发送Snapshot请求和handleAppend实现上的逻辑不统一。我实现Snapshot时如果暂时没有生成Snapshot，则会发送一个Entries为空的Append请求，本意是想借此更新Prs；但是follower实现的handleAppend处理Entries==nil时，因为该AppendMsg包含的Index和LogTerm与follower的日志一致，接受AppendMsg的同时也把follower的CommitIndex更新为AppendMsg.Index了。但是可能存在leader的AppendMsg.Index超过leaderCommitted的情况的，此时上述逻辑中的follower committed就超过leader，这显然不对的。  
    解决方法：接受AppendMsg时，论文中提到follower更新committed的逻辑为If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)；同时还需要正确处理AppendMsg中Entries为空的情况，此时只能将Committed更新为`min(AppendMsg.Index，AppendMsg.Committed)`

    3. 依然是Raft层实现Snapshot的问题。PeerStorage中apply Snapshot时，会删除当前所有的日志，但是handleSnapshot时一并追加的日志还是根据删除之前的来追加的，导致最后生成的Ready中包含的日志不完整。如Snapshot.Index=105, entries:\[{Index:106}, {Index:107}, {Index:108}...]，节点原本的日志为\[...{Index:106},{Index:107}]，我的错误实现里接受该snapshot之后生成的ready，只包含一个Snapshot和Index:108之后的日志，最终导致上述错误。  
    解决方法：接受snapshot时也接受消息中包含的全部日志。

[1]: https://pingcap.com/zh/blog/linearizability-and-raft
[2]: https://keys961.github.io/2020/11/06/etcd-raft-7/