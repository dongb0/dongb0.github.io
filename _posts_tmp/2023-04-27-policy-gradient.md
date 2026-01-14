---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - 
---


abstract
function approximation (是指逼近 什么的 函数 来着？ 真值？)

这个方法是保证至少收敛到 局部最优？


value-function approach，是要训练得到一个 value function（来对每个状态下不同选择的 reward,或者说 value 进行估计），然后选择策略根据 value 来使用某种贪心方法选择得到；
典型方法：Q-Learning
但是无法保证对任意的 MDP 任务都能收敛；

因此 policy-based 方法并不去尝试贴近 value function，而是直接训练一个 policy function 来表示选择


1 Policy Gradient Theorem

看不懂