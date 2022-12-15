---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - RL
---

> 不得已需要学习一些RL的知识，这里开始是一系列记录各种算法/模型的笔记

Q-learning

Q表，记录什么？

Bellman Equation:

如何计算 Q value:

new Q = current_Q + alpha * {R(s,a) + gama * maxQ' - current_Q}
  = (1 - alpha) * current_Q + alpha * R(s, a) + alpha * gama * maxQ'


agent
environment
reward
action
policy
