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


- SqlNode  
  什么是 SqlNode？
  > [文档][1]: A SqlNode is a SQL parse tree. 
  SqlNode 本身是个抽象类，其下有许多派生类

  （详见 https://zhuanlan.zhihu.com/p/65701467
  （虽然图画得不咋样，不过可理解

  可以在 [这里] // 截图，在picture 目录下，关于SqlNode的派生类
  看到 SELECT/DELETE/CREATE 以及其他的 WITH/JOIN/ORDER BY 等都有对应 SqlNode 派生类

  我们解析一个 `SELECT` 语句看下结果：
  // TODO

- RelNode
  > [文档][2]:
  > A RelNode is a relational expression.  
  > Relational expressions process data, so their names are typically verbs: Sort, Join, Project, Filter, Scan, Sample. A relational expression is not a scalar expression...


- RexNode
RexLiteral(常量), RexCall(函数)， RexInputRef(输入引用) 都继承自 RexNode


[1]: https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/SqlNode.html
[2]: https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelNode.html