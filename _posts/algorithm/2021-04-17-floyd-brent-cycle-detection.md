---
layout: post
title: "Floyd/Brent判圈算法"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
---

简单记一下Floyd判圈算法和Brent判圈算法 [leetcode-141][1] / [leetcode-142][2]，给出一条链的头节点，问链上是否存在环。

首先能想到一种简单的方法是用 `set/unordered_set` 记录已经访问过的节点，每遍历一个节点就塞入 `set` 中，如果遍历过程中碰到已经在集合中的节点，则表明有环；走到链的末尾则无环。

考虑 `unordered_set` 每次查询都是 $ O(1)$ 时间的话，这种方法的时间复杂度也是 $ O(n) $，但是空间复杂度也是 $ O(n) $，似乎有些题目会被卡。

#### Floyd 判圈算法[^1]

简单概括来说就是，如果存在环，那么朝同一个方向、以不同速度前进的两个指针必定会相遇。一般设指针  $p_1$ 步长为1，  $p_2$ 步长为2即可。当然也可以把  $p_2$ 的步长设得大一些，比如设置成3，4，5等，能更快“超圈”，这样就会变成brent算法。但是这里为了便于说明后续解释先不这么做。

如果只用判断是否存在环，则  $p_1$ 与  $p_2$ 相遇时函数就可以返回了。如果还要求环的起点，则需要进一步判断。

设从 $head$ 到环的起点要走 $d$ 步，设环的长度为 $k$，设相遇时  $p_1$ 走了 $x$步，则  $p_2$ 走了 $2x$ 步，又因为 $p_2$ 比 $p_1$ 多走了若干圈（不一定超1圈，可能$p_1$没进环内时$p_2$就在环上兜圈了），所以 $2x - x = x = nk$。此时 $p_1,p_2$ 的路程都是环长度的整数倍。

如果把 $p_1$ 放回起点 $p_2$ 位置不变，然后将 $p_1, p_2$ 步长都设置为1，同时前进，当 $p_1$ 走过 $d$ 步到达环的起点时，$p_1$共走过$d+x$步，$p_2$ 总共走过 $d + 2x$ 步，因为 $x$ 是环长度的整数倍所以此时 $p_2$ 也会恰好落在环的起点。因此找环的起点只要找到 $p_1, p_2$ 第二次相遇的位置就可以了，附上渣代码如下所示。

    /**
    * Definition for singly-linked list.
    * struct ListNode {
    *     int val;
    *     ListNode *next;
    *     ListNode(int x) : val(x), next(NULL) {}
    * };
    */
    ListNode *detectCycle(ListNode *head) {
        if(!head || !head->next) return nullptr;
        ListNode *p1 = head, *p2 = head;
        do{
            p1 = p1->next;
            p2 = p2->next->next;
        }while(p1 != p2 && p2 && p2->next);
        if(p1 != p2) 
            return nullptr;
        p1 = head;
        while(p1 != p2){
            p1 = p1->next;
            p2 = p2->next;
        }
        return p1;
    }


#### Brent 判圈算法

虽然Floyd判圈的复杂度是$ O(n) $，但是还有更快的Brent算法。

Brent算法的流程大致为：

1. $p_1, p_2$ 初始都指向 $head$，$p_2$先走 $step = 2$ 步，如果碰到 $p_1$，说明有环，终止；碰到链的末尾，无环，终止；
2. 如果算法没有终止，则令 $step = 2 * step$，将 $p_1$ 指向 $p_2$ 当前位置，继续重复步骤1。

与Floyd求起点略微不同的是，Brent算法中$p_1, p_2$ 相遇时 $p_1$ 的步数不一定等于环的长度了。但是因为每次我们都会先将 $p_1$ 移动到 $p_2$ 当前的位置，然后 $p_2$ 才开始前进，所以 $p_2$ 是一次性走一整圈的长度追上 $p_1$ 的，此时我们可以直接得到环的长度 $k$。参考 [这篇博客](https://blog.csdn.net/dpppBR/article/details/75477514)
，在 Floyd 判圈算法中，我们利用的是 $p_2$ 走过的距离是环长度的整数倍来求出环的起点；而现在已知环的长度 $k$，只要把 $p_1$ 放回链表头，$p_2$ 放到距离链表头 $k$ 的地方，就能用 Floyd 的步骤找到起点。下面就直接附上代码了。


------------



    ListNode *detectCycle(ListNode *head) {
        if(!head || !head->next) return nullptr;
        ListNode * p1 = head, *p2 = head;
        int step = 2, cnt = 0;
        while(p2){
            cnt++;
            p2 = p2->next;
            if(p1 == p2)
                break;
            if(cnt == step){
                step *= 2;
                cnt = 0; 
                p1 = p2;
            }
        }
        if(!p2) return nullptr;
        p1 = p2 = head;
        while(cnt--){
            p2 = p2->next;
        }
        while(p1 != p2)
            p1 = p1->next, p2 = p2->next;        
        return p2;
    }


-------------------------------------

感觉这题应该算是判断一个图是否存在环的特例，那么用来判别图上是否有环的 拓扑排序/DFS方法 也能够用在这里，只不过有些浪费而已（用 set 的方法应该就是一种DFS；至于拓扑排序那更麻烦了，在统计每个节点度数的时候就需要遍历所有节点，还不如直接用 set ）

文字似乎有些啰嗦，以后有心情的话再改吧。

The End

[1]: https://leetcode.com/problems/linked-list-cycle/
[2]: https://leetcode.com/problems/linked-list-cycle-ii/

[^1]: [Wikipedia - Floyd Cycle Detection Algo](https://zh.wikipedia.org/wiki/Floyd%E5%88%A4%E5%9C%88%E7%AE%97%E6%B3%95)


