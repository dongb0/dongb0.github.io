---
layout: post
title: "快速幂 | 取余运算 简略笔记"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
tags:
  - algorithm
---


关于快速幂，[OI Wiki][1] 介绍得十分清楚，不再重复。这里想写的是运用快速幂进行取模运算的思路。

模板题在此 [luogu P1226](https://www.luogu.com.cn/problem/P1226)，计算$x^n mod\ m$的值。[OI Wiki][1] 觉得这太简单，解释了一句就贴代码了。

> 我们知道取模的运算不会干涉乘法运算，因此我们只需要在计算的过程中取模即可。

可是我离散不好数学也不行啊，看不懂学霸跟我说“显然得证”的“显然”在哪里，所以打算自己动手推导一下为什么在运算过程中取模能够得到最终的结果。

----------

借用一下 OI Wiki 的例子，当我们使用快速幂计算$\ 3^{13} \ mod \ 11 \ $时，因为

<div>
$$ 
{ 3^{13} = 3^{(1101)_2} = 3^{8 \cdot 1} \cdot 3^{4 \cdot 1} \cdot 3^{2 \cdot 0} \cdot 3^1 }
$$
</div>

设$m$为所求余数，$x$为所求的幂次方，$t_i$为对应中间项的值
- 计算第$0$项$3^1,  x_0 = 3 , m_0 = 3 \% 11 = 3 , t_0 = 3 $
- 计算第$1$项$ 3^{2 \cdot 0} $为$ 1 $，这一项对结果没有产生影响，但是我们需要得到$t_1$来计算下一项，$ x_1 = 3, \ m_1 = 3,\  t_1 = {t_0}^2 = 9$
- 计算第$2$项$ 3^{4 \cdot 1} $，此时$ x_2 = x_1 \cdot 3^4 = 3 \cdot 3^4, \ t_2 = {t_1}^2 = 81 $，而计算$m_2$并不需要$x$来取模，而是$ m_2 = m_1 \cdot t_2 \% 11 = (3\cdot81)\% 11 = 1 $  
- 计算第$3$项$ 3^{8 \cdot 1} $，此时$ x_3 = x_2 \cdot t_2 = 3 \cdot 81 \cdot 81^2,\ t_3 = {t_2}^2 = 81^2, m_3 = m_2 \cdot t_3 \% 11 =  (1 \cdot 81^2) \% 11 = 5$

最后的$m_3 = 3$就是$\ 3^{13} \ mod \ 11 \ $的结果，其实$x$的值并没有用上，只是一并写出来帮助理解快速幂。大概做完这样的手动计算之后我发现，我的问题其实变成：为什么

<div>
$$ (A\cdot B) \% C = ((A \% C) \cdot (B \% C)) \% C $$
</div>

那这个就好证明了（虽然不知道对不对）

<div>
$$
设A = C\cdot t_1 + m_1, B = C\cdot t_2 + m_2  \\
设a = t_1 \cdot C, \ b = t_2 \cdot C  \\
\therefore A=a+m_1,\ B=b+m_2 \\
\therefore 左边= (a\cdot b + a\cdot m_2 + b\cdot m_1 + m_1\cdot m_2)\%C  \\ 
\because (a\cdot b)\%C = (a\cdot m_2)\%C = (b\cdot m_1)\%C = 0(用分配律还需要证明吗？)\\
\therefore 左边 = (m_1\cdot m_2) \% C \\
又\because右边((A \% C) \cdot (B \% C)) \% C = (m_1\cdot m_2)\% C \\
\therefore 左边=右边\\证毕
$$
</div>


自己想是想不出来的，这辈子都想不出来的，只能看别人的结论推导证明理解一下，勉强维持得了生活的样子。


------

附来自 [OI Wiki][1] 的代码


        long long binpow(long long a, long long b, long long m) {
            a %= m;     
            long long res = 1;
            while (b > 0) {
                if (b & 1) res = res * a % m;
                a = a * a % m;
                b >>= 1;
            }
            return res;
        }

第二行的`a %= m;` 把$ 5^2 mod\ 3 $变成计算$ 2^2 mod\ 3 $，其实是利用了$ (5 \cdot 5 ) mod\ 3 = (5mod\ 3)\cdot(5mod\ 3)mod\ 3$，写博客之前我还真是没想到。

The End


[1]: https://oi-wiki.org/math/quick-pow/#_9


