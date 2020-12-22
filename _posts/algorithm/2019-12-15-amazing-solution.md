---
layout: post
title: "神奇的解法"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
tags:
  - algorithm
  - note
---

这几天看到两道题，发现有些解法很有意思，记录一下。

### 传球问题

这道题[luogu P1057](https://www.luogu.com.cn/problem/P1057)有一个令人耳目一新的解法。通过转化成图，这道题就变成了从节点 1 走 m 步，有多少种不同的路径能够回到节点1。然后利用离散数学里面学到可达性矩阵，计算 $m-1$ 次矩阵乘法后，节点1对应的值就是答案。

    vector<vector<int>> g, tg, res;
    void  multiply(vector<vector<int>> &m1, vector<vector<int>> &m2, vector<vector<int>> &res)
    {
      for (int i = 0, n = g.size(); i < n; i++)
        for (int j = 0; j < n; j++)
          for (int k = 0; k < n; k++)
            res[i][k] += m1[i][j] * m2[j][k];
    }

    int main()
    {
      int n = 0, m = 0;
      cin >> n >> m;
      g.assign(n, vector<int>(n, 0));
      for (int i = 0; i < n; i++)
        g[i][(i + n - 1) % n] = g[i][(i + 1) % n] = 1;
      tg = g;

      for (int i = 1; i < m; i++)
      {
        res.assign(n, vector<int>(n, 0));
        multiply(tg, g, res);
        tg = res;
      }
      cout << tg[1][1];
      return 0;
    }


### 数列分段

[luogu P1182](https://www.luogu.com.cn/problem/P1182) 和 [leetcode 410](https://leetcode.com/problems/split-array-largest-sum/)

这道题的做法，题解基本都是某种应用了二分和贪心的方法，尤其是洛谷，高赞的代码都特别像，像是一个老师（教练）教出来的，我作为菜鸡实在没办法“一看到数列分段就想到用二分法”，只好尝试另一种比较接近常人思维的 DP。

先说DP的方法，分别在某[博客][1]和 leetcode [讨论区](https://leetcode.com/problems/split-array-largest-sum/discuss/141497/AC-Java-DFS-%2B-memorization)看到 bottum-up 和 top-down 的 DP 写法。后者跟我的比较像也比较普通，这里就记前者吧，想法还是比较巧妙的。

首先定义数组$ dp[\ ][\ ]$，其中$ dp\[i]\[j] $是前 $ j $ 个数划分成 $ i $ 组后$（j >= i）$，所有划分里得到的最大值最小的那个。比如对于数列 `4 2 4 5 1`，$dp\[1]\[3]$是将前3个数 `4 2 4` 分为1组，最大值为$4+2+4=10$，$\therefore dp[1][3]=10 $。$dp\[2]\[4] $是将前4个数分为2组，有3种分法

    [4][2 4 5] max_sum = 2+4+5 = 11 
    [4 2][4 5] max_sum = 4+5 = 9  
    [4 2 4][5] max_sum = 4+2+4 = 10  

$ \therefore dp\[2]\[4]=min\lbrace11,9,10\rbrace = 9 $

$dp[1][j]$这一行，表示将前 $j$ 个数分为1组，也就相当于前缀和数组。

现在我们的 **核心问题** 是，第二行开始的每个dp\[ ]\[ ]要怎么确定呢。

要将前 $j$ 个数分为$i$组，并不需要 $j$ 个数分为 $i-1 $组的结果来递推（这也没法递推，得重新划分），
我们需要的是前 $\[1, j-1]$ 个数分为$i-1$组的结果（实际上是$\[i-1, j-1]$，因为每组至少 $1$ 个元素，所以元素个数不能比组数少），然后把新加入的数作为 $1$ 组，这样就能枚举完所有的分组情况。

还是举个例子吧。

只要想象成，$dp[i-1][k], k < j$ 中包含了所有k个元素分为 $i-1$ 组的情况，而我们要增加组数到 $i$ 时，只要把$\[k+1, j]$的元素合成一组，遍历$k=[i-1, j-1]$，就相当于枚举了$j$个元素分为$i$组的情况。

比如，假设我们要计算 dp\[3]\[5]，那么你可以理解成 $ dp[2][2]=4 $ 包含了唯一的分组情况 $\lbrace[4],[2]\rbrace$，结果为4；$ dp[2][3]=6 $ 包含了分组情况 $ \lbrace[4,2],[4]\rbrace\ \lbrace[4],[2,4]\rbrace $，结果为6；$ dp[2][4]=9 $ 包含了分组情况 $ \lbrace[4,2,4],[5]\rbrace\ \lbrace[4,2],[4,5]\rbrace\ \lbrace[4],[2,4,5]\rbrace$，结果为9。 

因此要计算 dp\[3]\[5]，前一行我们已经得到$ dp[2][2]=4, dp[2][3]=6, dp[2][4]=9$，$dp[2][2]$表示的是将前2个数分为2组的情况，然后把剩下的3个数作为一组（4+5+1=10），与$dp[2][2]$比较，两者的最大值就是划分 $\lbrace [4],[2],[4,2,5] \rbrace$ 的最大值，得到10；$dp[2][3]$与剩下的2个数拼成的1组（5+1=6）比较，得到最大值6；$dp[2][4]$与最后1组比较，得到最大值9。所以$dp[3][5]=min\lbrace10, 6, 9 \rbrace = 6$

这个过程也就相当与我们检验了把5个数分为3组的所有情况

    dp[2][2]代表的1种{[4],[2]} + 追加新元素构成的一组[4,5,1]
    dp[3][2]代表的2种{[4],[2,4]} {[4,2],[4]} + [5,1]
    dp[4][2]代表的3种{[4],[2,4,5]} {[4,2],[4,5]} {[4,2,4],[5]} + [1]
    共6种情况

这样 dp\[ \]\[ \] 的每個元素的计算只需要与前一行的 n 个元素比较而不用重新计算了。最后计算到 dp\[m]\[n] 就是我们要求的答案

因为要反复计算区间和，所以使用前缀和数组来避免重复计算的开销（dp\[ \]\[ \]的第一行元素）。代码前面的[博客][1]有一份，不过另外开辟了一个前缀和数组 sum，属于浪费。

也可以对$ m \times n $的dp数组做一些空间上优化，~~不过无关紧要，这里就不多说了。~~还是重要的，不优化在洛谷上连二维数组都开不出来，但是太懒就不写了。

但是问题在于这种方法时间复杂度比较高，大约为$O(m \times n^2)$，leetcode 上的数据比较弱也许还可以过，但是洛谷的数据范围是$ N \leq 10^5，M \leq N$，显然没办法AC。（事实上数据比较大的时候单开二维数组就会MLE，也就只能过两个测试点的样子，就算用的是全局数组，还是会超时）

所以接下来介绍第二种方法。

思路其实不难理解，就是比较难往这个方向想。而且很多题解代码写得挺漂亮但是解释实在令人捉急，不如看代码来的快。首先，我们可以确定要找的答案必然在区间 $ [MAX_{A_i}, SUM_{A_i}] $ 中，也就是数组元素最大值与所有元素之和中间。然后一个很简单的思路就是，遍历该区间，找到符合分组数为 m 个，同时最大区间之和最小的那个值。但是遍历显然效率不够高，所以这里用的是二分法在该区间内查找。

然后有一个比较关键的问题是，当我们检验一个值不能满足时，应该如何确定继续往下搜索左区间还是右区间呢。

假设我们当前要检验的值为 t，也就是我们假设答案是 t，然后验证能不能恰好分成 m 组使得最大区间和不超过 t。而具体检验的方法我是真的服气（见`check()函数`），可能是我贪心学的太少。先来看代码吧。

    vector<int> num;
    bool check(int tmpAns,int m)
    {
      int cnt = 0, tmpSum = 0;
      for (int i = 0, sz = num.size(); i < sz; i++)
      {
        if (tmpSum + num[i] > tmpAns) tmpSum = num[i], cnt++;
        else tmpSum += num[i];
      }
      cnt++;
      return cnt <= m;
    }

    void solve(int n, int m)
    {
      int L = 0, R = 0, ans = 0;
      num.assign(n, 0);
      for (int i = 0; i < n; i++)
      {
        cin >> num[i];
        R += num[i];
        L = max(L, num[i]);
      }

      while (L <= R)
      {
        int mid = L + (R - L) / 2;
        if (check(mid, m)) R = mid - 1, ans = mid;
        else L = mid + 1;
      }
      cout << ans;
    }

    int main()
    {
      int n = 0, m = 0;
      cin >> n >> m;
      solve(n, m);
      return 0;
    }

如果设答案为 t，那么当某一段累加和即将超过 t 时，截断分为一组，然后开始划分下一组。全部分完后，有三种情况：

1. 如果组数 cnt < m，那么说明 t 设得太大了，可以减小 t
2. 如果 cnt == m，那么当前 t值可能是答案，可以先记录。然后我们同样再减小 t 看看有没有更小的 t 值可以满足
3. 如果 cnt > m，说明 t 设太大了，因此需要增大 t，在右区间查找

当二分查找完整个 $ [MAX_{A_i}, SUM_{A_i}] $ 我们也就得到了答案。

这个其实是 leetcode 评论区的解释，比洛谷的题解要更容易懂。（虽然他们代码基本是一样的）

The End


----

[1]: https://www.cnblogs.com/grandyang/p/5933787.html